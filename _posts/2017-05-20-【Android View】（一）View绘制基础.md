



****
# ViewRoot DecorView
1.View的绘制流程从ViewRoot的performTraversals方法开始。

2.performTraversals->performMeasure->performLayout->performDraw

3.performMeasure->measure->onMeasure->子measure依次循环，遍历整个View树

performLayout、performDraw类似

而performDraw是在draw方法中dispatchDraw实现的

4.DecorView包含 titlebar和content View层事件都是经过DecorView才传递给子View。

![View绘制流程图](http://img.blog.csdn.net/20160426121725483)

# MeasureSpec
1.MeasureSpec度量规格。 32位值，高两位代表SpecMode,低两位代表SpecSize

2.模式常量：
* public static final int UNSPECIFIED = 0 << MODE_SHIFT;//父不对View有任何限制，要多大给多大。 一般用于系统内部，表示一种**测量中**的状态。
~~（多表现为：一般是父布局不清楚自己有多大（wrap_content），而子布局又match_parent）~~
* public static final int EXACTLY = 1 << MODE_SHIFT;//View有确定值。具体数值和大部分的match_parent情况（如果父View的大小确定）都是EXACTLY。
* public static final int AT_MOST = 2 << MODE_SHIFT;//父View对它指定了一个大小。 子View不得大于这个值。 对应wrap_content。

3.DecorView、普通View的测量
* DecorView：Window窗口尺寸以及本身的LayoutParams。
```
**ViewRootImpl**

//参数备注：host是DecorView、 lp是窗体属性、desiredWindowWidth是屏幕宽度、desiredWindowHeight屏幕高度
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    //用来确定childView的属性。因为需要遍历整个树，所以需要先确定DecortView的大小
    int childWidthMeasureSpec;
    int childHeightMeasureSpec;
    boolean windowSizeMayChange = false;
    // Debug代码
    boolean goodMeasure = false;
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
       //....省略大量代码
       //窗体大小不确定的情况下才做的处理，一般DecorView对应的Activity窗体大小都是最大。 所以不需要看
    }

    if (!goodMeasure) {
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);//获取DecoView的宽和模式
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);//高和模式
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);//开始执行测量
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }

    // Debug代码

    return windowSizeMayChange;
}


private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);//RootView和窗体大小一样大，直接设定具体大小为窗体大小
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);//RootView只想刚好包含内容大小，并且不能大于Window窗口大小
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);//RootView规定了自己的大小，那么直接给它这个大小
        break;
    }
    return measureSpec;
}
```

* 普通的View：父View的MeasureSpec以及自身的LayoutParams有关
```
**ViewGroup**

protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    //测量孩子的WidthMeasureSpec。 传入父亲的widthMeasureSpec,传入父亲的左右padding以及孩子的左右margin之和,传入父亲已经被用去的width,传入孩子View的宽度
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    //同上
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    //测量完成
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;
    //根据父亲的模式来度量
    switch (specMode) {
    // Parent has imposed an exact size on us
    //父亲已经有规定好的大小
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {//给孩子要求的大小
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {//孩子要求和父亲一样大，那么就和父亲一样大
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {//孩子想包含自己的孩子，那么要规定大小不能超过父亲的大小
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    // 父亲有一个最大值得规定，说明父亲是wrap_content。并且这个最大值是它的父亲的大小。
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {//孩子有想要的确切大小，就给它
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {//孩子想和父亲一样大，但是父亲大小不确定，所以孩子的只能是AT_MOST模式，并且最大值不能大于父亲的父亲的大小
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) { //孩子想包含自己的孩子，所以孩子的只能是AT_MOST模式，并且最大值不能大于父亲的父亲的大小
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    // 父亲被要求看看自己的孩子们有多大？
    // 这个其实没看太懂
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {//孩子有自己固定的大小
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {//孩子想和父亲一样大，但是父亲也不知道自己有多大，所以还需要测量
            // Child wants to be our size... find out how big it should
            // be
            resultSize = 0;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {//孩子想包含自己孩子的大小，所以孩子也只能等待测量自己孩子的大小。
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize =  0;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```


## 总结
| childLayoutParams\parentSpecMode| EXACTLY| AT_MOST| UNSPECIFIED|
| :---- |:---|:-------|:-------------|
| dp/px | EXACTLY childSize | EXACTLY childSize | EXACTLY childSize |
| match_parent | EXACTLY parentSize| AT_MOST parentSize | UNSPECIFIED 0 |
| wrap_content | EXACTLY parentSize | AT_MOST parentSize | UNSPECIFIED 0 |

* 子View是固定宽和高：那么不管父View的MeaureSpec是什么，子View的MeasureSpec都是精确模式（EXACTLY），且大小为LayoutParams的大小
* 子View是match_parent：如果父View是精确模式（EXACTLY），那么子View也是精确模式（EXACTLY）并且大小为父View的剩余空间；如果父View是最大模式（AT_MOST），那么子View也是最大模式（AT_MOST）并且大小不会超过父View的剩余大小
* 子View是wrap_content：不管父View是精确模式（EXACTLY）还是最大模式（AT_MOST）,子View总是最大模式（AT_MOST）并且大小不会超过父View剩余空间
* UNSPECIFIED模式：**主要用于系统内部多次Measure情形，一般我们不关注该模式**


> 参考资料：《Android开发艺术探索》


