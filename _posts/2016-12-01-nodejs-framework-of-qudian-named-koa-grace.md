---
layout: post
title: 趣店前端团队基于koajs的前后端分离实践
author: 熊伟烈
tag: 前后端分离,Koa-grace
digest: 前后端分离已经是当下前端界老生常谈的话题，业界也有较好的推广和应用；本文主要通过趣店集团前端自主研发的框架「Koa-grace」实战性出发，和大家一起聊聊不一样的前后端分离。
---

## 前言

关于**前后端分离**，我的感觉其实也是：这么老土的话题，为什么还要拿出来老调重弹？

但越来越发现基于前后端分离的类RESTful架构，能很好的满足WebAPP的业务需求。尤其是WebAPP+NativeAPP产品为主的中小型公司，能让整个公司的服务端研发和部署更灵活。

**PS：**已经了解前后端分离和koajs(不喜欢看背景和扯淡)同学可以直接跳到：“三、如何实践”。

## 一、什么是前后端分离？

**前后端分离**的概念和优势在这里不再赘述，有兴趣的同学可以看各个前辈们一系列总结和讨论：
* 系列文章：[前后端分离的思考与实践(1-6)](http://blog.jobbole.com/71675/)
* slider: [淘宝前后端分离实践](http://2014.jsconf.cn/slides/herman-taobaoweb/#/)
* 知乎提问：
	* [如何评价淘宝 UED 的 Midway Framework 前后端分离？](https://www.zhihu.com/question/23512853) 
	*  [Web 前后端分离的意义大吗？](https://www.zhihu.com/question/28207685)

尤其是《前后端分离的思考与实践》系列文章非常全面的阐述了前后端分离的意义和技术实现。如果不想看上面的文章，可以在脑海里留下这样一个轮廓就好：

![前后端分离](https://github.com/xiongwilee/demo/blob/master/photo/%E5%89%8D%E5%90%8E%E7%AB%AF%E5%88%86%E7%A6%BB.png?raw=true)

本文主要阐述趣店团队基于Koajs的前后端分离实践方案。

## 二、为什么选择koa？

[koa](https://github.com/koajs/koa)是由Express 原班人马打造的一个更小、更健壮、更富有表现力的 Web 框架。（[koa官网](http://koajs.com)需用梯子，中文文档可以参考郭宇大神的翻译：[koa中文文档](https://github.com/guo-yu/koa-guide)，这里也不详细介绍。）

![koa](https://github.com/xiongwilee/demo/blob/master/photo/koa.png?raw=true)

看到koa之后，简单看了下文档和源码，立刻感觉**koa不就是为前后端分离而诞生的轻量级框架吗？**

因为：
* 如果要用koa实现一整套类似于PHP的Laravel或者Ruby on Rails的服务端MVC框架其实还有很长的路要走；
* koa中间件的思路可以很好的被前后端分离的思路利用起来，比如：路由、模板引擎、数据代理等等；
* koa基于ES6的generator特性(不了解generator的同学可以参考[阮一峰老师的文章](http://es6.ruanyifeng.com/#docs/async))，利用[co](https://github.com/tj/co)等模块让前端同学写一个路由再简单不过；
* ……

总之，koa给我的第一印象就是：如果用它来实现一套前后端分离的框架，非常`高效`、`轻量`、`易扩展`。

## 三、如何实践？

### 1、刀耕火种的年代

在谈如何实现基于koajs的前后端分离框架的之前，必须提一下在做分离之前趣店（原趣分期）的情况：
* 一个仓储托管了趣分期业务前后端所有代码，由于不断维护，这个仓储足足有**600M**；
* 后端为了满足NativeAPP的接口需求，**这个600M的仓储还包含了提供给客户端的代码**；
* 前后端开发流程为：**先由后端写一个路由 → 前端将这个路由写成可以交互的页面 → 前端提交代码后后端同学“套模板”填充数据**；
* 前后端开发都在一台远程开发机上，通过samba隐射到本地进行开发；
* 前端几乎没有做任何打包编译；有时候为了偷懒，甚至把CSS和JS直接以内嵌的方式写在模板里（不要去联想vue，两码事儿）

写到这儿想贴代码来着，想想还是算了……

### 2、基于koa的实现

无论使用什么工具要实现前后端分离框架，无非要满足这样几点：
* 更便捷地创建路由
* 更高效地代理数据请求
* 更灵活地环境部署

先看一个已经实现了demo应用的目录结构：

```javascript
.
└── demo
    ├── controller
    │   ├── data.js
    │   ├── defaultCtrl.js
    │   └── home.js
    ├── static
    │   ├── css
    │   ├── image
    │   └── js
    └── views
        └── home.html
```
这个结构大家再熟悉不过：`controller`提供路由；`static`包含静态文件；`views`提供模板文件。
这里可能大家会有疑问，如果我的前端构建是components的模式怎么办呢？

```javascript
├──components_a
│   ├── home.js
│   ├── home.css
│   └── home.html
├──components_b
...
```
没关系，do what u want！很多boilerplate可以直接拿到这个框架下直接使用，我们团队自己也正在使用了requirejs和vue的构建方案。

#### 1）更便捷地创建路由：`controller`

回到上文提到的案例：

```javascript
controller
├── data.js
└── home.js
```
这段代码生成的是一个`/data/*`,`/home/*`路由，用过Express的同学对这种路由实现肯定一点都不陌生。

但有一点不一样的是，koa下面每一个路由是一个独立的generator函数（很遗憾，generator函数不能使用箭头函数的语法来表示），参看`controller/home.js`：

```javascript
exports.index = function* () {
  yield this.bindDefault();
  yield this.render('home', {
    title: 'Hello , Grace!'
  });
}
```
你还可以用一个对象把一个controller包起来，，参看`controller/data.js`：

```javascript
exports.info = {
  repo: function*(){
    yield this.proxy('github:repos/xiongwilee/koa-grace')
  }
}
```

这种写法非常适用于仅仅用来做一个数据代理的ajax请求的控制器。当然，如果需要的话，你还可以在控制器里对数据做一些加工。

另外，还有几个细节与express的路由控制器不太一样：
1. 终止请求的动作都有koa来完成，在这里的controller里，你不需要通过`res.end()`来终止请求；
2. 这里的controller的作用域，就是koa的上下文，所以你可以通过`this.req`获取request对象；
3. 这里的controller里，异步回调的语法成为了往事，请尽情的使用`yield`。

我们对现在的路由方式做了一个压测：8核CPU/8G内存的机器，通过一个路由返回`hello world!`，结果显示：**8核CPU/8G至少能扛住1000QPS**的压力（cpu idle不到60%，cpu load接近8）。

#### 2）更高效地代理数据请求：proxy

前后端分离模式下的，数据代理主要有这两种使命：

* 一种是，拼装一个和多个数据结果，给模板渲染数据或者给Ajax接口
* 另外一种是，接受用户的数据请求，将结果交给一个后端接口

第一种场景，比如：一个用户中心的页面，需要展示用户的用户信息、订单列表、喜欢的商品推荐等等信息。由于后端的架构，用户、订单、商品可能都是属于不同的服务，这就意味着，后端同学需要给你同时拼装这么多数据，拼装还有可能是同步获取每个接口然后再一起返回给前端。

现在，就可以尽情的发挥**nodejs异步并发及koa的generator同步语法**的优势，我们还可以这么写：

```javascript
 repo: function*(){
   yield this.proxy({
    data1:'github:repos/xiongwilee/data1',
    data2:'github:repos/xiongwilee/data2',
    data3:'github:repos/xiongwilee/data3'
   })
 }
```

koa会**自动并发请求所有接口**，然后将结果返回给你，然后可以在上下文的`backData`中获取结果集。

另外一种场景，比如用户需要上传一张图片或者提交一篇文章，你也可以这样写：

```javascript
submit: function*(){
  yield this.proxy('http://test.com/submit')
}
```
不用做任何配置，koa会直接把request数据buffer直接通过管道pipe给后端接口。

值得一提的是：
* 有没有发现，这种proxy方式是没有同域机制的限制 了，前端面试常问的跨域方案都不是事儿；
* 另外，同学们可能觉得，本来一个请求都能搞定的事情非要用nodejs代理一次，会不会很慢？事实上，我们的测试结果显示：**内网访问的情况下，nodejs代理耗时不超过10ms**

最后，我们也做了一个压测：8核CPU/8G内存的机器，通过一个路由将请求代理到另外一个接口，结果显示：**8核CPU/8G至少能扛住300QPS**的压力，这个结果比纯路由的压测情况要低很多，值得注意。

不过有意思的一点是，我们压测发现：**proxy的性能与接口响应时间无关，与接口响应content大小有密切关系：接口响应内容越大，proxy性能越差**。

#### 3） 更灵活地环境部署

基于koa可以实现各类有意思的中间件。比如：可以拦截请求计算请求总耗时中间件、可以匹配路径实现本地的静态文件服务器、可以不需要再另开server实现一个mock数据的功能……

##### a) 开发环境

上文提到过一个DEMO的文件目录，具体在开发环境和生产环境中整体目录结构中其实是这样的：

```javascript
├── app				// 开发模式下应用模块目录
│   ├── blog
│   └── demo
├── cli				// 配套命令行工具
│   ├── bin
│   ├── lib
│   └── package.json
├── log				// 日志目录
└── server			// 服务器目录
	├── app			// 实际服务端引用的模块目录
	│   ├── blog
	│   └── demo
    ├── bin
    ├── config
    ├── src
    └── package.json
```

在开发环境下，你可以将`./app`下的项目源文件直接编译到`./server/app`目录，这就意味着：**`./app/`目录下的各个项目，你自己想怎么玩就怎么玩**。

我们目前有基于gulp+requirejs的模块化方案和基于webpack+vue的前端构建方案，其中gulp+requirejs的方案已经开源：https://github.com/xiongwilee/gulp-requirejs-boilerplate 。

基于上文提到的路由和数据代理的功能，现在我们的开发模式就可以优化为：**前后端确定接口→前后端同时开发→联调提测上线**。给开发带来的好处自不必说：
* 后端接口完全解耦，前端利用异步并发，性能可以达到最优；
* WebAPP和NativeAPP可以通用一套接口，后端省了不少成本；
* 前后端独立之后，前端构建的成本更低，灵活性更高；
* ……

##### b) 生产环境

生产环境的部署与大多数服务部署类似：经过`SLB`之后到`nginx`进行反向代理，走到nodejs服务响应请求；可以在nginx层进行负载均衡和监控；如果需要用到缓存，可以在nginx和nodejs之间再加一层`Varnish`。

这里必须要推荐下最牛哔的nodejs进程管理工具：[pm2](https://github.com/Unitech/PM2/)，完成了趣店生产环境下nodejs进程管理和nodejs日志切分。

然后提一下，我们的沙盒环境和测试环境部署。

沙盒只需要摘一台性能比较low的线上机器，提供给办公网络访问就行。因为数据代理的接口直接走的服务端对应环境的接口，整个沙盒环境和测试环境的搭建也非常愉快。

另外，我们对线上业务做了一个压测：**单台8核CPU/8G机器，能扛住500QPS左右的压力**。

最后，关于环境部署，再提两个场景：

* 一个场景是，**超越前后端分离框架的本职工作，实现团队文档系统**。

	因为前后端分离框架本身是不支持数据库和SESSION存储功能的，为了满足一些简单的数据库和文件上传需求，我们自己实现了两个中间件，一个能够利用mongoose很简单地与mongo打交道，一个利用`formidable`,`koa-send`模块轻易实现文件上传和下载；从而很快实现了团队博客和文档系统。
	
	当然了，这个文档系统和团队博客仅限在内网系统中使用。

* 另外一场景是，**后端环境使用Java MVC框架下的前后端独立开发**。

	我们团队与蚂蚁金服团队深度合作（事实上是作为TP，实现支付宝下的业务需求），必须使用蚂蚁金服的基于Java的sofalite MVC框架，而且坑爹的是：蚂蚁金服不能提供他们的前端构建框架！
	
	这就意味着：我们团队的前端同学需要回到刀耕火种时代，跟后端同学用一个仓储自己搭建Java环境进行本地开发了。
	
	但是，别忘了，我们现在有基于koa的前后端分离框架啊（知乎腔）！
	
	事实上，前后端独立开发的唯一瓶颈就在与Java的`velocity`模板引擎；而我们的前后端分离框架的模板引擎是可配的。
	
	所以，前后端定好了接口，然后前端就愉快地用上mock数据独立开发去了；**最后，整个项目提前四天开发联调完成**。

## 四、koa-grace

主（guang）角（gao）终于要出场了！

[![](https://github.com/xiongwilee/demo/blob/master/photo/koa-grace.png?raw=true)](https://github.com/xiongwilee/koa-grace)

这套前后端分离框架——**[koa-grace：基于koa的前后端分离框架](https://github.com/xiongwilee/koa-grace)**已经开源。欢迎各路大神`star`/`fork`/`提ISSUE`，谢谢！！！

* **项目主页：** https://github.com/xiongwilee/koa-grace 
* **示例页面：** http://grace.wilee.me/
* **详细文档：** https://github.com/xiongwilee/koa-grace/wiki/%E4%B8%AD%E6%96%87%E6%96%87%E6%A1%A3

另外，配套命令行工具暂时仅限内网使用，后续会开源出来。最后，关于koa-garce还得啰嗦两点：

1）koa-grace是基于`koa 1.x`版本

目前koa-grace所有的中间件都是基于`koa 1.x`，而`koa 2.x`正式发布之后会立刻跟进。

但koa-grace毕竟只是一个轻量级的框架：`Requiring babel to run your server is just not good developer experience.`（引自：[koa/issues/533](https://github.com/koajs/koa/issues/533)）。

2）koa-grace需要完善测试用例

vue的作者尤雨溪说过，`不写测试的开源项目不是合格的开源项目`。而koa-grace目前还没有测试用例覆盖，希望大家一起来完善。


## 最后

无论前端怎么玩、无论技术怎么发展，**任何架构最终都是为了服务于产品**。

**减少技术成本，提升迭代效率和产品体验**是Web技术的核心诉求（尤其是在做产品不快则死的创业公司）。

趣店产品项目在逐渐升级，我们也在随着产品升级而将业务迁移到这套前后端分离架构上来；目前来看，koa-grace给趣店产品和技术部署带来了不少便利。

就说到这儿，老板喊我去写PHP了…………