---

layout: post
title: 【JS】知识点归纳
subtitle: 等待补充
date: 2022-10-03
author: "H4kur31"
header-img: "img/bg-touhou-7.jpg"
tags: 

- JS

---

> 1995年，JavaScript 问世。当时，它的主要用途是代替 Perl
> 等服务器端语言处理输入验证。

初学 JS，只要理解了 BOM, DOM, ajax, XML, JSON,
事件这些概念，就能很轻易地完成前端表单输入验证、动态刷新页面等功能。但如果尝试系统性地将
JS 作为一门编程语言学习，会发现 JS
的语言特性十分迥异，从变量、作用域，到面向对象、this指针、异步和回调......

这里面既有 JS
作为动态和弱类型语言的原因，也有需要向前兼容的原因。归纳一下红宝书的一些知识点：

变量

1. var
   关键词声明的变量作用域为[**函数作用域**]{style="color: #ff6600;"}，且会自动[**提升**]{style="color: #3366ff;"}到函数作用域顶部，let
   关键词生命的变量作用域为[**块作用域**]{style="color: #ff6600;"}，在
   let 声明之前的执行瞬间被称为"暂时性死区"（temporal dead
   zone），在此阶段引用任何后面才声明的变量都会抛出
   ReferenceError。由于声明提升，var 可以**冗余声明**，let 则不行。
2. 在全局作用域声明的 var 变量会成为 **windows 对象属性**，let
   变量则不会。不过，let
   声明依然是在全局作用域发生的，相应变量会在页面的生命周期内存续。**在函数内定义变量时，省略
   var 操作符，可以创建一个全局变量。**
3. 在不确定页面之前是否出现某个变量时，使用 var
   直接再次声明不会有问题。不能直接使用 let 声明（可能抛出
   SyntaxError，已经声明过了），但由于 let
   变量的作用域是块作用域，因此也无法使用**条件声明**。
4. for 循环中的 var 声明会渗透到循环体外部，let
   声明则不会。好的声明风格和最佳实践要求尽量使用 const 和 let，而非
   var。
5. const 行为与 let
   基本相同，唯一一个重要的区别是用它声明变量时必须同时初始化变量。
6. for (const i = 0; i \< 10; ++i) 会报错，但 for (const value of \[1,
   2, 3, 4, 5\]) 是正确可行的。
7. 虽然不能修改 const
   声明的变量，但如果变量引用一个对象，则能够修改对象的内部属性。
8. 对于 JS 和 Python
   而言，可以将变量理解为[**标签**]{style="color: #ff0000;"}。Python
   中变量的值保存在内存的"堆"中，而变量作为引用保存在"栈"中。JS 与
   Python 类似，区别在于 Python 中的一切皆为对象，而 JS
   还有原始值直接保存在"栈"中。对两者而言，**重新赋值都相当于重新在内存中开辟空间**。JS
   变量复制分为**原始值**和**引用值**两种情况，后者实际上复制的是指针，而
   Python 的复制就相当于后者。

数据类型

1. typeof 会返回变量的数据类型，对于 null 会返回 Object，因为逻辑上
   null 表示一个空对象指针。
2. 严格来讲，function 在 ECMAScript 中被认为是对象，但 typeof function
   结果为 function。
3. 声明但未定义、显示给设置 undefined 值、未声明的变量，通过 typeof
   都会返回
   undefined。前两者是等价的，但与未声明的变量存在根本性差异。建议永远不要显式地给某个变量设置
   undefined 值（函数传参除外），声明变量时总进行初始化，这样当 typeof
   返回\"undefined\"时，就知道该变量是未声明，而不是声明了但未初始化。
4. 不同类型与布尔值之间的转换规则：![](/img/2022-10-03-JS-Summarizing-Difficulties/1.png){loading="lazy"}
5. Number类型使用 IEEE 754
   格式表示整数和浮点数（计算浮点值时可能出现误差）。由于存储浮点值使用的内存是存储整数值的两倍，所以
   ECMAScript 总是想方设法把值转换为整数。（如：1.  10.0）
6. 最小值：Number.MIN_VALUE；最大值：Number.MAX_VALUE。超出范围则自动转化为
   Infinity 或者 -Infinity，且无法进一步用于任何计算。isFinite()
   函数可用于判断参数是否为有限大。使用 Number.POSITIVE_INFINITY 和
   Number.NEGATIVE_INFINITY 也可获取 Infinity 和 -Infinity
7. 0 除 0 返回 NaN，0 除非 0 值返回 Infinity 或 -Infinity
8. 三种数值转换函数：Number()、parseInt()、parseFloat()，后两者主要用于字符串转数值
9. 字符串可以使用双引号、单引号或反引号，三者对字符串解释方式没有区别（不像PHP双引号和单引号之间），不过反引号能够定义**模板字面量**。
10. JS
    中的字符串是不可变的（实际上Number/Boolean等原始值都是不可变的，因为
    JS 不允许直接修改内存）
