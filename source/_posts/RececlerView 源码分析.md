---
title: RececlerView 源码分析
date: 2020-04-27
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428141843.png
tags: 
- RececlerView
categories:
- [Android, 框架]
---


> 本文基于 V7-25.3.1


## 图文总结

### RecyclerView优点
* 更好的灵活配置能力，在LayoutManager（布局）、Adapter（数据适配）和动画的兼容上都更优雅
* 缓存能力增强，离屏缓存相对较优，另增加了一层缓存池缓存
* 支持局部刷新，对于一些交互处理多的情况下，会带来更好的性能

### RecyclerView类图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428141843.png)


### RecyclerView绘制流程图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428141858.png)


### RecyclerView滑动流程图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428141904.png)


### RecyclerView缓存介绍图
![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428141917.png)



## 源码阅读
### RecyclerView使用方法分分析

#### RecyclerView 构造方法

* 进行View相关配置属性设置，触摸范围、滑动速度等
* 设置Item动画监听器
* 初始化AdapterManager，创建AdapterHelper（负责Adapter里的数据集发生变化时的预处理操作）
* 初始化ChildHelper（负责管理和访问 RecyclerView 的子视图）
* 如果配置了LayoutManager 则通过反射方法创建它

```
 public RecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        ...
        // View配置相关属性设置
        final ViewConfiguration vc = ViewConfiguration.get(context);
        mTouchSlop = vc.getScaledTouchSlop();
        mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
        mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
        setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);
        
        // 设置Item动画监听器
        mItemAnimator.setListener(mItemAnimatorListener);
        // 设置 AdapterManager
        initAdapterManager();
        // 设置 ChildrenHelper 
        initChildrenHelper();
        // 硬件加速相关属性设置
        if (ViewCompat.getImportantForAccessibility(this)
                == ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
            ViewCompat.setImportantForAccessibility(this,
                    ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_YES);
        }
        mAccessibilityManager = (AccessibilityManager) getContext()
                .getSystemService(Context.ACCESSIBILITY_SERVICE);
        setAccessibilityDelegateCompat(new RecyclerViewAccessibilityDelegate(this));
        
        // 如果attrs指定了LayoutManager，则创建LayoutManager
        boolean nestedScrollingEnabled = true;
        
        if (attrs != null) {
            int defStyleRes = 0;
            TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.RecyclerView,
                    defStyle, defStyleRes);
            String layoutManagerName = a.getString(R.styleable.RecyclerView_layoutManager);
            int descendantFocusability = a.getInt(
                    R.styleable.RecyclerView_android_descendantFocusability, -1);
            if (descendantFocusability == -1) {
                setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            }
            a.recycle();
            // 反射方法创建 LayoutManager
            createLayoutManager(context, layoutManagerName, attrs, defStyle, defStyleRes);

            if (Build.VERSION.SDK_INT >= 21) {
                // SDK >=21下 ，nestedScrollingEnabled状态支持变更
                a = context.obtainStyledAttributes(attrs, NESTED_SCROLLING_ATTRS,
                        defStyle, defStyleRes);
                nestedScrollingEnabled = a.getBoolean(0, true);
                a.recycle();
            }
        } else {
            setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        }

        // 重置nestedScrollingEnabled状态 SDK 21以下默认true
        setNestedScrollingEnabled(nestedScrollingEnabled);
    }

```

#### setLayoutManager
* 处理重新设置一个新的LayoutManager的一些逻辑
* 设置this给到LayoutManager，并如果attach了则执行LayoutManger的attach分发实践
* 跟新缓存大小，并请求重新布局

```
public void setLayoutManager(LayoutManager layout) {
        if (layout == mLayout) {
            return;
        }
        stopScroll();
        // 设置新的layout情况下的一些处理逻辑
        ... 
        
        mChildHelper.removeAllViewsUnfiltered();
        mLayout = layout;
        if (layout != null) {
            // layout只能绑定一个mRecyclerView 
            if (layout.mRecyclerView != null) {
                throw new IllegalArgumentException("LayoutManager " + layout +
                        " is already attached to a RecyclerView: " + layout.mRecyclerView);
            }
            // 设置this引用给LayoutManager
            mLayout.setRecyclerView(this);
            if (mIsAttached) {
                // 分发attach事件
                mLayout.dispatchAttachedToWindow(this);
            }
        }
        // 重新更新缓存大小 及请求重新布局
        mRecycler.updateViewCacheSize();
        requestLayout();
    }

```

#### setAdapter
* 接触frozen状态
* 设置新的Adapter，并触发一系列监听事件

```
public void setAdapter(Adapter adapter) {
   // 解除frozen状态
   setLayoutFrozen(false);
   // 替换到当前Adapter，并触发监听
   setAdapterInternal(adapter, false, true);
   requestLayout();
}

private void setAdapterInternal(Adapter adapter, boolean compatibleWithPrevious,
            boolean removeAndRecycleViews) {
        // 旧Adapter进行解绑数据监听 和 RecyclerView的引用
        if (mAdapter != null) {
            mAdapter.unregisterAdapterDataObserver(mObserver);
            mAdapter.onDetachedFromRecyclerView(this);
        }
        if (!compatibleWithPrevious || removeAndRecycleViews) {
            removeAndRecycleViews(); // 移除缓存的View
        }
        mAdapterHelper.reset();
        final Adapter oldAdapter = mAdapter;
        mAdapter = adapter;
        if (adapter != null) {
            // 处理新设置的Adapter的关联监听和RecyclerView
            adapter.registerAdapterDataObserver(mObserver);
            adapter.onAttachedToRecyclerView(this);
        }
        // 通知LayoutManager Adapter变更
        if (mLayout != null) {
            mLayout.onAdapterChanged(oldAdapter, mAdapter);
        }
        // 触发Recycler Adapter变更事件
        mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
        // 状态置为 mStructureChanged 
        mState.mStructureChanged = true;
        markKnownViewsInvalid();
    }


```

### RecyclerView绘制方法分析
#### onMeasure
* 未赋值layoutManager情况下，走默认measure，结果是无展示
* 系统提供的LayoutManager默认AutoMeasure。执行LayoutManger的onMeasure方法
* 如果未指定确定宽高的尺寸规格，则会进行布局，继而获得子View的大小。此过程可能执行两次

```

protected void onMeasure(int widthSpec, int heightSpec) {
   if (mLayout == null) {// 无LayoutManger
       defaultOnMeasure(widthSpec, heightSpec); 
       return;
   }
   // Android提供的三个LayoutManger，都是AutoMeasure
   if (mLayout.mAutoMeasure) {
       // 获取测量规格
       final int widthMode = MeasureSpec.getMode(widthSpec);
       final int heightMode = MeasureSpec.getMode(heightSpec);
       final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
               && heightMode == MeasureSpec.EXACTLY;
       // 执行LayoutManager的onMeasure方法
       mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
       
       if (skipMeasure || mAdapter == null) {
           return;
       }
       // 如果测量规格不确定 且设置了Adapter，则先执行一次layout
       if (mState.mLayoutStep == State.STEP_START) {
           dispatchLayoutStep1();
       }
       // 设置测量规格
       mLayout.setMeasureSpecs(widthSpec, heightSpec);
       mState.mIsMeasuring = true;
       dispatchLayoutStep2();

       // 设置 获取到子View的宽高
       mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

       //二次测量，宽高不确定情况下
       if (mLayout.shouldMeasureTwice()) {
           mLayout.setMeasureSpecs(
                   MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                   MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
           mState.mIsMeasuring = true;
           dispatchLayoutStep2();
           // now we can get the width and height from the children.
           mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
       }
   } else {
       ...
   }
}


```
#### RecyclerView.onLayout
* 执行DispatchLayout方法
* 根据不同State状态，分别执行Step1、Step2、Step3方法


```
protected void onLayout(boolean changed, int l, int t, int r, int b) {
   TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
   dispatchLayout();
   TraceCompat.endSection();
   mFirstLayoutComplete = true;
}
    void dispatchLayout() {
        mState.mIsMeasuring = false;
        // 如果State状态是 State.STEP_START 则执行 Step1 和Step2
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth() 
            // 直接执行 step2
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            mLayout.setExactMeasureSpecsFrom(this);
        }
        // 执行Step3 ，主要保存一些View信息和动画执行
        dispatchLayoutStep3();
    }

```


#### dispatchLayoutStep1
* 第一步layout方法
* 处理adapter变更
* 确定需要执行的动画
* 针对当前的Views进行信息缓存
* 如有必要，则进行预布局并缓存信息

```
private void dispatchLayoutStep1() {
   //  State状态断言 
   mState.assertLayoutStep(State.STEP_START);
   mState.mIsMeasuring = false;
   // 是否过滤掉 RequestLayout执行，需要过滤的时候 执行该方法，会使过滤次数+1
   eatRequestLayout();
   // 清楚 ViewInfo 所有状态和其存在的数据
   mViewInfoStore.clear();
   // 执行进入 layout或者scroll行为标志
   onEnterLayoutOrScroll();
   // 执行Adapter变更及计算那些需要执行的动画
   processAdapterUpdatesAndSetAnimationFlags();
   // 存储焦点信息
   saveFocusInfo();
   // state状态信息设置
   mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
   mItemsAddedOrRemoved = mItemsChanged = false;
   mState.mInPreLayout = mState.mRunPredictiveAnimations;
   mState.mItemCount = mAdapter.getItemCount();
   // 寻找 layout过程中position的最大和最小值
   findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
   
   if (mState.mRunSimpleAnimations) {
       ...  
       int count = mChildHelper.getChildCount();
       for (int i = 0; i < count; ++i) {
            // 遍历VieHolder
           final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
           if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
               continue;
           }
           // 创建 ItemHolderInfo
           final ItemHolderInfo animationInfo = mItemAnimator
                   .recordPreLayoutInformation(mState, holder,
                           ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                           holder.getUnmodifiedPayloads());
           // mViewInfoStore存储 holder及其对应animation信息
           mViewInfoStore.addToPreLayout(holder, animationInfo);
           if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
               // 如果holder确定要更新，就把它添加到 oldChangeHolders 集合中
               mViewInfoStore.addToOldChangeHolders(key, holder);
           }
       }  
   }
   if (mState.mRunPredictiveAnimations) {
      ... // 运行预布局
   }
   // 执行退出 layout或者scroll行为标志
   onExitLayoutOrScroll();
   // 对应 mEatRequestLayout -1
   resumeRequestLayout(false);
   // 状态进入 State.STEP_LAYOUT
   mState.mLayoutStep = State.STEP_LAYOUT;
}


private void processAdapterUpdatesAndSetAnimationFlags() {
   if (predictiveItemAnimationsEnabled()) {
       mAdapterHelper.preProcess(); // 预处理
   } else {
       mAdapterHelper.consumeUpdatesInOnePass();
   }
   boolean animationTypeSupported = mItemsAddedOrRemoved || mItemsChanged;
   // 计算 mRunSimpleAnimations 和 mRunPredictiveAnimations
   // mDataSetHasChangedAfterLayout 数据是否变化
   mState.mRunSimpleAnimations = mFirstLayoutComplete
           && mItemAnimator != null
           && (mDataSetHasChangedAfterLayout
           || animationTypeSupported
           || mLayout.mRequestedSimpleAnimations)
           && (!mDataSetHasChangedAfterLayout
           || mAdapter.hasStableIds());
   mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
           && animationTypeSupported
           && !mDataSetHasChangedAfterLayout
           && predictiveItemAnimationsEnabled();
}

```

#### dispatchLayoutStep2
* 执行最终的View布局操作，该过程由LayoutManager完成
* 该方法可能会被多次执行

```
private void dispatchLayoutStep2() {
   // 过滤掉 RequestLayout执行，需要过滤的时候 执行该方法，会使过滤次数+1。对应resumeRequestLayout方法进行消费
   eatRequestLayout();
   // 对应 onExitLayoutOrScroll
   onEnterLayoutOrScroll();
   mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
   // 跳过预处理过程，一次性执行完所有的update
   mAdapterHelper.consumeUpdatesInOnePass();
   mState.mItemCount = mAdapter.getItemCount(); // 赋值 itemCOunt
   mState.mDeletedInvisibleItemCountSincePreviousLayout = 0; 

   
   mState.mInPreLayout = false;
   // 执行 layout （执行 LayoutManager 布局）
   mLayout.onLayoutChildren(mRecycler, mState);

   mState.mStructureChanged = false;
   mPendingSavedState = null;
   mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
   // State状态进入 State.STEP_ANIMATIONS
   mState.mLayoutStep = State.STEP_ANIMATIONS;
   // 对应 onExitLayoutOrScroll
   onExitLayoutOrScroll();
   // 对应eatRequestLayout方法
   resumeRequestLayout(false);
}
```

#### dispatchLayoutStep3
layout过程最后一步，执行相关动画和一些清理事项

```
private void dispatchLayoutStep3() {
   ...
   if (mState.mRunSimpleAnimations) {
       // 执行相关动画
       ... 
   }
   // 一些清理动作   
}

```
#### draw
主要涉及Item装饰的绘制和动画

```
public void draw(Canvas c) {
   super.draw(c);

   final int count = mItemDecorations.size();
   for (int i = 0; i < count; i++) {
       mItemDecorations.get(i).onDrawOver(c, this, mState);
   }
  
   boolean needsInvalidate = false;
   ...
   if (!needsInvalidate && mItemAnimator != null && mItemDecorations.size() > 0 &&
           mItemAnimator.isRunning()) {
       needsInvalidate = true;
   }

   if (needsInvalidate) {
       ViewCompat.postInvalidateOnAnimation(this);
   }
}
    
public void onDraw(Canvas c) {
   super.onDraw(c);

   final int count = mItemDecorations.size();
   for (int i = 0; i < count; i++) {
       mItemDecorations.get(i).onDraw(c, this, mState);
   }
}

```

### LinearLayoutManager 填充子View过程

#### LinearLayoutManager.onLayoutChildren
* Child布局执行核心方法
* 布局方式，通过确定锚点，首先以锚点为基准上到下布局，在以锚点为基准从下往上布局。如果还有空间，继续从上到下布局。最后确认整个间隙是正确的。（反向布局及横向反之则可）
* 该方法为LayoutManager布局核心执行方法，Child的测量和添加工作在fill这个重要方法执行，接下来会阐述 

```
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    
       // 确定是否需要反向布局
       // 确定锚点及偏移量 (1. 优先焦点child 2. 如果是反向布局，则找recycler里面最最接近尾部的child 3. 如果是正向，则找最接近头部的child)
       // 计算额外的偏移量（RecyclerView padding） 
       ...
       // 锚点准备ready        
       onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
       // 临时 detach和回收当前的view 第一次 measure 的时候不会产生效果，因为此时 RecyclerView 还没有子 View。 而在第二第三次 layout 时，它会把子 View 从 RecyclerView 中 remove 或 detach ，并缓存子 View，以便之后重新 add 回来或 attach 回来，避免重复加载相同的子 View
       detachAndScrapAttachedViews(recycler); 
       mLayoutState.mInfinite = resolveIsInfinite(); 
       mLayoutState.mIsPreLayout = state.isPreLayout();
       
       // 开始填充view
       if (mAnchorInfo.mLayoutFromEnd) {
          ... // 反向填充
       } else { // 正向填充
           // （基于锚点位置先 由上到下||由左到右）更新锚点信息
           updateLayoutStateToFillEnd(mAnchorInfo);
           mLayoutState.mExtra = extraForEnd; // 额外的尾部偏移量
           // 开始填充 View布局主要方法
           fill(recycler, mLayoutState, state, false);
           // 尾部位移
           endOffset = mLayoutState.mOffset;
           final int lastElement = mLayoutState.mCurrentPosition;
           if (mLayoutState.mAvailable > 0) {
               extraForStart += mLayoutState.mAvailable;
           }
           // （基于锚点位置 由下到上||由右到左）更新锚点信息
           updateLayoutStateToFillStart(mAnchorInfo);
           mLayoutState.mExtra = extraForStart;
           mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
           // 二次填充
           fill(recycler, mLayoutState, state, false);
           startOffset = mLayoutState.mOffset;
           // 仍有可用空间
           if (mLayoutState.mAvailable > 0) {
               extraForEnd = mLayoutState.mAvailable;
               // 继续 （基于锚点位置先 由上到下||由左到右）更新信息并填充View
               updateLayoutStateToFillEnd(lastElement, endOffset);
               mLayoutState.mExtra = extraForEnd;
               fill(recycler, mLayoutState, state, false);
               endOffset = mLayoutState.mOffset;
           }
       }
    
       // 有滑动位置导致的gap间隙修复处理
       ...
       // 预布局动画处理
       ...
    }
        

```

#### LinearLayoutManager.fill
* 如果是滑动流程，则根据情况进行回收流程
* LayoutState中部分成员变量含义，mOffset：填充起始坐标，mCurrentPosition：填充起始数据的position，mAvailable：本次滑动可填充的距离，mScrollingOffset：滑动过的总量循环依次加载子View
* 确定可布局大小，直至布局大小消费完成
* 加载子View在 layoutChunk 中执行


```
 int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
            
   // 可布局的位移
   final int start = layoutState.mAvailable;
   // 滑动偏移的情况下
   if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
       if (layoutState.mAvailable < 0) {
           layoutState.mScrollingOffset += layoutState.mAvailable;
       }
       // 执行回收流程
       recycleByLayoutState(recycler, layoutState);
   }
   // 余量大小
   int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
   // 每次布局结果中间记录 方便运算  
   LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
   while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
       layoutChunkResult.resetInternal();
       // 加载子View
       layoutChunk(recycler, state, layoutState, layoutChunkResult);

       if (layoutChunkResult.mFinished) {
           break;
       }
       
       layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
       // 计算布局使用过的大小值
       if (!layoutChunkResult.mIgnoreConsumed || mLayoutState.mScrapList != null
               || !state.isPreLayout()) {
           layoutState.mAvailable -= layoutChunkResult.mConsumed;
           remainingSpace -= layoutChunkResult.mConsumed;
       }
        // 如果当前正在滚动屏幕
       if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
           layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
           if (layoutState.mAvailable < 0) {
               layoutState.mScrollingOffset += layoutState.mAvailable;
           }
           // 把移出屏幕的 View 缓存到 mCachedViews 里面
           recycleByLayoutState(recycler, layoutState);
       }
       if (stopOnFocusable && layoutChunkResult.mFocusable) {
           break;
       }
   }
   
   return start - layoutState.mAvailable;
}

```

#### LinearLayoutManager.layoutChunk
* 通过layoutState.next(recycler)获取目标布局View
* 获取目标View完毕后，进行含装饰的Margin计算，并执行布局

```
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        // 获取下一个布局View （核心方法）
        View view = layoutState.next(recycler);
        
        LayoutParams params = (LayoutParams) view.getLayoutParams();
        if (layoutState.mScrapList == null) { // 除非特殊指定，否则mScrapList为null
            // 执行 addView 
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                    // 添加到末尾
                addView(view); 
            } else {
                addView(view, 0); // 添加到第一个位置
            }
        } else {
            ...      
        }
        measureChildWithMargins(view, 0, 0); // 测量子View的Margins
        // 计算 含装饰的Margin值的大小
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
        int left, top, right, bottom;
        // 计算 r、l、t、b的值
        if (mOrientation == VERTICAL) {
            if (isLayoutRTL()) {
                right = getWidth() - getPaddingRight();
                left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
            } else {
                left = getPaddingLeft();
                right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
            }
            if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
                bottom = layoutState.mOffset;
                top = layoutState.mOffset - result.mConsumed;
            } else {
                top = layoutState.mOffset;
                bottom = layoutState.mOffset + result.mConsumed;
            }
        } else {
            ...
        }
        // 对View进行布局
        layoutDecoratedWithMargins(view, left, top, right, bottom);
        // 部分状态改变 
        if (params.isItemRemoved() || params.isItemChanged()) {
            result.mIgnoreConsumed = true;
        }
        result.mFocusable = view.hasFocusable();
}

```

#### LinearLayoutManager.next
通过RecyclerView.Recycler获取对应Pos的View

```
View next(RecyclerView.Recycler recycler) {
  if (mScrapList != null) { // 除非定制View，不然为null
      return nextViewFromScrapList();
  }
  通过RecyclerView.Recycler 获取目标position对应的View 
  final View view = recycler.getViewForPosition(mCurrentPosition);
  mCurrentPosition += mItemDirection; // 当前pos 增加
  return view;
}
```


### Recycler获取VH的缓存和创建过程

#### Recycler.getViewForPosition
根据Pos获取View方法，最终执行tryGetViewHolderForPositionByDeadline获取View

```
public View getViewForPosition(int position) {
  return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
  return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}

```

#### Recycler.tryGetViewHolderForPositionByDeadline
* 获取ViewHolder方法
* 如果是预布局，线通过ChangeScrap中获取
* 第一次尝试获取VH，依次从Scrap、Hidden、Cache中获取VH
* 第二次尝试获取VH，针对具有StableId的Adapter，根据id依次从Scrap和Cache获取
* 第三次尝试从自定义缓存中获取VH
* 第四次尝试从Recycler获取VH
* 最后直接创建VH

```
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
  
  boolean fromScrapOrHiddenOrCache = false;
  ViewHolder holder = null;
  // 0) 如果是预布局， 从mChangedScrap中获取 
  if (mState.isPreLayout()) {
      holder = getChangedScrapViewForPosition(position);
      fromScrapOrHiddenOrCache = holder != null;
  }
  // 1) 第一次尝试获取，依次从Scrap、Hidden、Cache中获取VH
  if (holder == null) {
        // 依次从Scrap、Hidden、Cache中获取VH
      holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
      ... 
  }
  if (holder == null) {
      final int offsetPosition = mAdapterHelper.findPositionOffset(position);
      final int type = mAdapter.getItemViewType(offsetPosition);
      // 2) 第二次尝试获取，当Adapter具备StableIds情况
      if (mAdapter.hasStableIds()) {
          holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                  type, dryRun);
          if (holder != null) {
              // update position
              holder.mPosition = offsetPosition;
              fromScrapOrHiddenOrCache = true;
          }
      }
      // 3) 第三次尝试从 自定义缓存获取
      if (holder == null && mViewCacheExtension != null) {
                   final View view = mViewCacheExtension
                  .getViewForPositionAndType(this, position, type);
          if (view != null) {
              holder = getChildViewHolder(view);
              ...
          }
      }
      // 4) 第四次尝试 从 RecyclerPool中获取
      if (holder == null) { // fallback to pool
          holder = getRecycledViewPool().getRecycledView(type);
          ...
      }
      // 5) 开始创建
      if (holder == null) {
          long start = getNanoTime();
          // 创建VH
          holder = mAdapter.createViewHolder(RecyclerView.this, type);
          ... 
      }
      
       boolean bound = false;
       if (mState.isPreLayout() && holder.isBound()) {
           holder.mPreLayoutPosition = position;
       } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
            // 为bind过，执行bind方法
           final int offsetPosition = mAdapterHelper.findPositionOffset(position);
           bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
       }
}

        private boolean tryBindViewHolderByDeadline(ViewHolder holder, int offsetPosition,
                int position, long deadlineNs) {
            ...
            // 执行Adapter bindViewHolder方法
            mAdapter.bindViewHolder(holder, offsetPosition);
            ...
            return true;
        }
```

#### 00 从ChangeScrap获取
针对的是预布局状态，从mChangedScrap中获取目标ViewHolder
ScrapView：View仍然attach在其父RecyclerView上且可以被重复绑定数据及重复使用。将View标记为Scrap过程中分为两大类mAttachedScrap 和 mChangedScrap。
mAttachedScrap：VH有ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID这两个Flag，或者VH是没有被更新过的，或者是可以被重新更新的VH。
其它则是mChangedScrap

```
ViewHolder getChangedScrapViewForPosition(int position) {
  // 必须是预布局状态，取mChangedScrap中的ViewHolder
  final int changedScrapSize;
  if (mChangedScrap == null || (changedScrapSize = mChangedScrap.size()) == 0) {
      return null;
  }
  // 通过position获取
  for (int i = 0; i < changedScrapSize; i++) {
      final ViewHolder holder = mChangedScrap.get(i);
      if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position) {
          holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
          return holder;
      }
  }
  // 如果Adapter是固定id，尝试从Adapter获取
  if (mAdapter.hasStableIds()) {
      final int offsetPosition = mAdapterHelper.findPositionOffset(position);
      if (offsetPosition > 0 && offsetPosition < mAdapter.getItemCount()) {
          final long id = mAdapter.getItemId(offsetPosition);
          for (int i = 0; i < changedScrapSize; i++) {
              final ViewHolder holder = mChangedScrap.get(i);
              if (!holder.wasReturnedFromScrap() && holder.getItemId() == id) {
                  holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                  return holder;
              }
          }
      }
  }
  return null;
}

// Mark an attached view as scrap.
void scrapView(View view) {
       final ViewHolder holder = getChildViewHolderInt(view);
       if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
               || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
           holder.setScrapContainer(this, false);
           mAttachedScrap.add(holder);
       } else {
           if (mChangedScrap == null) {
               mChangedScrap = new ArrayList<ViewHolder>();
           }
           holder.setScrapContainer(this, true);
           mChangedScrap.add(holder);
       }
}


```


#### 第一次尝试获取VH（AttachScrap、Hidden、CacheView）
* 先从 mAttachedScrap中获取VH
* 从隐藏且未移出的View中获取 View
* 从一级缓存CacheView中获取


```
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
       final int scrapCount = mAttachedScrap.size();

       // 先从 mAttachedScrap中获取VH
       for (int i = 0; i < scrapCount; i++) {
           final ViewHolder holder = mAttachedScrap.get(i);
           // 验证VH是否可用，若可用，则直接返回该VH
           if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                   && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
               holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
               return holder;
           }
       }
       // dryRun 传递是false（代表VH在scrap、cache中可以被Removed）
       if (!dryRun) {
           // 从隐藏且未移出的View中获取 View
           View view = mChildHelper.findHiddenNonRemovedView(position);
           if (view != null) {
               // View可用，则进行可视、detach、scrap缓存
               final ViewHolder vh = getChildViewHolderInt(view);
               mChildHelper.unhide(view);
               int layoutIndex = mChildHelper.indexOfChild(view);
               mChildHelper.detachViewFromParent(layoutIndex);
               scrapView(view);
               vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                       | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
               return vh;
           }
       }

       // 从第一级缓存View中获取
       final int cacheSize = mCachedViews.size();
       for (int i = 0; i < cacheSize; i++) {
           final ViewHolder holder = mCachedViews.get(i);
           // VH是有效的
           if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
               if (!dryRun) {
                   mCachedViews.remove(i); // 移出获取的cache
               }
               return holder; // 返回VH
           }
       }
       return null;
}

```
#### 第二次尝试获取VH（Adapter有稳定id情况）
* Adapter配置的id是稳定的，稳定指数据集变化的时候，对于同一数据对应的id是唯一的
* 先尝试从Scrap获取VH，非dryRun下，将未命中的从Scrap中移出，并加入到Cache或Pool缓存
* 在尝试从Cache获取VH，将未命中的从Cache中移出，并加入到Pool缓存

```
ViewHolder getScrapOrCachedViewForId(long id, int type, boolean dryRun) {
       // 从AttachedScrap中 尝试获取VH
       final int count = mAttachedScrap.size();
       for (int i = count - 1; i >= 0; i--) {
           final ViewHolder holder = mAttachedScrap.get(i);
           // id 相等 且 holder 非Scrap返回
           if (holder.getItemId() == id && !holder.wasReturnedFromScrap()) {
               if (type == holder.getItemViewType()) {
                   holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                   if (holder.isRemoved()) {
                        // 从事
                       if (!mState.isPreLayout()) {
                           holder.setFlags(ViewHolder.FLAG_UPDATE, ViewHolder.FLAG_UPDATE
                                   | ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED);
                       }
                   }
                   return holder;
               } else if (!dryRun) {
                   // 从AttachedScrap移除
                   mAttachedScrap.remove(i);
                   removeDetachedView(holder.itemView, false);
                   // 回收加入至 cache 或者 pool
                   quickRecycleScrapView(holder.itemView);
               }
           }
       }

       // 从CacheView中尝试获取
       final int cacheSize = mCachedViews.size();
       for (int i = cacheSize - 1; i >= 0; i--) {
           final ViewHolder holder = mCachedViews.get(i);
           if (holder.getItemId() == id) {
               if (type == holder.getItemViewType()) {
                   if (!dryRun) {
                       mCachedViews.remove(i); // 从Cache中移出
                   }
                   return holder;
               } else if (!dryRun) {
                   // 从Cache中移出，放到pool中
                   recycleCachedViewAt(i);
                   return null;
               }
           }
       }
       return null;
}


```

### RecyclerView滑动机制分析
根据View事件机制可以直接来看onTouchEvent方法。
重点查看move事件。move事件执行了scrollByInternal方法。该方法最后会执行LayoutManager的Scroll方法，以LinearLayoutManager为例，它的ScrollBy方法最终执行到fill方法。也就是上文提到的ItemView填充方法，滑动过程中会不断执行获取对应位置的ViewHolder，然后进行View的展示。从而实现RecyclerView的滑动

```
public boolean onTouchEvent(MotionEvent e) {
   ...

   switch (action) {
       case MotionEvent.ACTION_DOWN: {
          ...
       case MotionEventCompat.ACTION_POINTER_DOWN: 
          ...

       case MotionEvent.ACTION_MOVE: { // 触摸时间-move
           ...
           if (mScrollState == SCROLL_STATE_DRAGGING) {
               mLastTouchX = x - mScrollOffset[0];
               mLastTouchY = y - mScrollOffset[1];
                // 执行内部滑动方法
               if (scrollByInternal(
                       canScrollHorizontally ? dx : 0,
                       canScrollVertically ? dy : 0,
                       vtev)) {
                   getParent().requestDisallowInterceptTouchEvent(true);
               }
               ...           }
       } break;

       case MotionEventCompat.ACTION_POINTER_UP: {
           onPointerUp(e);
       } break;

       case MotionEvent.ACTION_UP: { // 触摸事件-up
           // 执行 fling方法 ，主要做一些item和scroller动画等操作
           if (!((xvel != 0 || yvel != 0) && fling((int) xvel, (int) yvel))) {
               setScrollState(SCROLL_STATE_IDLE);
           }
           resetTouch();
       } break;

       case MotionEvent.ACTION_CANCEL: {
           cancelTouch();
       } break;
   }
   ...
   return true;
}

```
#### scrollByInternal
* 内部Scroll执行方法，此处会执行LayoutManager的Scroll方法
* 其它处罚Nested、OnScroll等事件

```

boolean scrollByInternal(int x, int y, MotionEvent ev) {
   int unconsumedX = 0, unconsumedY = 0;
   int consumedX = 0, consumedY = 0;

   consumePendingUpdateOperations();
   if (mAdapter != null) {
       eatRequestLayout();
       onEnterLayoutOrScroll();
       TraceCompat.beginSection(TRACE_SCROLL_TAG);
       if (x != 0) {
           consumedX = mLayout.scrollHorizontallyBy(x, mRecycler, mState);
           unconsumedX = x - consumedX;
       }
       if (y != 0) {
            // LinearLayout 竖向布局为例，走LayoutManager滑动放啊放
           consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);
           unconsumedY = y - consumedY;
       }
       TraceCompat.endSection();
       repositionShadowingViews();
       onExitLayoutOrScroll();
       resumeRequestLayout(false);
   }
   if (!mItemDecorations.isEmpty()) {
       invalidate();
   }
    // 分发 NestedScroll事件
   if (dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset)) {
    ...
   }
   if (consumedX != 0 || consumedY != 0) {
       dispatchOnScrolled(consumedX, consumedY); // 分发onScrolled事件
   }
   if (!awakenScrollBars()) {
       invalidate();
   }
   return consumedX != 0 || consumedY != 0;
}

```

#### LinearLayoutManager执行滑动处理
* 执行scrollBy方法
* scrollBy方法最终走到 fill方法（上面提到的填充子View方法）
* 该方法则会进行 ItemView的填充。从而完成Recycler滑动时，View的重新创建或者重新绑定一系列过程
* 平移整个View的child，实现滑动效果

```
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler,
       RecyclerView.State state) {
   if (mOrientation == HORIZONTAL) {
       return 0;
   }
   return scrollBy(dy, recycler, state);
}


int scrollBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
   ...
   mLayoutState.mRecycle = true;
   ensureLayoutState();
   final int layoutDirection = dy > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
   final int absDy = Math.abs(dy);
   // 更新LayoutState，布局方向和偏移值。目的是让LayoutManager知道从开始还是末尾进行回收和填充
   updateLayoutState(layoutDirection, absDy, true, state);
   
   // 执行 LinearLayout的fill 方法
   final int consumed = mLayoutState.mScrollingOffset
           + fill(recycler, mLayoutState, state, false);
   ...
   // 平移整个view的child
   mOrientationHelper.offsetChildren(-scrolled);
   
   return scrolled;
}
```


```
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
    ...
    // 执行回收流程
    recycleByLayoutState(recycler, layoutState);
    ...
    // 执行填充流程（参考上面layoutChunk方法）
    layoutChunk(recycler, state, layoutState,layoutChunkResult);
}
```

#### LinearLayoutManager回收流程
* 根据不同的布局方向进行不同方向的回收。以Start为例介绍
* 计算位移limit值，根据limit

```
private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
   // 假设是 初始方向布局，则开始末尾View回收。反之亦然
   if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
       recycleViewsFromEnd(recycler, layoutState.mScrollingOffset);
   } else {
       recycleViewsFromStart(recycler, layoutState.mScrollingOffset);
   }
}

private void recycleViewsFromStart(RecyclerView.Recycler recycler, int dt) {
   final int limit = dt;
   final int childCount = getChildCount();
   if (mShouldReverseLayout) {
       ...
   } else {
        for (int i = 0; i < childCount; i++) {
           View child = getChildAt(i);
           // 遍历child，当超过限制大小时候，开始回收
           if (mOrientationHelper.getDecoratedEnd(child) > limit
                   || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
               recycleChildren(recycler, 0, i); // 执行Children回收流程
               return;
           }
       }        }
}
    
private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {
   if (endIndex > startIndex) {
       for (int i = endIndex - 1; i >= startIndex; i--) {
           removeAndRecycleViewAt(i, recycler); // 执行RecyclerView的移出和回收方法
       }
   } else {
       for (int i = startIndex; i > endIndex; i--) {
           removeAndRecycleViewAt(i, recycler);
       }
   }
}

```

#### RecyclerView.removeAndRecycleViewAt
* 移出和回收View方法
* 执行ChildHelper的移出View方法。内部Bucket移出和回掉CallBack进行View移出
* 执行Recycler回收方法

```
public void removeAndRecycleViewAt(int index, Recycler recycler) {
  final View view = getChildAt(index); // 获取目标View
  removeViewAt(index); // 执行ChildHelper移出
  recycler.recycleView(view); // 回收View
}

public void removeViewAt(int index) {
  final View child = getChildAt(index); 
  if (child != null) {
      mChildHelper.removeViewAt(index); // 执行ChildHelper移出
  }
}

public void recycleView(View view) {
  ViewHolder holder = getChildViewHolderInt(view); // 获取VH
  // ViewHolder 回收前，需要完全detach、且不是Scrap
  if (holder.isTmpDetached()) {
      removeDetachedView(view, false); 
  }
  if (holder.isScrap()) {
      holder.unScrap();
  } else if (holder.wasReturnedFromScrap()){
      holder.clearReturnedFromScrapFlag();
  }
  recycleViewHolderInternal(holder); // 执行回收
}



```


#### RecyclerView.recycleViewHolderInternal
* 内部缓存VH方法
* 如果CacheView满了，则移出一个Cache到Pool中
* 将目标VH缓存到Cache末尾
* 如果没有Cache成功，则直接缓存到Pool中

```
 void recycleViewHolderInternal(ViewHolder holder) {  
      if (mViewCacheMax > 0
              && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
              | ViewHolder.FLAG_REMOVED
              | ViewHolder.FLAG_UPDATE
              | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
          // Cache缓存个数超了，则直接回收CacheView到RecyclerPool
          int cachedViewSize = mCachedViews.size();
          if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
              recycleCachedViewAt(0);
              cachedViewSize--;
          }

          int targetCacheIndex = cachedViewSize;
          // 将VH缓存到CacheView中
          mCachedViews.add(targetCacheIndex, holder);
          cached = true;
      }
      // 如果未CacheView缓存，则直接缓存RecyclerViewPool中
      if (!cached) {
          addViewHolderToRecycledViewPool(holder, true);
          recycled = true;
      }
       ...
}
```


### 局部刷新


#### Adapter数据操作对外API
RecyclerView.Adapter提供局部数据变化通知方法，然后执行到RecyclerViewDataObserver对应的各种数据操作方法上。


#### RecyclerViewDataObserver
* 通过mAdapterHelper进行数据变化处理操作
* 然后触发更新处理
* 下面介绍下 ItemChanged操作

```
   public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {
       assertNotInLayoutOrScroll(null);
       if (mAdapterHelper.onItemRangeChanged(positionStart, itemCount, payload)) {
           triggerUpdateProcessor();
       }
   }

   public void onItemRangeInserted(int positionStart, int itemCount) {
       assertNotInLayoutOrScroll(null);
       if (mAdapterHelper.onItemRangeInserted(positionStart, itemCount)) {
           triggerUpdateProcessor();
       }
   }

   public void onItemRangeRemoved(int positionStart, int itemCount) {
       assertNotInLayoutOrScroll(null);
       if (mAdapterHelper.onItemRangeRemoved(positionStart, itemCount)) {
           triggerUpdateProcessor();
       }
   }

```

#### AdapterHelper.onItemRangeChanged

```
boolean onItemRangeChanged(int positionStart, int itemCount, Object payload) {
   // 添加一个更新操作 ，标志为update、记录pos、item相关信息
   mPendingUpdates.add(obtainUpdateOp(UpdateOp.UPDATE, positionStart, itemCount, payload));
   mExistingUpdateTypes |= UpdateOp.UPDATE;
   // 如果只有一个待处理操作则为true，true则执行后续更新处理。如果是多个，则会忽略，因为在第一次出发后，就会集中处理
   return mPendingUpdates.size() == 1;
}

```

#### RecyclerViewDataObserver.triggerUpdateProcessor
1. 当RecyclerView有固定大小，且已经Attached了。则走Runnable更新
2. 否则直接走requestLayout方式更新，即重新走绘制流程 onMeasure、onLayout等

```
void triggerUpdateProcessor() {
  if (POST_UPDATES_ON_ANIMATION && mHasFixedSize && mIsAttached) {
      // RecyclerView有固定大小的时候 会执行mUpdateChildViewsRunnable 来处理更新
      ViewCompat.postOnAnimation(RecyclerView.this, mUpdateChildViewsRunnable);
  } else {
      // 直接走 requestLayout方式来处理
      mAdapterUpdateDuringMeasure = true;
      requestLayout();
  }
}

```

#### triggerUpdateProcessor下requestLayout
requestLayout下 onMeasure -> dispatchLayout -> dispatchLayoutStep2 -> layoutChildren -> fill -> layoutChunk -> next -> tryGetViewHolderForPositionByDeadline
最终对Item进行重新绑定 实现局部刷新逻辑

```
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
                
           if (mState.isPreLayout() && holder.isBound()) {
                holder.mPreLayoutPosition = position;
            } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
                // 执行数据变化的Holder的重新bind，从而实现局部刷新              
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);
                bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
            }
      
                
}

```
#### triggerUpdateProcessor下mUpdateChildViewsRunnable
当RecyclerView有固定大小时，则不需要Measure，直接走dispatchLayout方法进行刷新操作

```
final Runnable mUpdateChildViewsRunnable = new Runnable() {
   @Override
   public void run() {
        ...
        // 消费 等待执行的操作 
        consumePendingUpdateOperations();
   }
}

void consumePendingUpdateOperations() {
        
      if (mAdapterHelper.hasAnyUpdateTypes(AdapterHelper.UpdateOp.UPDATE) && !mAdapterHelper
           .hasAnyUpdateTypes(AdapterHelper.UpdateOp.ADD | AdapterHelper.UpdateOp.REMOVE
                   | AdapterHelper.UpdateOp.MOVE)) {
        // update 情况下 逻辑
       
       eatRequestLayout();
       onEnterLayoutOrScroll();
       // 数据预处理 
       mAdapterHelper.preProcess();
       if (!mLayoutRequestEaten) {
            // 执行 dispatchLayout 进行局部刷新处理
           if (hasUpdatedView()) {
               dispatchLayout();
           } else {
               // no need to layout, clean state
               mAdapterHelper.consumePostponedUpdates();
           }
       }
       ...
   } else if (mAdapterHelper.hasPendingUpdates()) {
       // add、remove等操作，直接执行dispatchLayout
       dispatchLayout();
       TraceCompat.endSection();
   }
}
```

