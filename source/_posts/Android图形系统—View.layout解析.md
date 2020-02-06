
---
tags: 
- 源码
- View
categories:
- [Android, 系统]
---


### 图文概括
1. 单一View通过layout方法确定位置，ViewGroup子类通过重写抽象onLayout方法来实现子视图以及自己的位置分配逻辑
2. getWidth|getHeight与getMeasuredWidth|getMeasureHeight区别


| 方法 | 概念 | 时机 | 场景 |
| --- | --- | --- | --- |
| getMeasuredWidth和Height | 测量的宽和高 | measure过程 | onMeasure使用 |
| getWidth和Height | 计算的宽和高 | layout过程 | onLayout使用 |



![](https://upload-images.jianshu.io/upload_images/9696036-0ce23ae06edca160.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/9696036-4b862fec211de964.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 源码分析
#### View.measure过程

```
// 计算View的四个顶点为孩子（left、top、right、bottom）
public void layout(int l, int t, int r, int b) {
    // 判断是否需要重新测量
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
      onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
      mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }
    // 保存上一次View的四个位置
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    // 设置当前视图View的l、t、r、b位置，及判断布局是否有改变
    boolean changed = isLayoutModeOptical(mParent) ?
          setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    // View大小 或者 位置发生变化 则重新布局View
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        // 1. 对于单一View，onLayout是一个空实现
        // 2. 对于ViewGroup，因为位置的确定与布局有关，所以onLayout是一个抽象方法，需要重写实现
        onLayout(changed, l, t, r, b);
        ...
    }
    
   ...    
}

// 通过传入的4个顶点位置设置View的四个顶点位置 ，返回位置是否变化
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    // 是否改变
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
      changed = true;
    
      // Remember our drawn bit
      int drawn = mPrivateFlags & PFLAG_DRAWN;
    
      int oldWidth = mRight - mLeft;
      int oldHeight = mBottom - mTop;
      int newWidth = right - left;
      int newHeight = bottom - top;
      boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);
    
      // 位置变化，执行invalidate
      invalidate(sizeChanged);
      // 设置新的顶点位置
      mLeft = left;
      mTop = top;
      mRight = right;
      mBottom = bottom;
      mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
      ...    

    }
    return changed;
}

// 单一View在layout已经确定了位置，所以onLayout空实现
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

}

```

#### ViewGroup.measure
```
// measure过程调用的View.measure，不同在于onLayout
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

// 抽象方法，子类需要实现以实现child的layout计算
protected abstract void onLayout(boolean changed,
       int l, int t, int r, int b);

// 大体过程
protected void onLayout(boolean changed,
       int l, int t, int r, int b){
    for (int i=0; i<getChildCount(); i++) {
       View child = getChildAt(i);  
       // 计算子View的测量宽高
       // 执行child的layout方法，child.layout 
    }       
}

```



-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)


