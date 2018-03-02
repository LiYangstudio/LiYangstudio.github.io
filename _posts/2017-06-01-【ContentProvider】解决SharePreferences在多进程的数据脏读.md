
---
layout: post
title: 【ContentProvider】解决SharePreferences在多进程的数据脏读
desc: 我的博客系统介绍
keywords: 'blog'
date: 2017-6-02T00:00:00.000Z
categories:
- blog
tags:
- blog
icon: fa-blog
---
## 问题描述

由于WebView内存泄露的原因，在工程中一般都会是用多进程。

当用户第一次登陆完毕之后，在WebView所在的进程，通过`SharePreferences`获取token的时候，会发现获得到的数据为空。

而重新启动App就正常了，查明原因是由`SharePreferences`脏读引起的。
<!-- more -->


## 解决思路

**通过`ContentProvider`将`SharePreferences`放在一个进程里面进行操作**



具体的解决方法已经在这篇文章里面有写:

[多进程中安全的使用SharedPreferences](http://melodyxxx.com/2016/08/04/%E5%A4%9A%E8%BF%9B%E7%A8%8B%E4%B8%AD%E5%AE%89%E5%85%A8%E7%9A%84%E4%BD%BF%E7%94%A8SharedPreferences/)



简而言之过程是：

* 重写`ContentProvider`的`call`方法

* 调用 

```

context.getContentResolver().call(sUri, PreferencesProvider.METHOD_PUT_BOOLEAN, null, data);
```

* 本质是通过`ContentProvider`的`aidl`实现的



## 本质

通过`SharePreferences`源码分析，得出结果：

* `SharePreferences`在创建的时候，就会讲数据加载到内存。内部采用`Map`集合进行数据的内存管理

* 若子进程的`SharePreferences`率先初始化

* 之后主进程通过`SharePreferences`修改了当前的数据，主进程的内存`Map`和磁盘都得到了修改

* 而子进程的`SharePreferences`实际上是不会有任何改变的



源代码分析可以看我的另外一篇笔记：[【SharePreferences】源码分析](http://www.hanszone.xyz/2016/10/27/%E3%80%90SharePreferences%E3%80%91%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

**-Hans 2016.10.27**

