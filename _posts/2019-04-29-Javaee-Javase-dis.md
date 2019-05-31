---
layout: post
title: "Java SE、Java EE区别、联系"
subtitle: "JDK、Jre的区别"
author: "Autu"
header-style: text
tags:
  - Java基础
---

### javaME，javaSE和javaEE是什么

---

①Java SE：标准版Java SE（Java Platform，Standard Edition）。JavaSE以前成为J2SE。 
<font color="red">**它语序开发和部署在桌面，服务器，嵌入式环境和实时环境中使用Java应用程序。**</font>
JavaSE包含了支持JavaWeb服务的开发的类，并为Java Platform,Enterprise Edition(Java EE)提供了基础。

②Java EE：企业版Java EE（Java Platform，Enterprise Edition）。这个版本以前成为J2EE。<font color="red">**企业版本帮助开发和部署可移植，健壮，可伸缩切安全的服务器端Java应用程序。**</font>

③Java ME：微型版Java ME（Java Platform，Micro Edition）。这个版本以前称为J2ME。<font color="red">**Java ME为在移动设备和嵌入式设备（笔记手机，PDA，电视机顶盒和打印机）上运行的应用程序提供一个健壮且灵活的环境。**</font>

</br>

<font size=3>

网络上普遍认为javaME就是用来开发嵌入式的，javaSE就是用来开发桌面的，javaEE就是用来开发企业端的。这也许没错，但是为什么我们采用SSH框架和SSM框架的时候使用的是javaEE的技术，为什么下载的是jdk就可以了呢。

现在企业里面，真正常用的JavaEE规范有什么？Servlet、JSP、JMS、JNDI。这些技术都只是充当了一个程序的入口而已。

oracle上提供的java EE是官方指定的javaEE规范，里面都是符合官方指定的javaEE组件，我们用SSM，SSH开发后台时使用到的只有Servlet、JSP、JMS等少量的java EE规范，没有必要使用orcale提供的java EE版本，直接使用jdk就可以（当然还需要maven等管理第三方的jar包来实现功能）

</br>

<!--<div align=center>-->

![image](F:/markdown/1554217315.png)

<!--</div>-->


<!--<img src="/F:/markdown/1554217315.png" width="100" height="100" />-->


</br>

Java SE是Java的标准版，主要用于桌面应用开发，同时也是Java的基础，它包含Java语言基础、JDBC（Java数据库连接性）操作、I/O（输出输出）操作、网络通信、多线程等技术。

</br>

java SE是基础，javaEE是基于java SE的基础知识的

</font>

</br>
</br>

### JDK 与 JRE 的区别：

---

![image](F:/markdown/1554218226.png)

</br>

<font color="red" size=3>**JRE(Java Runtime Enviroment)是Java的运行环境。面向Java程序的使用者，而不是开发者** </font>
<font size=3>
。如果你仅下载并安装了JRE，那么你的系统只能运行Java程序。JRE是运行Java程序所必须环境的集合，包含JVM标准实现及Java核心类库。它包括Java虚拟机、Java平台核心类和支持文件。
</font>
<font color="red" size=3> **它不包含开发工具(编译器、调试器等)。**</font>
    
</br>


<font color="red" size=3>**JDK(Java Development Kit)又称J2SDK(Java2 Software Development Kit)，是Java开发工具包，**</font>
<font size=3>
它提供了Java的开发环境(提供了编译器javac等工具，用于将java文件编译为class文件)和运行环境(提供了JVM和Runtime辅助包，用于解析class文件使其得到运行)。如果你下载并安装了JDK，那么你不仅可以开发Java程序，也同时拥有了运行Java程序的平台。
</font>
<font color="red" size=3>**JDK是整个Java的核心，包括了Java运行环境(JRE)，一堆Java工具tools.jar和Java标准类库 (rt.jar)。**
</font>

</br>
</br>
</br>
</br>
</br>