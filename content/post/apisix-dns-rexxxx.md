---
author: "Merlinlin"
date: 2022-03-08
title: "APISIX源码阅读 - 域名解析问题"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---


# 背景
在openresty写过涉及第三方服务请求相关逻辑的同学应该遇到过，默认情况下openresty在建链时无法解析域名，需要借助如`resty.dns.resolver`模块做域名解析的工作。

apisix则在服务初始化时解决了这个问题

# 替换默认的ngx.socket tcp、udp函数

不管是和mysql、redis还是其他第三方服务建链均离不开四层协议。

apisix在服务启动时替换掉默认的ngx.socket.tcp、udp函数
```
ngx_socket.tcp = function ()
    local phase = get_phase()
    if phase ~= "init" and phase ~= "init_worker" then
        return patch_tcp_socket(original_tcp())
    end

    return luasocket_tcp()
end

ngx_socket.udp = function ()
    return patch_udp_socket(original_udp())
end
```

在tcp、udp的connect、setpeername函数执行之前解析域名。使得依赖这两个接口的上层封装（如resty-http、resty-redis）无感修改，可以正常使用
```lua
--   ............
     elseif not ipmatcher.parse_ipv4(host) and not ipmatcher.parse_ipv6(host) then
         local err
         host, err = resolver.parse_domain(host)
         if not host then
             return nil, "failed to parse domain: " .. err
         end
     end
 end
 -- 此时host已替换为ip
 return old_tcp_sock_connect(sock, host, port, opts)
```
