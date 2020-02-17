---
layout:     post
title: "OVS审计模式"
subtitle: "添加删除查看镜像口"
date:       2020-02-17 23:00:00
author:     "Autu"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 6_Cloud
  - CSEC
  - ovs
---

1: 创建mirror.  名字固定为csecmirror_ljq_net

ovs-vsctl -- set bridge ljq_net mirrors=@m -- --id=@vnet6 get port vnet6 -- --id=@m create mirror name=csecmirror_ljq_net  output-port=@vnet6



ovs-ofctl mod-port ljq_net vnet6 NO-FLOOD



删除Mirror 

ovs-vsctl remove Bridge vs_business mirrors 2179581d-57de-4d59-b58c-67c0f6f94a4d



2: 加保护一个虚机网卡. 假设其对应的port 名字是vnet6

UUID=$(ovs-vsctl get port vnet6 _uuid)

ovs-vsctl add Mirror csecmirror_ljq_net  select_dst_port $UUID

ovs-vsctl add Mirror csecmirror_ljq_net  select_src_port $UUID





3: 去保护一个虚机网卡. 假设其对应的port 名字是vnet6

UUID=$(ovs-vsctl get port vnet6 _uuid)

ovs-vsctl remove Mirror csecmirror_ljq_net  select_dst_port $UUID

ovs-vsctl remove Mirror csecmirror_ljq_net  select_src_port $UUID



4.查看mirror

ovs-vsctl list mirror



查看当前虚拟机接口是否在mirror中

1.virsh list 查看虚拟机列表

![img](/img/6-cloud/Csec/ovs/clipboard01.png)

2.根据id查看配置文件

virsh dumpxml 77

看接口名

![img](/img/6-cloud/Csec/ovs/clipboard02.png)

3.通过接口名查uuid

 ovs-vsctl get Port vnet3 _uuid

![img](/img/6-cloud/Csec/ovs/clipboard03.png)

\4. 查看镜像口配置

 ovs-vsctl list mirror

![img](/img/6-cloud/Csec/ovs/clipboard04.png)







  970  virsh list

  971  virsh dumpxml 7

  972  (ovs-vsctl get port vnet2 _uuid

  973  ovs-vsctl get port vnet2 _uuid

  974  ovs-vsctl list mirror

  975  virsh list

  976  virsh dumpxml 1