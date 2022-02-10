# 从 chromium 源码来窥探浏览器的渲染

### **传统回答**

首先我们可以看下比较传统的回答方式，我们不去考虑 js 文件对⻚面解析造成的影响，在最简单的⻚面中，浏览器仅仅拿到 **HTML 文件** 和 **CSS 文件** ，便可以渲染出一个⻚面。在浏览器引擎中，会分别使用 HTML 解析器以及 CSS 解析器将接收到的二进制流数据转化为浏览器能够识别的 DOM 树和 CSS 规则树，随后将两者进行结合生成 Render(Layout Object)树，浏览器在拿到 (Layout Object)树后再经历分层，绘制等，我们才能在屏幕上看到最终的⻚面。

## Chrome 多进程机制

我们从另一个⻆度，Chrome 多进程的⻆度也可以进行探索，大家都知道 Chrome 采用的是一个 **多进程架构。** 详情参考现代浏览器架构。例如浏览器进程，负责浏览器主框架，提供一些通用的能力，GPU 进程则负责将渲染进程上传到 GPU 中的位图纹理进行处理随后呈现到屏幕上等。而浏览器渲染关联的渲染进程，当然是我们最关心的，它究竟是由哪些线程组成的，以及各个线程之间是如何通信合作来完成渲染的呢？

### **渲染进程组成**

渲染进程主要由如下几个线程组成：

1. GUI 渲染主线程： 解析 html，css，构建 DOM 树和 LayoutObject。
2. JS 引擎线程： 执行，解析 JS 代码。
3. 合成器线程：进行分块操作，同时也负责接受用戶的滚动，输入，分发回调事件等。
4. 栅格化线程：将绘制命令转换为位图或者 GPU 能识别的纹理。

我们可以通过 **chrome://tracing** 记录在一个⻚面渲染过程中，各个线程之间的通信：

![图片](https://mmbiz.qpic.cn/mmbiz_png/oAe2PlNm3ibibDyRiczwibUtLR1fbhLmV6sQ2qrJ61ibOQp9V8Dge0X1pv2mZoRibqcK8zM65mhalJmVYRSrTOICXiccg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上所示：CRFRenderMain 表示的是渲染主线程，主要进行一些计算的操作。Compositior 表示合成器线程，主要进行合成操作。Compositior Tile Workder 表示栅格化线程，现代浏览器往往有 2-4 个栅格化线程，浏览器会根据资源情况合理分配栅格化线程资源。

### **线程间通信过程**

我们从一帧渲染开始，来看各个线程之间的通信过程，如下所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/oAe2PlNm3ibibDyRiczwibUtLR1fbhLmV6sQxToa0C2tjt1qfCJYC3FRMCQ9DYicy5HxMERetN2YOfuJcQg6XwwxictA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

同时**Blink**内核为了清晰区分各个阶段，也定义了一个类**DocumentLifeCycle** 来确保各个阶段不会发生来回的跳转。类似于 React 当中的生命周期，一帧的渲染是一个 **原子操作**， 只要开始渲染便会一路执行到底，而不会进行回滚操作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/oAe2PlNm3ibibDyRiczwibUtLR1fbhLmV6sQRDmicW5LeULcJGxaic1UOH6n354gX6iceJIemLx6O08m2aqPMStsJaQGg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

整体的渲染流程如下：

1. 合成器线程接收到**Vsync**信号，开始新的一帧绘制。
2. 我们知道合成器线程可以处理用戶的输入，如果一些输入事件中存在一些回调事件，例如滚动的回调事件，那么合成器线程在上一帧收集完这些事件之后，会在当前帧将这些事件交给渲染主线程进行处理。
3. 执行 requestAnimationFrame 相关的动画操作。
4. 解析 HTML 数据，形成 DOM 树，这是 HTML 解析器(HTMLParser)的主要工作。浏览器接收到的html 数据也是字节流，因此要将其转换成浏览器能认识及转换的 token 标签，在这之中主要经历了如下步骤：

- 4.1. 解码：浏览器将接收的字节流(Bytes)基于编码方式解析为字符(characters)。
- 4.2. 分词：通过分词器（词法分析）将字符转换为 Token，分为 Tag Token 和文本 Token。详情可参考 vue 源码中的模板解析过程，大部分还是相同的。
- 4.3. 将 tokens 标签转换为 nodes 节点，随后将 nodes 节点添加至 DOM 树上，这两步是并行执行的，在这期间，主要是通过**栈**的数据结构来进行维护(类似于常⻅面试题-括号匹配)，当遇到开标签时，将对应 node 推入栈中，并且添加至 DOM 树上，当遇到文本标签时，就直接将文本 node 添加至 DOM 树上即可，当遇到闭合标签时，就进行出栈操作。另外 html 是一⻔**友好语言**，对于开闭标签不匹配的场景，或者是自定义的标签，都有自己的处理方式，在这里就不做具体展开。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1. 在有了 DOM 树之后，就需要去计算样式，计算样式的主要过程就不展开了，我们主要来看 CSS 解析器的产物，便是 styleSheets，在控制台使用**document.styleSheets**可以看到：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

关于 stylesheets 的具体属性，可参考stylesheets 详解

我们引入 css 的方式主要有行内样式，行内样式表，外部样式表(最经常使用)，这里的 stylesheets 便是一个个引入方式的最终解析产物。

在将各个引入方式进行解析后，我们就要将这些样式赋予我们的 DOM 节点，浏览器会结合 CSS 的 **继承** ， **优先级层叠** 等规则，形成 CSS 规则树，可以通过浏览器的**Element->Computed** 查看一个DOM 节点上的具体样式。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1. Layout，计算布局，这里主要是将⻚面中真正需要渲染的元素在 Layout Object 树中进行展示。
2. 更新 Layer Tree，这里主要是进行一些分层的操作，例如，更新 Paint Layer Tree 及 Graphic Layer Tree，在下文会进行具体展开。
3. Paint，生成绘画指令，以及记录需要执行哪些绘画调用和调用顺序，将其序列化记录进 SkPicture 数据结构中。
4. Composite，计算出每个合成层在合成时所需要的数据，包括位移（Translation）、缩放（Scale）、旋转（Rotation）混合等操作的参数。

以上这些都主要是在浏览器主线程中进行的操作，可看出主要进行的都是些复杂度不是很高的计算操作，而 JS 的解析执行则会放到专⻔的 JS 解析线程中去执行。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1. 提交至合成器线程，合成器线程主要在做的就是一个分块操作，大家都知道，若我们对一个⻚面中的所有元素进行绘制的话是非常消耗性能的(因为可视区域外的渲染根本没有必要)，因此我们会首先进行**分块操作**，将⻚面中的可视元素进行摘取并绘制。这样可以大大地节省资源。
2. 栅格化，栅格化主要进行的操作便是将上面产生的绘制指令转换成 GPU 能识别的位图或者纹理，主要有以下 2 种方式。

- a. 基于 CPU，使用 Skia 库的**软件加速(Software Rasterization)**，首先绘制进位图里，然后再作为纹理上传到 GPU。
- b. 基于 GPU，采用**硬件加速(Hardware Rasterization)**，这个过程是借助 OpenGL 直接在 GPU 纹理中进行绘制和光栅化，填充像素，也就是 GPU Raster。

1. 提交至 GPU 进程，进行渲染和前后缓冲区的交换，将结果展示至屏幕中。

## 分层阶段

在上面阐述的线程间通信中，我们主要忽略了分层这个过程，下面我们就主要来研究分层过程中产生的 3 棵树究竟是用来干嘛的。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### **Layout Object Tree (布局树)**

作用：DOM 节点可以分为可视化节点(div,p),非可视化节点(script,meta,head)等。Render 树的作用就是展现⻚面上真正需要渲染的元素，**忽略掉不可⻅元素，例如(display:none)，添加不存在 DOM 树中但需要显示的内容(例如伪元素)。**

#### 布局树形成概览

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

产生的过程如上图所示，便是将 DOM 树和 CSS 规则树进行结合，忽略掉不可⻅元素，以及添加可⻅元素，来计算出各个元素在⻚面中的位置。

在 Chrominum 中的源码也较简单，通过判断 display 来产生不同的 Layout Object。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### 映射关系

各个 html 节点与 Layout Object 的映射关系如下所示，他们都继承自同一个基类 LayoutObject，在其 基础上衍生出了不同的盒子模型，例如我们熟知的块级元素，行内元素以及行内块元素等，这些不同的类都定义了对其子元素以及兄弟元素之间是如何进行布局的，在这不做具体展开。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### 例子

我们也可以拿一个最简单的布局代码举例，来看看他会产生怎样的一颗 Render 树。

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <div>
      <p>123</p>
    </div>
    <div>
      1
      <p>456</p>
    </div>
  </body>
</html>
```

##### Content Shell

我们利用的是 Chromium 官方提供的 Content Shell 命令行工具，该工具拥有 Chrome 内核，但是没有 UI，详情参考，运行如下命令后，便可以看到上述代码产生的 Render 树。

```
out/mychromium/Content\ Shell.app/Contents/MacOS/Content\ Shell --run-web-tests ~Desktop/层演示/Layout/index.html
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这里要注意的一点便是文本1外部了一个匿名块元素，因为行内元素不能和块级元素相邻，所以为了布局方便，产生了一个匿名块元素包裹在文本元素外围。

### **Paint Layer(渲染层)**

一般来说，在 Render 树的基础上，我们会将拥有相同**z 坐标空间**的 Layout Objects，归属到同一个渲染层（Paint Layer）中。Paint Layer 最初是用来实现**stacking context（层叠上下文）**，类似于画一张蓝天白云图，我们要确定究竟是先画蓝天，还是白云。若先画白云，再画蓝天，会出现白云不可⻅的错误。层叠上下文的作用亦是如此，它主要来保证⻚面元素以正确的顺序合成。

#### 渲染层分类

渲染层也可以主要分为以下 3 类，各个渲染层的主要形成原因如下所示：

1. **kNormalPaintLayer**

- 根元素（HTML）

- position 值为 absolute 或 relative，且 z-index 不为 auto 的元素

- position 值为 fixed 或 sticky 的元素

- flex 容器的子元素，且 z-index 值不为 auto

- grid 容器的子元素，且 z-index 值不为 auto

- mix-blend-mode 属性值不为 normal 的元素

- 以下任意属性值不为 none 的元素：

- - transform
  - filter
  - perspective
  - clip-path
  - mask/mask-image/mask-border
  - isolation 属性值为 isolate 的元素

1. **kOverflowClipPaintLayer**

- overflow 不为 visible

1. **KNoPaintLayer**

- 不需要 paint 的 PaintLayer，比如一个没有视觉属性（背景、颜色、阴影等）的空 div

#### 举例

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <div style="position: absolute; background-color: yellow;z-index:1">
      <div style="opacity: 0.1">opacity</div>
      <div style="filter: blur(5px)">filter</div>
      <div style="transform: translateX(20px)">tranform</div>
      <div style="mix-blend-mode: multiply">mix-blend-mode</div>
      <div style="overflow: hidden">overflow</div>
    </div>
  </body>
</html>
```

我们也可以利用 Content Shell 查看上述代码产生的，Paint Layer 分层情况。

```
out/mychromium/Content\ Shell.app/Contents/MacOS/Content\ Shell --run-web-tests ~Desktop/层演示/Paint/index.html
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

可看到为各个满足生成渲染层条件的 html 元素都形成了一个渲染层。

#### 整体流程

整体源码中生成渲染层的调用流程如下所示：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1. **StyleDidChange 入口函数**我们主要关心的是 LayerTypeRequired 函数，在这个函数当中，我们会得到实际要产生哪种类型的渲染层。

   ```
   void LayoutBoxModelObject::StyleDidChange(StyleDifference diff,
                                             const ComputedStyle* old_style) {
    ···
    代码省略
    ···
     if (old_style && IsOutOfFlowPositioned() && Parent() &&
         (StyleRef().GetPosition() == old_style->GetPosition()) &&
         (StyleRef().IsOriginalDisplayInlineType() !=
          old_style->IsOriginalDisplayInlineType()))
       Parent()->SetNeedsLayout(layout_invalidation_reason::kChildChanged,
                                kMarkContainerChain);
     // 判断 Paint Layer 的类型
     PaintLayerType type = LayerTypeRequired();
     // 不为 kNoPaintLayer 
     if (type != kNoPaintLayer) {
       if (!Layer()) {
         // In order to update this object properly, we need to lay it out again.
         // However, if we have never laid it out, don't mark it for layout. If
         // this is a new object, it may not yet have been inserted into the tree,
         // and if we mark it for layout then, we risk upsetting the tree
         // insertion machinery.
         if (EverHadLayout())
           SetChildNeedsLayout();
           // 创建PainterLayer 并且插入
         CreateLayerAfterStyleChange();
       }
     }
   }
   ```

2. 1. **LayerTypeRequired 判断函数**

这个函数的主要作用便是判断产生哪种类型的 PaintLayer，这里的判断函数  **IsStacked** 顾名思义，便是用来判断会不会产生层叠上下文的。

```
PaintLayerType LayoutBox::LayerTypeRequired() const {
    NOT_DESTROYED();
      if (IsStacked() || HasHiddenBackface() ||
          (StyleRef().SpecifiesColumns() && !IsLayoutNGObject()) ||
          IsEffectiveRootScroller())
        return kNormalPaintLayer;

      if (HasNonVisibleOverflow())
        return kOverflowClipPaintLayer;

      return kNoPaintLayer;
}
```

IsStacked 函数的核心流程如下，我们主要去判断的就是 IsStackingContextWithoutContainment 属性，在样式解析过程中会有许多地方去设置这个属性。

```
inline bool IsStacked(const ComputedStyle& style) const {
    NOT_DESTROYED();
    // 判断定位不为 static 以及style样式中某些属性是否满足堆叠上下文
    return style.GetPosition() != EPosition::kStatic &&
           IsStackingContext(style);
  }
  
   inline bool IsStackingContext(const ComputedStyle& style) const {
    NOT_DESTROYED();
    // This is an inlined version of the following:
    // `IsStackingContextWithoutContainment() ||
    //  ShouldApplyLayoutContainment() ||
    //  ShouldApplyPaintContainment()`
    // The reason it is inlined is that the containment checks share
    // common logic, which is extracted here to avoid repeated computation.
    // 判断style的 IsStackingContextWithoutContainment 属性
    return style.IsStackingContextWithoutContainment() ||
           ((style.ContainsLayout() || style.ContainsPaint()) &&
            (!IsInline() || IsAtomicInlineLevel()) && !IsRubyText() &&
            (!IsTablePart() || IsLayoutBlockFlow()));
  }
```

整体的调用链路为:

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

设置 IsStackingContextWithoutContainment 属性的地方有很多，例如拥有 transform3D 属性，便会设置 IsStackingContextWithoutContainment 为 true，我们可以去观察下HasStackingGroupingProperty 这个函数。

```
void ComputedStyle::UpdateIsStackingContextWithoutContainment(
    bool is_document_element,
    bool is_in_top_layer,
    bool is_svg_stacking) {
  if (IsStackingContextWithoutContainment())
    return;
  // Force a stacking context for transform-style: preserve-3d. This happens
  // even if preserves-3d is ignored due to a 'grouping property' being present
  // which requires flattening. See:
  // ComputedStyle::HasGroupingPropertyForUsedTransformStyle3D().
  // This is legacy behavior that is left ambiguous in the official specs.
  // See https://crbug.com/663650 for more details.
  // transform 3D 样式设置为true
  if (TransformStyle3D() == ETransformStyle3D::kPreserve3d) {
  // 设置 IsStackingContextWithoutContainment
    SetIsStackingContextWithoutContainment(true);
    return;
  }
  // document 根元素或者含有 StackingGroupingProperty 属性
  if (is_document_element || is_in_top_layer || is_svg_stacking ||
      StyleType() == kPseudoIdBackdrop || HasTransformRelatedProperty() ||
      HasStackingGroupingProperty(BoxReflect()) ||
      HasViewportConstrainedPosition() || GetPosition() == EPosition::kSticky ||
      HasPropertyThatCreatesStackingContext(WillChangeProperties()) ||
      /* TODO(882625): This becomes unnecessary when will-change correctly takes
      into account active animations. */
      ShouldCompositeForCurrentAnimations()) {
     // 设置 IsStackingContextWithoutContainment
    SetIsStackingContextWithoutContainment(true);
  }
}
```

HasStackingGroupingProperty 函数中，为各个有特殊样式设置的元素都提升为了渲染层。

```
 bool HasStackingGroupingProperty(bool has_box_reflection) const {
    // opcaity 属性设置为 true
    if (HasNonInitialOpacity())
      return true;
    // filter 属性设置
    if (HasNonInitialFilter())
      return true;
    if (has_box_reflection)
      return true;
    if (HasClipPath())
      return true;
    if (HasIsolation())
      return true;
      // mask 属性设置
    if (HasMask())
      return true;
      // mix-blend-mode 属性
    if (HasBlendMode())
      return true;
    if (HasNonInitialBackdropFilter())
      return true;
    return false;
  }
```

1. **Paint Layer 插入函数**

在创建了渲染层之后，我们便要将其插入到已有渲染层树当中，核心流程如下：

```
void PaintLayer::InsertOnlyThisLayerAfterStyleChange() {
  if (!parent_ && GetLayoutObject().Parent()) {
    // We need to connect ourselves when our layoutObject() has a parent.
    // Find our enclosingLayer and add ourselves.
    // 调用 Enclosing Layer
    PaintLayer* parent_layer = GetLayoutObject().Parent()->EnclosingLayer();
    DCHECK(parent_layer);
    // 调用 FindNextLayer
    PaintLayer* before_child = GetLayoutObject().Parent()->FindNextLayer(
        parent_layer, &GetLayoutObject());
    // 采用头插法进行插入
    parent_layer->AddChild(this, before_child);
  }
  ```
  代码省略
  ```
}
```

- 调用 EnclosingLayer，找到新创建的 PaintLayer 关联的 LayoutObject 的父节点对应的 Paint Layer。
- 调用 FindNext Layer，找到要插入的 Paint Layer 在父 Paint Layer 的 Child List 中的位置 before_child。
- 采用头插入的方法将创建的 Paint Layer 插入进 Child List 链表中。

详情可参考：[paint layer的创建与插入](http://www.4k8k.xyz/article/tornmy/81737603)

### **GraphicLayer（合成层）**

在渲染层树的基础上，浏览器又会将某些特殊的渲染层会被提升为合成层（G raphicLayers），合成层拥有单独的 GraphicsLayer，而其他不是合成层的渲染层，则和其第一个拥有  GraphicsLayer 父层共用一个，如下所示：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### 优势

每个 GraphicsLayer 实际都有一个 **Graphics Context** ，可以理解为一个缓存。GraphicsContext会缓存该层的位图，在下次进行绘制的时候，合成层便可以利用这些缓存直接进行绘制，而不用走绘制指令的生成等过程。位图是存储在共享内存中，作为纹理上传到 GPU 中。

#### 合成层形成举例

**注意这些都在 chorme94 以下版本中才能生效，原因后续会说明、**

##### 直接原因（direct reason)

- 有 3dtranform 相关设置 (translateZ,rotate 一些属性的设置）
- 不同域的 iframe 元素，会使用硬件加速，使用合成层。

iframe 的主域和子域之间拥有不同的渲染进程，而渲染进程默认有一个根合成层，因此 iframe 元素会单独形成一个合成层。

```
<!DOCTYPE html>
<html>
  <head>
    <title>Composited frame test</title>
    <style type="text/css" media="screen">
      #iframe {
        position: absolute;
        left: 100px;
        top: 100px;
        padding: 0;
        height: 300px;
        width: 400px;
        background-color: red;
      }
    </style>
  </head>
  <iframe id="iframe" src="composited-subframe.html" frameborder="0"></iframe>
</html>
```

- video 元素，2d 或者 3d 的 canvas 层
- backface-visibility 为 hidden， **元素的背面朝向观察者是否可⻅** 。详情参考
- 对 opacity、transform、fliter、backdropfilter 应用了 animation 或者 transition（需要是 active 的 animation 或者 transition，也就是在动画中的元素，当 animation 或者 transition 效果未开始或结束后，提升合成层也会失效）

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Test Animation</title>
    <style>
      .target {
        width: 80px;
        height: 80px;
        background-color: green;
        -webkit-animation: swing 5s linear 1;
      }
      @-webkit-keyframes swing {
        from {
          transform: rotate(0deg);
        }
        to {
          transform: rotate(90deg);
        }
      }
    </style>
  </head>
  <body>
    <div class="target"></div>
  </body>
</html>
```

- will-change 设置为 opacity、transform、top、left、bottom、right（其中 top、left 等需要设置明确的定位属性，如 relative 等，默认定位 static 不会产生合成层。

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Test Will-change</title>
    <style>
      div {
        width: 150px;
        height: 30px;
        margin: 20px;
        padding: 10px;
        background-color: red;
        color: #fff;
      }

      .left {
        position: relative;
        will-change: left;
      }

      .top {
        position: relative;
        will-change: top;
      }

      .bottom {
        position: relative;
        will-change: bottom;
      }

      .right {
        position: relative;
        will-change: right;
      }

      .opacity {
        position: relative;
        will-change: opacity;
      }

      .transform {
        position: relative;
        will-change: transform;
      }
    </style>
  </head>
  <body>
    <div class="left">will-change: left</div>
    <div class="top">will-change: top</div>
    <div class="bottom">will-change: bottom</div>
    <div class="right">will-change: right</div>
    <div class="opacity">will-change: opacity</div>
    <div class="transform">will-change: transform</div>
  </body>
</html>
```

##### 后代元素原因（kComboCompositedDescendants）

- 有合成层后代同时本身设置了 opactiy（小于 1）、mask、fliter、reflection 等属性。

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Test Descendant</title>
    <style>
      .outer {
        width: 100px;
        height: 100px;
        border: 1px solid #000;
        margin: 20px;
      }

      .inner {
        width: 50px;
        height: 50px;
        margin: 20px;
        background-color: green;
      }

      .composited {
        will-change: transform;
      }

      .mask {
        -webkit-mask: linear-gradient(transparent, black);
      }

      .opacity {
        opacity: 0.5;
      }

      .reflection {
        -webkit-box-reflect: right 10px;
      }

      .filter {
        -webkit-filter: drop-shadow(-25px -25px 0 gray);
      }
    </style>
  </head>
  <body>
    <div class="outer filter">
      filter
      <div class="inner composited">inner</div>
    </div>
    <div class="outer reflection">
      reflection
      <div class="inner composited">inner</div>
    </div>
    <div class="outer opacity">
      opacity
      <div class="inner composited">inner</div>
    </div>
    <div class="outer mask">
      mask
      <div class="inner composited">inner</div>
    </div>
  </body>
</html>
```

- 有 3d transfrom 的合成层后代同时本身有 preserves-3d 属性。

```
<!DOCTYPE html>
<html>
<head>
  <style>
    .box {
      position: relative;
      height: 100px;
      width: 100px;
      margin: 10px;
      left: 0;
      top: 0;
      background-color: silver;
    }
    
    .preserve3d {

      width: 300px;
      border: 1px solid black;
      padding: 20px;
      margin: 10px;
      -webkit-transform-style: preserve-3d;
    }
  </style>

</head>
<body>

  <div class="preserve3d">
    This layer should not be composited.
    <div class="box"></div>
    </div>
  </div>

  <div class="preserve3d">
    This layer should not be composited.
    <div class="box" style="transform: rotate(10deg)"></div>
    </div>
  </div>

  <div class="preserve3d">
    This layer should be composited.
    <div class="box" style="transform: rotateY(10deg)"></div>
  </div>

</body>
</html>
```

- 有 3dtransfrom 的合成层后代同时本身有 perspective 属性

##### 重叠原因（overlap）

我们重点来看当某个元素与合成层产生重叠时，为什么也会产生合成层。原有视图中，top 和 bottom 元素共同享用父级元素的合成层资源。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

当底部 bottom 因为某些原因形成合成层时，若按照原有渲染规则，top 元素会和父级元素首先被渲染，随后是 bottom 元素的渲染，这样就打破了层叠上下文的准则，如下所示：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

因此顶部 top 元素也必须也提升为合成层，这样才能保证按照正确的顺序进行渲染。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

会有如下几个重叠原因来被迫产生合成层：

- filter 效果同合成层重叠

```
<!DOCTYPE html>
<html>
  <head>
    <title>Test Overlap Filter</title>
    <!-- If the test passes, the light green drop shadow should appear over the the black div where they intersect. -->
    <style>
      #software {
        background-color: green;
        -webkit-filter: drop-shadow(25px 25px 0 lightgreen);
        position: absolute;
        top: 0;
        left: 0;
        width: 100px;
        height: 100px;
        color: #fff;
      }
      #composited {
        background-color: black;
        position: absolute;
        top: 105px;
        left: 105px;
        width: 100px;
        height: 100px;
        transform: translate3d(0, 0, 0);
        color: #fff;
      }
    </style>
  </head>
  <body>
    <div id="composited">composited</div>
    <div id="software">overlap</div>
  </body>
</html>
```

- transform 变换后同合成层重叠
- overflow scroll 情况下同合成层重叠。即如果一个 overflow scroll（不管 overflow:auto 还是 overflow:scroll，只要是能 scroll 即可） 的元素同一个合成层重叠，则其可视子元素也同该合成层重叠
- 假设重叠在一个合成层之上（assumedOverlap）

比如一个元素的 CSS 动画效果，动画运行期间，元素是有可能和其他元素有重叠的。针对于这种情况，于是就有了 assumedOverlap 的合成层产生原因，在本 demo 中，动画元素视觉上并没有和其兄弟元素重叠，但因为 assumedOverlap 的原因，其兄弟元素依然提升为了合成层。

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Test Overlap Animation</title>
    <style>
      @-webkit-keyframes slide {
        from {
          transform: none;
        }
        to {
          transform: translateX(100px);
        }
      }

      .animating {
        width: 100px;
        height: 100px;
        background-color: orange;
        color: #fff;
        -webkit-animation: slide 10s alternate linear infinite;
      }

      .overlap {
        width: 100px;
        height: 100px;
        color: #fff;
        position: relative;
        margin: 10px;
        background-color: blue;
      }
    </style>
  </head>
  <body>
    <div class="animating">composited animating</div>
    <div class="overlap">overlap</div>
  </body>
</html>
```

##### 层压缩

当满足一些条件时，浏览器也会进行层的压缩，以防止出现“太多合成层”的情况，如下所示：

```
<!DOCTYPE html>
<head>
<style>
.composited {
transform: translateZ(0);
}

.box {
  width: 100px;
  height: 100px;
}

.behind {
  position: absolute;
  z-index: 1;
  top: 100px;
  left: 100px;
  background-color: blue;
}

.middle {
  position: absolute;
  z-index: 1;
  top: 180px;
  left: 180px;
  background-color: lime;
}

.middle2 {
  position: absolute;
  z-index: 1;
  top: 260px;
  left: 260px;
  background-color: magenta;
}

.top {
  position: absolute;
  z-index: 1;
  top: 340px;
  left: 340px;
  background-color: cyan;
}

div:hover {
  background-color: green;
  transform:translatez(0);
}

</style>
</head>
<body>
  <div class="composited box behind"></div>
  <div class="box middle"></div>
  <div class="box middle2"></div>
  <div class="box top"></div>
</body>
```

当 hover 至蓝色元素时，蓝色元素会被提升为合成层，而其余 3 个元素，若按照重叠原因，则会产生 3个合成层，但是浏览器会将他们划分到一个合成层中，避免资源的过度浪费(因为创建一个合成层，便会产生一个上下文来进行记录，这样会加大浏览器的资源开销)。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

##### 无法进行层压缩

但有些情况下，浏览器也不能进行层压缩，必须拥有不同的合成层，主要有如下几种情况：

- 当渲染层同合成层有不同的裁剪容器（clippingcontainer）时，该渲染层无法压缩（squashingClippingContainerMismatch）

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Test</title>
    <style>
      .clipping-container {
        overflow: hidden;
        height: 10px;
        background-color: blue;
      }

      .composited {
        transform: translateZ(0);
        height: 10px;
        background-color: red;
      }

      .box1 {
        position: absolute;
        top: 0px;
        height: 100px;
        width: 100px;
        background-color: green;
        color: #fff;
      }
      .box2 {
        overflow: hidden;
        position: relative;
        top: -10px;
      }
    </style>
  </head>
  <body>
    <div class="clipping-container">
      <div class="composited"></div>
    </div>
    <div class="box1">第一个不会被压缩到 composited div 上</div>

    <div class="box2">第二个不会被压缩到 composited div 上</div>
  </body>
</html>
```

上述代码中，composited 元素与 box1，以及 box2 有不同的裁剪容器，因此 box1，box2 与composited 产生重叠时，会被迫单独产生一个合成层，而不能进行共享。

- 无法进行会打破渲染顺序的压缩（ksquashingWouldBreakPaintOrder）
- video 元素的渲染层无法被压缩同时也无法将别的渲染层压缩到 video 所在的合成层上（ksquashingVideoIsDisallowed）
- iframe、plugin的渲染层无法被压缩同时也无法将别的渲染层压缩到其所在的合成层上（ksquashingLayoutPartIsDisallowed）
- 无法压缩有 reflection 属性的渲染层（ksquashingReflectionDisallowed）
- 无法压缩有 blendmode 属性的渲染层（ksquashingBlendingDisallowed）

源码详解

#### 整体流程

整体合成层的判断及创建过程如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1. ##### UpdateAssignmentsIfNeeded 入口函数

```
void PaintLayerCompositor::UpdateAssignmentsIfNeeded(
    DocumentLifecycle::LifecycleState target_state) {
    // 更新 dom 生命周期函数
  DCHECK(target_state >= DocumentLifecycle::kCompositingAssignmentsClean);
  ···
  代码省略
  ···
  if (update_type >= kCompositingUpdateAfterCompositingInputChange) {
    CompositingRequirementsUpdater(*layout_view_).Update(update_root);

    CompositingLayerAssigner layer_assigner(this);
    // 1. 创建 CompositingLayerMapping (Paint Layer 到 Graphic Layer 的映射）
    layer_assigner.Assign(update_root, layers_needing_paint_invalidation);
    CHECK_EQ(compositing_, (bool)RootGraphicsLayer());

    if (layer_assigner.LayersChanged())
      update_type = std::max(update_type, kCompositingUpdateRebuildTree);
  }

#if DCHECK_IS_ON()
  if (update_root->GetCompositingState() != kPaintsIntoOwnBacking) {
    AssertWholeTreeNotComposited(*update_root);
  }
#endif

  GraphicsLayerUpdater updater;
  // 2.更新 Graphic Layer 的一些属性
  updater.Update(*update_root, layers_needing_paint_invalidation);
  // 3. 根据 NeedsRebuildTree 判断是否需要重新创建 Graphics Layers
  if (updater.NeedsRebuildTree())
    update_type = std::max(update_type, kCompositingUpdateRebuildTree);
  if (update_type >= kCompositingUpdateRebuildTree) {
    GraphicsLayerVector child_list;
    {
  
      TRACE_EVENT0("blink", "GraphicsLayerTreeBuilder::rebuild");
      GraphicsLayerTreeBuilder().Rebuild(*update_root, child_list);
    }
```

其中核心的便是 ComputeCompositedLayerUpdate，该函数的主要作用便是将多个渲染层映射为一个合成层。

1. ComputeCompositedLayerUpdate 函数

   ![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

```
CompositingLayerAssigner::ComputeCompositedLayerUpdate(PaintLayer* layer) {
  CompositingStateTransitionType update = kNoCompositingStateChange;
  if (NeedsOwnBacking(layer)) {
    if (!layer->HasCompositedLayerMapping()) {
     // 1. 可以创建 Graphic Layers
      update = kAllocateOwnCompositedLayerMapping;
    }
  } else {
    if (layer->HasCompositedLayerMapping())
     // 2. 删除 Graphic Layers
      update = kRemoveOwnCompositedLayerMapping;

    if (!layer->SubtreeIsInvisible() && layer->CanBeComposited() &&
        RequiresSquashing(layer->GetCompositingReasons())) {
      // We can't compute at this time whether the squashing layer update is a
      // no-op, since that requires walking the paint layer tree.
      update = kPutInSquashingLayer;
    } else if (layer->GroupedMapping() || layer->LostGroupedMapping()) {
     // 3. 可以进行层压缩
      update = kRemoveFromSquashingLayer;
    }
  }
  return update;
}
```

在 ComputeCompositedLayerUpdate 的 needsOwnBacking 函数中，主要进行的就是合成层原因的判断。

**形成 Graphic Layers 原因**

形成合成层的原因在 Chromium 的 compositing_reason 中有详细的说明，和上面的分类差不多，主 要有直接原因，层堆叠原因以及后代元素原因。

compositing_reason.h(合成层原因概述)

```
namespace blink {

using CompositingReasons = uint64_t;

#define FOR_EACH_COMPOSITING_REASON(V)                                        \
  /* Intrinsic reasons that can be known right away by the layer. */   
  //1. 直接原因       \
  V(3DTransform)                                                              \
  V(Trivial3DTransform)                                                       \
  V(Video)                                                                    \
  V(Canvas)                                                                   \
  V(Plugin)                                                                   \
  V(IFrame)                                                                   \
  V(DocumentTransitionContentElement)                                         \
  /* This is used for pre-CompositAfterPaint + CompositeSVG only. */          \
  V(SVGRoot)                                                                  \
  V(BackfaceVisibilityHidden)                                                 \
  V(ActiveTransformAnimation)                                                 \
  V(ActiveOpacityAnimation)                                                   \
  V(ActiveFilterAnimation)                                                    \
  V(ActiveBackdropFilterAnimation)                                            \
  V(AffectedByOuterViewportBoundsDelta)                                       \
  V(FixedPosition)                                                            \
  V(StickyPosition)                                                           \                                           \
  V(OutOfFlowClipping)                                                        \
  V(VideoOverlay)                                                             \
  V(WillChangeTransform)                                                      \
  V(WillChangeOpacity)                                                        \
  V(WillChangeFilter)                                                         \
  V(WillChangeBackdropFilter)                                                 \
                                                                              \
  /* Reasons that depend on ancestor properties */                            \
  V(BackfaceInvisibility3DAncestor)                                           \
  /* TODO(crbug.com/1256990): Transform3DSceneLeaf today depends only on the  \
     element and its properties, but in the future it could be optimized      \
     to consider descendants and moved to the subtree group below. */         \
  V(Transform3DSceneLeaf)                                                     \
  /* This flag is needed only when none of the explicit kWillChange* reasons  \
     are set. */                                                              \
  V(WillChangeOther)                                                          \
  V(BackdropFilter)                                                           \
  V(BackdropFilterMask)                                                       \
  V(RootScroller)                                                             \
  V(XrOverlay)                                                                \
  V(Viewport)                                                                 \
  // 2. 层的堆叠                                                                            \
  /* Overlap reasons that require knowing what's behind you in paint-order    \
     before knowing the answer. */                                            \
  V(AssumedOverlap)                                                           \
  V(Overlap)                                                                  \
  V(NegativeZIndexChildren)                                                   \
  V(SquashingDisallowed)                                                      \
                                                                              \
  /* Subtree reasons that require knowing what the status of your subtree is  \
     before knowing the answer. */   
  3.// 后代元素原因                                         \
  V(OpacityWithCompositedDescendants)                                         \
  V(MaskWithCompositedDescendants)                                            \
  V(ReflectionWithCompositedDescendants)                                      \
  V(FilterWithCompositedDescendants)                                          \
  V(BlendingWithCompositedDescendants)                                        \
  V(PerspectiveWith3DDescendants)                                             \
  V(Preserve3DWith3DDescendants)                                              \
  V(IsolateCompositedDescendants)                                             \
  V(FullscreenVideoWithCompositedDescendants)                                 \
                                                                              
```

compositing_reason.cc(映射关系)

在 compositing_reason.cc 中也描述了各个形成合成层的原因，这与我们在控制台的 Layer 模块中看到的合成原因一一对应。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 绘制阶段

### **当前算法流程**

```
from layout
  |
  v
+------------------------------+
| LayoutObject/PaintLayer tree |-----------+
+------------------------------+           |
  |                                        |
  | PaintLayerCompositor::UpdateIfNeeded() |
  |   CompositingInputsUpdater::Update()   |
  |   CompositingLayerAssigner::Assign()   |
  |   GraphicsLayerUpdater::Update()       | PrePaintTreeWalk::Walk()
  |   GraphicsLayerTreeBuilder::Rebuild()  |   PaintPropertyTreeBuider::UpdatePropertiesForSelf()
  v                                        |
+--------------------+                   +------------------+
| GraphicsLayer tree |<------------------|  Property trees  |
+--------------------+                   +------------------+
      |                                    |              |
      |<-----------------------------------+              |
      | LocalFrameView::PaintTree()                       |
      |   LocalFrameView::PaintGraphicsLayerRecursively() |
      |     GraphicsLayer::Paint()                        |
      |       CompositedLayerMapping::PaintContents()     |
      |         PaintLayerPainter::PaintLayerContents()   |
      |           ObjectPainter::Paint()                  |
      v                                                   |
    +---------------------------------+                   |
    | DisplayItemList/PaintChunk list |                   |
    +---------------------------------+                   |
      |                                                   |
      |<--------------------------------------------------+
      | PaintChunksToCcLayer::Convert()                   |
      v                                                   |
+--------------------------------------------------+      |
| GraphicsLayerDisplayItem/ForeignLayerDisplayItem |      |
+--------------------------------------------------+      |
  |                                                       |
  |    LocalFrameView::PushPaintArtifactToCompositor()    |
  |         PaintArtifactCompositor::Update()             |
  +--------------------+       +--------------------------+
                       |       |
                       v       v
        +----------------+  +-----------------------+
        | cc::Layer list |  |   cc property trees   |
        +----------------+  +-----------------------+
                |              |
  +-------------+--------------+
  | to compositor
```

当前 chorme 的渲染流程大致如上所示，我们在分层之后，便会在一个合成层上进行操作，来产生一个合成层上的绘制指令，这些绘制指令之间的顺序说明了当前合成层首先绘制什么，其次绘制什么，来保证一个合成层上的多个渲染层(也就是多个层叠上下文)按照既定的规则进行有顺序的渲染，如下所示：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

我们会首先去渲染根层叠上下文，随后再去绘制负 z-index 的层叠上下文，其次是一些布局元素，例如 float，块级元素等，随后是正 z-index 以及行内元素(因为内容是最重要的，是需要被用戶看⻅的)。

#### 举例

```
<!DOCTYPE html>
<html lang="en">
  <meta http-equiv="Content-Type" content="text/html; Charset=UTF-8" />
  <head>
    <style type="text/css">
      * {
        margin: 0;
        padding: 0;
      }
      div {
        width: 200px;
        height: 100px;
        text-align: center;
        line-height: 100px;
      }
      p {
        height: 40px;
        line-height: 40px;
        font-size: 20px;
        margin-bottom: 30px;
      }
      .level-default {
        position: absolute;
        background: #f5cec7;
        top: 60px;
      }
      .level1 {
        background: #ffb284;
        position: absolute;
        z-index: 2;
        top: 160px;
      }
      .level2 {
        background: #e79796;
        position: absolute;
        z-index: 1;
        top: 260px;
      }
      .composite-1 {
        position: relative;
        transform: translateZ(0);
        width: 300px;
        height: 400px;
        background: #ddd;
        margin-bottom: 20px;
      }
    </style>
  </head>
  <body>
    <div class="composite-1">
      <p>合成层一</p>
      <div class="level-default">默认层</div>
      <div class="level1">渲染层1：z-index:2</div>
      <div class="level2">渲染层2：z-index:1</div>
    </div>
    
  </body>
</html>
```

我们以上述代码为例，来看 composite-1 这个合成层上的渲染层，是不是以上面提到的绘制顺序进行绘制。最终的绘制产物如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### **查看过程**

通过 Layers 工具中的 PaintProfiler 可以查看到绘制指令。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### **绘制指令**

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

各指令参考也可参考 flutter 中的指令解析，https://api.flutter.dev/flutter/dart- ui/Canvas/restore.html，大致和浏览器中的绘制指令定义是差不多的。

## 性能优化

上面讲了这么多合成层形成原因，以及合成层的优势，我们当然要进行合理的利用，来提升我们的⻚面性能，可以分为如下 2 个层面：

### **代码层面**

1. 不同 css 属性会触发的流程 https://csstriggers.com/，从该网站中可以看出修改各个属性的值究竟会触发哪些流程，例如我们熟悉的top,一旦修改的话，便会**经历布局，绘制，合成**等，相当于分层阶段的过程都会走一遍，因此是极度损耗性能的。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

而 transform 属性的修改，我们只要将 **合成层进行一些变换操作** 即可，例如进行位移或者透明度的转换，而布局和绘制的命令都可以使用缓存中的数据，因此极大的提升了性能。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

position 动画：

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
    <style>
      .ball-running {
        width: 100px;
        height: 100px;
        animation: run-around 2s linear alternate 100;
        background: red;
        position: absolute;
        border-radius: 50%;
      }
      @keyframes run-around {
        0% {
          top: 0;
          left: 0;
        }
        25% {
          top: 0;
          left: 200px;
        }
        50% {
          top: 200px;
          left: 200px;
        }
        75% {
          top: 200px;
          left: 0;
        }
        100% {
          top: 0px;
          left: 0px;
        }
      }
    </style>
  </head>
  <body>
    <div class="ball-running"></div>
  </body>
</html>
```

transform 动画：

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
    <style>
      .ball-running {
        width: 100px;
        height: 100px;
        animation: run-around 2s linear alternate 100;
        background: red;
        position: absolute;
        border-radius: 50%;
      }
      .ball-running {
        width: 100px;
        height: 100px;
        animation: run-around 2s linear alternate 100;
        background: red;
        border-radius: 50%;
      }
      @keyframes run-around {
        0% {
          transform: translate(0, 0);
        }
        25% {
          transform: translate(200px, 0);
        }
        50% {
          transform: translate(200px, 200px);
        }
        75% {
          transform: translate(0, 200px);
        }
      }
    </style>
  </head>
  <body>
    <div class="ball-running"></div>
  </body>
</html>
```

##### 性能对比

我们也可以将 left 动画与 transform 动画在相同的 CPU 算力下进行一个对比，如下可⻅，left 动画的后半程会出现丢帧的情况，整个⻚面的 fps 也降低了很多，而这仅仅是一个元素，若是一堆元素进行重新布局的话，⻚面势必会变得十分卡顿，相反transform 动画则会保持较高的 f ps 以及不会出现丢帧的情况，保证了⻚面的流畅度。

left:

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

transform:

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

##### 创建合成层

创建合成层的过程也相对简单，满足我们上面讲的创建合成层的原因即可。

```
#target {
  transform: translateZ(0);
}
#target {
  will-change: transform;
}
```

##### 合成层爆炸

而我们还在某些情况下，会因为操作不当，导致产生了过多的合成层，极大的消耗了⻚面的资源，而产生⻚面卡顿等问题，如下所示。

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Layer Explosion</title>
    <style>
      @-webkit-keyframes slide {
        from {
          transform: none;
        }
        to {
          transform: translateX(100px);
        }
      }
      .animating {
        width: 300px;
        height: 30px;
        background-color: orange;
        color: #fff;
        -webkit-animation: slide 5s alternate linear infinite;
      }

      ul {
        padding: 5px;
        border: 1px solid #000;
      }

      .box {
        width: 600px;
        height: 30px;
        margin-bottom: 5px;
        background-color: blue;
        color: #fff;
        position: relative;
        /* 会导致无法压缩：squashingClippingContainerMismatch */
        overflow: hidden;
      }

      .inner {
        position: absolute;
        top: 2px;
        left: 2px;
        font-size: 16px;
        line-height: 16px;
        padding: 2px;
        margin: 0;
        background-color: green;
      }
    </style>
  </head>
  <body>
    <div class="animating">composited animating</div>
    <ul id="list"></ul>
    <script>
      var template = function (i) {
        return [
          '<li class="box">',
          '<p class="inner">asume overlap, 因为 squashingClippingContainerMismatch 无法压缩</p>',
          "</li>",
        ].join("");
      };
      var size = 200;
      var html = "";
      for (var i = 0; i < size; i++) {
        html += template(i);
      }
      document.getElementById("list").innerHTML = html;
    </script>
  </body>
</html>
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

上面的各个 li 元素会因为假设重叠原因，被迫提升为合成层，而他们之间拥有不同的裁剪容器，所以不能进行层压缩，因此，每个 li 元素都产生了一个合成层。出现了层爆炸情况。。

所以我们必须打破层爆炸，第一种方法便是将动画元素的 z-index 提升，确认将其覆盖在 li 元素之上，这样各个 li 元素便不会因为假设重叠而导致提升为为合成层。

- 产生层叠上下文，将 z-index 属性提升：

```
.animating {
  
  ...
  /* 让其他元素不和合成层重叠 */
  position: relative;
  z-index: 1;
}
```

- 去除 squashingClippingContainerMismatch：

第二种方法便是将 overflow 去除，让各个 li 元素拥有相同的裁剪容器，这样满足层压缩的条件后，便可以避免出现层爆炸的情况。

```
.box {
        width: 600px;
        height: 30px;
        margin-bottom: 5px;
        background-color: blue;
        color: #fff;
        position: relative;
        /* 会导致无法压缩：squashingClippingContainerMismatch */
        overflow: hidden;
      }
```

### **浏览器层面**

在浏览器层面，也已经帮我们做了很大的优化，例如在 chrome94 及以上版本中，新的浏览器渲染算法已经被全量使用，与原先的渲染算法不同，新的算法会在绘制指令生成后，再进行分层，这样可以极大的降低因为假设重叠而被迫提升为合成层的概率，下图便是新的浏览器绘制算法流程：

```
from layout
  |
  v
+------------------------------+
| LayoutObject/PaintLayer tree |
+------------------------------+
  |     |
  |     | PrePaintTreeWalk::Walk()
  |     |   PaintPropertyTreeBuider::UpdatePropertiesForSelf()
  |     v
  |   +--------------------------------+
  |<--|         Property trees         |
  |   +--------------------------------+
  |                                  |
  | LocalFrameView::PaintTree()      |
  |   FramePainter::Paint()          |
  |     PaintLayerPainter::Paint()   |
  |       ObjectPainter::Paint()     |
  v                                  |
+---------------------------------+  |
| DisplayItemList/PaintChunk list |  |
+---------------------------------+  |
  |                                  |
  |<---------------------------------+
  | LocalFrameView::PushPaintArtifactToCompositor()
  |   PaintArtifactCompositor::Update()
  |
  +---+---------------------------------+
  |   v                                 |
  | +----------------------+            |
  | | Chunk list for layer |            |
  | +----------------------+            |
  |   |                                 |
  |   | PaintChunksToCcLayer::Convert() |
  v   v                                 v
+----------------+ +-----------------------+
| cc::Layer list | |   cc property trees   |
+----------------+ +-----------------------+
  |                  |
  +------------------+
  | to compositor
  v
```

上述的层爆炸例子在 chorme94 版本中的表现是这样的，li 元素之间共享了一个合成层。只能说非常感谢 google 爸爸。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

在 chrome94 版本中还有非常多的新功能，例如 SchedulerAPI，让开发者可以去控制任务的优先级，也就是在干 React 框架中， **调度模块** 在做的事情，详情可参考卡颂大佬的文章，chrome94 的发布日志如下所示：https://blog.chromium.org/2021/10/renderingng.html