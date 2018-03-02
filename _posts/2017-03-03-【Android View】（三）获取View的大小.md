---
layout: post
title: 【Android View】（三）获取View的大小
desc: 我的博客系统介绍
keywords: 'blog'
date: 2017-1-28T00:00:00.000Z
categories:
- life
tags:
- life
icon: fa-life
---
<!-- more -->
[【Android View】（一）View绘制基础](http://www.hanszone.xyz/2016/04/25/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%B8%80%EF%BC%89View%E7%BB%98%E5%88%B6%E5%9F%BA%E7%A1%80/)
[【Android View】（二）View Measure过程](http://www.hanszone.xyz/2016/04/26/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%BA%8C%EF%BC%89View%20Measure%E8%BF%87%E7%A8%8B/)
[【Android View】（三）获取View的大小](http://www.hanszone.xyz/2016/04/26/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%B8%89%EF%BC%89%E8%8E%B7%E5%8F%96View%E7%9A%84%E5%A4%A7%E5%B0%8F/)
[【Android View】（四）View Layout过程](http://www.hanszone.xyz/2016/05/12/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E5%9B%9B%EF%BC%89View%20Layout%E8%BF%87%E7%A8%8B/)
[【Android View】（五）View Draw过程](http://www.hanszone.xyz/2016/05/13/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%BA%94%EF%BC%89View%20Draw%E8%BF%87%E7%A8%8B/)
<!-- more -->

****
# 获取View大小
在一些情况下，我们需要获取View的大小。
但是在onCreate、onStart、onResume中是无法正确获得View的大小的。 因为measure的过程，不是和Activity生命周期同步执行的。 所以无法保证View已经被测量完成了。

## 1.Activity/View#onWindowFocusChanged
```
public void onWindowsFocusChanged(boolean hasFocus) {
    super.onWindowFoucusChanged(hasFoucus);
    if(hasFocus) {
        int width = view.getMeasuredWidth();
        int height  = view.getMeasuredHeight();
    }
}
```
当Activity窗口获取或者失去焦点的时候，均会被调用一次。 所以这个会被频繁调用。

## 2.view.post(runnable)
```
view.post(new Runnable() {
    
    @Override
    public void run() {
        int width  = view.getMeasuredWidth();
        int height = view.getMeasuredHeight();
    }
});
```
放入消息队列，当Looper就会调用这个方法时候，View也已经初始化好了。 （具体实现我目前还不知道，有待研究）

## 3.ViewTreeObserver
需要注意的是，现在这个方法已经不建议使用了。
```
ViewTreeObserver observer = view.getViewTreeObserver();
observer.addOnGlobalLayoutListener(new OnGloballLayoutListener() {
    @SuppressWarnings("deprecation")
    @Override
    public void onGlobalLayout() {
        view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
        int width  = view.getMeasuredWidth();
        int height = view.getMeasuredHeight();
    }
});
```
这个方法是，当View树发生改变的时候或者内部View的可见性发生改变，那么这个方法就会被执行。 

此外，当View树状态变化改变等，这个方法会被调用多次，所以用完之后就移除`view.getViewTreeObserver().removeGlobalOnLayoutListener(this);`


> 参考资料：《Android开发艺术探索》

**-Hans 2016.04.26 20:00**
