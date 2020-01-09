---
title: "浅谈JavaScript的执行机制"
subtitle: "JavaScript的执行机制"
header-img: "img/post-bg-apple-event-2015.jpg"
header-mask: 0.2
layout: post
author: "wendy"
tags:
  - JavaScript
  - 执行机制
  - 作用域链
  - 内存
  - V8引擎
---

- JavaScript的执行机制 － eventloop
- 作用域链与引用类型
- V8引擎内存问题

JavaScript的执行机制 － eventloop
----------------------

执行顺序：

> 开始执行  ＝> 逐步执行代码  => 有代码异步操作 ＝>  异步操作插入到异步队列 => 全部执行完毕 => 询问是否有异步 => 异步队列 => 异步任务的回调回到主任务执行


异步队列里包含微任务和宏任务
- 宿主环境（常见的两种宿主环境有浏览器和node）提供的叫宏任务，由语言标准（比如ES6提供的promise）提供的叫微任务
- 微任务：Promise, process.nextTick
- 宏任务：整体代码script、setTimeout、setInterval
- async本身不是异步操作， await是等待，后面的代码都不执行，await从使用上来说，必须等待一个promise

###### 学习微宏任务，异步

```js
// 一、学习微宏任务，异步
setTimeout(function(){
  console.log('set1');
  new Promise(function(resolve){
    resolve();
  }).then(function(){
    console.log('then3');
  });
});

// 微任务
new Promise(function(resolve){
  console.log('pr1');
  resolve();
}).then(function(){
  console.log('then1');
});

// 宏任务
setTimeout(function(){
  console.log('set2');
});

// 微任务
new Promise(function(resolve){
  resolve();
}).then(function(){
  console.log('then2');
});

console.log(3);

// 执行结果 pr1 3 then1 then2 set1 then3 set2
```

###### 学习async、异步

```js
// 二、学习async、异步
// async本身不是异步函数
async function a(){
  console.log('async')
  // await是等待，后面的代码都不执行，await从使用上来说，必须等待一个promise
  const b = await new Promise(function(resolve){
    resolve(7);
  }); 
  console.log(5);
  console.log(b);
}
a();
console.log(3)
// 执行结果 async 3 5 7

```

###### 循环中异步处理

```js
for(var i = 0; i < 10; i++){
  setTimeout(()=>{
    console.log(i); // 执行都是10
  });
}

for(var i = 0; i < 10; i++){
  (function(i){
    setTimeout(()=>{
      console.log(i); // 执行都是0123456789
    });
  })(i);
}

// node读取文件、网络都是异步的
var arr = ['url', 'url', 'url'];
// 把这个数组里面的文件都读取出来
for(var i = 0; i < arr.length; i++){
  // 这里是错的，不对的，这里都是最后一条
  fs.readFile(arr[i], function(){

  });
}
```

微任务会先于宏任务； 微任务队列空了，才会执行下一个宏任务，异步队列执行的时候，只要有微任务，都优先执行微任务


作用域链与引用类型
----------------------

- 对象、数组是引用类型
- 参数在方法内，相当于是个局部变量
- js查找变量会从当前作用域逐级向上查找，直到window，如果window没有，则为undefined

###### 对象、数组使用
```js
var a = [1, 2, 3];
function f(){
  a[3] = 4;
  a=[100]
}

f();
console.log(a); // 执行结果［100］

```

###### 参数在方法内，相当于是个局部变量

```js
// 添加一个参数
var a = [1, 2, 3];
function f(a){
  // a是局部变量和外面的a是没有关系的, 这里a只是内存地址和外面的a是一样的
  a[3] = 4; // 引用类型，
  a = [100]; // 对局部变量赋予给了新的数组，切断了和外面的联系
  a[1] = 5; // 引用类型
  console.log(a) // 执行结果［100，5]
}

f(a);
console.log(a); // 执行结果［1, 2, 3, 4］
```

###### 对象、数组是引用类型

```js
// 对象、数组是引用类型
var a = [1, 2, 3];
var b = a; // 将数组a的内存地址指向b
a[3] = 4;
console.log(b); // 执行结果［1, 2, 3, 4］
```

```js
var a = { n:1 };
var b = a;
// a.x  .号运算优先级最高，从右往左运行
a.x = a = {n:2};
console.log(a.x); // undefined
console.log(b.x); // {n:2}

// 假设{n:1}内存地址是1000
// a.x要申请一块新的内存地址
```

###### js查找变量作用域逐级向上查找

```js
var c = 123;
function a(){
   // js查找变量会从当前作用域逐级向上查找，直到window，如果window没有，则为undefined
  console.log(c);
}
```

javascript随便扩容，删减

扩展：Javascript的数组并不是数据结构意义上的数组，为什么？
- 1 数据结构意义上的数组是，连续想等内存变量。必须规定大小、类型
- 2 真的数组是不可以扩容的

> 思考：数据结构上，扩容一个数组，内存做了什么？

V8引擎内存问题
----------------------

v8引擎，浏览器可以拿到最多1.4g的内存可以支配（64位），32位是700mb<br>
node它可以使用c++的内存, 最大可以达到2000多MB

> 回收的时候，会暂停js执行，回收会100MB内存大概需要10毫秒，so内存快接近满才回收，除非主动回收

内存如何回收
- 内存快接近满 ＝> 不回收(全局变量)
- 内存快接近满 ＝> 回收(局部变量且失去引用)

内存查看
- 浏览器：window.performance;
- node-process.memoryUsage();

```js
function getme(){
  var mem = process.memoryUsage();
  var format = function(bytes){
    return (bytes/1024/1024).toFixed(2)+'MB';
  }
  console.log("heapTotal:" + format(mem.heapTotal) +
   'heapUsed:' + format(mem.heapUsed));
}

// JS stack trace 内存不足
var size = 20 * 1024 * 1024;
var arrall = [];
// setTimeout(function(){
//   arrall.push(new Array(size));
// });

for(var i = 0; i< 20; i++){
  getme();
  // 给全局变量大小限制
  if(arrall.length > 4) {
    arrall.shift();
  }
  arrall.push(new Array(size));
}

function b(){
  var arr1 = new Array(size);
  var arr2 = new Array(size);
  var arr3 = new Array(size);
  var arr4 = new Array(size);
  var arr5 = new Array(size);
  var arr6 = new Array(size);
  var arr7 = new Array(size);
  var arr8 = new Array(size);
  var arr9 = new Array(size);
  var arr10 = new Array(size);
}
b();
getme();
var arr11 = new Array(size);
arr11 = null;
arr11 = undefined;
var arr12 = new Array(size);
var arr13 = new Array(size);
var arr14 = new Array(size);
getme();

```

> 思考：如何避免，针对全局变量，对全局变量限制, 如果用node服务，全局变量，只要服务开着，全局变量不会被回收

容易引发内存使用
- 1 滥用全局变量，记得及时回收
- 2 缓存不限制
- 3 操作大文件

