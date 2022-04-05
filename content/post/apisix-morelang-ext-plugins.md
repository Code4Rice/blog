---
author: "Merlinlin"
date: 2022-04-05
title: "APISIX源码阅读 - 多语言插件ExtPlugins支持"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---



apisix支持多语言开发插件。除了原生的lua语言以外，其他语言通过runner或者wasm实现。

本系列文章将通过两篇文章分别介绍runner以及wasm的实现。

wasm实现详见[xxxxxxxxxx]()


# runner总流程

![](http://code4rice.com/sz_mmbiz_png/vibjb3KpB2azd2xTibnN4orG1eVqq4lVH8NYusp7ic54EQibddp9tX5Jw8clALTqryQTsff227fEwZoFRxLYiaibYHaQ/0)


> 1. init_worker阶段创建特权进程，并通过特权进程维护各语言runner
> 2. 通过worker间事件通知runner退出事件，用于重建缓存
> 3. ext插件执行阶段，通过多次rpc与runner交互完全请求
> 4. 根据回包类型设置response



下面我们进行详细源码解读

# runner创建和维护

apisix在init阶段启动特权进程，在init_worker阶段注册runner退出事件，在特权进程init_worker阶段启动各语言runnner。

```lua
-- apisix/init.lua line.78
-- 启动特权进程
local ok, err = process.enable_privileged_agent()

-- apisix/init.lua line.102
-- 初始化event
local we = require("resty.worker.events")

-- apisix/init.lua line.130
-- 初始化ext plugins
require("apisix.plugins.ext-plugin.init").init_worker()

-- apisix/plugins/ext-plugin/init.lua line.800
function _M.init_worker()
    -- 获取配置文件，获取语言runner cmd
    local local_conf = core.config.local_conf()
    local cmd = core.table.try_read_attr(local_conf, "ext-plugin", "cmd")
    if not cmd then
        return
    end
    -- event list
    events_list = events.event_list(
        "process_runner_exit_event",
        "runner_exit"
    )

    -- 注册runner退出事件 用于重建缓存
    events.register(recreate_lrucache, events_list._source, events_list.runner_exit)

    -- 特权进程时启动并维护runner
    if process.type() == "privileged agent" then
        setup_runner(cmd)
    end
end
```

ext-plugin在配置文件中配置为多条可执行文件路径，apisix通过`ngx.pipe.spawn`批量启动。
```lua
-- apisix/plugins/ext-plugin/init.lua line.732
local function spawn_proc(cmd)
    -- 写入环境变量
    must_set("APISIX_CONF_EXPIRE_TIME", helper.get_conf_token_cache_time())
    must_set("APISIX_LISTEN_ADDRESS", helper.get_path())

    local opt = {
        merge_stderr = true,
    }
    -- 批量执行
    local proc, err = ngx_pipe.spawn(cmd, opt)
    if not proc then
        error(str_format("failed to start %s: %s", core.json.encode(cmd), err))
        -- TODO: add retry
    end
    -- 不超时
    proc:set_timeouts(nil, nil, nil, 0)
    return proc
end
```

启动runner后，特权进程通过定时器维护runner状态

循环打印runner log

**当runnner退出时广播触发刷新缓存**

```lua
-- apisix/plugins/ext-plugin/init.lua line.751
local function setup_runner(cmd)
    -- 创建runner进程
    runner = spawn_proc(cmd)
    -- 创建定时器维护runner状态
    ngx_timer_at(0, function(premature)
        -- ...
        -- 当当前worker进程没有退出时
        while not exiting() do
            -- 日志 or 退出循环
            while true do
                -- drain output
                local max = 3800 -- smaller than Nginx error log length limit
                local data, err = runner:stdout_read_any(max)
                if not data then
                    if exiting() then
                        return
                    end

                    if err == "closed" then
                        break
                    end
                else
                    -- we log stdout here just for debug or test
                    -- the runner itself should log to a file
                    core.log.warn(data)
                end
            end
            -- 函数本意是等待进程退出
            -- 但执行到这里runner已经退出了
            -- 获取runner返回状态
            local ok, reason, status = runner:wait()
            if not ok then
                core.log.warn("runner exited with reason: ", reason, ", status: ", status)
            end

            runner = nil
            -- 广播runner退出事件
            local ok, err = events.post(events_list._source, events_list.runner_exit)
            if not ok then
                core.log.error("post event failure with ", events_list._source, ", error: ", err)
            end

            core.log.warn("respawn runner 3 seconds later with cmd: ", core.json.encode(cmd))
            core.utils.sleep(3)
            core.log.warn("respawning new runner...")
            -- 重启runner
            runner = spawn_proc(cmd)
        end
    end)
end
```


# 接口请求流程

apisix将ext plugin执行阶段分为pre和post。意为请求处理之前和之后。

实际区别在于
**pre在rewrite阶段，以最高优先级运行**
**post在access阶段，以最低优先级运行**

这样设计可以有效覆盖大多数需要处理的阶段，**但对于一些仍需要apisix收发请求，处理回包body以及header的场景并不适合。**


## apisix rpc请求runner

当启用ext plugin时，apisix和runner会进行**至少1次rpc请求**，
数量取决于是否需要获取token以及runner插件的逻辑是否多次获取第一次rpc没有传递的参数。


```lua
-- apisix/plugins/ext-plugin/init.lua line.688
function _M.communicate(conf, ctx, plugin_name)
    local ok, err, code, body
    local tries = 0
    -- rpc通信重试至多3次
    while tries < 3 do
        tries = tries + 1
        -- rpc远端请求
        ok, err, code, body = rpc_call(constants.RPC_HTTP_REQ_CALL, conf, ctx, plugin_name)
        if ok then
            -- stop类型，通过ngx.say设置回包数据
            if code then
                return code, body
            end
            -- rewrite类型，仍执行ngx请求数据
            return
        end
        -- 到这里则有异常，需重试
        if not core.string.find(err, "conf token not found") then
            core.log.error(err)
            if conf.allow_degradation then
                core.log.warn("Plugin Runner is wrong, allow degradation")
                return
            end
            return 503
        end
        -- 重刷缓存token
        core.log.warn("refresh cache and try again")
        recreate_lrucache()
    end

    core.log.error(err)
    -- 配置allow_degradation则允许runner逻辑失败时走正常apisix逻辑
    if conf.allow_degradation then
        core.log.warn("Plugin Runner is wrong after " .. tries .. " times retry, allow degradation")
        return
    end
    return 503
end
```

rpc_call在runner启动的首次请求（重启后也算）会进行两次tcp请求

第一次获取token，并存储在cache中，第二次才是真正的逻辑请求

```lua
-- apisix/plugins/ext-plugin/init.lua line.652
rpc_call = function (ty, conf, ctx, ...)
    -- 获取unix sock路径
    local path = helper.get_path()
    -- 建立tcp请求
    local sock = socket_tcp()
    sock:settimeouts(1000, 60000, 60000)
    local ok, err = sock:connect(path)
    if not ok then
        return nil, "failed to connect to the unix socket " .. path .. ": " .. err
    end
    -- 调用rpc handler
    -- 首次调用时会因为create cache重新进入到当前路径
    -- 并且调用rpc_handlers[ty]函数创建token
    local res, err, code, body = rpc_handlers[ty + 1](conf, ctx, sock, ...)
    if not res then
        sock:close()
        return nil, err
    end
    -- 放入连接池
    local ok, err = sock:setkeepalive(180 * 1000, 32)
    if not ok then
        core.log.info("failed to setkeepalive: ", err)
    end

    return res, nil, code, body
end
```

`rpc_handlers`是一个函数数组，下标1为nil，2为获取token逻辑，3是逻辑请求。

首次进入`rpc_call -> rpc_handlers[ty+1]`，会因需创建cache而再次进入`rpc_call`，不同的是这次调用的是`rpc_handlers[ty]`获取token


```lua
-- apisix/plugins/ext-plugin/init.lua line.441
-- 当前函数为`rpc_handlers[ty + 1]`函数
-- 调用lrucache.plugin_ctx传入的rpc_call，constants.RPC_PREPARE_CONF
-- 用于获取token
function (conf, ctx, sock, entry)
    local lrucache_id = core.lrucache.plugin_ctx_id(ctx, entry)
    local token, err = core.lrucache.plugin_ctx(lrucache, ctx, entry, rpc_call,
                                                constants.RPC_PREPARE_CONF, conf, ctx,
                                                lrucache_id)
```

获取token后，apisix打包uri、path、srcip、uri args、headers参数发往runner。
> 这里少了**请求body**以及**ngx变量**，主要考虑到这两个数据使用几率低，且一起打包会使得包体过大
> apisix采取了按需获取的设计，仅在插件需要的时候**再次通过rpc获取**


数据协议采用google的flatbuffers，感兴趣的同学可以[自行查阅](https://github.com/google/flatbuffers)

```lua
-- apisix/plugins/ext-plugin/init.lua line.450
builder:Clear()
local var = ctx.var
local uri
-- 写入uri
if var.upstream_uri == "" then
    -- use original uri instead of rewritten one
    uri = var.uri
else
    uri = var.upstream_uri
    -- the rewritten one may contain new args
    local index = core.string.find(uri, "?")
    if index then
        local raw_uri = uri
        uri = str_sub(raw_uri, 1, index - 1)
        core.request.set_uri_args(ctx, str_sub(raw_uri, index + 1))
    end
end
-- 写入path
local path = builder:CreateString(uri)
-- 写入srcip
local bin_addr = var.binary_remote_addr
local src_ip = builder:CreateByteVector(bin_addr)
local args = core.request.get_uri_args(ctx)
-- 写入uri args
local textEntries = {}
for key, val in pairs(args) do
    local ty = type(val)
    if ty == "table" then
        for _, v in ipairs(val) do
            core.table.insert(textEntries, build_args(builder, key, v))
        end
    else
        core.table.insert(textEntries, build_args(builder, key, val))
    end
end
local len = #textEntries
-- 写入headers
http_req_call_req.StartArgsVector(builder, len)
for i = len, 1, -1 do
    builder:PrependUOffsetTRelative(textEntries[i])
end
local args_vec = builder:EndVector(len)
local hdrs = core.request.headers(ctx)
core.table.clear(textEntries)
for key, val in pairs(hdrs) do
    local ty = type(val)
    if ty == "table" then
        for _, v in ipairs(val) do
            core.table.insert(textEntries, build_headers(var, builder, key, v))
        end
    else
        core.table.insert(textEntries, build_headers(var, builder, key, val))
    end
end
local len = #textEntries
http_req_call_req.StartHeadersVector(builder, len)
for i = len, 1, -1 do
    builder:PrependUOffsetTRelative(textEntries[i])
end
local hdrs_vec = builder:EndVector(len)
-- 写入id，method
local id = generate_id()
local method = var.method
http_req_call_req.Start(builder)
http_req_call_req.AddId(builder, id)
http_req_call_req.AddConfToken(builder, token)
http_req_call_req.AddSrcIp(builder, src_ip)
http_req_call_req.AddPath(builder, path)
http_req_call_req.AddArgs(builder, args_vec)
http_req_call_req.AddHeaders(builder, hdrs_vec)
http_req_call_req.AddMethod(builder, encode_a6_method(method))
local req = http_req_call_req.End(builder)
builder:Finish(req)
-- 发送到runner
local ok, err = send(sock, constants.RPC_HTTP_REQ_CALL, builder:Output())
```

ext plugin的rpc包结构比较简单，通过头部1个字节描述包类型，3个字节表示包长度以及追加包体内容来表示。

![](http://code4rice.com/sz_mmbiz_png/vibjb3KpB2azd2xTibnN4orG1eVqq4lVH842hb8HRqsddfG4iczT7KlaD8bxFjyV6G8NsPvTFdltEVuzUN1wrdJrw/0)


```lua
-- apisix/plugins/ext-plugin/init.lua line.112
local send
do
    -- 申请4个字节长度
    local hdr_buf = ffi.new("unsigned char[4]")
    -- 用于存放头部和包体
    local buf = core.table.new(2, 0)
    local MAX_DATA_SIZE = lshift(1, 24) - 1

    function send(sock, ty, data)
        -- 请求类型
        hdr_buf[0] = ty
        -- 包体长度
        local len = #data

        -- ...

        -- 大端字节序
        for i = 3, 1, -1 do
            hdr_buf[i] = band(len, 255)
            len = rshift(len, 8)
        end
        -- 1 头部
        -- 2 包体
        buf[1] = ffi_str(hdr_buf, 4)
        buf[2] = data
        -- send方法接受table，一同发送内容
        -- 避免字符串连接操作
        return sock:send(buf)
    end
end
```

## 插件逻辑

apisix请求runner后，业务逻辑由runner插件接管。

当插件需要获取ngx变量或者请求body时，通过rpc告知apisix，从apisix获取数据。

**这个过程取决于逻辑实现，可能会触发多次rpc请求，使用方需权衡性能损耗**

apisix侧获取变量处理逻辑
```lua
-- apisix/plugins/ext-plugin/init.lua line.532
local ty, resp
while true do
    -- 收包
    -- 逻辑同send，解析头部四字节、获取body
    ty, resp = receive(sock)
    if ty == nil then
        return nil, "failed to receive RPC_HTTP_REQ_CALL: " .. resp
    end
    -- 非获取数据则跳过当前逻辑
    if ty ~= constants.RPC_EXTRA_INFO then
        break
    end
    -- 获取变量数据
    local out, err = handle_extra_info(ctx, resp)
    if not out then
        return nil, "failed to handle RPC_EXTRA_INFO: " .. err
    end
    -- 再次发送给runner
    local ok, err = send(sock, constants.RPC_EXTRA_INFO, out)
    if not ok then
        return nil, "failed to reply RPC_EXTRA_INFO: " .. err
    end
end
-- apisix/plugins/ext-plugin/init.lua line.296
-- 获取变量数据
local function handle_extra_info(ctx, input)
    -- 反序列化
    local buf = flatbuffers.binaryArray.New(input)
    local req = extra_info_req.GetRootAsReq(buf, 0)

    local res
    local info_type = req:InfoType()
    -- 获取ngx变量
    if info_type == extra_info.Var then
        local info = req:Info()
        local var_req = extra_info_var.New()
        var_req:Init(info.bytes, info.pos)

        local var_name = var_req:Name()
        res = ctx.var[var_name]
    -- 获取req body
    elseif info_type == extra_info.ReqBody then
        local info = req:Info()
        local reqbody_req = extra_info_reqbody.New()
        reqbody_req:Init(info.bytes, info.pos)

        local err
        res, err = core.request.get_body()
        if err then
            core.log.error("failed to read request body: ", err)
        end
    else
        return nil, "unsupported info type: " .. info_type
    end

    -- 序列化回包
    builder:Clear()

    local packed_res
    if res then
        res = tostring(res)
        packed_res = builder:CreateByteVector(res)
    end
    extra_info_resp.Start(builder)
    if packed_res then
        extra_info_resp.AddResult(builder, packed_res)
    end
    local resp = extra_info_resp.End(builder)
    builder:Finish(resp)
    return builder:Output()
end
```

## 回包处理

ext plugins回包支持rewrite和stop两种模式。

- rewrite
  重写上游url、覆盖or新增req header、设置uri参数
- stop
  写入resp header、resp body、resp code

```lua
-- apisix/plugins/ext-plugin/init.lua line.296
-- 反序列化
local buf = flatbuffers.binaryArray.New(resp)
local call_resp = http_req_call_resp.GetRootAsResp(buf, 0)
local action_type = call_resp:ActionType()
-- stop类型
if action_type == http_req_call_action.Stop then
    local action = call_resp:Action()
    local stop = http_req_call_stop.New()
    stop:Init(action.bytes, action.pos)
    -- 设置response headers
    local len = stop:HeadersLength()
    if len > 0 then
        for i = 1, len do
            local entry = stop:Headers(i)
            core.response.set_header(entry:Name(), entry:Value())
        end
    end
    -- 获取response body
    local body
    local len = stop:BodyLength()
    if len > 0 then
        -- TODO: support empty body
        body = stop:BodyAsString()
    end
    -- 获取code
    local code = stop:Status()
    -- avoid using 0 as the default http status code
    if code == 0 then
         code = 200
    end
    -- 最终通过ngx.say、ngx.exit设置body和code
    return true, nil, code, body
end
-- rewrite类型
if action_type == http_req_call_action.Rewrite then
    -- 反序列化
    ctx.request_rewritten = constants.REWRITTEN_BY_EXT_PLUGIN
    local action = call_resp:Action()
    local rewrite = http_req_call_rewrite.New()
    rewrite:Init(action.bytes, action.pos)
    -- 设置上游uri
    local path = rewrite:Path()
    if path then
        path = core.utils.uri_safe_encode(path)
        var.upstream_uri = path
    end
    -- 设置req headers
    local len = rewrite:HeadersLength()
    if len > 0 then
        for i = 1, len do
            local entry = rewrite:Headers(i)
            local name = entry:Name()
            core.request.set_header(ctx, name, entry:Value())
            if str_lower(name) == "host" then
                var.upstream_host = entry:Value()
            end
        end
    end
    -- 设置uri参数
    local len = rewrite:ArgsLength()
    if len > 0 then
        local changed = {}
        for i = 1, len do
            local entry = rewrite:Args(i)
            local name = entry:Name()
            local value = entry:Value()
            if value == nil then
                args[name] = nil
            else
                if changed[name] then
                    if type(args[name]) == "table" then
                        core.table.insert(args[name], value)
                    else
                        args[name] = {args[name], entry:Value()}
                    end
                else
                    args[name] = entry:Value()
                end
                changed[name] = true
            end
        end
        core.request.set_uri_args(ctx, args)
        if path then
            var.upstream_uri = path .. '?' .. var.args
        end
    end
end
return true
```

至此apisix的ext plugins请求结束
