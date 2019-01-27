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

> 前两年用过基于官方 [vue-hackernews-2.0](https://github.com/vuejs/vue-hackernews-2.0) 改造过的项目，目的是能够正常工作，当时理解的不够深入，配置上比较繁琐。最近有个新工程项目需要用到SSR，如果基于老的一套模块也能正常工作，但是考虑到半年前发布的CLI3更简单的配置，更快的编译速度，所以考虑基于CLI3做一个更好的SSR项目模板。

基于SEO的需求，通常一个项目中部分页面需要支持SSR，以便搜索引擎更好的爬取和检索。一般有2种处理方案，预渲染和服务端渲染。

- 预渲染：页面变更频度不高可采用 [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin)
- 服务端渲染：页面内容是动态生成，则需要基于`vue-server-render`进行实现SSR。

基于VUE的服务端渲染目前有2种比较常用的方案实现，一种是开箱即用的 [nuxtjs](https://nuxtjs.org/) 它基于VUE技术栈抽象了很多模板并提供了额外的功能。如果倾向更直接控制应用程序解构，有更大的自主权，那么可以考虑第二种方案，基于`vue-server-render`和`webpack`进行改造实现服务端渲染。官方的 [VUE SSR 指南](https://ssr.vuejs.org/)，提供了很好的指导。

#### 开始前问题&思考

由于CLI3本身不包含SSR功能，所以基于SSR需求做改造情况下会有一些问题需要提前思考并确认后面是否可用。

- CLI3的配置更简单，也意味着对比原来增加SSR复杂度更高，需要深入了解CLI3的内置配置进行改造
- CLI3的底层封装了 DEV-SERVER，意味着没法抽取出来改造成SSR，需要自行添加SSR使用的DEV/PROD 使用的SERVER
- 考虑到项目的多样性，期望基于改造之后不影响纯前端渲染，即(支持纯前端渲染，服务端渲染以及基于服务端渲染的前端渲染)
- CLI3 build 后默认资源文件过于分类，CDN指向不太方便，重新指定static 会更利于后续使用
- CLI3 可根据--modle自动读取.env文件，但是有2个问题，1. envName被限制，必须以VUE_APP开头，不太友好   2. 实际CLI3内部不少地方都用了production,dev等NODE_ENV环境变量，如果有多套环境，那么基于--modle 映射环境可能导致工程跑起来会有异常，需要处理


#### 初步构建

开始第一步是找现有的解决方案，根据掘金上的两篇篇文章 [通过vue-cli3构建一个SSR应用程序](https://juejin.im/post/5b98e5875188255c8320f88a) 和 [基于vue-cli3 SSR 程序实现热更新功能](https://juejin.im/post/5bc4321b6fb9a05d1e0e824b) 进行同步执行和修改，文章感觉循序渐进，引导的比较好，基本上基于上两篇文章完成初步构建，作者是一个刚毕业一年学生，表述上相当不错。
虽然基于文章初步构建了项目，但是距离工程化还有一些距离，罗列部分需要处理的列表：

- 无Proxy处理
- 通过修改baseUrl，修改 服务端渲染页面的前端静态资源地址，不够优雅，并且可能导致前端渲染刷新页面导致404无法访问
- 无AsyncData 等处理，文章只提供了页面渲染，实际业务中有接口请求，有loading等功能，这些通常是工程化项目需要具备
- 增加是否前后端渲染控制
- TDK支持
- PM2支持
- DIST处理

#### ENV Config 构建
背景：CLI3默认读取 --mode NODE_ENV作为环境变量，提供了.ENV文件作为匹配环境。使用中发现CLI3内置一些处理以及写死处理`NODE_ENV= production || development`，如果我们增加了一种环境，比如test环境，用于测试环境部署，有测试环境的配置等等，测试环境部署的时候却要依赖production来构建。
`会导致本地开发/服务端部署 与 实际的多套环境冲突`，这样的场景并不利于开发梳理逻辑。另外基于.ENV环境获取ENV的时候被强制以VUE_APP作为开头的变量名不是很友好，使用起来稍显别扭。

解决方案：基于`cross-env`提供设置脚本环境变量，增加`NODE_DEPLOY`作为环境变量，工程根据这个环境变量提取配置的config文件中的env配置。这样好处就是把本地开发和部署 以及 与实际部署环境拆分开来不再互相干扰。实现方案如下：
1. `npm install cross-env -D`
2. 修改package.json中的脚本，根据项目实际环境设置`NODE_DEPLOY`，比如：`cross-env NODE_DEPLOY=test npm run dev`
3. 增加config文件夹，并增加 index.js 以及 env
代码&目录参考如下，基于`NODE_DEPLOY`获取到环境变量，config/index.js根据环境变量获取env配置文件，默认合并env.js，这与cli3提供env基本一致，但不受限于命名。
![java-javascript](/img/vue-cli3-ssr-project/env_config.jpg)

#### 项目源码

项目源码：[vue-cli3-ssr-project](https://github.com/EDiaos/vue-cli3-ssr-project) , 欢迎 STAR 和 提 ISSUES .






