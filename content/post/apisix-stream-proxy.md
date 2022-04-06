

# APISIX源码阅读 - 四层协议的支持

> apisix主要是以api网关的身份被大家熟知，大多数场景都是应用在七层协议代理中。不过apisix的内核还是Nginx，所以天然就支持四层协议的代理模式，在nginx中也称之为stream proxy



## 配置示例

在了解实现原理之前，我们先看一个简单的配置示例，在config.yaml中，如下配置

```yaml
apisix:
  stream_proxy: # TCP/UDP proxy
    tcp: # TCP proxy address list
      - 9100 # by default uses 0.0.0.0
      - "127.0.0.10:9101"
```

如此，apisix启动时，便会启动对以上两组tcp地址的动态代理，但此时并没有对应的路由和upstream可以进行寻址，所以我们还需要进行一次路由的配置，如下

```bash
curl http://127.0.0.1:9080/apisix/admin/stream_routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "server_addr": "127.0.0.10",
    "server_port": 9101,
    "upstream": {
        "nodes": {
            "127.0.0.1:3306": 1
        },
        "type": "roundrobin"
    }
}'
```

以上配置的含义是，当apisix服务端收到地址为127.0.0.10，端口为9101的请求时，会转发到upstream的nodes中(127.0.0.1:3306)，假设我们有一组MySQL的服务，可以用apisix作为tcp proxy



## 源码分析

下面我们来看看apisix在代码层面是如何实现整个stream proxy的逻辑的

### 配置生成

我们知道，stream proxy是nginx的核心功能之一，apisix也是基于nginx的stream模块进行封装，所以整个stream proxy的配置，还是要从nginx.conf开始进行。

apisix里生成nginx.conf的核心文件是[apisix/cli/ngx_tpl.lua](https://github.com/apache/apisix/blob/master/apisix/cli/ngx_tpl.lua)，他是一个模板文件，根据apisix的config.yaml，生成功能相对应的nginx.conf。第一部分我们看到config.yaml配置中，四层相关的代理是在stream_proxy配置项下，我们在ngx_tpl.lua中找到对应逻辑。

```lua
{% if stream_proxy then %}
stream {

    // 定义后端upstream
    upstream apisix_backend {
        server 127.0.0.1:80;
        balancer_by_lua_block {
            apisix.stream_balancer_phase()
        }
    }

    // 初始化stream模块
    init_by_lua_block {
				...
        apisix.stream_init(args)
    }

    init_worker_by_lua_block {
        apisix.stream_init_worker()
    }

    // nginx server配置
    server {
      	// 根据config.yaml里的配置，生成所有需要代理的四层协议地址
        {% for _, item in ipairs(stream_proxy.tcp or {}) do %}
        listen {*item.addr*} {% if item.tls then %} ssl {% end %} {% if enable_reuseport then %} reuseport {% end %} {% if proxy_protocol and proxy_protocol.enable_tcp_pp then %} proxy_protocol {% end %};
        {% end %}
        {% for _, addr in ipairs(stream_proxy.udp or {}) do %}
        listen {*addr*} udp {% if enable_reuseport then %} reuseport {% end %};
        {% end %}
      
      	...

        {% if proxy_protocol and proxy_protocol.enable_tcp_pp_to_upstream then %}
        proxy_protocol on;
        {% end %}

        preread_by_lua_block {
            apisix.stream_preread_phase()
        }

        proxy_pass apisix_backend;
      
      	...

        log_by_lua_block {
            apisix.stream_log_phase()
        }
    }
}
{% end %}
```

根据这个模板，我们可以模拟出第一部分我们的配置示例所对应的nginx.conf，如下

```bash
...
   // 这里只列出了server的核心部分
   server {
        listen 9100 ssl reuseport;
        listen 127.0.0.1:9101 reuseport;
        
        ...

        preread_by_lua_block {
            apisix.stream_preread_phase()
        }

        proxy_pass apisix_backend;

        log_by_lua_block {
            apisix.stream_log_phase()
        }
    }
...
```

### 代理流程

服务启动时，进行stream代理相关的初始操作，首先进行dns resolver的初始化，stream_init中进行配置相关的初始化

```lua
init_by_lua_block {
    require "resty.core"
    apisix = require("apisix")
    local dns_resolver = {"xxx.xxx.xxx.xxx"}
    local args = {
        dns_resolver = dns_resolver,
    }
    apisix.stream_init(args)
}
```

然后进行stream worker中的初始化

```lua
init_worker_by_lua_block {
    apisix.stream_init_worker()
}

```

```lua
function _M.stream_init_worker()
    core.log.info("enter stream_init_worker")
    local seed, err = core.utils.get_seed_from_urandom()
    if not seed then
        core.log.warn('failed to get seed from urandom: ', err)
        seed = ngx_now() * 1000 + ngx.worker.pid()
    end
    math.randomseed(seed)
    -- for testing only
    core.log.info("random stream test in [1, 10000]: ", math.random(1, 10000))

    plugin.init_worker()
    router.stream_init_worker()
    apisix_upstream.init_worker()

    if core.config == require("apisix.core.config_yaml") then
        core.config.init_worker()
    end

    load_balancer = require("apisix.balancer")

    local_conf = core.config.local_conf()
end
```

在stream_init_worker中，对plugin、router、upstream、config、balancer等核心模块进行初始化。

在所有依赖模块初始化完成后，便可以进行完整的tcp/udp协议代理

请求到来后，首先会执行stream_preread_phase，也是stream proxy的主要逻辑

```
preread_by_lua_block {
    apisix.stream_preread_phase()
}
```

我们来看看stream_preread_phase做了些什么事情

```lua
function _M.stream_preread_phase()

		...

  	// 根据stream route的配置，匹配该次请求对应的apisix route对象
    local ok, err = router.router_stream.match(api_ctx)
    if not ok then
        core.log.error(err)
        return ngx_exit(1)
    end

    local matched_route = api_ctx.matched_route
    if not matched_route then
        return ngx_exit(1)
    end

    // 获取该路由对应的upstream对象
    local up_id = matched_route.value.upstream_id
    if up_id then
        api_ctx.matched_upstream = get_upstream_by_id(up_id)
    else
        // 如果upstream对象包含域名，则进行dns解析
        if matched_route.has_domain then
            local err
            matched_route, err = parse_domain_in_route(matched_route)
            if err then
                core.log.error("failed to get resolved route: ", err)
                return ngx_exit(1)
            end

            api_ctx.matched_route = matched_route
        end

        local route_val = matched_route.value
        api_ctx.matched_upstream = (matched_route.dns_value and
                                    matched_route.dns_value.upstream)
                                   or route_val.upstream
    end

    // 执行绑定在改路由中的所有插件
    local plugins = core.tablepool.fetch("plugins", 32, 0)
    api_ctx.plugins = plugin.stream_filter(matched_route, plugins)

    ...

    plugin.run_plugin("preread", plugins, api_ctx)

    local code, err = set_upstream(matched_route, api_ctx)
    if code then
        core.log.error("failed to set upstream: ", err)
        return ngx_exit(1)
    end

    // 调用负载均衡模块选取upstream中的一个server
    local server, err = load_balancer.pick_server(matched_route, api_ctx)
    if not server then
        core.log.error("failed to pick server: ", err)
        return ngx_exit(1)
    end

    api_ctx.picked_server = server

  	// 执行apisix的before_proxy阶段
    common_phase("before_proxy")
end
```

1. 根据请求进行路由匹配
2. 获取路由的upstream对象，如果upstream中包含域名，则进行dns解析
3. 执行改路由绑定的所有插件
4. 调用负载均衡模块进行upstream的server选取
5. 执行接下来的before_proxy阶段

请求的最后，会进行log阶段

```lua
log_by_lua_block {
    apisix.stream_log_phase()
}
```

在stream_log_phase中，会调用log相关的插件，进行相关日志的打印，至此，一次完整的tcp/udp四层协议的请求代理就完成了。





