---
author: "Spawnris"
date: 2022-03-05
title: "APISIX源码阅读 - 概述"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---


# APISIX 源码阅读 - 概述



### 背景

随着互联网的高速发展，流量时代中的后台技术的发展变化，从过去的单体应用模式到SOA到微服务理念的提出，直到现在结合容器化等云原生技术的落地， 在架构中的网关角色也发生了巨大的变化，从传统网关演变成了现在的API网关。
  

先看下网关的定义，维基百科中这样描述

> 网关（英语：Gateway）是转发其他服务器通信数据的服务器，接收从客户端发送来的请求时，它就像自己拥有资源的源服务器一样对请求进行处理。有时客户端可能都不会察觉，自己的通信目标是一个网关。

可见传统意义的网关更多是透明无感知，承载了数据流量转发，协议转换等工作这样的角色。但微服务的出现导致服务拆分粒度更小，服务数量变的庞大，网关作为服务的统一入口，也要承担起更艰巨的工作，那就是对服务，对服务暴露的API的整个生命周期都进行管理，所以API网关的实现也成了整体架构中的重中之重。

当下的API网关的技术方案也是百花齐放，从技术栈层面划分有Java系列的ZUUL，Go的Envoy，Traefik，基于Openresty的Kong等等，但是无论在网关的性能，灵活性，还是到代码设计，社区活跃，云原生友好都或多或少有些缺陷，直到APISIX的出现，基于KONG设计理念的优化设计，与云原生的结合，解决了很多历史项目存在的不足，也让它成为了我的不二之选。

言归正传，所以本着更好的掌握学习了解方案本身，也为了在业务重有着最佳实践的目的，我们决定来一起阅读和学习APISIX的源码，也通过文章的形式呈现出来，与大家更好的互动和交流。



### 整体架构


![架构图](https://cdn.jsdelivr.net/gh/apache/apisix@release/2.12/docs/assets/images/flow-software-architecture.png)

先看下APISIX的整体架构，如下是官方给出的架构图，APISIX是一个基于OpenResty构建的API网关。
OpenResty是通过ngx_lua模块的方式将lua嵌入到Nginx中，通过将脚本的IO操作委托给Nginx的事件处理机制来实现对请求的并发，非阻塞的处理，而基于OpenResty构建的APISIX项目从而很好的得到了性能的保证。
同时OpenResty对Nginx作出了很充分的扩展，预埋了很多hook点，覆盖了整个请求的生命周期，而APISIX则通过对预先hook的实现来完成对网关的动态化配置能力。

![生命周期](https://moonbingbing.gitbooks.io/openresty-best-practices/content/images/openresty_phases.png)

APISIX在不同阶段的实现
Initialization阶段
```Go
init_by_lua -> http_init // 初始化参数
init_worker_by_lua -> http_init_worker // 初始化worker
```

Rewrite/Acess阶段
```Go
ssl_certificate_by_lua_block -> http_ssl_phase //证书处理
access_by_lua -> http_access_phase // 鉴权，过滤等
```

Content阶段
```Go
balancer_by_lua -> http_balancer_phase // 负载均衡
header_filter_by_lua - > http_header_filter_phase // 过滤header
body_filter_by_lua -> http_body_filter_phase // 过滤body
```

Log阶段
```Go
log_by_lua -> http_log_phase // 日志记录

```

在数据面的设计上，APISIX参考并延续了KONG中的数据模型，包含了Route，Service， Upstream，Consumer，Plugin，Script等

-   Upstream，上游服务节点的抽象，可配置负载均衡，健康检查等
-   Route， 转发给Upsteam的规则，可绑定插件(限流限速等)
-   Service, 相同规则的一组路由抽象
-   Consumer，服务消费者的用户认证体系，key-auth，JWT等
-   Plugin，插件，可绑定Route，Service，Consumer
-   Script，请求生命周期中执行的脚本，与Plugin互斥



### 环境配置
阅读源码前我们要先对APISIX以及相应的实验环境进行安装配置
首先我们启用一个Ubuntu的容器，做到环境的隔离，方便我们做各种学习调试同时对其他程序带来影响
```Go
// 拉取镜像
docker pull ubuntu
// 创建容器
docker create -it --privileged --name apisix -v ~/code:/code ubuntu bash -l

// 版本
cat /etc/lsb_release
DISTRIB_DESCRIPTION="Ubuntu 20.04.3 LTS"

```

采用源码的方式安装APISIX
```Go
// 项目克隆
git clone https://github.com/apache/apisix.git

// 安装依赖
cd utils
install-dependencies.sh

// 安装APISIX
make deps

// 启动APISIX
./bin/apisix start

```
到这里我们的实验环境算是正式部署完成，APISIX也成功启动了，接下来让我们看看整个项目的代码结构是怎样的，也好有个全局的认识，更好地帮助我们去理解他的设计和结构

### 源码结构
代码主要在项目中的apisix文件夹下，我们通过tree -L 1来看下

```Go
.
|-- admin // Admin API
|-- api_router.lua
|-- balancer // 负载均衡器
|-- balancer.lua
|-- cli // 项目中用到的脚本工具
|-- constants.lua
|-- consumer.lua // Consumer
|-- control // Control API
|-- core // 项目中用到的核心公共方法
|-- core.lua
|-- debug.lua
|-- discovery // 服务发现相关实现
|-- error_handling.lua
|-- http 
|-- init.lua
|-- patch.lua
|-- plugin.lua // 插件
|-- plugin_config.lua
|-- plugins // 自带插件
|-- router.lua // 路由
|-- schema_def.lua // json schema的相关定义
|-- script.lua // 脚本
|-- ssl // ssl证书相关
|-- ssl.lua
|-- stream // 流处理相关
|-- timers.lua // 定时器的封装
|-- upstream.lua
|-- utils
`-- wasm.lua
```

结合目录结构我们后续大概的后续相关内容会划分为如下
-   [启动与配置的生成]()
-   [初始化]()
-   [负载均衡 & 服务发现]()
-   [路由是怎么匹配的]()
-   [core中有哪些核心功能于优化]()
-   [四层协议的支持]()
-   [如何实现多语言的支持]()
-   [如何管理APISIX，Admin API]()
-   [如何编写单元测试]()

### 从start开始
对项目结构有了整体的了解之后，这么多文件，我们从哪里开始入手呢，别急我们先看下进程的情况
```Go
// ps查看下进程
ps aux | grep apisix

root        1383  0.0  0.0 287488  5004 ?        Ss   20:25   0:00 nginx: master process openresty -p /code/apisix -c /code/apisix/conf/nginx.conf
```
由此可以看出实际上apisix的启动也就是上面的start就是通过openresty加载了apisix生成的独有nginx配置文件来而运行起来的nginx，那么这个apisix命令在启动时到底做了哪些事情，nginx.conf又是如何生成的呢？别急，我们从start出发，一探整个项目代码的真相。

