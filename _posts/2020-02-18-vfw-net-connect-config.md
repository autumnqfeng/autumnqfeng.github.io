---
layout:     post
title: "vfw网络配置"
subtitle: "防火墙配置网络连接"
date:       2020-02-18 14:14:00
author:     "Autu"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 6_Cloud
  - CSEC
  - vfw
---

#### 1. 登录

- 输入：<code>u c</code> ,回车登录
- 输入password，<code>admin</code> 

![img](/img/6-cloud/Csec/vfw/vfw_net_config/1582006558184.png)



#### 2. 进入管理口网络视图

- 首先输入<code>config</code> 进入config视图
- 输入<code>interface Ten-Ge 0/0/0</code> 进入管理口视图

![img](/img/6-cloud/Csec/vfw/vfw_net_config/1582006826280.png)



#### 3. 配置网络

- 配置ip地址：<code>ip address 172.22.100.191 24</code> ，此处ip为vfw管理口ip
- 配置静态路由和网关：<code>ip static-route 0.0.0.0 0 gateway 172.22.100.1</code> 

![img](/img/6-cloud/Csec/vfw/vfw_net_config/1582007052989.png)

