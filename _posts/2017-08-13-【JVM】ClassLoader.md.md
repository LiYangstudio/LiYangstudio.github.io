
---
layout: post
title:【JVM】ClassLoader.md
desc: 我的博客系统介绍
keywords: 'blog'
date: 2016-11-07T00:00:00.000Z
categories:
- life
tags:
- life
icon: icon-life
---

<!-- more -->
## 参考资料

* 深入理解Java虚拟机

## 分类

Java中有三种类加载器：

* 1.Bootstrap ClassLoader：

    * 启动类加载器

    * JVM的一部分,C++实现

    * 负责加载<JAVA_HOME>\lib目录中的类，并且按文件名识别

    * 开发者不可调用，如果想通过双亲委派给Bootstrap，直接返回null即可

* 2.Extension ClassLoader：

    * 扩展类加载器，由ExtClassLoader实现

    * 负责加载了<JAVA_HOME>\lib\ext目录中的类

* 3.AppClassLoader：

    * 应用程序类加载器，由AppClassLoader实现

    * 调用ClassLoader.getSystemClassLoader()即可获取，所以一般也称为系统类加载器

    * 开发者可以直接使用，如果开发者没有自定义加载器，一般默认使用该类加载器



## 双亲委派模式

如图所示：

工作过程：

* 1.ClassLoader收到类加载，调用父类加载（partent.loadClass()）

* 2.父类加载成功，则自己不加载

* 3.父类加载失败，则自己加载



双亲委派实际上非常简单，很好理解。



这样设计的一些理由：

* 1.安全、安全、安全  可以避免如java.lang.String核心类被恶意替换，因为由最顶级的BootStrapClassLoader加载

* 2.保证类的加载逻辑清晰，不会出先同包名的类被同一个ClassLoader加载两次。注：但是可以被两个ClassLoader同时加载，但是他们两个并不是同一个类了。

* 3.一个类是否相等，除了要考虑全限定名之外，还要保证是被同一个ClassLoader加载



## 破坏双亲委派模型

双亲委派模型是Java推荐的模型，但他并不是一个强制要求

通过破坏双亲委派模型，在某些特定场景下会有更好的用途：

* 使Parent最后加载，达到动态替换某个具体类的版本（通过线程上下文类加载器 ThreadContextClassLoader）

* 修改ClassLoader行为，到达热修复（Android中）

* ....





可以参考下面的文章：

[CSDN - Java类加载器（Class Loader）之详解](http://blog.csdn.net/Radic_Feng/article/details/6897898)

[CSDN - java classLoader体系结构使用详解](http://blog.csdn.net/yaerfeng/article/details/51052576)

线程类加载器的使用：

[Java类加载器之线程上下文类加载器(ContextClassLoader)](http://blog.leanote.com/post/medusar/Java%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6)


**-Hans**
