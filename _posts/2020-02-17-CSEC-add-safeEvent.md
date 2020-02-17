---
layout:     post
title: "云盾操作流程-添加安全事件"
subtitle: "H3C版本"
date:       2020-02-17 23:10:00
author:     "Autu"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 6_Cloud
  - CSEC
  - ovs
---

### 1. 部署防火墙（vfw）

配置节点：安全网络一为镜像网络，该防火墙只能保护安全网络一中的虚机

![img](/img/6-cloud/Csec/operation_add_safe_event/1581952222292.png)



### 2. 资源定义

添加地址对象

![img](/img/6-cloud/Csec/operation_add_safe_event/1581952664094.png)



### 3. 添加安全策略

![img](/img/6-cloud/Csec/operation_add_safe_event/1581952788770.png)



### 4. 发送安全事件命令

在 kali 机器中执行 <code># nikto -host 172.22.170.111</code> ，ip为目的ip，用户名：root，密码：toor

![img](/img/6-cloud/Csec/operation_add_safe_event/1581953014056.png)