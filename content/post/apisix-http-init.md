---
author: "Merlinlin"
date: 2022-03-07
title: "APISIX源码阅读 - http init、init worker阶段"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---

# 前言
笔者所在公司团队在20年网关选型上选择了apisix。经过一年多的踩坑和开发，算是对apisix小有了解。但尚未系统性的阅读一遍apisix的源码，目前计划对apisix2.12.1版本源码进行详细解读并更新于当前系列文章（希望不会鸽

# 简单介绍
有调研过apisix的同学应该知道，apisix是基于openresty之上，使用lua封装了一层框架并提供动态路由、动态上游、插件等能力的网关。

实际上他就是一个在openresty上写了很多lua代码的项目（起码一开始是这样的，现在的apisix已经发展为有定制化的openresty、wasm、多语言支持的项目）


本系列文章从nginx.conf上的配置，到各个处理阶段中lua代码的逻辑逐一详细剖析。

# nginx配置

apisix的动态一词在 nginx.conf 配置便开始体现。通过模板描述 nginx.conf，以及自定义的配置文件，在服务启动时自动生成nginx.conf。

apisix的配置统一管理在conf目录下的 config-default.yaml,和 config.yaml 文件中，两个文件的配置在apisix启动时候会做merge操作，冲突配置以 config.yaml 配置为准。

这样的设计是希望开发者能更直观清晰地关注需要修改的配置。

**ps: 直到生成nginx.conf之前，apisix都是通过脚本完成上述过程。在生成nginx.conf之后才真正启动openresty服务**


# http初始化逻辑

从 nginx.conf 的模板文件开始，http模块的初始化包含了`init_by_lua`阶段和`init_worker_by_lua`阶段。


```lua
# apisix/cli/ngx_tpl.lua
init_by_lua_block {
        require "resty.core"
		# 第三方模块代码注入
		# apisix允许在模板文件许多地方注入第三方逻辑
        {% if lua_module_hook then %}
        require "{* lua_module_hook *}"
        {% end %}
        apisix = require("apisix")
		# dns服务器列表
		# dns_resolver数据从配置文件或默认系统/etc/resolv.conf文件读取
        local dns_resolver = { {% for _, dns_addr in ipairs(dns_resolver or {}) do %} "{*dns_addr*}", {% end %} }
        local args = {
            dns_resolver = dns_resolver,
        }
		# 主进程初始化阶段
        apisix.http_init(args)
    }

	# worker进程初始化阶段
    init_worker_by_lua_block {
        apisix.http_init_worker()
    }
```

写过openresty请求相关逻辑的同学应该知道，通常在openresty中我们需要自己完成dns解析的工作，自行维护dns服务器列表。

apisix在init阶段完成相应工作
```lua
-- apisix/init.lua line.74
core.resolver.init_resolver(args)
```
`init_resolver`的逻辑并不复杂，仅仅是将传入的dns服务器列表存储在全局变量。那么涉及到openresty与第三方服务（例如mysql、redis、http服务器等）建联时的dns解析逻辑在哪里？

可以阅读另一篇文章[《APISIX源码阅读--域名解析问题》](https://iwiki.woa.com/pages/viewpage.action?pageId=1504846100)

处理完dns相关逻辑后，apisix对当前服务从配置文件读取或生成一个唯一id。这在集群部署下具备apisix实例身份标识等作用。
```lua
-- apisix/init.lua line.75
-- 支持从conf/apisix.uid文件中读取id
-- 或随机生成uuid并写入apisix.uid,
-- 这使得服务重启时uid仍能保持不变
core.id.init()
```

apisix不仅支持原生的lua语言，还支持通过local rpc方式使用golang、java、python等语言开发插件。

这需要apisix在启动时同时启动相关语言runner并维护它。
```lua
-- apisix/init.lua line.77
-- apisix使用nginx的特权进程来维护各个语言的runner
-- 关于特权进程具体工作在后续文章描述
local process = require("ngx.process")
-- 启用特权进程
local ok, err = process.enable_privileged_agent()
if not ok then
    core.log.error("failed to enable privileged_agent: ", err)
end
```

在init阶段的最后初始化配置，这次的配置不同于上面使用在生成nginx.conf的配置，这次的配置用于apisix实际工作的配置，例如路由、插件、上游服务等配置，关于功能，而不是服务本身。


对于使用默认的存储etcd来说，还会检查是否在apisix启动之前完成对etcd路径初始化的工作（不需要自己动手，apisix会执行脚本完成etcd的初始化）

```lua
-- apisix/init.lua line.83
if core.config.init then
    -- core的config根据config.yaml赋值
    local ok, err = core.config.init()
    if not ok then
        core.log.error("failed to load the configuration: ", err)
    end
end
```

**至此，apisix的http模块init阶段处理完毕，接下来进入init_worker阶段。**

nginx是多进程单线程设计，最大程度上提高cpu利用率。当有worker进程之间需要同步消息时采用共享内存或事件等功能实现。

apisix目前维护了一个全局的事件模块。主要使用在**服务发现的数据同步**（截至2.12.1版本，consul_kv、nacos使用了）、**通知所有worker进行插件配置重载**、以及在**重启apisix时重新创建ext-plugin的lrucache**

```lua
-- apisix/init.lua line.83
-- 借助系统/dev/urandom文件生成随机种子
local seed, err = core.utils.get_seed_from_urandom()
if not seed then
    core.log.warn('failed to get seed from urandom: ', err)
    seed = ngx_now() * 1000 + ngx.worker.pid()
end
math.randomseed(seed)
-- for testing only
core.log.info("random test in [1, 10000]: ", math.random(1, 10000))
local we = require("resty.worker.events")
-- shm配置进程间共享内存名称
-- 每interval秒拉取事件
local ok, err = we.configure({shm = "worker-events", interval = 0.1})
if not ok then
    error("failed to init worker event: " .. err)
end
```

nginx提供了基于事件驱动的定时器模型， 我们借助定时器实现各种功能，例如更新数据的后台任务，在无法执行某种api操作的阶段仍触发某段逻辑（我们可以用ngx.timer.at，设置0秒后运行，让人有一种他好像就在当前阶段运行了的感觉）

但当注册的定时器越来越多时，nginx需要消耗更多的资源去维护定时器结构。

对此， apisix创建了**一个全局定时器**，定时器中执行多个任务，且为非阻塞的协程模式运行，大大减少了nginx维护大量定时器的成本，且这对使用apisix的开发者来说是无感的。

坏处是默认定时器执行轮询时间为1秒，且每次执行需要等待全部任务执行结束后才算完成。**对于时间敏感的任务不适合使用全局统一的定时器， 且全局定时器没有出入参处理**

2.12.1版本的apisix使用全局定时器的居多是插件中的数据上报等对时间精确性没有过多要求的功能

```lua
-- apisix/init.lua line.107
-- 批量初始化服务发现, 建联、跑定时器之类的
local discovery = require("apisix.discovery.init").discovery
if discovery and discovery.init_worker then
    discovery.init_worker()
end
-- 对于这个版本的来说是空的，没啥此操作
require("apisix.balancer").init_worker()
load_balancer = require("apisix.balancer")
-- 初始化admin api路由，注册reload插件的事件以及同步配置文件和etcd的插件配置
require("apisix.admin.init").init_worker()
-- 初始化定时器模块
-- 仅创建一个nginx层面的定时器
require("apisix.timers").init_worker()
```

由于apisix提供服务过程中涉及许多配置，例如upstream、plugins、plugins_config等。使得apisix在触发reload时经常需要主动释放不需要的全局变量。

在plugins模块调用init_worker函数时，apisix会遍历全局变量`local_plugins_hash`调用已载入的插件的destroy函数完成卸载。

接着释放lua全局变量`package.loaded`中各个插件变量
> local pkg_loaded = package.loaded
> pkg_loaded[pkg_name] = nil


最后再载入新的插件，运行各个插件的init方法，根据优先级排序。

```lua
-- apisix/init.lua line.117
-- 若开启debug模式将可配置打印阶段输入输出、添加header等功能
require("apisix.debug").init_worker()
-- 初始化插件
plugin.init_worker()
-- 初始化路由模块，
-- 包括初始化匹配模式、通过config模块watch路由配置和全局规则配置
router.http_init_worker()
-- 配置读取到内存并watch
require("apisix.http.service").init_worker()
-- 初始化plugin_config模块，watch service配置信息
-- 插件配置，类似于upstream在route仅配置一个id的功能一样, watch
plugin_config.init_worker()
-- 配置读取到内存并watch
require("apisix.consumer").init_worker()

if core.config == require("apisix.core.config_yaml") then
    -- 当配置模式是yaml时，启用定时器定时读取文件数据
    core.config.init_worker()
end
-- 配置读取到内存并watch，洗数据
apisix_upstream.init_worker()
-- 读取ext插件配置
-- 特权进程运行、管理各runner程序
require("apisix.plugins.ext-plugin.init").init_worker()
-- 提示apisix版本在header中
local_conf = core.config.local_conf()
if local_conf.apisix and local_conf.apisix.enable_server_tokens == false then
    ver_header = "APISIX"
end
```

至此apisix的http模块就初始化结束啦
