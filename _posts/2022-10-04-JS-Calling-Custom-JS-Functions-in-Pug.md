---

layout: post
title: 【JS】Pug调用自定义函数
date: 2022-10-04
author: "H4kur31"
header-img: "img/bg-touhou-7.jpg"
tags: 

- JS

---

当我用 node.js 从数据库中查询 datetime
类型的日期字段，并输出到网页上时，发现JS自动进行了类型转换：

由 2023-01-07 21:47:00 变成了 Sat Jan 07 2023 21:47:00 GMT+0800
(中国标准时间)

我希望简单保持 ISO 格式的日期输出：

    function formatDate(date) {
        var d = new Date(date);
        return d.toISOString().slice(0, 19).replace('T', ' ');
      }

这种转换既能在前端也能在后端处理，由于我没有根据用户浏览器来输出当地时间的需求，因此选择在后端处理。

在 Pug
中调用自定义JS函数可以通过传递上下文对象来实现（也就是在路由中将函数作为对象方法传递给
pug 文件），类似于下面这个例子。

将函数作为模块导出：

    /** calculateAge.js **/
    
     module.exports = function(birthYear) {
       var age = new Date().getFullYear() - birthYear;
    
       if (age > 0) {
         return age + " years old";
       } else {
         return "Less than a year old";
       }
     };
    /** end calculateAge.js **/

路由中间件：

    const calculateAge = require("calculateAge"); // change path accordingly
    
    router.get("/main", function(){
      let pageInfo = {};
      pageInfo.title = "Demo";
      pageInfo.calculateAge = calculateAge;
      res.render("main", pageInfo);
    });

参考：[javascript - Pug call js function from another file inside
template - Stack
Overflow](https://stackoverflow.com/questions/43838511/pug-call-js-function-from-another-file-inside-template)

 
