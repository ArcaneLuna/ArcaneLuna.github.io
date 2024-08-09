---

layout: post
title: 【JS】Primitive Type是值类型吗？存储在栈上吗？
subtitle: 应用层语言不能按照系统编程语言理解
date: 2022-10-01
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- JS

---

## 0x01 Immutable

在讨论 Primitive Type（原始类型）是否为值类型和存储在栈上前，先要理解JS原始类型的一个特殊性质：immutable

《JavaScript 高级程序设计》中有一段对字符串的描述：

> ECMAScript中的字符串是不可变的（immutable）

同时附带了一个简单的例子：

```{.highlighter-hljs
let lang = "Java";
lang = lang + "Script";
```

这其中的过程是：变量 lang
一开始包含字符串\"Java\"，然后**重新分配**一个容纳 10
个字符的空间，填充上\"Java\"和\"Script\"，最后**销毁**字符串\"Java\"和\"Script\"。

实际上不止 String，Undefined、Null、Boolean、Number、String 和 Symbol
这些直接存储在[**栈**]{style="color: #ff6600;"}（姑且按照书上这么说，后面展开讨论）上的**[原始值]{style="color: #ff6600;"}**（primitive
value）同样是**不可变**的（《JavaScript权威指南》："原始值是不可更改的"），这里的"不可变"指的并非是变量，而是作为[**字面量**]{style="color: #ff0000;"}的\"Java\"\"Script\"和\"JavaScript\"。所以对原始值重新赋值会分配新的地址，而不像
c/c++ 在原地址上重写。

再举一个  Number 类型的例子：

```{.highlighter-hljs
let num = 11;
num++;
```

[num
在执行自增操作后，分配了一个**新的地址。**]{style="background-color: #ffff00; color: #ff0000;"}

## [0x02 Value Type & Reference Type]{style="color: #000000; background-color: #ffffff;"}

维百上关于值类型和引用类型的定义：

> A value is a fully self-describing piece of data. Equal values are
> indistinguishable at program runtime. That is, they [**lack
> identity**]{style="color: #3366ff;"}.
> 
> Reference types are represented as a reference to another value, which
> may itself be either a value or reference type. Reference types are
> often implemented using **pointers**, though many high-level
> programming languages such as Java and Python do not expose these
> pointers to the programmer.

在复制时更能体现这种区别，对a=b，如果是值类型的复制，a和b相互独立，如果是引用类型的复制，a或b修改后会相互影响。

![](/img/2022-10-01-JS-Are-Primitive-Types-Value-Types-and-Stack-Stored/1.png){width="828"
height="351"}

正如图上划分的那样，JS原始值（primitive
value）表现出值类型的特点，而引用值（reference
valule）表现出引用类型的特点：

```{.highlighter-hljs
let num1 = 5;
let num2 = num1;  //num2++不会影响到num1

let obj1 = new Object();
let obj2 = obj1;
obj1.name = "Nicholas";
console.log(obj2.name);  //"Nicholas"
```

尽管对外表现出的一致，但严格意义上讲，出于优化考虑，**部分**JS引擎是通过[**引用类型**[+]{style="color: #000000;"}]{style="color: #ff6600;"}[**字面量的不可变性**]{style="color: #3366ff;"}来实现值类型的特点。

<div>

- V8 中只有小整数（smi）为值类型，undefined、null、boolean
  是特殊的引用类型，一般的 number 也是引用类型。
- QuickJS 中 undefined、null、boolean、number
  是值类型，其余都为引用类型。
- SpiderMonkey 和 JavaScriptCore 也这样做，但具体实现不同。

</div>

（在0x04中我们对于V8中的实现进一步讨论）

[变量都是对不可变字面量（11）的引用，当变量改变时，就指向了另一个不可变的字面量（12），如此一来，两个存储相同值的变量不会相互影响[（这样当我们重复定义长字符串时能够节省很多的空间）]{style="color: #000000; background-color: #ffffff;"}]{style="color: #ff0000; background-color: #ffff00;"}

这里引用他人的话先下一个结论：

> 具体到 V8 引擎，**String
> 类型是以接近引用类型的方式实现的**。一般而言，有性能追求的引擎会用某种
> variant 或 fancy pointer
> 实现。这样让特殊值「undefined」、「null」和小整数（例如 V8 的
> Smi），甚至是一般 Number 类型值或短的 String
> 类型值等足够小的对象，能存活在调用栈上或嵌入到闭包/对象实例中。效果上类似于
> escape analysis 优化。**由此观之，初略地认为 JavaScript
> 变量/字段的类型全部都是引用类型是没有问题的**。当然，这些都属于实现的细节。JavaScript
> 规范并未规定必须如何。**你也可以认为 Primitive 类型全是值类型**，只是
> JavaScript
> 性质使然，引擎必定启用某种优化，又允许将栈上数据挪到堆上，因此表现出了部分引用类型的性质。

这个结论可能会与红宝书中的说法产生矛盾：

> 引用值（或者对象）是某个特定引用类型的实例。(P103)
> 
> 在很多语言中，字符串都是使用对象表示的，因此被认为是引用类型。ECMAScript
> 打破了这个惯例。(P83)

然而按照我们上面的结论，如果 String 类型是引用类型，那么 String
到底是不是对象？

这里最好把 **JS
语境**下的引用类型和其他语言的引用类型（*之前我们一直讨论的是广义的引用类型*）区分开来，前者与之对应的是
[**Primitive Type**]{style="color: #ff6600;"}，后者与之对应的是 [**Value
Type**]{style="color: #3366ff;"}。

[String 类型在实现上是 Reference Type，但表现出 Value Type 的性质，在 JS
中我们不把它当作 Reference Type，而是 Primitive
Type。]{style="color: #ff0000; background-color: #ffff00;"}

[*对象是某个引用类型的实例 *]{style="color: #808080;"}这话在 JS
语境下虽然没错（因为 JS 只有原始类型和引用类型），但最好永远只把它当作
JS 语境下的。

**对象**应该理解为**数据和功能的集合**，根据是否具有**属性**和**方法**来判断一个变量是不是对象要更准确。所以，[**JS
中的 String 不是对象**]{style="color: #339966;"}。

接受这个概念更容易理解"[**原始值包装类型**]{style="color: #ff6600;"}"：每当用到某个原始值的**方法**或**属性**时，后台都会创建一个相应原始包装类型的**对象**，从而暴露出操作原始值的各种方法。这个自动创建的原始值包装对象只存在于访问它的那行代码执行期间。

## 0x03 执行上下文栈

> JS的原始值存储在栈中，但V8的默认栈区为984KiB，为什么可以定义一个500MiB的字符串？

根据之前的分析我们可以说：JS引擎（比如V8）会在堆存储着字符串，栈上存储着指向字符串的指针。

需要明确：

> 栈和堆的分配是指 C 或 C++ 编译的程序
> 
> 通常有这几部分组成：
> 
>   1、栈区（stack） 由编译器自动分配释放
> ，存放函数的参数值，局部变量的值等
> 
>   2、堆区（heap）一般由程序员分配释放，使用 malloc 或 new 等
> 
>   ...
> 
> 但是由于JS脚本引擎是一种由 C 或 C++ 开发的"应用"
> ，而且这种脚本"应用"并不再经过 C/C++
> 编译器编译，所以这种"应用"内变量所处位置并不好说，因为这些变量可能只是
> C 或 C++ 内结构体或者某种Script类型实例后结果。
> 
> 实际 Native
> 的实现肯定是某种具体类对象，[**那么它们肯定会在堆内**]{style="color: #ff6600;"}。而由于要使用堆内对应的值，**栈区就会有对应的对内值地址，此时栈区存储的是指针**，其大小是固定的，可以被放置在有限的栈空间内。

JS
语境下的栈应该指的是**执行上下文栈**（在更底层的函数调用栈中可能只有到执行栈中变量的指针/引用，讨论这个层次没有意义）。

每个上下文都有一个关联的变量对象 ( variable object
)，而这个上下文中定义的所有变量和函数都存在于这个对象上。

## 0x04 V8 引擎实现细节

V8官方指出：

> V8 中的 JavaScript **值**表示为对象并在 V8
> 堆上分配，无论它们是对象、数组、数字还是字符串。这允许我们将任何值表示为指向对象的**指针**。

JS不提供对地址的直接访问，所以我们很难直观感受到一个 Primitive
类型是如何进行存储的。

浏览器提供的调试工具查看堆快照（heap snapshot），能够看到一个 JavaScript
引擎（虚拟机）中的等价地址。

[JavaScript中变量存储在堆中还是栈中？ - 知乎
(zhihu.com)](https://www.zhihu.com/question/482433315/answer/2083349992) 本文很大程度上参考了这篇回答，回答的大佬查阅了
V8 的源代码并进行了一些小的测试实验，这里不再重复。

现象是两个函数对象中的字符串属性都是对某一字面量的**[引用]{style="color: #ff6600;"}。**

结论：

> <div>
> 
> [**字符串并没有存到栈中**]{style="color: #3366ff;"}，而是存到了一个别的地方，再把这个地方的地址存到了栈中。
> 
> </div>
> 
> 1. v8内部有一个名为stringTable的hashmap缓存了所有字符串，在V8阅读我们的代码，转换抽象语法树时，每遇到一个字符串，会根据其特征换算为一个hash值，插入到hashmap中。在之后如果遇到了hash值一致的字符串，会优先从里面取出来进行比对，一致的话就不会生成新字符串类。
> 2. 缓存字符串时，根据字符串不同采取不同hash方式。
> 
> <div>
> 
> <div>
> 
> ......在我们创建字符串的时候，V8会先从内存中（哈希表）查找是否有已经创建的完全一致的字符串，如果存在，直接[**复用**]{style="color: #ff6600;"}。如果不存在，则开辟一块新的内存空间存进这个字符串，然后把地址赋到变量中。这也是为什么我们不能直接用下标的方式修改字符串:
> V8中的字符串都是不可变的。
> 
> </div>
> 
> </div>

这符合我们最开始关于Primitive
类型不可变的看法。（不过拼接字符串后的结果可能不太一样，暂不讨论）

之前我们提到 V8 中有一个值类型的特例：smi（小整数）。

实际上这是V8的一种优化，使用指针标记（pointer
tagger）技术，最低位为0时，会直接存储 smi 而不是指针。

## 0x05 总结 & 参考

最开始只是好奇为什么原始值为什么总要重新分配地址而不是在地址上更改（这大概和底层语言使用
const 有关？），不曾想查阅资料发现 V8 中的值竟然都存储在堆而不是栈。

这段时间被执行上下文、Primitive Type、Reference Type、Value Type、Object
这些概念折磨的够呛，以后尽量把 Engine 对 JS 的实现当成黑箱。

学习过程中第一次尝试用浏览器调试 JS：f12-\>源代码-\>片段-\>新片段

【参考】

- [在JavaScript中存在值类型（Value Type）吗？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/451383686/answer/1803004448)
- [JavaScript中变量存储在堆中还是栈中？ - 六耳的回答 - 知乎
  (zhihu.com)](https://www.zhihu.com/question/482433315/answer/2083349992)
- [JavaScript中变量存储在堆中还是栈中？ - 李杭帆的回答 - 知乎
  (zhihu.com)](https://www.zhihu.com/question/482433315/answer/2157491981)
- [Value type and reference type -
  Wikipedia](https://en.wikipedia.org/wiki/Value_type_and_reference_type#cite_note-9)
- [ECMAScript® 2023 Language Specification
  (tc39.es)](https://tc39.es/ecma262/#sec-execution-contexts)
- [JavaScript中变量存储在堆中还是栈中？- 貘吃馍香的回答 - 知乎
  (zhihu.com)](https://www.zhihu.com/question/482433315/answer/2110323193)
- [Javascript中堆栈到底是怎样划分的？- 貘吃馍香的回答 - 知乎
  (zhihu.com)](https://www.zhihu.com/question/42231657/answer/102552732)
- [javascript - 请问JavaScript
  里面怎么获取某个变量的内存地址？并打印出来 - SegmentFault
  思否](https://segmentfault.com/q/1010000015236330)
- [如何理解js中基本数据类型的值不可变_WinstonLau的博客-CSDN博客_js
  基本类型的值都是不可改变的](https://blog.csdn.net/WinstonLau/article/details/88754704) （虽然但是。。。这篇文章用的是Java对栈中数据的共享来解释JS原始值不可变）

【劝退指南】

- [JavaScript的学习难点在哪？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/34262414/answer/169999132)
- [js中0=='0'、0==\[\]为true 为什么'0'==\[\]为false？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/430070261)

 
