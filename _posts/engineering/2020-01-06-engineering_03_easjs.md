---
title: "模块加载seajs源码分析"
subtitle: "「前端工程化」- 模块加载器设计"
header-img: "img/post-bg-halting.jpg"
header-mask: 0.2
layout: post
author: "wendy"
tags:
  - 前端工程化
  - 模块化
  - 模块加载器
  - seajs
---

- 模块加载器设计
- seajs源码分析

模块加载器设计
----------------------

模块加载器核心部分

- `模块部分`：`数据初始化`、`模块存储`。每个模块创建都先初始化数据，存储在缓存对象中
- `资源部分`：`依赖管理`、`资源定义`、`动态加载script文件`。资源定位依赖管理是加载器设计两大核心

### 1 数据初始化

* 加载器中设计了一个名为Module构造函数，每个模块都是此构造函数实例对象。
* 构造函数中给实例对象扩展了， ‘未来’所需要的用到的属性以及方法，
* url：当前模块的绝对路径地址
* deps：模块的依赖列表。
* ...

自定义seajs源码分析如下：

```coq
(function(global){
    var seajs = global.seajs = {
      version: '1.0.0'
    };
    var data = {};
    var cache = {}; // 缓存对象，模块的信息
    var anonymousMeta = {};
    var status = {
      FETCHEN: 1,
      SAVED:2,
      LOADING: 3,
      LOADED: 4,
      EXECUTING: 5,
      EXECUTED: 6
    }
    // 构造函数 模块 => 实例化
    function Module(uri, deps){
      this.uri = uri; // 当前模块的绝对路径地址
      this.deps = deps || []; // 模块的依赖列表
      this.exports = null; // 导出的接口对象
      this.status = 0; // 生命周期 状态码 1(初始化) => 6（加载完毕） (避免重复加载)
      this._waitings = {}; // 检测机制，谁会依赖于我，我会依赖谁 （循环依赖问题）
      this._remain = 0; // 依赖模块的个数
    }
})(this);
```

### 2 模块存储

加载器中设计了一个名为cache缓存对象，每个文件（模块）都会存储在cache对象中<br/>
具体存储方式：`｛"当前模块绝对路径": new Module()｝`<br/>
`注意`：当前模块的绝对路径是通过资源部分，资源定位方式实现的<br/>

自定义seajs源码分析如下：

```coq
(function(global){
    var seajs = global.seajs = {
      version: '1.0.0'
    };
    var cache = {}; // 缓存对象，模块的信息

    Module.prototype.save = function(uri, meta){
      var mod = Module.get(uri);
      mod.uri =  uri;
      mod.deps = meta.deps || [];
      mod.factory = meta.factory;
      mod.status = status.SAVED;
    }

    Module.get = function(uri, deps){
      return cache[uri] || (cache[uri] = new Module(uri, deps));
    }
  
})(this);
```


### 3 资源定位
加载器中设计了一个resolve()的方法把模块名解析成绝对路径格式；<br/>
模块化名的获取<br/>
`seajs.use['a.js', 'b.js']`;<br/>
`seajs.use()`加载器启动方法，启动时会调用传入数组列表中的模块<br/>
seajs源码如下：<br/>

进行资源定位代码如下：
```coq
    // 资源定位 resolve("a") 
    seajs.resolve = function(child, parent){
        if(!child) return "";
        // alias:{"a": "common/js/a"} 
        child = parseAlias(child); // 是否有模块的短名称配置
        child = parsePaths(child); // 是否有路径的短名称配置
        child = normalize(child); // 是否有后缀，处理后缀js
        return addBase(child, parent); // 生成最终的路径地址
    }

```

用use获取依赖如下：

```coq
  Module.use = function(deps, callback, uri){
    var m = Module.get(uri, isArray(deps) ? deps : [deps]);
    m.callback = function(){
      var exports = [];
      var uris = m.resolve();
      for(var i = 0; i < uris.length; i++){
        exports[i] = cache[uris[i]].exec();
      }
      if(callback){
        callback.apply(global, exports);
      }
    }
    m.load();
  }

  var _cid = 0;

  function cid(){
    return _cid++;
  }

  data.preload = [];
  data.cwd = document.URL.match(/[^?#]*\//)[0];
  Module.preload = function(callback){
    var length = data.preload.length;
    if(!length) callback();
  }

  seajs.use = function(list, callback){
    Module.preload(function(){
      Module.use(list, callback, data.cwd + "_use_" + cid());
    });
  } 

```

### 4 动态加载script文件

- 通过加载器resolve()方法，把模块名解析成绝对路径格式
- 动态创建script document.create('script'); src 指向当前模块绝对路径地址
- 加载文件，模块加载器解析当前模块所依赖模块以数组的形式存储

自定义seajs源码分析如下：

```coq
(function(global){
    var seajs = global.seajs = {
      version: '1.0.0'
    };
    // 异步加载
    seajs.request = function(url, callback){
      var node = document.createElement('script');
      node.src = url;
      document.body.appendChild(node);
      node.onload = function(){
        document.body.removeChild(node);
        callback();
      }
   }
})(this);
```

### 5 依赖管理（核心）

比如 a.js依赖了b.js  b.js依赖了c.js<br/>
解析a依赖，找到b，解析b依赖， 找到c依赖，解析c依赖，c是否还有依赖。（`从前到后`）<br/>
加载c依赖、b依赖、a依赖（`从后到前`）<br/>
已知当前模块的cache中的形式 `{"当前模块绝对路径": new Module()｝`<br/>

换算：
* 1 ｛"当前模块绝对路径":{uri:"当前模块绝对路径", deps:[]}}
* 2 deps存储当前模块的依赖列表，依赖泪奔通过动态加载script文件正则解析获取。

> 解析依赖 => 获取依赖依赖绝对路径地址 => 动态加载 => 提取依赖 => 解析依赖。递归方式加载所有模块，直到模块全都加载完毕。

> `思考`：怎么知道a.js依赖了b.js

'懒加载'的书写方式，用到了谁就加载谁, 异步加载

```coq
  define(function(require, exports, module){
    // 模块代码
    var b = require('b.js');
    // 逻辑代码
    ...
  });
```
如何解析呢? 静态化 require('b.js')， 提取参数b.js ＝> deps: ['b.js']<br/>

自定义seajs源码分析如下：
```coq
(function(global){

  Module.define = function(factory){
      var deps;
      if(isFunction(factory)){
        // 解析依赖，变成字符串
        deps = parseDependencies(factory.toString());
      }
      var meta = {
        id: '',
        uri: "",
        deps: deps,
        factory: factory
      };
      anonymousMeta = meta;
  }

  seajs.config = function(options){
    var key, currl
    for(key in options){
      curr = options[key];
      data[key] = curr;
    }
  }

  function commentReplace(match, multi, multiText, singlePrefix){
    return singlePrefix;
  }


   // 解析依赖，变成字符串
  function parseDependencies(code){
    var ret = [];
    // 获取require依赖
    code.replace(commentRegExp, commentReplace).replace(REQUIRE_RE, function(m, m1, m2){
      if(m2) ret.push(m2);
    });
    return ret;
  }

  var commentRegExp = /(\/*())/mg;
  var REQUIRE_RE = /\brequire\s*\(\s*(["'])(.+?)\1\s*\)/g; 
 
  global.define = Module.define;

})(this);
```


seajs源码总结关键词
----------------------

数据初始化、数据存储（缓存， 地址）、资源定位（resolve）、动态加载script文件（request）、依赖管理<br/>
status状态，类似生命周期、cache存储<br/>

config、 define、 use(加载器启动)， require(引入) require.async
- `exports`对外开发接口，方法和变量对象
- `require` 加载外部模块，读取并执行
- `module` 模块

方法： fetch、save、load、onload、exec、error<br/>
正则匹配require，对应的参数 ，资源定位，uri<br/>

[Seajs源码来源](https://cdn.bootcss.com/seajs/3.0.3/runtime-debug.js)