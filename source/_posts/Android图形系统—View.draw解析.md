
---
tags: 
- 源码
- View
categories:
- [Android, 系统]
---


### 图文概括
#### 流程
1. 绘制背景
2. 绘制内容
3. 分发子View绘制
4. 绘制装饰

![](https://upload-images.jianshu.io/upload_images/9696036-0bfd5198cdefa19c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 源码分析
#### View.draw
```
public void draw(Canvas canvas) {
    .... 
    // 1. 绘制本身View背景
    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        // 2. 绘制内容，默认空实现 需复写
        if (!dirtyOpaque) onDraw(canvas);
        
        // 3. 绘制 children 
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // 4. 分发Draw （单一View空实现，ViewGroup见下面分析）
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
    
        // 5. 绘制装饰 (前景色，滚动条)
        onDrawForeground(canvas);

        return;
    }
    ....
}

//绘制背景
private void drawBackground(Canvas canvas) {
    final Drawable background = mBackground;
    if (background == null) {
        return;
    }
    
    // 根据在 layout 过程中获取的 View 的位置参数，来设置背景的边界
    setBackgroundBounds();

    // 先尝试用HWUI绘制
    if (canvas.isHardwareAccelerated() && mAttachInfo != null
            && mAttachInfo.mThreadedRenderer != null) {
        mBackgroundRenderNode = getDrawableRenderNode(background, mBackgroundRenderNode);

        final RenderNode renderNode = mBackgroundRenderNode;
        if (renderNode != null && renderNode.isValid()) {
            setBackgroundRenderNodeProperties(renderNode);
            ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
            return;
        }
    }

    final int scrollX = mScrollX;
    final int scrollY = mScrollY;
    if ((scrollX | scrollY) == 0) {
        //调用 Drawable 的 draw 方法来进行背景的绘制
        background.draw(canvas);
    } else {
        // 若 mScrollX 和 mScrollY 有值，则对 canvas 的坐标进行偏移平移画布
        canvas.translate(scrollX, scrollY);
        //调用 Drawable 的 draw 方法来进行背景的绘制
        background.draw(canvas);
        canvas.translate(-scrollX, -scrollY);
    }
}

// 绘制View本身内容，空实现，子类必须复写
protected void onDraw(Canvas canvas) {
}
// 单一View无子View，空实现
protected void dispatchDraw(Canvas canvas) {
}

public void onDrawForeground(Canvas canvas) {
    //绘制指示器
    onDrawScrollIndicators(canvas);
    //绘制滚动条
    onDrawScrollBars(canvas);

    final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
    if (foreground != null) {
        if (mForegroundInfo.mBoundsChanged) {
            mForegroundInfo.mBoundsChanged = false;
            final Rect selfBounds = mForegroundInfo.mSelfBounds;
            final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

            if (mForegroundInfo.mInsidePadding) {
                selfBounds.set(0, 0, getWidth(), getHeight());
            } else {
                selfBounds.set(getPaddingLeft(), getPaddingTop(),
                        getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
            }

            final int ld = getLayoutDirection();
            Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                    foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
            foreground.setBounds(overlayBounds);
        }
        //调用 Drawable 的 draw 方法，绘制前景色
        foreground.draw(canvas);
    }
}

```
#### ViewGroup.draw
```
public void draw(Canvas canvas) {
    // 调用View.draw 方法
}


@Override
protected void dispatchDraw(Canvas canvas) {
    ...
    // 1. 遍历子View
    final int childrenCount = mChildrenCount;
    ...
    for (int i = 0; i < childrenCount; i++) {
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                transientIndex = -1;
            }
        }

        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
            // 调用 drawChild 方法，进行子元素绘制
            more |= drawChild(canvas, child, drawingTime);
        }
    }
    ....
}
-

```

-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)



