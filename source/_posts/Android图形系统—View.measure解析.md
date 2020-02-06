
---
tags: 
- 源码
- View
categories:
- [Android, 系统]
---

### 图文概览
#### View测量流程
1. measure
    * 性质：final方法，View测量初始方法  
    * 作用：基本逻辑计算，是否重新测量，调用onMeasure，缓存测量规格
2. onMeasure 
    * 性质：测量方法，子类可复写实现自己的测量方式
    * 作用：进行测量规格计算，调用getDefaultSize和setMeasureDimension
3. getDefaultSize
    * 作用：根据View的测量规格计算宽/高值
4. setMeasureDimension
    * 作用：存储测量后的宽/高值

View测量规格参考：[View.MeasureSpec分析](https://www.jianshu.com/p/66eb92cca405)


#### ViewGroup测量大概流程
![View.Measure](media/View.Measure.png)


### 源码分析
#### View.measure流程

```
// view measure入口， final类型
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 是否需要再次计算等计算
    ... 
    if (cacheIndex < 0 || sIgnoreMeasureCache) {   
        // 计算视图大小
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }else { 
        setMeasuredDimensionRaw((int) (value >> 32), (int) value);
    }
}


protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 存储测量后的View宽 / 高
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
          getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

// View的宽度为mMinWidth和mBackground.getMinimumWidth()中的最大值,高度类似
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth,mBackground.getMinimumWidth());
}

// 根据View宽/高的测量规格计算View的宽/高值
public static int getDefaultSize(int size, int measureSpec) {
    // 设置默认大小
    int result = size;
    // 获取宽/高测量规格的模式 & 测量大小
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    // 模式为AT_MOST,EXACTLY时，使用View测量后的宽/高值 = measureSpec中的Size
    
    case MeasureSpec.AT_MOST:  
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}

// 存储测量后的View宽 / 高 
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

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
    
    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}

```

#### ViewGroup.onMeasure

参考：[Android图形系统—View之LinearLayout.measure分析](https://www.jianshu.com/p/e893950d6cb3)

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // ViewGroup 继承类，大体都要充写次方法，大体模式如下
    
    // 遍历子View，对子控件一一measure
    for(int i=0;i<size;++i){
        measureChild(child, widthMeasureSpec, heightMeasureSpec)
    }
    
    // 合并所有子控件大小，计算出父控件宽高测量规格
    mergeChildSpec() ; // 也有可能与第一步同时计算
    
    // 设置父控件的宽/高值
    setMeasuredDimension(widthMeasure,  heightMeasure);
}


```




