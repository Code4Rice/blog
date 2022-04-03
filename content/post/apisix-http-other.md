---
author: "Merlinlin"
date: 2022-04-03
title: "APISIX源码阅读 - http balance、body header filter、log阶段"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---
**
上篇文章我们介绍了承载apisix大部分逻辑的`access`阶段。本篇文章讲述access后的剩余阶段。


# 负载均衡阶段（balancer）

该阶段主要从众多上游ip中通过一定策略选出合适的ip用于本次请求。apisix在该阶段附加健康检查以及上游超时重试的逻辑。


> balancer调用时机：
> **balancer阶段并不在每次nginx请求都会执行，它和content阶段是互斥的。**
> 当nginx请求自行产生数据时（content阶段或者是在balancer之前调用`ngx.say`、`ngx.print`）
> 即不需要通过请求上游获得数据。就不会执行balancer阶段，并且不请求上游。直接到header_filter阶段。


同时，balancer阶段可能调用多次。这源于nginx的proxy next upstream设计，即当请求的上游超时时会再次调用balancer选出下一个上游进行重试。


```lua
function _M.http_balancer_phase()                                  
    local api_ctx = ngx.ctx.api_ctx                                
    if not api_ctx then                                            
        core.log.error("invalid api_ctx")                          
        return core.response.exit(500)                             
    end                                                            
                                                                   
    -- 最终会通过 balancer.set_current_peer设置上游地址            
    load_balancer.run(api_ctx.matched_route, api_ctx, common_phase)
end                                                                
```

具体实现可以关注xxxx，本期文章不做赘述。

# header_filter

balancer阶段结束后，nginx对上游发起请求。往后就到header_filter阶段，主要处理response的header。


```lua
-- apisix/init.lua line.588
function _M.http_header_filter_phase()
    -- 当上下文打包数据存在时（详见[上下文](http://code4rice.com/post/apisix-http-access/#%E4%B8%8A%E4%B8%8B%E6%96%87)）
    if ngx_var.ctx_ref ~= '' then
        -- 从ngx变量中获取，并且解包
        local stash_ctx = fetch_ctx()

        -- 当当前请求为转发请求时候，重新注入上下文（原有无法携带）
        if ngx_var.from_error_page == "true" then
            ngx.ctx = stash_ctx
        end
    end

    -- 标记apisix处理的该请求
    core.response.set_header("Server", ver_header)

    -- 当上游请求错误时讲错误码写入response header
    local up_status = get_var("upstream_status")
    if up_status and #up_status == 3
       and tonumber(up_status) >= 500
       and tonumber(up_status) <= 599
    then
        set_resp_upstream_status(up_status)
    elseif up_status and #up_status > 3 then
        -- the up_status can be "502, 502" or "502, 502 : "
        local last_status
        if str_byte(up_status, -1) == str_byte(" ") then
            last_status = str_sub(up_status, -6, -3)
        else
            last_status = str_sub(up_status, -3)
        end

        if tonumber(last_status) >= 500 and tonumber(last_status) <= 599 then
            set_resp_upstream_status(up_status)
        end
    end

    -- 执行插件的header filter阶段
    common_phase("header_filter")

    local api_ctx = ngx.ctx.api_ctx
    if not api_ctx then
        return
    end

    -- 当开启debug模式时写入插件数据
    local debug_headers = api_ctx.debug_headers
    if debug_headers then
        local deduplicate = core.table.new(#debug_headers, 0)
        for k, v in pairs(debug_headers) do
            core.table.insert(deduplicate, k)
        end
        core.response.set_header("Apisix-Plugins", core.table.concat(deduplicate, ", "))
    end
end

```

# body_filter

继header_filter之后是body_filter阶段，该阶段会调用多次，分块处理nginx读取回包，并且通过`ngx.arg[2]`判断是否读到末尾（ngx.arg[1]可以获取当前分块）

apisix在主流程中没有对body filter做特殊处理，直接执行各个插件的body filter逻辑

# 日志阶段（log）

log是请求的最后阶段，一般用于日志上报、各种状态刷新、内存释放。

apisix在log阶段主流程中主要做这几件事：
1. 上报上游健康状态

apisix支持配置upstream的健康检查，log阶段上报当前请求健康状态

```lua
-- apisix/init.lua line.713
healthcheck_passive(api_ctx)

-- apisix/init.lua line.644
local function healthcheck_passive(api_ctx)
    -- 获取当前请求上游健康检查器
    local checker = api_ctx.up_checker
    if not checker then
        return
    end

    local up_conf = api_ctx.upstream_conf
    -- 获取被动检查配置
    local passive = up_conf.checks.passive
    if not passive then
        return
    end

    core.log.info("enabled healthcheck passive")
    -- 获取主动检查配置
    -- 主要获取host用于主机配置
    local host = up_conf.checks and up_conf.checks.active
                 and up_conf.checks.active.host
    local port = up_conf.checks and up_conf.checks.active
                 and up_conf.checks.active.port

    local resp_status = ngx.status
    -- 获取被动检查配置中 健康的状态码列表
    local http_statuses = passive and passive.healthy and
                          passive.healthy.http_statuses
    core.log.info("passive.healthy.http_statuses: ",
                  core.json.delay_encode(http_statuses))
    if http_statuses then
        for i, status in ipairs(http_statuses) do
            -- 当当前请求码在健康列表中时上报当前请求
            if resp_status == status then
                checker:report_http_status(api_ctx.balancer_ip,
                                           port or api_ctx.balancer_port,
                                           host,
                                           resp_status)
            end
        end
    end
    -- 获取不健康状态码列表
    http_statuses = passive and passive.unhealthy and
                    passive.unhealthy.http_statuses
    core.log.info("passive.unhealthy.http_statuses: ",
                  core.json.delay_encode(http_statuses))
    if not http_statuses then
        return
    end

    for i, status in ipairs(http_statuses) do
        -- 当当前请求状态码在不健康状态码列表中则上报
        if resp_status == status then
            checker:report_http_status(api_ctx.balancer_ip,
                                       port or api_ctx.balancer_port,
                                       host,
                                       resp_status)
        end
    end
end
```

2. 执行负载均衡算法的负载后回调
主要用于释放当balancer多次调用时用于**记录请求失败的上游信息table**。

3. 释放各种变量table
在lua中，table是一个复杂变量，频繁的创建销毁会带来巨大的资源消耗。

apisix大量采用table池记录变量。在请求结束时均需释放table回table池。
```lua
-- apisix/init.lua line.719
-- 释放apisix自行维护的变量池
core.ctx.release_vars(api_ctx)
-- 释放当前路由匹配到的插件对象池子
if api_ctx.plugins then
    core.tablepool.release("plugins", api_ctx.plugins)
end
-- 释放当前匹配路由数据
if api_ctx.curr_req_matched then
    core.tablepool.release("matched_route_record", api_ctx.curr_req_matched)
end
-- 释放apisix维护的上下文
core.tablepool.release("api_ctx", api_ctx)
```

至此，一个http请求在apisix的主线处理流程就结束了。各个模块更深入逻辑讲解请关注本系列文章~**
