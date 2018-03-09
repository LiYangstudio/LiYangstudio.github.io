

# View的measure过程
前面说了，测量过程是：
![measure过程](http://img.blog.csdn.net/20160426125856290)
但这里明明是先测量ViewGroup，怎么先讲View了呢？

其实要明确一点的是，ViewGroup是继承自View的。 而且我们搜索ViewGroup里面的方法，发现并没有measure方法。 所以也就是说measure方法是调用View的measure方法的！

```
**View**
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int oWidth  = insets.left + insets.right;
        int oHeight = insets.top  + insets.bottom;
        widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
        heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
    }

    // Suppress sign extension for the low bytes
    long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
    if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {

        // first clears the measured dimension flag
        mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

        resolveRtlPropertiesIfNeeded();

        int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                mMeasureCache.indexOfKey(key);
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back       
            onMeasure(widthMeasureSpec, heightMeasureSpec);  // 测量我们自己
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } else {
            long value = mMeasureCache.valueAt(cacheIndex);
            // Casting a long to int drops the high 32 bits, no mask needed
            setMeasuredDimensionRaw((int) (value >> 32), (int) value);
            mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("View with id " + getId() + ": "
                    + getClass().getName() + "#onMeasure() did not set the"
                    + " measured dimension by calling"
                    + " setMeasuredDimension()");
        }

        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;

    mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
            (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
}

/**
 * Return true if o is a ViewGroup that is laying out using optical bounds.
 * @hide
 */
public static boolean isLayoutModeOptical(Object o) {
    return o instanceof ViewGroup && ((ViewGroup) o).isLayoutModeOptical();
}
```
看到方法定义：final。 也就是说这个方法是不能够被子类重写的。

然后看到里面有一个`onMeasure(widthMeasureSpec, heightMeasureSpec); `  通过它的注释可以看出来`measure ourselves, this should set the measured dimension flag back` 它通过onMeasure方法测量自己的大小。 所以也符合上文流程图的顺序。

那么看看View的onMeasure默认实现：
```
//View的onMeasure的默认实现
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));//这里注意 调用了setMeasuredDimension传入两个参数就是View 测量完成之后设置进入的大小。 这个在自定义View的时候，传入自己设置的大小。 这个是必须调用的方法。
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) { //这个模式是在ViewGroup中调用measureChild的时候就限定好的了。
    case MeasureSpec.UNSPECIFIED: 
        result = size;//这个size大小是上文 getDefaulSize返回的值大小。 这个情况一般是用在系统内部测量的情况下。
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY: 
        result = specSize;//这个值是两种情况。 如果是AT_MOST的话就是，父View剩余空间的大小；如果是EXACTLY的话，是View本身的具体大小。
        break;
    }
    return result;
}

//获取最小。 （备注mMinWidth的大小是android:minWidth设置的）
public static getSuggestedMinimumWidth() {
    //如果没有背景，那么则返回View的minWidth；如果有的话，那么就返回 两者中，较大的那一个
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
//同上
public static getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight: max(mMinHeight, mBackground.getMinimumHeight());
}

/**
 * <p>This method must be called by {@link #onMeasure(int, int)} to store the
 * measured width and measured height. Failing to do so will trigger an
 * exception at measurement time.</p>
 *  //这个方法必须在onMeasure调用。 用来存储已经测量好的View 宽高的模式和大小。 如果不被调用，在测量时抛出异常。
 *
 * @param measuredWidth The measured width of this view.  May be a complex
 * bit mask as defined by {@link #MEASURED_SIZE_MASK} and
 * {@link #MEASURED_STATE_TOO_SMALL}.
 * @param measuredHeight The measured height of this view.  May be a complex
 * bit mask as defined by {@link #MEASURED_SIZE_MASK} and
 * {@link #MEASURED_STATE_TOO_SMALL}.
 */
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

/**
 * Sets the measured dimension without extra processing for things like optical bounds.
 * Useful for reapplying consistent values that have already been cooked with adjustments
 * for optical bounds, etc. such as those from the measurement cache. 
*  // ?蹩脚翻译和理解。
 * // 保存没有经过额外处理（如optical bounds 光边界？）的测量值。 这个值是用来与已经被光边界（optical bounds）调整的值保持一致的。 比如那些从 缓存来的。
 *
 * @param measuredWidth The measured width of this view.  May be a complex
 * bit mask as defined by {@link #MEASURED_SIZE_MASK} and
 * {@link #MEASURED_STATE_TOO_SMALL}.
 * @param measuredHeight The measured height of this view.  May be a complex
 * bit mask as defined by {@link #MEASURED_SIZE_MASK} and
 * {@link #MEASURED_STATE_TOO_SMALL}.
 */
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}

/**
 * Return true if o is a ViewGroup that is laying out using optical bounds.
 * @hide
 */
public static boolean isLayoutModeOptical(Object o) {
    return o instanceof ViewGroup && ((ViewGroup) o).isLayoutModeOptical();
}
```

所以其实经过上面的分析我们可以得出一个道理:
**如果我们自定义View的时候，直接继承于View的话，那么我们需要在onMeasure里面做一些自己的处理。 因为如果不这样做的话，无论你设置match_parent还是wrap_cotent的话，它的大小都是match_parent。即直接占据父View的剩余空间**
这里给出一个简单处理例子：
```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
    if(widthSpecMode == MeasureSpc.AT_MOST && heightSpecMode == MeasureSpc.AT_MOST) {
        setMeasuredDimension(mWidth,mHeight);
    }else if(widthSpecMode  == MeasureSpec.AT_MOST){
        setMeasuredDimension(mWidth,heightSpecSize);
    }else if(heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSpecSize ,mHeight);
    }
}
```
主要是对AT_MOST进行处理，因为AT_MOST说明我们的View属性是被设置成wrap_content。 如上面**黑体加粗**所说，如果我们不处理的话，那么就会是match_parent了。 这和我们的原意不符。 所以我们会定义宽和高，mWidth和mHeight来限定在wrapc_content的情况。

# ViewGroup的measure过程
## 1.measure自身
经过View的分析，测量自身的过程就是调用onMeasure方法。 但是会发现，ViewGroup里面没有onMeasure方法。 

这是为什么？

其实也很容易理解：RelativeLayout、LinearLayout这些都是差异很大的ViewGroup。 所以onMeasure方法当然是交给它们去具体实现了。

以后再分析一下LinearLayout的源代码。

## 2.measure子View
作为ViewGroup，除了测量自己的大小之外。 当然也要担起测量自己子View的重任。

下面看看它的具体方法：
```
**ViewGroup**
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {//遍历自己的子View
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}

protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);//测量子View 宽的MeasureSpec
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);//测量子View 高的MeasureSpec
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);//调用子的measure方法
}

//这个方法上次分析的是一样的。
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
其实之前的看懂了，那么这里的代码也是很容易理解了。

我们来理一下ViewGroup的调用顺序：
```flow
start=>start: ViewGroup#onMeasure
op1=>operation: measureChildren//触发遍历子View
op2=>operation: measureChildren //子View根据父View的大小进行测量
op3=>operation: getChildMeasureSpec //具体获取子View MeasureSpec的方法
op4=>operation: child.measure//触发子View测量。
end=>end: View#measure

start->op1->op2->op3->op4->end
```
现在再看看开头的这个过程，发现完全一致。
![measure过程](http://img.blog.csdn.net/20160426125856290)


> 参考资料：《Android开发艺术探索》

## 文章解读和评价
