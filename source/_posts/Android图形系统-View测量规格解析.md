---
cover: https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428150503.png
tags: 
- 源码
- View
categories:
- [Android, 系统]
---

### 图文概括

![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428150503.png)


![](https://raw.githubusercontent.com/afree8909/pictures/master/blog20200428152307.png)

### 源码分析
#### View.Measure 类分析
```
public static class MeasureSpec {
    // 进位大小 = 2的30次方
    private static final int MODE_SHIFT = 30;  
        
    // 运算遮罩：0x3为16进制，二进制为11，向左进30位
    // = 11 00 0000 0000 0000 0000 0000 0000 0000
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;  
    
    // UNSPECIFIED的模式设置：0向左进位30 = 00后跟30个0，即00 00000000000
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;  
      
    // EXACTLY的模式设置：1向左进位30 = 01后跟30个0 ，即01 00000000000
    public static final int EXACTLY = 1 << MODE_SHIFT;  
    
    // AT_MOST的模式设置：2向左进位30 = 10后跟30个0，即10 0000000000
    public static final int AT_MOST = 2 << MODE_SHIFT;  

 
    // 根据提供的size和mode得到测量规格，即measureSpec
    public static int makeMeasureSpec(int size, int mode) {  
     // 设计目的：使用一个32位的二进制数，其中：32和31位代表测量模式（mode）、后30位代表测量大小（size）
         return size + mode;  
     }  
    
     
    // 通过measureSpec获得测量模式（mode）
    public static int getMode(int measureSpec) {  
      //位运算：保留measureSpec的高2位（即测量模式）、使用0替换后30位
         return (measureSpec & MODE_MASK);  
    }  
      
    // 通过measureSpec获得测量大小size    
    public static int getSize(int measureSpec) {  
      // 位运算，即 将MODE_MASK取反，也就是变成了00 111111(00后跟30个1)，将32,31替换成0也就是去掉mode，保留后30位的size  
         return (measureSpec & ~MODE_MASK);  
        
    } 

```

#### ViewGroup.getChildMeasureSpec 方法分析
```
// 根据父视图的MeasureSpec & 布局参数LayoutParams，计算单个子View的MeasureSpec
// @param spec 父view的详细测量值(MeasureSpec) 
// @param padding view当前尺寸的的内边距和外边距(padding,margin) 
// @param childDimension 子视图的布局参数（宽/高）
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    // 父view的测量模式  父view的大小
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    // 通过父view计算出的子view = 父大小-边距（父要求的大小，但子view不一定用这个值） 
    int size = Math.max(0, specSize - padding);
    
    // 声明子view想要的实际大小和模式
    int resultSize = 0;
    int resultMode = 0;
    
    switch (specMode) {
        // 当父view的模式为EXACITY时，父view强加给子view确切的值
        // 一般是父view设置为match_parent或者固定值的ViewGroup 
        case MeasureSpec.EXACTLY:  
             // 当子view的LayoutParams>0，即有确切的值  
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
             // 当子view的LayoutParams为MATCH_PARENT时(-1) 
             // 子view大小为父view大小，模式为EXACTLY  
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                //子view决定自己的大小，但最大不能超过父view，模式为AT_MOST  
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
    
    
    // 当父view的模式为AT_MOST时，父view强加给子view一个最大的值。（一般是父view设置为wrap_content）  
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
            }
            break;
    
         // 当父view的模式为UNSPECIFIED时，父容器不对view有任何限制，要多大给多大
        // 多见于ListView、GridView  
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }

    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);

}

```


-------

推荐阅读：[图形系统总结](https://www.jianshu.com/p/238eb0a17760)


