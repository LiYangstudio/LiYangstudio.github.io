

[【Android View】（一）View绘制基础](http://www.hanszone.xyz/2016/04/25/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%B8%80%EF%BC%89View%E7%BB%98%E5%88%B6%E5%9F%BA%E7%A1%80/)
[【Android View】（二）View Measure过程](http://www.hanszone.xyz/2016/04/26/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%BA%8C%EF%BC%89View%20Measure%E8%BF%87%E7%A8%8B/)
[【Android View】（三）获取View的大小](http://www.hanszone.xyz/2016/04/26/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%B8%89%EF%BC%89%E8%8E%B7%E5%8F%96View%E7%9A%84%E5%A4%A7%E5%B0%8F/)
[【Android View】（四）View Layout过程](http://www.hanszone.xyz/2016/05/12/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E5%9B%9B%EF%BC%89View%20Layout%E8%BF%87%E7%A8%8B/)
[【Android View】（五）View Draw过程](http://www.hanszone.xyz/2016/05/13/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%BA%94%EF%BC%89View%20Draw%E8%BF%87%E7%A8%8B/)
<!-- more -->
****
# View的layout过程
ViewGroup的测量过程分两个阶段一个是通过layout测量自己本身的位置，然后调用onLayout方法来确定所有子元素的位置。

先放一张时序图：
![View layout的时序图](http://img.blog.csdn.net/20160512235140635)

## View的layout方法
```
/**
 * Assign a size and position to a view and all of its
 * descendants //为View以及它的子类指定了一个大小和位置
 *
 * <p>This is the second phase of the layout mechanism.
    //这是一个布局机制的第二部
 * (The first is measuring). In this phase, each parent calls
 * layout on all of its children to position them.
   // （第一步是测量） 在这一步，每一个父布局调用layou方法让它子View测量它自己的位置
 * This is typically done using the child measurements
 * that were stored in the measure pass().</p>
   //这通常是使用子View它们本身在measure测量的大小来确定的     
 *
 * <p>Derived classes should not override this method.
 * Derived classes with children should override
 * onLayout. In that method, they should
 * call layout on each of their children.</p>
  // 继承View不应该重写这个方法。 但有包含子View的子类（指ViewGroup）应该复写onLayout，并且它应该调用它的每个子View的layout方法
 *
 * @param l Left position, relative to parent
 * @param t Top position, relative to parent
 * @param r Right position, relative to parent
 * @param b Bottom position, relative to parent
 */
@SuppressWarnings({"unchecked"})
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);//setFrame方法来确定自身的位置

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);//调用onLayout方法 （这个方法每个ViewGroup的实现不一样）
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            //通知布局已经变化的监听器
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```
从上面的源代码和注释可以知道：
1.View的子类不应该重写该方法。
2.ViewGroup应该复写onLayout，并且它应该调用它的每个子View的layout方法（View类本身的onLayout是一个空方法，因为ViewGroup才有子View）


然后我们看看setFrame和setOpticalFrame这两个方法：
```
/**
 * Return true if o is a ViewGroup that is laying out using optical bounds.
 * @hide
 */
public static boolean isLayoutModeOptical(Object o) {
    return o instanceof ViewGroup && ((ViewGroup) o).isLayoutModeOptical();
}

private boolean setOpticalFrame(int left, int top, int right, int bottom) {
    Insets parentInsets = mParent instanceof View ?
            ((View) mParent).getOpticalInsets() : Insets.NONE;
    Insets childInsets = getOpticalInsets();
    return setFrame(
            left   + parentInsets.left - childInsets.left,
            top    + parentInsets.top  - childInsets.top,
            right  + parentInsets.left + childInsets.right,
            bottom + parentInsets.top  + childInsets.bottom);
}
/**
 * Assign a size and position to this view. //指定这个View的大小和位置
 *
 * This is called from layout.
 *
 * @param left Left position, relative to parent
 * @param top Top position, relative to parent
 * @param right Right position, relative to parent
 * @param bottom Bottom position, relative to parent
 * @return true if the new size and position are different than the
 *         previous ones
 * {@hide}
 */
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;

    if (DBG) {
        Log.d("View", this + " View.setFrame(" + left + "," + top + ","
                + right + "," + bottom + ")");
    }

    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;

        // Remember our drawn bit
        int drawn = mPrivateFlags & PFLAG_DRAWN;

        int oldWidth = mRight - mLeft;
        int oldHeight = mBottom - mTop;
        int newWidth = right - left;
        int newHeight = bottom - top;
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

        // Invalidate our old position
        invalidate(sizeChanged);

        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);

        mPrivateFlags |= PFLAG_HAS_BOUNDS;


        if (sizeChanged) {
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }

        if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {
            // If we are visible, force the DRAWN bit to on so that
            // this invalidate will go through (at least to our parent).
            // This is because someone may have invalidated this view
            // before this call to setFrame came in, thereby clearing
            // the DRAWN bit.
            mPrivateFlags |= PFLAG_DRAWN;
            invalidate(sizeChanged);
            // parent display list may need to be recreated based on a change in the bounds
            // of any child
            invalidateParentCaches();
        }

        // Reset drawn bit to original value (invalidate turns it off)
        mPrivateFlags |= drawn;

        mBackgroundSizeChanged = true;
        if (mForegroundInfo != null) {
            mForegroundInfo.mBoundsChanged = true;
        }

        notifySubtreeAccessibilityStateChangedIfNeeded();
    }
    return changed;
}
```
发现setOpticalFrame方法实际上还是调用了setFrame方法。从setFrame方法的注释可以看出，这个方法就是真正用来指定View大小和位置的方法。

## ViewGroup的layout方法
刚刚看到View说实际上它的子类是不应该重写这个方法的。 我一开始很疑惑，因为如果不重写那么直接final关键字规定不就好了。  然后我看了一下TextView、ImageView发现的确是没有重写的。

然后看到ViewGroup的layout方法才恍然大悟。
```
/**
 * {@inheritDoc}
 */
@Override
public final void layout(int l, int t, int r, int b) {
    if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
        if (mTransition != null) {
            mTransition.layoutChange(this);
        }
        super.layout(l, t, r, b);
    } else {
        // record the fact that we noop'd it; request layout when transition finishes
        mLayoutCalledWhileSuppressed = true;
    }
}
```
这里面针对ViewGroup进行了一些代码的书写。

然后现在看一下ViewGroup的onLayout方法
```
/**
 * {@inheritDoc}
 */
@Override
protected abstract void onLayout(boolean changed,
        int l, int t, int r, int b);
```
发现是抽象方法，也就是每一个ViewGroup都必须重写这个方法。 

这也很合理，因为每一个ViewGroup确定子View的方法肯定是不同的。

瞄一眼LinearLayout的onLayout方法：
```
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```
这里就看到它本身还分成垂直方向和水平方向来测量。 

上面还说到:
> ViewGroup应该复写onLayout，并且它应该调用它的每个子View的layout方法

以layoutVertical为例：
```
/**
 * Position the children during a layout pass if the orientation of this
 * LinearLayout is set to {@link #VERTICAL}.
 *
 * @see #getOrientation()
 * @see #setOrientation(int)
 * @see #onLayout(boolean, int, int, int, int)
 * @param left
 * @param top
 * @param right
 * @param bottom
 */
void layoutVertical(int left, int top, int right, int bottom) {
   //.....

    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            
            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();
            
            //....

            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            childTop += lp.topMargin;
            // 设置child View的位置
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
```
看到这句重点的：
```
setChildFrame(child, childLeft, childTop + getLocationOffset(child),
```
是不是有点似曾识。 刚刚上面View其实际上确定位置就是通过setFrame来确定自身的位置的。

那这个方法很容易就知道它就是用来设置子View位置的:
```
private void setChildFrame(View child, int left, int top, int width, int height) {        
    child.layout(left, top, left + width, top + height);
}
```
很简单，发现它调用了child的layout方法。 

# 小结
实际上整个流程还是比较简单和清晰的。

可以配合时序图理解:
![View layout的时序图](http://img.blog.csdn.net/20160512235140635)

> 参考资料：《Android开发艺术探索》

