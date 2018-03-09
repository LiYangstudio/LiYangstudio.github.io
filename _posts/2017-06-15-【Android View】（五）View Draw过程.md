

[【Android View】（一）View绘制基础](http://www.hanszone.xyz/2016/04/25/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%B8%80%EF%BC%89View%E7%BB%98%E5%88%B6%E5%9F%BA%E7%A1%80/)
[【Android View】（二）View Measure过程](http://www.hanszone.xyz/2016/04/26/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%BA%8C%EF%BC%89View%20Measure%E8%BF%87%E7%A8%8B/)
[【Android View】（三）获取View的大小](http://www.hanszone.xyz/2016/04/26/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%B8%89%EF%BC%89%E8%8E%B7%E5%8F%96View%E7%9A%84%E5%A4%A7%E5%B0%8F/)
[【Android View】（四）View Layout过程](http://www.hanszone.xyz/2016/05/12/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E5%9B%9B%EF%BC%89View%20Layout%E8%BF%87%E7%A8%8B/)
[【Android View】（五）View Draw过程](http://www.hanszone.xyz/2016/05/13/%E3%80%90Android%20View%E3%80%91%EF%BC%88%E4%BA%94%EF%BC%89View%20Draw%E8%BF%87%E7%A8%8B/)
<!-- more -->

****
# View draw
实际上View从measure到layout再到draw，源代码的分析难度是一步一步下降的。
直接上源代码:
```
/**
 * Manually render this view (and all of its children) to the given Canvas.
 * The view must have already done a full layout before this function is
 * called.  When implementing a view, implement
 * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
 * If you do need to override this method, call the superclass version.
 *
 * @param canvas The Canvas to which the View is rendered.
 */
@CallSuper
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // we're done...
        return;
    }
```
从注释就可以看出它就四大步:
* 1.Draw the background 绘制背景
* 2.Draw view's content 绘制内容
* 3.Draw children 绘制子View 具体的是通过调用onDraw方法
* 4.Draw decorations (scrollbars for instance) 绘制装饰品 如滚动条

过程非常简单。

为了研究完，看一下是如何绘制子View的。
```
// Step 4, draw the children
dispatchDraw(canvas);
```
看到这一步，可以知道只有ViewGroup才会有具体实现，View肯定是空方法的。
```
/**
 * Called by draw to draw the child views. This may be overridden
 * by derived classes to gain control just before its children are drawn
 * (but after its own view has been drawn).
 * @param canvas the canvas on which to draw the view
 */
protected void dispatchDraw(Canvas canvas) {

}
```
不出所料，果然是空实现。直接看ViewGroup的dispatchDraw:
# ViewGroup dispatchDraw
```
/**
 * {@inheritDoc}
 */
@Override
protected void dispatchDraw(Canvas canvas) {
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    int flags = mGroupFlags;

    //...省略大量代码

    // Draw any disappearing views that have animations
    if (mDisappearingChildren != null) {
        final ArrayList<View> disappearingChildren = mDisappearingChildren;
        final int disappearingCount = disappearingChildren.size() - 1;
        // Go backwards -- we may delete as animations finish
        for (int i = disappearingCount; i >= 0; i--) {
            final View child = disappearingChildren.get(i);
            more |= drawChild(canvas, child, drawingTime);//绘制child
        }
    }
    if (usingRenderNodeProperties) canvas.insertInorderBarrier();

    if (debugDraw()) {
        onDebugDraw(canvas);
    }

    if (clipToPadding) {
        canvas.restoreToCount(clipSaveCount);
    }

    // mGroupFlags might have been updated by drawChild()
    flags = mGroupFlags;

    if ((flags & FLAG_INVALIDATE_REQUIRED) == FLAG_INVALIDATE_REQUIRED) {
        invalidate(true);
    }

    if ((flags & FLAG_ANIMATION_DONE) == 0 && (flags & FLAG_NOTIFY_ANIMATION_LISTENER) == 0 &&
            mLayoutAnimationController.isDone() && !more) {
        // We want to erase the drawing cache and notify the listener after the
        // next frame is drawn because one extra invalidate() is caused by
        // drawChild() after the animation is over
        mGroupFlags |= FLAG_NOTIFY_ANIMATION_LISTENER;
        final Runnable end = new Runnable() {
           public void run() {
               notifyAnimationListener();
           }
        };
        post(end);
    }
}
```
看到重点`drawChild(canvas, child, drawingTime)`
```
/**
 * Draw one child of this View Group. This method is responsible for getting
 * the canvas in the right state. This includes clipping, translating so
 * that the child's scrolled origin is at 0, 0, and applying any animation
 * transformations.
 *
 * @param canvas The canvas on which to draw the child
 * @param child Who to draw
 * @param drawingTime The time at which draw is occurring
 * @return True if an invalidate() was issued
 */
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```
还是和layout一样的套路~ 

# 小结
draw流程分析起来还是非常简单的~~ 时序图就不画了。 这代码看多一两次就明白了~~。

> 参考资料：《Android开发艺术探索》


