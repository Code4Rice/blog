---
author: "Spawnris"
date: 2022-03-06
title: "APISIX源码阅读 - 启动"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---

# APISIX源码阅读 - 启动


### apisix命令
书接上文，我们从apisix start命令启动了APISIX，并通过ps查看到实际启动的进程的指令及参数却是` openresty -p /code/apisix -c /code/apisix/conf/nginx.conf`，那么我们就从这个apisix命令看一看，到底start的过程做了什么

### apisix.sh
打开apisix文件发现该文件实际上是一个shell脚本，代码块如下，主要分为三部分

```bash
# 确定项目中cli/apisix.lua的位置
if [ -s './apisix/cli/apisix.lua' ]; then
    APISIX_LUA=./apisix/cli/apisix.lua
elif [ -s '/usr/local/share/lua/5.1/apisix/cli/apisix.lua' ]; then
    APISIX_LUA=/usr/local/share/lua/5.1/apisix/cli/apisix.lua
else
    APISIX_LUA=/usr/local/apisix/apisix/cli/apisix.lua
fi

# 确认openresty位置，版本以及lua的版本

OR_BIN=$(which openresty || exit 1)
OR_EXEC=${OR_BIN:-'/usr/local/openresty-debug/bin/openresty'}
OR_VER=$(openresty -v 2>&1 | awk -F '/' '{print $2}' | awk -F '.' '{print $1"."$2}')
LUA_VERSION=$(lua -v 2>&1| grep -E -o  "Lua [0-9]+.[0-9]+")

# 确认cli/apisix.lua的执行方式
# 如果Openresty版本为1.19则使用LuaJIT的方式执行
# 否则如果Lua版本为5.1，则使用lua直接执行apisix.lua
# 如果既没有OpenResty1.19也没有Lua5.1那么报错

if [[ -e $OR_EXEC && "$OR_VER" =~ "1.19" ]]; then
    LUAJIT_BIN=$(${OR_EXEC} -V 2>&1 | grep prefix | grep -Eo 'prefix=(.*)/nginx\s+--' | grep -Eo '/.*/')luajit/bin/luajit
    echo "$LUAJIT_BIN $APISIX_LUA $*"
    exec $LUAJIT_BIN $APISIX_LUA $*
elif [[ "$LUA_VERSION" =~ "Lua 5.1" ]]; then
    echo "lua $APISIX_LUA $*"
    exec lua $APISIX_LUA $*
else
    echo "ERROR: Please check the version of OpenResty and Lua, OpenResty 1.19 + LuaJIT or OpenResty before 1.19 + Lua 5.1 is required for Apache APISIX."
fi

```

综上所述，apisix.sh文件中主要做的就是判断`cli/apisix.lua`的位置，并通过环境中的LuaJIT或者lua来解释执行cli/apisix.lua，所以真正的入口文件是在项目中cli目录下的apisix.lua文件，那么我们进一步看看cli/apisix.lua到底又做了哪些


### apisix.lua

````Lua
local pkg_cpath_org = package.cpath
local pkg_path_org = package.path

local apisix_home = "/usr/local/apisix"
local pkg_cpath = apisix_home .. "/deps/lib64/lua/5.1/?.so;"
                  .. apisix_home .. "/deps/lib/lua/5.1/?.so;"
local pkg_path = apisix_home .. "/deps/share/lua/5.1/?.lua;"

-- 将apisix默认的模块路径和c模块路径添加到全局的搜索路径中
package.cpath = pkg_cpath .. pkg_cpath_org
package.path  = pkg_path .. pkg_path_org

-- 通过引入cli下的env模块来确认整个项目运行的相关环境参数
local env = require("apisix.cli.env")(apisix_home, pkg_cpath_org, pkg_path_org)

-- 引入cli下的ops模块，ops模块中则实现了apisix命令中所有的动作选项
local ops = require("apisix.cli.ops")

# 具体动作的执行入口，目前支持help，version，init，start，realod等等指令
ops.execute(env, arg)
````
通过cli下apisix.lua的代码内容可以看出主要逻辑只有三部分，一部分是设置默认模块路径，第二部分是获取当前项目环境的变量，通过cli/env.lua实现，第三部分则是整个apisix命令的入口，通过cli/ops.lua实现，那我们继续了解看看env.lua中都包含了哪些环境变量，ops.lua又是如何执行启动动作的

#### env.lua
````lua

return function (apisix_home, pkg_cpath_org, pkg_path_org)

    -- 获取内核可同时打开文件描述符的最大值
    local res, err = util.execute_cmd("ulimit -n")
    if not res then
        error("failed to exec ulimit cmd \'ulimit -n \', err: " .. err)
    end
    local ulimit = tonumber(util.trim(res))
    if not ulimit then
        error("failed to fetch current maximum number of open file descriptors")
    end
    
    -- 判断项目的根目录在什么位置
    -- 如果脚本执行cli/apisix.lu的文职是由./这样的相对路径开头，那么获取当前路径作为项目的根目录
    local is_root_path = false
    local script_path = arg[0]
    if script_path:sub(1, 2) == './' then
        apisix_home = util.trim(util.execute_cmd("pwd"))
        if not apisix_home then
            error("failed to fetch current path")
        end

       --   判断项目根目录是否在系统的根目录(/root)下
        if str_find(apisix_home .. "/", '/root/', nil, true) == 1 then
            is_root_path = true
        end

        local pkg_cpath = apisix_home .. "/deps/lib64/lua/5.1/?.so;"
                          .. apisix_home .. "/deps/lib/lua/5.1/?.so;"

        local pkg_path = apisix_home .. "/?/init.lua;"
                         .. apisix_home .. "/deps/share/lua/5.1/?/init.lua;"
                         .. apisix_home .. "/deps/share/lua/5.1/?.lua;;"

        -- 根据确认的项目根目录路径设置lua模块路径以及c模块路径，并将路径放入全局路径中
        -- 不清楚为什么不把默认路径的拼接逻辑放到此代码块中，而是放到了入口的apisix.lua中
        package.cpath = pkg_cpath .. package.cpath
        package.path  = pkg_path .. package.path
    end

    do
        -- 通过pcall执行require table.new来判断LuaJIT版本是否大于2.1（因为LuaJIT在2.1才引入table.new反法)
        local ok = pcall(require, "table.new")
        if not ok then
             -- 通过pcall来判断引入cjson模块是否成功
            -- 如果LuaJIT版本不大于2.1且引入cjson模块成功，需要删除lua本身的cjson模块，而是使用OpenResty中的cjson模块
            local ok, json = pcall(require, "cjson")
            if ok and json then
                stderr:write("please remove the cjson library in Lua, it may "
                            .. "conflict with the cjson library in openresty. "
                            .. "\n luarocks remove lua-cjson\n")
                exit(1)
            end
        end
    end

     -- OpenResty真正的启动执行命令和参数
    local openresty_args = [[openresty -p ]] .. apisix_home .. [[ -c ]]
                           .. apisix_home .. [[/conf/nginx.conf]]

    -- 所欲etcd的最小版本
    local min_etcd_version = "3.4.0"

   -- env.lua主要确定的环境变量返回并传入到ops中
    return {
        apisix_home = apisix_home,
        is_root_path = is_root_path,
        openresty_args = openresty_args,
        pkg_cpath_org = pkg_cpath_org,
        pkg_path_org = pkg_path_org,
        min_etcd_version = min_etcd_version,
        ulimit = ulimit,
    }
end
````
 env.lua主要确认了项目运行中所需要的环境变量主要包含
-  apisix_home, 项目真正的根目录（这很关键）
-  is_root_path, 项目是否在系统根目录下，用来确认woker的启动权限
-  openresty_args, 真正的进程启动命令
-  pkg_cpath_org, 项目默认的c模块路径
- pkg_path_org, 项目默认的lua模块路径
- min_etcd_version, 项目所依赖etcd的最小版本
- ulimit, 系统内核设置的可同时打开文件描述符的最大值

除了确认这些环境所需要的变量外，出于性能原因，还用openresty中的cjson模块替换了Lua默认的cjson模块
        

### ops.lua
````lua
local function start(env, ...)
    -- Because the worker process started by apisix has "nobody" permission,
    -- it cannot access the `/root` directory. Therefore, it is necessary to
    -- prohibit APISIX from running in the /root directory.
    
    -- worker进程只有nobody权限，在root目录下没有权限
    if env.is_root_path then
        util.die("Error: It is forbidden to run APISIX in the /root directory.\n")
    end

   -- 创建日志目录
    local cmd_logs = "mkdir -p " .. env.apisix_home .. "/logs"
    util.execute_cmd(cmd_logs)

    -- check running
    
    -- 检测是否已经启动的Nginx
    local pid_path = env.apisix_home .. "/logs/nginx.pid"
    
    --  通过pid文件获取进程号
    local pid = util.read_file(pid_path)
    pid = tonumber(pid)
    if pid then
        -- 通过lsof -p pid的方式来判断进程是否在运行    
        -- 如果pid没有进程在使用，则logs/nginx.pid会被新的pid文件覆盖
        local lsof_cmd = "lsof -p " .. pid
        local res, err = util.execute_cmd(lsof_cmd)
        if not (res and res == "") then
            if not res then
                print(err)
            else
                print("APISIX is running...")
            end

            return
        end
        
        print("nginx.pid exists but there's no corresponding process with pid ", pid,
              ", the file will be overwritten")
    end

   -- 引入命令行解析器，可以通过-c选项自定义配置文件路径
    local parser = argparse()
    parser:argument("_", "Placeholder")
    parser:option("-c --config", "location of customized config.yaml")
    -- TODO: more logs for APISIX cli could be added using this feature
    parser:flag("--verbose", "show init_etcd debug information")
    local args = parser:parse()

   -- 判断是否有自定义配置文件路径
    local customized_yaml = args["config"]
    if customized_yaml then
        profile.apisix_home = env.apisix_home .. "/"

        --  备份默认配置文件
        local local_conf_path = profile:yaml_path("config")
        local err = util.execute_cmd_with_error("mv " .. local_conf_path .. " "
                                                .. local_conf_path .. ".bak")
        if #err > 0 then
            util.die("failed to mv config to backup, error: ", err)
        end
        
        -- 用自定义配置文件替换默认配置文件
        err = util.execute_cmd_with_error("ln " .. customized_yaml .. " " .. local_conf_path)
        if #err > 0 then
            util.execute_cmd("mv " .. local_conf_path .. ".bak " .. local_conf_path)
            util.die("failed to link customized config, error: ", err)
        end

        print("Use customized yaml: ", customized_yaml)
    end

    --  初始化apisix
    init(env)
    
    --  初始化etcd
    init_etcd(env, args)

    -- 通过启动OpenResty运行APISIX
    util.execute_cmd(env.openresty_args)
end

--  当前实现的动作
local action = {
    help = help,
    version = version,
    init = init,
    init_etcd = etcd.init,
    start = start,
    stop = stop,
    quit = quit,
    restart = restart,
    reload = reload,
    test = test,
}

-- 选项动作的执行，传入环境和参数
function _M.execute(env, arg)
    -- 没选择动作默认返回help
    local cmd_action = arg[1]
    if not cmd_action then
        return help()
    end

    -- 所选动作不在定义范围内，返回异常信息
    if not action[cmd_action] then
        stderr:write("invalid argument: ", cmd_action, "\n")
        return help()
    end

    -- 执行具体函数并传入环境和参数
    action[cmd_action](env, arg[2])
end


return _M
````

终于看到了命令执行的真正入口，就是execute这个方法，execute方法通过命令行传进来的动作参数来进一步执行对应的函数，目前已经支持的不仅仅有`start`, 还有`init`, `init_etcd`, `stop`,`quit`,`restart`, `reload`, `test`, `help`, `version`, 而我们一直关注的start主要完成了日志的创建，nginx进程的判断，自定义配置文件确认，进而完成apisix配置的初始化，etcd的初始化，最后执行了` openresty -p /code/apisix -c /code/apisix/conf/nginx.conf`从而启动apisix，我们也通过一张图来总结整个apisix在OpenResty层面是怎么启动的

### 整体流程图
![流程图](https://picabstract-preview-ftn.weiyun.com/ftn_pic_abs_v3/f43e2abf7f2315a8c29062bdab17b463a6a412c03fed6a8360da3e1233d21b39c2237419a05fea9b2b8e3b50267924b8?pictype=scale&from=30013&version=3.3.3.3&uin=584800952&fname=apisix%E5%90%AF%E5%8A%A8.png&size=750)


通过上面的流程图和源码的注释想必已经能很清晰的描述出apisix start到openresty启动的过程了，但是apisix的那些功能又是怎么实现的，前篇文章说的各个阶段的hook又在哪里？其实从最终执行的指令` openresty -p /code/apisix -c /code/apisix/conf/nginx.conf`不难猜出，这一切都在nginx.conf整个配置文件中，那这个配置文件又是如何生成的呢，答案就在指令执行前的init和init_etcd的初始化流程中，所以我们将在后面的篇幅继续从源码来剖析apisix的初始化过程，以及etcd的初始化过程。