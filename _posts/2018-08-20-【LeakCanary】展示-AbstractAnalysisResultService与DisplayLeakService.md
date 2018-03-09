

## AbstractAnalysisResultService

* 继承自`IntentService`

* 从名字可以看出是定义一个分析内存泄露结果的`Service`

<!-- more -->


### sendResultToListener方法

```

public static void sendResultToListener(Context context, String listenerServiceClassName, HeapDump heapDump, AnalysisResult result) {
    Class listenerServiceClass;
    try {
        listenerServiceClass = Class.forName(listenerServiceClassName);
    } catch (ClassNotFoundException var6) {
        throw new RuntimeException(var6);
    }

    Intent intent = new Intent(context, listenerServiceClass);
    intent.putExtra("heap_dump_extra", heapDump);
    intent.putExtra("result_extra", result);
    context.startService(intent);
}
```

* `listenerServiceClassName `可以从代码看出，使用反射获取了对应的`Class`，然后传入到`Intent`。 

* 这里的作用，是启动对应的实现`Service`。 后面分析可以知道该子类就是`DisplayLeakService`

* `HeapDump` 根据`HeapDump`的成员变量

```

public final File heapDumpFile;
public final String referenceKey;
public final String referenceName;
public final ExcludedRefs excludedRefs;
public final long watchDurationMs;
public final long gcDurationMs;
public final long heapDumpDurationMs;
```

可以知道，这个类主要是包含了当前内存分析的内容。比如File heapDumpFile保存了具体的内存信息

* `AnalysisResult` 根据名字和成员变量

```

public final boolean leakFound;
public final boolean excludedLeak;
public final String className;
public final LeakTrace leakTrace;
public final Throwable failure;
public final long retainedHeapSize;
public final long analysisDurationMs;
```

可以知道，这个类就是记录分析结果的 如boolean leakFound可以看出来



### 其他方法

```

public AbstractAnalysisResultService() {
    super(AbstractAnalysisResultService.class.getName());
}

protected final void onHandleIntent(Intent intent) {
    HeapDump heapDump = (HeapDump)intent.getSerializableExtra("heap_dump_extra");
    AnalysisResult result = (AnalysisResult)intent.getSerializableExtra("result_extra");

    try {
        this.onHeapAnalyzed(heapDump, result);
    } finally {
        heapDump.heapDumpFile.delete();
    }

}

protected abstract void onHeapAnalyzed(HeapDump var1, AnalysisResult var2);
```

* `onHandleIntent`获取从`sendResultToListener`的`HeapDump`和`AnalysisResult`

* 通过抽象方法`onHeapAnalyzed`让子类去具体处理



## DisplayLeakService

* 直接继承`AbstarctAnalysisResultService` 实现了`onHeapAnalyzed`方法

* 在`onHeapAnalyzed`方法具体对`HeapDump`和`AnalysisResult`进行相关展示处理



```

protected final void onHeapAnalyzed(HeapDump heapDump, AnalysisResult result) {
    //...
    int notificationId1 = (int)(SystemClock.uptimeMillis() / 1000L);
    LeakCanaryInternals.showNotification(this, contentTitle, contentText, pendingIntent, notificationId1);
    this.afterDefaultHandling(heapDump, result, leakInfo);
}

```
* `LeakCanaryInternals.showNotification`可以知道，显示在状态栏内存泄露提示的地方。

* 其他细节就不具体去看了，如Notification的title、content的内容、以及是否需要保存`HeadDump`文件，都是具体的细微逻辑



## 小结

* 这样子的设计，可以让开发者自定义对于`HeapDump`和`AnalysisResult`的具体流程

* 扩展性好
