---
layout:     post
title:      "VUE-CLI3 SSR 改造之旅"
subtitle:   "基于SEO的需求，一个项目中部分页面需要支持SSR，基于CLI3改造成可用于工程的SERVER-RENDER项目"
date:       2019-01-27
author:     "EDiaos"
header-img: "img/post-bg-js-version.jpg"
tags:
    - JavaScript
    - Vue
    - Cli3
    - SSR
---

## 背景
> 基于SEO的需求，通常一个项目中部分页面需要支持SSR，以便搜索引擎更好的爬取和检索。一般有2种处理方案，预渲染和服务端渲染。
- 预渲染：页面变更频度不高可采用[prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin)
- 服务端渲染：页面内容是动态生成，则需要基于`vue-server-render`进行实现SSR。

基于VUE的服务端渲染目前有2种比较常用的方案实现，一种是开箱即用的 [nuxtjs](https://nuxtjs.org/) 它基于VUE技术栈抽象了很多模板并提供了额外的功能。如果倾向更直接控制应用程序解构，有更大的自主权，那么可以考虑第二种方案，基于`vue-server-render`和`webpack`进行改造实现服务端渲染。官方的 [VUE SSR 指南](https://ssr.vuejs.org/)，提供了很好的指导。

前两年用过基于官方 [vue-hackernews-2.0](https://github.com/vuejs/vue-hackernews-2.0)改造过的项目，目的是能够正常工作，当时理解的不够深入，配置上比较繁琐。最近有个新工程项目需要用到SSR，如果基于老的一套模块也能正常工作，但是考虑到半年前发布的CLI3更简单的配置，更快的编译速度，所以考虑基于CLI3做一个更好的SSR项目模板。

## 开始前问题&思考
> 由于CLI3本身不包含SSR功能，所以基于SSR需求做改造情况下会有一些问题需要提前思考并确认后面是否可用。
- CLI3的配置更简单，也意味着对比原来增加SSR复杂度更高，需要深入了解CLI3的内置配置进行改造
- CLI3的底层封装了 DEV-SERVER，意味着没法抽取出来改造成SSR，需要自行添加SSR使用的DEV/PROD 使用的SERVER
- 考虑到项目的多样性，期望基于改造之后不影响纯前端渲染，即(支持纯前端渲染，服务端渲染以及基于服务端渲染的前端渲染)
- CLI3 build 后默认资源文件过于分类，CDN指向不太方便，重新指定static 会更利于后续使用
- CLI3 可根据--modle自动读取.env文件，但是有2个问题，1. envName被限制，必须以VUE_APP开头，不太友好   2. 实际CLI3内部不少地方都用了production,dev等NODE_ENV环境变量，如果有多套环境，那么基于--modle 映射环境可能导致工程跑起来会有异常，需要处理


## 初步构建
开始第一步是找现有的解决方案，根据掘金上的两篇篇文章 [通过vue-cli3构建一个SSR应用程序](https://juejin.im/post/5b98e5875188255c8320f88a) 和 [基于vue-cli3 SSR 程序实现热更新功能](https://juejin.im/post/5bc4321b6fb9a05d1e0e824b) 进行同步执行和修改，文章感觉循序渐进，引导的比较好，基本上基于上两篇文章完成初步构建。
虽然基于初步构建了，基于这个构建发现还有不少可以进一步优化，这里列举主要剩余处理的问题：
- 无Proxy处理
- 通过修改baseUrl，修改 服务端渲染页面的前端静态资源地址，不够优雅，并且可能导致前端渲染刷新页面导致404无法访问
- 无AsyncData 等处理，文章只提供了页面渲染，实际业务中有接口请求，有loading等功能，这些通常是工程化项目需要具备
- 增加是否前后端渲染控制
- TDK支持
- PM2支持
- DIST处理


## 总结

## 源码
项目源码：[vue-cli3-ssr-project](https://github.com/EDiaos/vue-cli3-ssr-project) , 欢迎 Start 和 提 Issues






