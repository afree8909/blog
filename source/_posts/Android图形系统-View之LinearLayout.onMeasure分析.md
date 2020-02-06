
---
tags: 
- 源码
- LinearLayout
categories:
- [Android, 系统]
---

### 图文概括
**总结**
1. 【父非EXACTLY & 子控件有weight并且指定高度为0】 则会执行二次测量方法
2. 【不满足1条件 & weight>0】的控件因为没有跳过第一次测量，而在第二次测量方法中也会进行测量
3. 【父非EXACTLY & 子height未指定0 】则weight属性不生效

**流程**
![](https://upload-images.jianshu.io/upload_images/9696036-f6bec68f9afc5729.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 源码分析
> 基于API23分析
#### 概览
```

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
   if (mOrientation == VERTICAL) {
       measureVertical(widthMeasureSpec, heightMeasureSpec);
   } else {
       measureHorizontal(widthMeasureSpec, heightMeasureSpec);
   }
}

void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 变量声明
    // 第一次测量，循环测量（非EXACTLY & lp.height==0 & lp.weight>0,父不定高且子高度为0还有权重）
    // 忽略，useLargestChild=true下的测量
    // 第二次测量 （weight相关）
    // 设置宽高值
}

```
#### 变量声明
```
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 变量声明
    // 成员变量，核心目的，求总高度
    mTotalLength = 0;
    // 记录子空间中最大宽度
    int maxWidth = 0;
    // 子控件的测量状态，会在遍历子控件测量的时候通过combineMeasuredStates来合并上一个子控件测量状态与当前遍历到的子控件的测量状态，采取的是按位相或
    int childState = 0;
    
    // layout_weight<=0的子控件最大宽度
    int alternativeMaxWidth = 0;
    // layout_weight>0的子控件最大宽度
    int weightedMaxWidth = 0;
    // 以上两个最大宽度跟上面的maxWidth最大的区别在于matchWidthLocally这个参数
    // 当matchWidthLocally为真，那么以下两个变量只会跟当前子控件的左右margin和相比较取大值
    // 否则，则跟maxWidth的计算方法一样
    
    // 是否子控件全是match_parent的标志位，用于判断是否需要重新测量
    boolean allFillParent = true;
    // 所有子控件的weight之和
    float totalWeight = 0;
    
    // 子控件数目（不含孙及以下控件）
    final int count = getVirtualChildCount();
    // 测量模式
    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    // 当子控件为match_parent的时候，该值为ture，同时判定的还有上面所说的matchWidthLocally，这个变量决定了子控件的测量是父控件干预还是填充父控件（剩余的空白位置）。
    boolean matchWidth = false;
    // 是否跳过第一次测量
    boolean skippedMeasure = false;
    
    final int baselineChildIndex = mBaselineAlignedChildIndex;        
    final boolean useLargestChild = mUseLargestChild;
    
    int largestChildHeight = Integer.MIN_VALUE;

}
```
#### 第一次测量
```
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 第一次测量
    // See how tall everyone is. Also remember max width.
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        
        ...
        // 添加 Divider高度
        if (hasDividerBeforeChildAt(i)) {
         mTotalLength += mDividerHeight;
        }
        
        LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();
        // 总weight 添加weight
        totalWeight += lp.weight;
        
        if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
           // EXACTLY 且 高度为0 且 weight>0 则先跳过该控件的测量
           final int totalLength = mTotalLength;
           mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
           skippedMeasure = true;
        } else {
            int oldHeight = Integer.MIN_VALUE;
               
            if (lp.height == 0 && lp.weight > 0) {
                // 此条件下，防止height为0
                oldHeight = 0;
                lp.height = LayoutParams.WRAP_CONTENT;
            }
        
            // 测量child（已经使用的height通过padding实现剔除）
            measureChildBeforeLayout(
                 child, i, widthMeasureSpec, 0, heightMeasureSpec,
                 totalWeight == 0 ? mTotalLength : 0);
        
            // ...
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                 lp.bottomMargin + getNextLocationOffset(child));
       
        }
        // 计算 宽度 和最大宽度等
        // ...
    
        i += getChildrenSkipCount(child, i);
    }
    
}

```
#### 调用子测量方法
```
void measureChildBeforeLayout(View child, int childIndex,
       int widthMeasureSpec, int totalWidth, int heightMeasureSpec,
       int totalHeight) {
    measureChildWithMargins(child, widthMeasureSpec, totalWidth,
      heightMeasureSpec, totalHeight);
}

protected void measureChildWithMargins(View child,
       int parentWidthMeasureSpec, int widthUsed,
       int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    // child测量时，会将已经用过的height和weight当作measure的padding来剔除
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
          mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                  + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
          mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                  + heightUsed, lp.height);
    
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
#### useLargestChild相关测量
```

void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 下面的这一段代码主要是为useLargestChild属性服务的，忽略
    if (mTotalLength > 0 && hasDividerBeforeChildAt(count)) {
      mTotalLength += mDividerHeight;
    }
    
    if (useLargestChild &&
          (heightMode == MeasureSpec.AT_MOST || heightMode == MeasureSpec.UNSPECIFIED)) {
      mTotalLength = 0;
    
      for (int i = 0; i < count; ++i) {
          final View child = getVirtualChildAt(i);
    
          if (child == null) {
              mTotalLength += measureNullChild(i);
              continue;
          }
    
          if (child.getVisibility() == GONE) {
              i += getChildrenSkipCount(child, i);
              continue;
          }
    
          final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                  child.getLayoutParams();
          // Account for negative margins
          final int totalLength = mTotalLength;
          mTotalLength = Math.max(totalLength, totalLength + largestChildHeight +
                  lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
      }
    }
       
    // Add in our padding
    mTotalLength += mPaddingTop + mPaddingBottom;
    
    int heightSize = mTotalLength;
    
    // Check against our minimum height
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
       
    // Reconcile our calculated size with the heightMeasureSpec
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK;


}

```
#### 第二次测量
⚠️ **weight**

1. 【父非EXACTLY & 子控件有weight并且指定高度为0】 则会执行二次测量方法
2. 【不满足1条件 & weight>0】的控件因为没有跳过第一次测量，而在第二次测量方法中也会进行测量
3. 【父非EXACTLY & 子height未制定0 则weight属性不生效】

```
公式：
第一次测量：mTotalLength = (是否跳过第一次测量) ？ 子.h ：0 
剩余高度： delta = heightSize - mTotalLength 
分配子高度： share = (int) (childExtra * delta / weightSum) 
剩余权重： weightSum -= childExtra
剩余分配高度：  delta -= share 
子高度： childHeight = child.getMeasuredHeight() + share
``` 
假设： LinearLayout.h = 100 , A.w : B.w = 2 :8 (权重)
结果： 子控件设置不同的height，结果如下（-1 = match_parent）

|  | A.h=0 | B.h=0 | A.h=-1 | B.h=-1 |
| --- | --- | --- | --- | --- |
| mTotalLength | 0 | 0 | 200 | 200 |
| delta | 100 | 80  | -100 | -80 |
| share | 20 | 80 | -20 | -80 |
| delta | 80 | 0 | -80 | 0 |
| childHeight | 20 | 80 | 80 | 20 |



```
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 计算剩余高度 
    int delta = heightSize - mTotalLength;
    // 上面skippedMeasure为true，剩余高度!=0 ，且totalWeight>0
    // 有weight的会测量两次，测量两次的控件是上面没有跳过测量的控件而跳过测量的控件这里进行测量（父非EXACTLY且子高度为0还有权重）
    if (skippedMeasure || delta != 0 && totalWeight > 0.0f) {
    	// 限定weight总和范围，如果设置过weighSum范围，那么子控件的weight为设置的weight
      float weightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;
    
      mTotalLength = 0;
    
      for (int i = 0; i < count; ++i) {
          final View child = getVirtualChildAt(i);
          
          if (child.getVisibility() == View.GONE) {
              continue;
          }
          
          LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();
          
          float childExtra = lp.weight;
          // weight  > 0 的控件进行测量
          if (childExtra > 0) {
            // 公式 = 剩余高度*（子控件的weight/weightSum）
            int share = (int) (childExtra * delta / weightSum);
            // weightSum计余
            weightSum -= childExtra;
            // 剩余高度
            delta -= share;
            // 计算child 的测量规格	
            final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                   mPaddingLeft + mPaddingRight +
                           lp.leftMargin + lp.rightMargin, lp.width);
            // 如果child height没有指定0（wrap或者match） 且非 EXACTLY
            if ((lp.height != 0) || (heightMode != MeasureSpec.EXACTLY)) {
                // childHeight = 测量高度+分配高度
                // ⚠️ 此处share为 负数
                int childHeight = child.getMeasuredHeight() + share;
                if (childHeight < 0) {
                    childHeight = 0;
                }
                // child测量
                child.measure(childWidthMeasureSpec,
                        MeasureSpec.makeMeasureSpec(childHeight, MeasureSpec.EXACTLY));
            } else {
                // 用分配高度进行child 测量
                child.measure(childWidthMeasureSpec,
                   MeasureSpec.makeMeasureSpec(share > 0 ? share : 0,
MeasureSpec.EXACTLY));
              }
    
    
              childState = combineMeasuredStates(childState, child.getMeasuredState()
                      & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
          }
            
        // 计算宽度 
        // ...  省略
      }
    
    
      mTotalLength += mPaddingTop + mPaddingBottom;
    
    }  else {
        // 没有weight，只有usrLargestChild为true的计算，省略 
    }
}

```
#### 设置宽高值
```
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 设置宽高
    if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
      maxWidth = alternativeMaxWidth;
    }
       
    maxWidth += mPaddingLeft + mPaddingRight;
    
    // Check against our minimum width
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
    // 设置 LinearLayout的宽、高值
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
          heightSizeAndState);
    
    if (matchWidth) {
      forceUniformWidth(count, heightMeasureSpec);
    }   
}

```

-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)

