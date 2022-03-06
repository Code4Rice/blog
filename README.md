# blog
The blog of Code4Rice Team

# 如何参与blog编写

## 1.下载项目
```
git clone --recurse https://github.com/Code4Rice/blog
```

## 2.编写文章

- 在content/post文件夹下面创建md文件即可
- 本地校验, blog使用[hugo](https://gohugo.io/)生成，可通过hugo本地编译验证，也可以使用自己习惯的md编辑器验证(如typora, mou等）

文章头部基础信息, tags参数可以让文章通过tag检索，categories参数可以将文章聚合到相同目录下
```
---
author: "Spawnris"
date: 2022-03-06
title: "APISIX源码阅读 - 概述"
tags: [
    "apisix",
    "cloudnative",
]
categories: [
    "APISIX",
]
---

```

## 3.文章发布
- push文章到仓库即可，已添加webhook自动更新
