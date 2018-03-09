


# 问题根源
在使用Handler发送消息的时候，Message有一个target字段引用了我们发送消息的Handler。 如果队列消息过多，这个Message迟迟得不到处理，若此时Activity被finish()，那么将会造成内存泄漏。 （Message保持有Handler的引用，所以也保持有Activity的引用，导致Activity无法销毁）
<!-- more -->
# 解决措施
首先要明白，为何Message持有Handler的引用，而也会持有Activity的引用
>在Java中，非静态的内部类和匿名内部类都会隐式地持有其外部类的引用。静态的内部类不会持有外部类的引用。

开发时，这样的写法就会导致内存泄漏：
```
private final Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ... 
    }
  }
```
解决办法：
```
public class MainActivity extends BaseActivity {
    private ImageView mImageView;

    private final Handler mHander = new MainActivityHandler(this);

    public static class MainActivityHandler extends Handler {
        private WeakReference<Activity> mActivity;

        public MainActivityHandler(MainActivity activity) {
            mActivity = new WeakReference<Activity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            MainActivity activity = (MainActivity) mActivity.get();
            if (activity != null)
                activity.mImageView.setImageBitmap(null);
        }
    }
}
```
说明：
1.将类声明为static，解决了内部类持有外部类的引用问题。 但带来了一个新的问题，就是因为不持有外部类的引用，所以静态内部类无法操作外部类的内容。 具体表现为：handleMessage方法，无法直接使用Activity的成员变量和方法。

2.解决无法使用外部类的成员变量和方法的途径就是：保持外部类的弱引用。 这样既做到避免内存泄漏，又做到了操作外部类成员变量的问题。

3.实际上，静态内部类和单独写一个外部类，达到的效果是一样的。不一定要用静态内部类实现。即将MainActivityHandler独立成一个类。

# 注意匿名内部类
开发中往往为了方便，总会直接new Runnable对象，然后交给线程池去execute。 

但上文也提到了，匿名内部类也会持有外部类的引用。 所以当Runnable对象迟迟没有被执行，这样也会导致内存泄漏。

解决措施还是和上文提及的一样：1.静态内部类 2.外部类

# 总结一句
当组件生命周期理应是结束时，而因为内部类或匿名内部类长时间处于等待处理，且它持有组件对象的引用（持有引用使得其生命无法结束），这样就可能导致内存泄漏。

