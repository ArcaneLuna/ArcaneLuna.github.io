---

layout: post
title: 【CSS】基础布局
date: 2022-09-30
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Encoding

---

## 0. 前言

最开始接触CSS时，没太注意文档布局这一块，学会用选择器后就迫不及待跑去美化文本、表格、表单等内容了。

等到需要从零开始构建一个网页时才发觉布局不仅很重要，而且由于涉及到浏览器内部渲染文档的原理，理解起来也不是那么容易。

另外，CSS实现同一种页面布局效果能有很多种不同的方法，选择合适的方法也能方便以后修改和适应不同设备和分辨率。

故对CSS布局相关的知识进行了梳理，对于 document flow、block container
box（块容器盒/块框）、BFC的知识点加入了自己的理解。如有错误，敬请指正。

## 1. 盒模型

### 1.1 概述

当对一个文档进行[**布局（lay
out）**]{style="color: #ff6600;"}的时候，浏览器的渲染引擎会根据标准之一的[
**CSS 基础框盒模型（CSS basic box
model）**]{style="color: #3366ff;"}，将所有元素表示为一个个矩形的盒子（box）。

盒模型由内到外由四个部分组成：content area, padding area, border area,
margin area

![](/img/2022-09-30-CSS-layout-basis/1.jpg){width="486"
height="304"}

- **内容区域 content area** ，由内容边界 (content edge)
  限制，容纳着**[元素的"真实"内容]{style="color: #339966;"}**，例如文本、图像，或是一个视频播放器。它的尺寸为
  content-box width 和 content-box height。如果 box-sizing 为
  [content-box]{style="background-color: #ffff00; color: #ff0000;"}（默认），则内容区域的大小可明确地通过
  width、min-width、max-width、height、min-height，和 max-height
  控制。
- **内边距区域 padding
  area**，由内边距边界限制，扩展自内容区域，负责延伸内容区域的背景，填充元素中内容与边框的间距。它的尺寸是
  padding-box width 和 padding-box height。内边距的粗细可以由
  padding-top、padding-right、padding-bottom、padding-left，和简写属性
  padding 控制。
- **边框区域 border area**
  由边框边界限制，扩展自内边距区域，是容纳边框的区域。其尺寸为
  border-box width 和 border-box height。**边框的粗细由 border-width
  和简写的 border 属性控制。**如果 box-sizing 属性被设为
  [border-box]{style="background-color: #ffff00; color: #ff0000;"}，那么边框区域的大小可明确地通过
  width、min-width, max-width、height、min-height，和 max-height
  属性控制。假如框盒上设有背景（background-color 或
  background-image），背景将会一直延伸至边框的外沿（默认为在边框下层延伸，边框会盖在背景上）。此默认表现可通过
  CSS 属性 background-clip 来改变。
- **外边距区域 margin area**
  由外边距边界限制，用空白区域扩展边框区域，以分开相邻的元素。它的尺寸为
  margin-box 宽度 和 margin-box 高度。外边距区域的大小由
  margin-top、margin-right、margin-bottom、margin-left，和简写属性
  margin
  控制。在发生外边距合并的情况下，由于盒之间共享外边距，外边距不容易弄清楚。

【width、height与min-width、max-width、min-height、max-height冲突时，以后者为准】

【行内可替换元素可通过width、height定义占用空间，**普通行内元素的width和height无效，只能通过line-height定义占用空间**】

### 1.2 Margin Collapse

- [深入理解CSS外边距折叠（Margin Collapse）
  (youzan.com)](https://tech.youzan.com/css-margin-collapse/#:~:text=%E4%BB%80%E4%B9%88%E6%98%AF%E5%A4%96%E8%BE%B9%E8%B7%9D%E5%8F%A0%E5%8A%A0%20%E5%85%88%E6%9D%A5%E7%9C%8B%E7%9C%8B%20W3C%20%E5%AF%B9%E4%BA%8E%E5%A4%96%E8%BE%B9%E8%B7%9D%E5%8F%A0%E5%8A%A0%E7%9A%84%E5%AE%9A%E4%B9%89%20%EF%BC%9A%20In%20CSS%2C%20the,resulting%20combined%20margin%20is%20called%20a%20collapsed%20margin.)
- [外边距重叠 - CSS（层叠样式表） \| MDN
  (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Box_Model/Mastering_margin_collapsing)

### 1.3 box-sizing

为了兼顾IE怪异模式，CSS3对盒模型进行了改善，定义了box-sizing属性，该属性能够**[事先定义盒模型的尺寸解析方式]{style="color: #ff6600;"}**。

> box-sizing:content-box \| border-box \| inherit;

- [理解标准盒模型和怪异模式&box-sizing属性 - 腾讯云开发者社区-腾讯云
  (tencent.com)](https://cloud.tencent.com/developer/article/1171658#:~:text=CSS3%E7%9A%84box-sizing%E5%B1%9E%E6%80%A7%201%20%E5%BD%93%E4%B8%BAcontent-box%E6%97%B6%EF%BC%8C%E5%B0%86%E9%87%87%E5%8F%96%E6%A0%87%E5%87%86%E6%A8%A1%E5%BC%8F%E8%BF%9B%E8%A1%8C%E8%A7%A3%E6%9E%90%E8%AE%A1%E7%AE%97,2%20%E5%BD%93%E4%B8%BAborder-box%E6%97%B6%EF%BC%8C%E5%B0%86%E9%87%87%E5%8F%96%E6%80%AA%E5%BC%82%E6%A8%A1%E5%BC%8F%E8%A7%A3%E6%9E%90%E8%AE%A1%E7%AE%97%203%20%E5%BD%93%E4%B8%BAinherit%E6%97%B6%EF%BC%8C%E5%B0%86%E4%BB%8E%E7%88%B6%E5%85%83%E7%B4%A0%E6%9D%A5%E7%BB%A7%E6%89%BFbox-sizing%E5%B1%9E%E6%80%A7%E7%9A%84%E5%80%BC)
- [CSS盒模型（重点） - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/145153467?from_voters_page=true)

## 2. 显示类型

### 2.1 块级盒子（Block box）和内联盒子（Inline box）

在常规网页设计中，CSS把标签分为两种基本显示形态：Block（块状）和
Inline（行内）。这两种盒子会在**页面流（page
flow）**和元素之间的关系方面表现出不同的行为。

 block box特点：

- 默认100%宽度，即占用父容器在内联方向上的所有可用空间
- width、height有效
- 产生换行（即使并没有占用一整行）
- padding、margin、border会应用且会把其他元素从当前盒子周围推开

 inline box特点：

- 不产生换行
- width、height无效（行内替换元素除外）
- 垂直方向的padding、margin以及border会被应用但是不会把其他处于 inline
  状态的盒子推开
- 水平方向的padding、margin以及border会被应用且会把其他处于 inline
  状态的盒子推开

### 2.2 常规流/普通流（normal flow）

上述的block和inline都是指的盒子的[**外部显示类型**]{style="background-color: #ffffff; color: #3366ff;"}。同时盒模型还有**[内部显示类型]{style="color: #008000;"}**，它决定了盒子内部是如何布局的。默认情况下（内部）是按照 [**normal
flow **]{style="color: #ff6600;"}布局。

normal flow
常被称作**常规流**、**普通流**、[*正常文档流、文档流*]{style="text-decoration: line-through;"}。

CSS2.1的英文文档中本身没有 document flow（文档流）的概念，只有作为
positioning schemes 的 normal flow。有时会在一些英文文章中看到 normal
document flow 的说法，实际上说的就是 normal
flow。而"文档流"这个翻译针对的就是 normal document
flow。所以如果面试官说"脱离文档流"，指的就是"脱离常规流"。

以上概念只要理解了一般不会产生误解，但无意间看到篇 blog 写的很具迷惑性：

[What Is Document Flow?
(soulandwolf.com.au)](https://soulandwolf.com.au/blog/what-is-document-flow/)[ [其中对
documnet flow
做了一个**非官方**的总结：]{style="background-color: #ffff00;"}]{style="color: #ff0000;"}

> 文档流是页面元素的排列（arrangement），由 CSS 定位语句（CSS
> positioning statements）和 HTML
> 元素的顺序（order）定义。也就是说，每个元素如何占用空间以及其他元素如何相应地定位自己。 也就是说，每个元素如何占用空间以及其他元素如何相应地定位自己。

一些国内的翻译（[什么是文档流？ - 掘金
(juejin.cn)](https://juejin.cn/post/7069342426875297806#heading-4)）更是据此总结道：

> **文档流一共分为4种**：
> 
> 1. 正常文档流 normal document flow
> 2. 显示类型 display type
> 3. 浮动框 float
> 4. 定位 position

[*原来除了 normal document flow，其他几种都算 abnormal document flow
的吗？*]{style="text-decoration: line-through;"}

以上说法固然不能算错（毕竟没有官方定义的document
flow），但却使得理解变得更加困难：**如果把页面元素的排列统统归为
document flow，又该怎样理解"脱离文档流"的定位元素**：

是只脱离了 normal flow 但仍属于document flow*[（所谓的 abnormal document
flow）]{style="text-decoration: line-through;"}*？还是彻底脱离了
document flow？

官方文档（[Visual formatting model
(w3.org)](https://www.w3.org/TR/CSS21/visuren.html#positioning-scheme)）的定义：

> Absolutely positioned boxes are taken out of the **normal flow**.

答案自然是前者，看得出 document flow 的概念在这里显得有些多余。*\
*

### 2.3 定位元素脱离 normal flow

与浏览器的渲染规则有关：normal flow 中非 positoned element
元素，总是先于 positioned element 元素渲染，所以表现就是在 positioned
element 下方，跟它在HTML中出现的顺序无关。而 positioned element 又会根据
stacking context、z-index、back-to-front 等规则进行堆叠

- [CSS定位机制之一：普通流
  (swordair.com)](https://swordair.com/css-positioning-schemes-normal-flow/)

### 2.4 内部和外部显示类型

> 在CSS中，可以使用display属性来改变元素的显示类型。在CSS2.1中，display属性共有18个选项值。
> 
> **常用显示类型都可以划归为block和inline两种基本形态，[其他类型都是这两种类型的特殊显示]{style="color: #ff6600;"}。**其中真正能够应用并获得所有浏览器支持的取值只有4个：block、none、inline、list-item。

有些国内工具书上有以上两段内容（比如：《CSS商业网站布局之道》《HTML5+CSS3+JavaScript从入门到精通》），但其中有些错误。

[Visual formatting model
(w3.org)](https://www.w3.org/TR/2011/REC-CSS2-20110607/visuren.html#display-prop)，在CSS2.1中，display属性只有16个属性值：（删除了CSS2中的
compact 和 marker 选项）

> value: 　inline \| block \| list-item \| inline-block \| table \|
> inline-table \| table-row-group \| table-header-group \|
> table-footer-group \| table-row \| table-column-group \| table-column
> \| table-cell \| table-caption \| none \| inherit

　　[CSS3](https://www.w3.org/TR/css-display-3/#the-display-properties){target="_blank"} 规范了display属性的取值：

> value: \[ \<display-outside\> \|\| \<display-inside\> \] \|
> \<display-listitem\> \| \<display-internal\> \| \<display-box\> \|
> \<display-legacy\>

给一个元素设置display后，将会决定这个盒子的两个基础显示类型：

（1）outer display
type：决定该元素本身如何布局的，即参与何种[格式化上下文]{style="color: #ff0000; background-color: #ffff00;"}

（2）inner display
type：将该元素当成**容器**，规定了其内部子元素如何布局，参与何种格式化上下文。container
box 类型依据 display 不同，分为四种：

【注：这段我参考了[1.5 万字 CSS 基础拾遗（核心知识、常见需求） - 掘金
(juejin.cn)](https://juejin.cn/post/6941206439624966152)，这里的说法不够准确，首先
inner display type 不一定会生成 container box，也可能生成一个 inline
box，其次 block container box 也不一定会建立BFC】

【[CSS Display Module Level 3
(w3.org)](https://www.w3.org/TR/css-display-3/#the-display-properties) \<short
\'display\', full \'display\', generated box\>那张表可以了解一下】

- block container box: *[建立 BFC 或
  IFC]{style="text-decoration: line-through;"}*
- flex container box：建立 FFC
- grid container box：建立GFC
- ruby container box：建立 [ruby formatting
  context](https://www.w3.org/TR/css-ruby-1/#ruby-formatting-context){#ref-for-ruby-formatting-context
  link-type="dfn"}

注：img这种替换元素（replaced element）声明为block不会产生container
box，因为设计初衷就没有考虑将它作为容器。

- [CSS 基础框盒模型介绍 - CSS（层叠样式表） \| MDN
  (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Box_Model/Introduction_to_the_CSS_box_model)
- [常规流中的块和内联布局 - CSS（层叠样式表） \| MDN
  (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flow_Layout/Block_and_Inline_Layout_in_Normal_Flow#%E5%9D%97%E6%A0%BC%E5%BC%8F%E5%8C%BA%E5%9F%9F%E4%B8%AD%E7%9A%84%E5%85%83%E7%B4%A0)
- [盒模型 - 学习 Web 开发 \| MDN
  (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Learn/CSS/Building_blocks/The_box_model#%E5%9D%97%E7%BA%A7%E7%9B%92%E5%AD%90%EF%BC%88block_box%EF%BC%89%E5%92%8C_%E5%86%85%E8%81%94%E7%9B%92%E5%AD%90%EF%BC%88inline_box%EF%BC%89)
- [CSS属性参考 \|
  height_jQuery之家-自由分享jQuery、html5、css3的插件库
  (htmleaf.com)](http://www.htmleaf.com/ziliaoku/qianduanjiaocheng/height.html)

### 2.5  关于 block-level box 与 block container box

[Visual formatting model
(w3.org)](https://www.w3.org/TR/2011/REC-CSS2-20110607/visuren.html#block-formatting) 明确：

> Except for table boxes, which are described in a later chapter, and
> replaced elements,[ **a block-level box is also a block container
> box**.]{style="color: #ff0000;"} A block container box either contains
> only block-level boxes or establishes an inline formatting context and
> thus contains only inline-level boxes. **[Not all block container
> boxes are block-level boxes]{style="color: #ff0000;"}: non-replaced
> inline blocks and non-replaced table cells are block containers but
> not block-level boxes.** Block-level boxes that are also block
> containers are called block boxes.

即除了 table box 和可替换元素，block-level box 同时也是 block container
box，但反过来，block container box 不一定是 block-level box，比如
inline-block 和 table-cell。

注意到，block container box 中只包含 block-level
boxes（只包含block-level boxes 并不表示就是 BFC）或者 IFC（二选一）。

### 2.6 匿名块框

block container box
内部同时书写块级元素和内联元素的情况很常见，为什么会说 block container
box 只包含 block-level boxes 或者 IFC 呢？

来看下面这种情况：

    <DIV>
        Some text
        <p>More text
    </DIV>

 默认\<div\>和\<p\>均为block。这时在 Some text
外会生成一个匿名块框（anonymous block box）。

![](/img/2022-09-30-CSS-layout-basis/2.png)

CSS2.1规定：[**如果一个block container
box（比如上面为DIV生成的）里面有一个 block-level
box（比如前面的P），那么我们[强制]{style="color: #ff6600;"}它里面只有
block-level box。**]{style="color: #3366ff;"}

当内联框中存在 in-flow block-level box
时，将会围绕块框断开，前后两部分分别被包含在匿名块框中：

    <P>
        This is anonymous text before the SPAN.
        <SPAN>This is the content of SPAN.</SPAN>
        This is anonymous text after the SPAN.  
    </P>

注意：如果上面DIV中的匿名块框的子级需要知道其包含块的高度以解析百分比高度，那么它将使用DIV形成的包含块的高，而不是匿名块框。

### [2.7 匿名内联框]{style="text-decoration: line-through;"}

> Any text that is directly contained inside a block container element
> (not inside an inline element) must be treated as an **[anonymous
> inline element]{style="color: #ff0000;"}**.

文本直接包含在块框元素（注意不是块级元素）时被作为匿名内联元素处理。

    <p>Some <em>emphasized</em> text</p>

这里＜p＞生成一个块框，其中有几个内联框。"emphasized"框是由内联元素（＜em＞）生成的内联框，但其他框（"Some
"和"
text"）是由块级元素（＜p＞）生成。后者被称为匿名内联框，因为它们没有关联的内联级别元素。

回到匿名块框的第一个例子，如果块框元素同时包含了文本和块级元素，优先生成匿名块框而不是匿名内联框，更准确地说，是[**具有内联格式上下文（IFC）的匿名块框**]{style="color: #ff6600;"}（[css -
Identifying Anonymous Block Boxes - Stack
Overflow](https://stackoverflow.com/questions/52981136/identifying-anonymous-block-boxes)），"Some
text"在匿名块框中表现为匿名内联元素。

**值得一提，CSS不止这两种匿名框，在格式化table时会产生更多的匿名框。**

### 2.8 更多

[CSS Display Module Level 3
(w3.org)](https://www.w3.org/TR/css-display-3/#:~:text=The%20display%20property%20defines%20an%20element%E2%80%99s%20display%20type,the%20principal%20box%20itself%20participates%20in%20flow%20layout.) 对
block container box 的概念进行了明确：

> A block container either contains only inline-level boxes
> participating in an [inline formatting
> context](https://www.w3.org/TR/css-display-3/#inline-formatting-context){#ref-for-inline-formatting-context①
> link-type="dfn"}, or contains only block-level boxes participating in
> a [block formatting
> context](https://www.w3.org/TR/css-display-3/#block-formatting-context){#ref-for-block-formatting-context⑦
> link-type="dfn"} (possibly generating anonymous block boxes to ensure
> this constraint, as defined
> in [CSS2§9.2.1.1](https://www.w3.org/TR/CSS2/visuren.html#anonymous-block-level)).
> 
> A block container that contains only inline-level content establishes
> a new [inline formatting
> context](https://www.w3.org/TR/css-display-3/#inline-formatting-context){#ref-for-inline-formatting-context②
> link-type="dfn"}. The element then also generates a [root inline
> box](https://www.w3.org/TR/css-inline-3/#root-inline-box){#ref-for-root-inline-box
> link-type="dfn"} which wraps all of its inline content. [Note,
> this [root inline
> box](https://www.w3.org/TR/css-inline-3/#root-inline-box){#ref-for-root-inline-box①
> link-type="dfn"} concept effectively replaces the \"anonymous inline
> element\" concept introduced
> in [CSS2§9.2.2.1](https://www.w3.org/TR/CSS2/visuren.html#anonymous).]{.note}
> 
> A block container establishes a new [block formatting
> context](https://www.w3.org/TR/css-display-3/#block-formatting-context){#ref-for-block-formatting-context⑧
> link-type="dfn"} if its parent formatting context is *not* a [block
> formatting context; otherwise, when participating in a [block
> formatting context itself, it either establishes a new [block
> formatting context for its contents or continues the one in which it
> participates, as determined by the constraints of other properties
> (such
> as [overflow](https://www.w3.org/TR/CSS2/visufx.html#propdef-overflow){#ref-for-propdef-overflow①
> .property .css
> link-type="property"} or [align-content](https://www.w3.org/TR/css-align-3/#propdef-align-content){#ref-for-propdef-align-content
> .property .css
> link-type="property"}).]{#ref-for-block-formatting-context①①}]{#ref-for-block-formatting-context①⓪}]{#ref-for-block-formatting-context⑨}

与之前的标准变化不大，block container box
要么只包含参与内联格式上下文的内联框，要么只包含参与块格式化上下文的块框（需要生成匿名块框）。但
**[root inline box
的概念有效取代了匿名内联元素的概念]{style="color: #ff6600;"}**。（想想也确实，只产生一个root
inline box，比产生多个匿名内联框要好很多）

措辞很严谨，当block container
box只包含内联内容时，会**[生成（generate）]{style="color: #339966;"}**新的IFC，但想要生成新的BFC要考察两点：

1. 父格式化上下文不是BFC，直接生成BFC
2. 父格式化上下文是BFC，由溢出和对齐等属性决定是否生成BFC，否则继续其参与的BFC

## 3. 格式化上下文

[常规流中的块和内联布局 - CSS（层叠样式表） \| MDN
(mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flow_Layout/Block_and_Inline_Layout_in_Normal_Flow#the_display_property_and_flow_layout%E6%98%BE%E7%A4%BA%E5%B1%9E%E6%80%A7%E5%92%8C%E6%B5%81%E5%B8%83%E5%B1%80%E7%BC%96%E8%BE%91)：

> [常规流中的任何一个盒子总是某个***格式区域*（*formatting
> context*）**中的一部分。]{style="color: #ff0000;"}块级盒子是在*块格式区域*（*block
> formatting
> context*）中工作的盒子，而内联盒子是在*内联格式区域*（*inline
> formatting
> context*）中工作的盒子。任何一个盒子总是块级盒子或内联盒子中的一种。

可以说格式化上下文是CSS布局的基础，只有在格式化上下文中的盒子才会产生布局。

### 3.1 BFC

BFC的本质是一片独立的渲染区域，隔离了外部（可理解为作用域）。

最初接触到BFC的概念时，一个令人困惑的问题是：BFC究竟是由内部相邻的许多盒子生成还是一个具有特殊属性的外部盒子指定？

回顾关于 inner display type 和匿名块框的说法，**block container
box**生成新的BFC**需要根据【①内部盒子类型；②父格式化上下文；③溢出对齐等属性】决定**。

再看看MDN的解释：

> 除了文档的根元素 (\<html\>) 之外，还将在以下情况下创建一个新的 BFC：
> 
> - **使用float 使其浮动的元素**
> - **绝对定位的元素 (包含 position: fixed 或position: sticky**
> - **使用以下属性的元素 display: inline-block**
> - 表格单元格或使用 display: table-cell, 包括使用 display: table-\*
>   属性的所有表格单元格
> - 表格标题或使用 display: table-caption 的元素
> - **块级元素的 overflow 属性不为 visible**
> - 元素属性为 display: flow-root 或 display: flow-root list-item
> - 元素属性为 contain: layout, content, 或 strict
> - flex items
> - 网格布局元素
> - multicol containers
> - 元素属性 column-span 设置为 all

**[默认所有的块级盒子都属于\<html\>产生的BFC，然后记住常用创建BFC的方法，这比试图去理解机制要容易得多。]{style="color: #3366ff;"}**

应用场景：

- 自适应两栏布局：BFC
  的区域不会和浮动区域重叠，所以就可以把侧边栏固定宽度且左浮动，而对右侧内容触发
  BFC，使得它的宽度自适应该行剩余宽度。
- 清除内部浮动：浮动造成的问题就是父元素高度坍塌，所以清除浮动需要解决的问题就是让父元素的高度恢复正常。而用
  BFC 清除浮动的原理就是：计算 BFC
  的高度时，浮动元素也参与计算。只要触发父元素的 BFC 即可。
- 防止垂直 margin 合并：同一个 BFC 下的垂直 margin
  会发生合并。所以如果让 2 个元素不在同一个 BFC 中即可阻止垂直 margin
  合并。那如何让 2 个相邻的兄弟元素不在同一个 BFC
  中呢？可以给其中一个元素外面包裹一层，然后触发其包裹层的
  BFC，这样一来 2 个元素就不会在同一个 BFC 中了。

### 3.2 IFC

IFC存在于其他的格式化上下文中，即当 block container box
中仅包含内联级别元素时，会产生IFC。

应用场景：

<div>

<div>

- 水平居中：当一个块要在环境中水平居中时，设置其为 inline-block
  则会在外层产生 IFC，通过 text-align 则可以使其水平居中。
- 垂直居中：创建一个 IFC，用其中一个元素撑开父元素的高度，然后设置其
  vertical-align: middle，其他行内元素则可以在此父元素下垂直居中。

</div>

</div>

 

 
