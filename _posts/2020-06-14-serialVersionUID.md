---
layout:     post
title: "serialVersionUID作用"
subtitle: "serialVersionUID"
date:       2020-06-14 16:38:00
author:     "Autu"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
  - java
  - 面试题
---

## 面试题：serialVersionUID

**问：serialVersionUID有何用途？如果没定义会有什么问题？**

- 序列化是将对象的状态信息转换为可存储或传输的形式的过程。我们都知道，Java对象是保存在JVM的堆内存中的，也就是说，如果JVM堆不存在了，那么对象也就跟着消失了。

- 而序列化提供了一种方案，可以让你在即使JVM停机的情况下也能把对象保存下来的方案。就像我们平时用的U盘一样。把JAVA对象序列化成可存储或传输的形式（如二进制流），比如保存在文件中。这样，当再次需要这个对象的时候，从文件中读取出二进制流，再从二进制流中反序列化出对象。
- 但是，虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化ID是否一致，即**serialVersionUID**要求一致。
- 在进行反序列化时，JVM会把传来的字节流中的**serialVersionUID**与本地相应实体类的**serialVersionUID**进行比较，如果相同就认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常，即使**InvalidCastException**。这样做是为了保证安全，因为文件存储中的内容可能被篡改。
- 当实现**<code>java.io.Serializable接口</code>**的类<code>没有显式地定义</code>一个**serialVersionUID**变量时候，JAVA序列化机制会根据编译的Class自动生成一个**serialVersionUID**作序列化版本比较用，这种情况下，如果Class文件没有发生变化，就算再编译多次，**serialVersionUID**也不会变化的。但是，如果发生了变化，那么这个文件对应的**serialVersionUID**也就会发生变化。
- 基于以上原理，如果我们一个类实现了**Serializable接口**，但是没有定义**serialVersionUID**，然后序列化。在序列化之后，由于某些原因，我们对该类做了变更，重新启动应用后，我们相对之前序列化过的对象进行反序列化的话就会报错。