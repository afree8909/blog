---

cover: https://fblitho.com/static/images/incremental-mount.png
tags: 
- Litho
categories:
- [Android, 框架]

---

### Litho 是什么
[Litho官网](https://fblitho.com/)
Litho是Facebook推出的一套高效构建Android UI的声明式框架，主要目的是提升RecyclerView复杂列表的滑动性能和降低内存占用。

**声明式组件**
允许用户使用声明式的API（注解）来构建符合Flexbox规范的布局。

**异步布局**
支持组件挂载，异步线程执行measure和layout操作，UI线程完成绘制工作

**扁平化布局**
Litho使用Yoga来完成组件布局的测量和计算，而且支持将View转换成Drawable渲染，降低了View层级，实现了布局的扁平化

**细粒度组件复用**
支持Text、Image或者Video细粒度组件的复用，尤其在多ItemType的列表中有显著性能提升

### Litho特性原理粗析
#### 声明式组件
Litho采用声明式的API来定义UI组件，组件通过一组不可变的属性来描述UI。这种组件化的思想灵感来源于React，实践过ReactNative或React都深有体会。不过声明式组件也有弊端，开发者不能直接预览布局编写，从而影响开发效率。

#### 异步布局
Android系统在绘制时为了防止页面错乱，页面所有View的测量（Measure）、布局（Layout）以及绘制（Draw）都是在UI线程中完成的。当页面UI非常复杂、视图层级较深时，难免Measure和Layout的时间会过长，从而导致页面渲染时候丢帧出现卡顿情况。
Litho鉴于此，提出了异步布局思想，最终实现了控件维度下异步线程完成measure和layout操作，在主线程完成draw操作。

首先，我们了解下 Android原生为什么不支持异步布局呢？

1. Android 视图是有状态和可变的。例如，TextView 必须跟踪它正在显示的文本，同时暴露一个允许开发人员对文本进行改变的 setText（）方法。这意味着如果 Android UI 框架决定把布局计算之类的操作放在在辅助线程中执行，则必须解决用户从另一个线程调用 setText（）并同时在布局计算发生时改变当前文本的问题。
2. 提前异步布局就意味着要提前创建好接下来要用到的一个或者多个条目的视图，而Android原生的View作为视图单元，不仅包含一个视图的所有属性，而且还负责视图的绘制工作。如果要在绘制前提前去计算布局，就需要预先去持有大量未展示的View实例，大大增加内存占用。

Litho的对策是怎样的呢 ？

1. Litho设计的组件的属性是不可变的，所以对于一个组件来说，它的布局计算结果是唯一且不变的。这样就不存在多线程问题
2. Litho组件只包含视图属性，仅负责计算布局，将绘制工作丢给父布局组件来完成。这样的拆分使得提前创建多个组件及子线程完成布局并不会带来太多的性能问题

##### Litho 布局绘制流程
View onMeasure时，在LayoutState中的clollectResults() 递归调用了nodeTree中的结点，最终调用yoga来计算结点的位置，目标就算是把所有的节点最终绘制到一个View中，降低View的层级。

View onLayout时，在MountState中，对所有的Component进行Mount调用，准备好需要绘制的具体内容，以待绘制调用。

​​同时Litho引入了缓存机制，可以提前在异步线程中进行measure和layout的操作，这样在绘制过程中只需要draw就可以了，理论提升效率35%（FaceBook宣传）
​​

![](http://q9j7c7ivg.bkt.clouddn.com/blog/20200429204925.png)

#### 扁平化的视图
使用Litho布局，我们可以得到一个极致扁平的视图效果。它可以减少渲染时的递归调用，加快渲染速度。

下面是同一个视图在Android和Litho实现下的视图层级效果对比。可以看到，同样的样式，使用Litho实现的布局要比使用Android原生实现的布局更加扁平。
![](https://fblitho.com/static/images/not-flat.png)
![](https://fblitho.com/static/images/flat.png)

##### 实现原理
1. 使用了Yoga来进行布局计算，Yoga会将Flexbox的相对布局转成绝对布局。经过Yoga处理后的布局没有了原来的布局层级，变成了只有一层。虽然不能解决过度绘制的问题，但是可以有效地减少渲染时的递归调用。
2. 前面介绍过Litho的视图渲染由绘制单元来完成，绘制单元可以是View或者更加轻量的Drawable，Litho自己实现了一系列挂载Drawable的基本视图组件。通过使用Drawable可以减少内存占用，同时相比于View，Android无法检查出Drawable的视图层级，这样可以使视图效果看起来更加扁平。

虽然平坦的视图结构对内存使用和绘图时间有重要的好处，但它们并非万能的灵丹妙药。Litho 有一个非常通用的系统来自动“ unflatten（堆叠起来） ” 已挂载组件的层次结构，当使用 Android View 里一些不可或缺的功能，如触摸事件的处理，无障碍功能，或限制失效等。例如，如果要在示例中启用单击图像或文本，则框架会自动将它们包装在视图中。

#### 细粒度组件回收
Litho中的所有组件都可以被回收，并在任何位置进行复用。这种细粒度的复用方式可以极大地提高内存使用率，尤其适用于多Item类型的复杂列表，内存优化非常明显。

RecyclerView 支持显示多样内容。为此，它根据视图的类型将视图保存在不同的缓冲池中。虽然这个概念在简单的情况下工作得很好，但是对于具有许多不同视图类型的 UI 来说，它可能会出现问题。在包含许多视图类型的场景中，在滚动事件之后进入窗口的视图更有可能是 RecyclerView 第一次显示的视图。如果发生这种情况，RecyclerView 必须分配一个新的视图。发生分配的 16ms 时间槽内 RecyclerView 也必须同时对即将显示的视图执行绑定、测量和布局操作

Litho 提供更具扩展性和高效率的回收系统，同时我们希望从 API 中消除视图类型的复杂性。 Litho 布局的表示与在屏幕上渲染布局的 View 和Drawable 完全脱节。这意味着，当我们需要在屏幕上放置一个新的 RecyclerView 视图时，我们已经知道该项目的内容以及它相对于其余 UI 的位置。 这使 Litho 完全摆脱 View 类型的概念。在 RecyclerView 滚动时，我们可以递增地使用文本或图像等构建块，而不是重新使用表示RecyclerView中项目的整个 View。 这对于传统的 Android 视图来说是不可能的，因为布局计算在完整的视图树上运行，并且当我们知道所有视图在一行中的位置时，所有视图都已经被实例化了。

##### 组件回收代码粗析
InternalNode：由Component+（View）NodeInfo转化得到的表示布局信息的节点
DiffNode：InternalNode的轻量级表示形式，用于在两个布局树的计算之间缓存测量结果，如果不需要更新，那么直接复用DiffNode即可
MountSpec：对应的Component包含纯粹的View或者Drawable，比如TextSpec，ImageSpec
LayoutSpec：对应的Component类似于ViewGroup，内部组织管理了很多子Component，比如CardSpec，SpinnerSpec

什么条件复用DiffNode至InternalNode ？

1. 两个节点的类型不会产生冲突
2. 两个节点的所有子节点类型不会产生冲突
3. 不需要更新Component，这通过每个自定义的Component都会重写的isEquivalentTo()方法判断

```
  // 当InternalNode和DiffNode都含有同一种Component类型或者都不含有Component时返回true
  private static boolean hostIsCompatible(InternalNode node, DiffNode diffNode) {
    if (diffNode == null) {
      return false;
    }

    return isSameComponentType(node.getTailComponent(), diffNode.getComponent());
  }
  
  static void applyDiffNodeToUnchangedNodes(InternalNode layoutNode, DiffNode diffNode) {
    try {
      final boolean isTreeRoot = layoutNode.getParent() == null;
      if (isLayoutSpecWithSizeSpec(layoutNode.getTailComponent()) && !isTreeRoot) {
        layoutNode.setDiffNode(diffNode);
        return;
      }

      // 复用条件1：两个节点包含的Component类型必须相同
      if (!hostIsCompatible(layoutNode, diffNode)) {
        return;
      }
		// 将DiffNode绑定到InternalNode中
      layoutNode.setDiffNode(diffNode);

      final int layoutCount = layoutNode.getChildCount();
      final int diffCount = diffNode.getChildCount();

      // 复用条件2：对两个节点的所有子节点做递归判断，他们的类型也必须兼容
      if (layoutCount != 0 && diffCount != 0) {
        for (int i = 0; i < layoutCount && i < diffCount; i++) {
          applyDiffNodeToUnchangedNodes(layoutNode.getChildAt(i), diffNode.getChildAt(i));
        }

        
      } else if (!shouldComponentUpdate(layoutNode, diffNode)) {
        // 如果不需要更新Component（根据每个Component都会重写的isEquivalentTo()方法判断），将DiffNode应用到InternalNode中的叶子节点（比如MountSpec）
        applyDiffNodeToLayoutNode(layoutNode, diffNode);
      }
    } catch (Throwable t) {
  }
  
   // 如果我们有一个cachedLayout，那么onPrepare和onMeasure之前就已经被调用了
  public @Nullable ConcurrentHashMap<Long, InternalNode> mThreadIdToLastMeasuredLayout;
  
  static InternalNode resolveNestedTree(
      ComponentContext context, InternalNode holder, int widthSpec, int heightSpec) {
			......
      // 检查是否有缓存可以使用
      final InternalNode cachedLayout =
          consumeCachedLayout(component, holder, widthSpec, heightSpec);
			......
  }
```


-------

参考
[Litho的使用及原理剖析](https://mp.weixin.qq.com/s/RS7O7prvkCvKyxkK3YQxtA?utm_source=androidweekly.io&utm_medium=website)
[LithoGitBook](https://fujianyi.gitbook.io/litho/)

