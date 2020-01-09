---
title: "JavaScript内存管理机制"
subtitle: "「JavaScript基础原理」- JavaScript内存管理机制"
header-img: "img/post-bg-apple-event-2015.jpg"
header-mask: 0.2
layout: post
author: "wendy"
tags:
  - JavaScript
  - 垃圾回收机制
  - 内存
  - 内存泄漏
---

- 内存分配：内存模型
- 生命周期：内存的生命周期、内存泄漏、内存常驻
- 垃圾回收：垃圾回收机制

内存模型
----------------------

- 堆：引入数据类型: Object, Array
- 栈：基本数据类型：undefined、null、boolean、number、 string
- 池：常量:const

> 连续性内存，存在栈顶和栈底

堆栈： 进栈、出栈，先进后出、后进先出<br>
引用类型, 新建对象存在内存，是个内存地址。

######  队列 先进先出

```js
let arr = new Array();
// 队列 先进先出
arr.unshift(1);
arr.unshift(2);
arr.unshift(3);
arr.pop();
console.log(arr); // [3, 2]
```

######  堆栈
```js
let arr2 = new Array();
arr2.push(1);
arr2.push(2);
arr2.push(3);
arr2.pop();
console.log(arr2); // [1, 2]
```

###### 基本类型

```js
let a = 10;
let b = a;
b = 20;
console.log(a); // 10
```

###### 引用类型

```js
let obj = {aa:20, bb:30};
let obk = obj;
obk.aa = 10;
console.log(obj.aa); // 10
```


内存的生命周期、内存泄漏、内存常驻
----------------------

###### 1 内存泄漏, 无声明
```js
// 内存泄漏
function fn(){
  // 无声明就是全局变量, 相对于window.demo，导致内存泄漏
 demo = "测试";
 // let demo = "测试";
}
fn(); 
```

###### 指向全局window，导致内存泄漏

```js
function fn2() {
  // 指向全局window，导致内存泄漏
  this.demo2 = 123;
}
fn2(); 
```


垃圾回收机制
----------------------

垃圾回收算法主要依赖于引用的概念。<br>
在内存管理的环境中，一个对象如果有访问另一个对象的权限（隐式或者显式），叫一个对象引用另一个对象。“对象” 的概念不仅特指javascript对象，还包括函数作用域（或者全局词法作用域）

```js
 // 垃圾回收机制
  let o = {
    a: {
       b: 2
    } 
  }
  // 以上o没有回收，引用操作
  let o2 = o; // o2引用了o
  // o是一个零引用状态
  o = 1;
  // 没有被回收，因为属性a还在被调用
  let oa = o2.a; 
  console.log(o); // 1
  o2 = 'wendy';
  // 进行手动主动回收
  oa = null; // 释放内存

```

###### 1 引用计数垃圾收集
初级的垃圾收集算法，对象是否不再需要即对象有没有其他对象引用到它，如果没有引用指向该对象（零引用），对象将被垃圾回收机制回收。<br>
缺点：循环引用不会被回收

```js
  // 引用计数垃圾收集
  function fn3(){
    let o = {};
    let o2 = {};
    // 循环引用，没被释放
    // 相互引用，不会被回收
    o.a = o2;
    o2.a = o;
  }
  fn3();

```

###### 2 标记 － 清除算法
设置一个叫做根（root）的对象（在javascript里，根是全局变量）。垃圾回收器将定期从根开始，找所有从根开始引用的对象，然后找这些对象引用的对象...从根开始，垃圾回收器将找到所有可以获得的对象和收集所有不能获得的对象<br>
缺点： 无法从根对象查询到的对象都将被清除（可被忽略）

```js
// <p id="domP"><p>  <button id="btn">移除p节点<button>
window.onload = () => {
  let domP = document.getElementById('domP');
  const timeInerval = setInterval(()=>{
    let time = new Date();
    if(domP){
      domP.innerHTML = JSON.stringify(time);
    }
  }, 1000);

  let btn = document.getElementById('btn');
  btn.onclick = () => {
    // 显示的移除
    clearInterval(timeInerval);
    domP.remove();
  }

  // 闭包，作用域。全局访问不了局部，通过闭包return
  // 闭包不能保持太多变量，会导致内存溢出。需要及时对闭包变量进行回收
  function findGf(){
    this.name = '名字';
    this.let = 160;
  }
  findGf.prototype.selectName = function(){
    // 这里会存在大量的变量
    // let _this = this;
    // 单独处理，并回收
    let name = this.name;
    return function(){
      return name;
    }
    // 进行回收
    name = null;
  }
  let gf = new findGf();
  console.log(gf.selectName()());
  // node
  console.log(process.memoryUsage());
}
```

> 温馨提示：少定义全局变量，定时器要及时清除、对一些全局变量（常驻内存）及时释放。
