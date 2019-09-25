---
layout:     post
title:      "IDEA中删除项目再新建同名项目-解决方案"
subtitle:   "微服务-bug"
date:       2019-09-25 12:00:00
author:     "Autu"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 分布式
    - 微服务
    - BUG
---

#第一步：打开Project Structrue/Modules
此时我们发现该项目与其他项目有些不同，如图

![](/img/micro_service/bug/bug_01.png)

#第二步：把缺少的2个组件加入进来，点击左上角的加号

![](/img/micro_service/bug/bug_02.png)

#第三步：此时发现还是不行，打开settings/maven/Ignored Files

![](/img/micro_service/bug/bug_03.png)

#第四步：刷新maven项目