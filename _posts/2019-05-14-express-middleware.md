---
layout:     post
title:      "Express中间件原理解析"
subtitle:   " \"Node.js学习笔记\""
date:       2019-05-10 12:00:00
author:     "mingjunw"
header-img: "img/home-about-bg.jpg"
tags:
    - JavaScript
    - Node.js
    - Express
---

## 前言

* 什么是Node？
Node.js基于Chrome V8引擎的JS开源Web服务器项目

* 什么是Express？
Express是一个简洁、灵活的Node Web应用开发框架,和Koa、Koa2都是目前比较流行的Node开发框架

## 正文

### 中间件的功能

Express中间件本质上就是一个函数,可以用来在请求过程中处理某些事情,比如记录日志,错误处理等等

### 中间件的分类

Express中间件根据功能可以大致上分为以下几类:

* 应用级中间件
* 路由中间件
* 内置中间件: Express框架内置的中间件, 比如static
* 第三方中间件: 插件类型的中间件, 比如morgan日志中间件

### 应用举例

```javascript
// 这是中间件的几种常用处理方法
var express = require('express');
var logger = require('morgan');
var createError = require('http-errors');
var router = express.Router();
app.use(logger('short'));
app.use(function(req, res, next) {
  next(createError(404));
});
app.use('/', router.get('/', function(req, res, next) {
  res.render('index', { title: 'Express' });
}););
app.listen(8080)
module.exports = app;
```

### 源码解析

* 首先看看Express的目录结构
<img class="shadow" width="720" src="/img/in-post/express-middleware/express.png" />

先来看看application.js中对use函数的定义

```javascript
app.use = function use(fn) {
    // 这部分是对传入参数进行处理,判断传入参数的第一个是函数还是字符串
    var offset = 0;
    var path = '/';
    if (typeof fn !== 'function') {
        var arg = fn;
        while (Array.isArray(arg) && arg.length !== 0) {
            arg = arg[0];
        }
        if (typeof arg !== 'function') {
            offset = 1;
            path = fn;
        }
    }
    // 在这里对整个arguments处理,截取出所有传入的中间件函数并进行扁平化,这个slice就是Array.prototype.slice
    var fns = flatten(slice.call(arguments, offset));
    if (fns.length === 0) {
        throw new TypeError('app.use() requires a middleware function')
    }
    // this._router是一个全局的路由对象
    // 在调用之前会先检查是否有这个全局路由
    this.lazyrouter();
    var router = this._router;
    // 遍历截取出来的中间件函数数组并且通过调用router.use传递给Router
    fns.forEach(function (fn) {
        if (!fn || !fn.handle || !fn.set) {
            return router.use(path, fn);
        }
        debug('.use app under %s', path);
        fn.mountpath = path;
        fn.parent = this;
        router.use(path, function mounted_app(req, res, next) {
            var orig = req.app;
            fn.handle(req, res, function (err) {
                setPrototypeOf(req, orig.request)
                setPrototypeOf(res, orig.response)
                next(err);
            });
        });
        fn.emit('mount', this);
    }, this);
    return this;
};
app.lazyrouter = function lazyrouter() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};

// 这个methods是个http方法的集合, 参考methods.js
// 遍历整个methods数组把方法挂载到app上
methods.forEach(function(method){
  app[method] = function(path){
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();
    // 从这里可以看到在调用比如app.get等方法其实是也会被当成中间件处理
    // 会把这个方法当做是一个路由中间件处理
    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

从use函数的定义中我们可以看到在注册中间件的时候是可以一次性传入多个函数的比如:

```javascript

    app.use(logger('short'),logger('short'),logger('short'),logger('short'),logger('short'));

```

* 然后看看Router类对use的定义

```javascript
// Router的构造函数定义
var proto = module.exports = function(options) {
    var opts = options || {};

    function router(req, res, next) {
        router.handle(req, res, next);
    }

    setPrototypeOf(router, proto)

    router.params = {};
    router._params = [];
    router.caseSensitive = opts.caseSensitive;
    router.mergeParams = opts.mergeParams;
    router.strict = opts.strict;
    // 这个在Router类中定义的stack数组, 其实就是用来存储定义的中间件的
    router.stack = [];

    return router;
};
// 上文中提到的通过调用router.use()把注册的中间件函数传递给了Router
// 这里就是router上的use定义
// 调用app.use,其实就是把中间件函数传递到this._router.use
// 最后push进this._router.stack中,这里的this代表的就是app
proto.use = function use(fn) {
    // 这里和app.use函数一样, 也是先对参数进行处理, 这两个函数第一个参数都有可能是个字符串
    // 如果是字符串的话,会被当成中间件生效的路径, 后续参数则会做基本类型判断
    // 只有function才会被当成中间件,其他类型则会直接抛出错误
    var offset = 0;
    var path = '/';
    if (typeof fn !== 'function') {
        var arg = fn;

        while (Array.isArray(arg) && arg.length !== 0) {
        arg = arg[0];
        }
        if (typeof arg !== 'function') {
        offset = 1;
        path = fn;
        }
    }

    var callbacks = flatten(slice.call(arguments, offset));

    if (callbacks.length === 0) {
        throw new TypeError('Router.use() requires a middleware function')
    }

    for (var i = 0; i < callbacks.length; i++) {
        var fn = callbacks[i];

        if (typeof fn !== 'function') {
        throw new TypeError('Router.use() requires a middleware function but got a ' + gettype(fn))
        }
        debug('use %o %s', path, fn.name || '<anonymous>')
        // 实例化一个Layer类用来保存中间件函数
        var layer = new Layer(path, {
        sensitive: this.caseSensitive,
        strict: false,
        end: false
        }, fn);
        // route如果是undefined的就表示这个中间件是个普通中间件
        layer.route = undefined;
        // 最后这个实例化的layer被push进了stack数组中
        this.stack.push(layer);
    }
    return this;
};
// Router上定义的这个方法是用来挂载路由中间件的
proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));
  // 在保存中间件的时候把route属性指向了一个route实例
  layer.route = route;
  // 不管是路由中间件还是普通中间件都被保存在同一个地方
  this.stack.push(layer);
  return route;
};

methods.concat('all').forEach(function(method){
  proto[method] = function(path){
    var route = this.route(path)
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});

```

* 再来看看route.js

```javascript
// 这个是Route也就是路由中间件的构造函数
function Route(path) {
  this.path = path;
  this.stack = [];

  debug('new %o', path)

  this.methods = {};
}
// 遍历methods方法挂载到Route的原型上
// Router上也有做类似的操作
// 这样router也可以使用比如router.get,router.post等方法注册中间件
methods.forEach(function(method){
  Route.prototype[method] = function(){
    var handles = flatten(slice.call(arguments));

    for (var i = 0; i < handles.length; i++) {
      var handle = handles[i];

      if (typeof handle !== 'function') {
        var type = toString.call(handle);
        var msg = 'Route.' + method + '() requires a callback function but got a ' + type
        throw new Error(msg);
      }

      debug('%s %o', method, this.path)

      var layer = Layer('/', {}, handle);
      // route上挂载的method就是http方法 比如get,post等等
      layer.method = method;

      this.methods[method] = true;
      // 在route上也有一个stack数组, 这里保存的也是中间件函数
      // 两者的区别是Router中保存的layer有个route对象, 用来判断是否是路由中间件
      // 如果route = undefined, 表示是个普通中间件函数
      // 如果route保存的是一个Route实例, 表示这个中间件是个路由中间件
      this.stack.push(layer);
    }
    return this;
  };
});
```

* 最后看看layer.js

```javascript
// 其实就是给中间件挂载一堆私有属性
// handle就是函数本身
// path是路径
// regexp是路径对应的正则
function Layer(path, options, fn) {
  if (!(this instanceof Layer)) {
    return new Layer(path, options, fn);
  }

  debug('new %o', path)
  var opts = options || {};

  this.handle = fn;
  this.name = fn.name || '<anonymous>';
  this.params = undefined;
  this.path = undefined;
  
  this.regexp = pathRegexp(path, this.keys = [], opts);

  this.regexp.fast_star = path === '*'
  this.regexp.fast_slash = path === '/' && opts.end === false
}
```

## 总结

整个中间件的关系可以用一张图说明

<img class="shadow" width="720" src="/img/in-post/express-middleware/router.png" />

* Router主要是用来创建一个普通中间件或者一个路由中间件的引用, Router上的stack储存的都是中间件, 可以是普通中间件也可以是路由中间件, 通过layer.route可以判断, 当route是一个Route实例的时候表示这个中间件是一个路由中间件
* Route用来创建路由中间件, 在Route实例上的stack虽然也是layer对象, 但是这里的layer对象里面的方法都是http方法, 比如get, post等