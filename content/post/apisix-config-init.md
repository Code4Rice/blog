---
author: "Kyros"
date: 2022-03-09
title: "APISIX 源码阅读 - config本地配置初始化"
tags: [
"apisix",
"cloudnative",
]
categories: [
"APISIX",
]
---

# 前言
继 [《APISIX 源码阅读 - http init、init worker阶段》](http://code4rice.com/post/apisix-http-init) 阶段中
的配置初始化部分，配置初始化的流程/细节。

# core.config 初始化
首先我们可以看到，apisix 的初始化分了三种类型，目录下也分为三种配置文件。

```shell
|-- apisix
|   |-- core
|       |-- config_etcd.lua
|       |-- config_local.lua
|       |-- config_yaml.lua
```

而在实际的初始化过程中，并没有指明 config 的初始化是基于 etcd 还是 yaml，
或者是其他类型的配置中心，如下：

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

为此，我们继续深入 core ，看看对应的 schema，core.config 是如何初始化的：

```lua
-- apisix/core.lua 19
-- 获取本地配置
local local_conf, err = require("apisix.core.config_local").local_conf()
if not local_conf then
    error("failed to parse yaml config: " .. err)
end

-- 如果本地配置中有指定配置中心的话，就按配置中心的来，如果没有指定，则默认走 etcd
local config_center = local_conf.apisix and local_conf.apisix.config_center
        or "etcd"
log.info("use config_center: ", config_center)

-- 对应配置中心的初始化
local config = require("apisix.core.config_" .. config_center)
config.type = config_center
```

这里我们可以看到，apisix 的配置初始化是分了以下几个步骤：

1. 初始化本地配置 （local_config）
2. 获取环境中心的变量（config_center）
3. 对应环境中心变量的内容去做指定类型的初始化（etcd/yaml）

这里我们主要是看第一步，也就是 apisix 如何去初始化本地配置的。

# 本地配置初始化

可以看到，在 config_local 中，主要的核心代码如下：
```lua
-- apisix/core/config_local.lua 33
function _M.local_conf(force)
    if not force and config_data then
        return config_data
    end

    -- 将 yaml 配置读取，并合并
    local default_conf, err = file.read_yaml_conf()
    if not default_conf then
        return nil, err
    end

    -- fill the default value by the schema
    -- 配置结构检查
    schema.validate(default_conf)

    config_data = default_conf
    return config_data
end
```

这里主要介绍 apisix 如何读取 yaml 配置的：

apisix 本地配置由 用户配置（config.yaml） + 系统默认配置（config-default.yaml）组合而成的。

所以它的本地配置初始化流程，从代码上依次如下：
1. 配置 apisix 项目所在路径。
2. 读取路径下系统默认配置 config-default.yaml 并序列化。
3. 读取用户配置 config.yaml 并结合 **环境变量动态初始化** 。
4. 合并用户配置和系统默认配置。

```lua
-- 读取 yaml 配置
function _M.read_yaml_conf(apisix_home)
    -- 配置 apisix 项目所在地址
    if apisix_home then
        profile.apisix_home = apisix_home .. "/"
    end

    -- 拼接 config-default 配置文件所在路径
    local local_conf_path = profile:yaml_path("config-default")

    -- 读取文件
    local default_conf_yaml, err = util.read_file(local_conf_path)
    if not default_conf_yaml then
        return nil, err
    end

    -- yaml 文件序列化
    local default_conf = yaml.parse(default_conf_yaml)
    if not default_conf then
        return nil, "invalid config-default.yaml file"
    end

    -- 拼接 config 配置文件所在路径
    local_conf_path = profile:yaml_path("config")

    -- 读取文件
    local user_conf_yaml, err = util.read_file(local_conf_path)
    if not user_conf_yaml then
        return nil, err
    end

    local is_empty_file = true
    -- 如果 line 为空，或者当前 line 最开头的字符为 # 或者 $。将 is_empty_file 设为 false
    -- is_empty_file 字面意思，主要用来判断用户配置是否为空
    for line in str_gmatch(user_conf_yaml .. '\n', '(.-)\r?\n') do
        if not is_empty_yaml_line(line) then
            is_empty_file = false
            break
        end
    end

    -- 用户配置不为空再做动态配置 + 合并
    if not is_empty_file then
        -- yaml 文件序列化
        local user_conf = yaml.parse(user_conf_yaml)
        if not user_conf then
            return nil, "invalid config.yaml file"
        end


        -- 结合环境变量将 yaml 动态参数初始化
        local ok, err = resolve_conf_var(user_conf)
        if not ok then
            return nil, err
        end

        -- 将 user_conf 内容合入 default_conf 里
        ok, err = merge_conf(default_conf, user_conf)
        if not ok then
            return nil, err
        end
    end

    return default_conf
end
```

补充一下动态变量初始化的流程：

1. 通过正则表达式（%$%{%{%s*([%w_]+[%:%=]?.-)%s*%}%}）扫描用户配置文件。
2. 将扫描到的配置（例：http://${{ETCD_HOST:=localhost}}:2379）单独提取。
3. 从环境变量中找到对应的（例：ETCD_HOST）并替换。
4. 回到第一步继续扫描。

补充：

> 为了避免源码过多导致的阅读困难，这一篇主题主要讲的是，在初始化的阶段，
基于 apisix 配置中心的配置初始化，而这一系列初始化之后，对应的配置
中心初始化才可以继续进行下去（etcd/yaml），感兴趣的话可以持续关注。

