<!--
author: leo
date: 2017-08-08
title: Koa-router 优先级问题
tags: koa,router,nodejs
category: backend
status: publish
summary: Koa-router 优先级问题
-->

问题描述
----
在使用Koa-router作为路由遇到了一个优先级问题.如下代码
```javascript
// routerPage.js file
const router = require("koa-router")
router.get("/test", ctx => { ctx.body = "test" })
router.get("/router/test", ctx => { ctx.body = "router test" })
module.exports = router

// routerIndex.js file
const router = require("koa-router")
const routerPage = require("./routerPage")
router.use(routerPage.routes(), routerPage.allowedMethods())
module.exports = router
```
 在访问`"/router/test"`时路由会优先匹配到`"/test"`路由,返回`ctx.body = "test"`,这个问题就很尴尬了,项目空闲下来去翻看源码终于找到了原因

问题原因
----
Koa-router的源码并不长,layer.js和router.js两个文件加起来共一千多行代码.建议可以结合[这篇文章][1]阅读.
其中造成这个问题的原因就是router.js中`router.use`这个方法,方法源码如下
```javascript
// 主要作用: 给path添加中间件
Router.prototype.use = function () {
  var router = this;
  var middleware = Array.prototype.slice.call(arguments);
  var path = '(.*)';

  // 如果path为array则递归调用use方法
  if (Array.isArray(middleware[0]) && typeof middleware[0][0] === 'string') {
    middleware[0].forEach(function (p) {
      router.use.apply(router, [p].concat(middleware.slice(1)));
    });

    return this;
  }
  //如果传入了path,则只对此path操作
  var hasPath = typeof middleware[0] === 'string';
  if (hasPath) {
    path = middleware.shift();
  }
  // 如果传入参数为一个路由数组,则遍历为每个路由添加前缀,中间件,并将此路由放入全局的路由数组
  middleware.forEach(function (m) {
    if (m.router) {
      m.router.stack.forEach(function (nestedLayer) {
        if (path) nestedLayer.setPrefix(path);
        if (router.opts.prefix) nestedLayer.setPrefix(router.opts.prefix);
        router.stack.push(nestedLayer);
      });

      if (router.params) {
        Object.keys(router.params).forEach(function (key) {
          m.router.param(key, router.params[key]);
        });
      }
    } else {
      router.register(path, [], m, { end: false, ignoreCaptures: !hasPath });
    }
  });

  return this;
};
```
问题就出在`router.use(routerPage.routes(), routerPage.allowedMethods())`时没有设置前缀,
路由就自动添加了默认的前缀`"(.*)"`,这里的path发生了改变,在路由后续的操作中,将path使用`pathToRegExp`转换成正则表达式时`"/test"`这个path本应该是`/^\/test...../`就会变成`/(.*)/\/test...`(大概是这个意思)
那么原本以`/test`开头的路由就会匹配包含`/test`的路由
所以request path 为`/router/test`时会被`/test`路由先匹配中,路由也就不会往下匹配

解决方式
----

- 将条件更精确的路由放到前面
- 在`/test`那个路由中加一个中间件,当匹配到`/router/test`时 `await next()`继续向下执行
- 更改源码`Router.propertype.use` 中 `path = "(.*)"` 为 `path = false`
- 在使用`router.use`时代码做一定更改,代码如下

```javascript
    // routerPage.js file
    const router = require("koa-router")
    router.get("test", ctx => { ctx.body = "test" })
    router.get("router/test", ctx => { ctx.body = "router test" })
    module.exports = router
    
    // routerIndex.js file
    const router = require("koa-router")
    const routerPage = require("./routerPage")
    router.use("/", routerPage.routes(), routerPage.allowedMethods())
    module.exports = router
```
  [1]: https://segmentfault.com/a/1190000007468233
