
---
layout: post
title:【LeakCanary】入口-ActivityRefWatcherdesc: 我的博客系统介绍
keywords: 'blog'
date: 2016-11-07T00:00:00.000Z
categories:
- life
tags:
- life
icon: icon-life
---
## 前提概念

要弄明白一个前提概念：

内存泄露实际上的本质是观察一个对象的引用的对象是否存活。那么这个对象必须是要有具体的生命周期的，在Android 的体现就是四大组件。

<!-- more -->


## ActivityLifecycleCallbacks

这个是一个Android Api 14以上允许在`Application`监听每一个`Activity`生命周期的回调方法。

```

private final ActivityLifecycleCallbacks lifecycleCallbacks = new ActivityLifecycleCallbacks() {
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
    }

    public void onActivityStarted(Activity activity) {
    }

    public void onActivityResumed(Activity activity) {
    }

    public void onActivityPaused(Activity activity) {
    }

    public void onActivityStopped(Activity activity) {
    }

    public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
    }

    public void onActivityDestroyed(Activity activity) {
        ActivityRefWatcher.this.onActivityDestroyed(activity);
    }
};
```

* 在`onActivityDestoryed`中，`LeakCanary`库基于这个生命回调，自动对`Activity`被结束之后，进行了后续`Activity`以及其引用对象是否被回收的情况。



```

private final Application application;
private final RefWatcher refWatcher;

public static void installOnIcsPlus(Application application, RefWatcher refWatcher) {
    if(VERSION.SDK_INT >= 14) {
        ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
        activityRefWatcher.watchActivities();
    }
}

public ActivityRefWatcher(Application application, RefWatcher refWatcher) {
    this.application = (Application)Preconditions.checkNotNull(application, "application");
    this.refWatcher = (RefWatcher)Preconditions.checkNotNull(refWatcher, "refWatcher");
}

void onActivityDestroyed(Activity activity) {
    this.refWatcher.watch(activity);
}

public void watchActivities() {
    this.stopWatchingActivities();
    this.application.registerActivityLifecycleCallbacks(this.lifecycleCallbacks);
}

public void stopWatchingActivities() {
    this.application.unregisterActivityLifecycleCallbacks(this.lifecycleCallbacks);
}
```

* `refWatcher.watch`对`Activity`进行具体的监听。

* 有上述代码可见，`RefWatcher`就是`LeakCanary`的原理所在了。

