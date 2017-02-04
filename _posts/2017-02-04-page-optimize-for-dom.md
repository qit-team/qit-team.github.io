---
layout: post
title: 高频dom操作和页面性能优化探索
author: 高雪婷
tag: DOM操作,性能优化,导师点评
digest: 新同学完成一个年会抽奖项目后，对项目中遇到的些许技术问题进行深度思考和总结；对于前端老司机来说，这些并不是什么困难的技术点，但对于新同学来说，能有这样的思考和总结，是非常值得赞许的。这里尤其值得关注的，是导师对新人总结的细心点评，及其负责任的帮助新同学成长。
---

> 新同学完成一个年会抽奖项目后，对项目中遇到的些许技术问题进行深度思考和总结；对于前端老司机来说，这些并不是什么困难的技术点，但对于新同学来说，能有这样的思考和总结，是非常值得赞许的。这里尤其值得关注的，是导师对新人总结的细心点评，及其负责任的帮助新同学成长。

## 前言：导师点评

这篇文章是 @雪亭 春节期间整理的，态度很赞，值得学习。
通读本篇文章，能看出有花时间去收集整理问题，但整体还有明显欠缺：

### 0.1 每部分都挺清晰，但从总体结构上看是散的：
* 第一部分
    * 提出问题，但全文没有看到针对问题的解决和数据反馈；
* 第二部分
    * 标题是分析操作DOM的性能分析，但内容偏网络加载和页面初始化、渲染等过程；
* 第三部分
    * 仅从几个片段讨论了操作DOM的优化方法，没有总结；
    * 每个优化点中demo不够具象，也没有数据/实操说明优化效果如何；
    * demo的技术栈比较零散，看起来思路比较跳跃；
* 第四部分
    * 转而讨论第二部分的网页加载性能，感觉没有实质内容。

### 0.2 每部分问题的发掘及解决的方式还不够细致：
* 高频操作DOM会导致的问题
    * 有哪些行为会导致`重绘`和`重构`？
        * 有没有深究一下为何会导致重构？
        * 除了js,css的加载和改变是否也会阻塞渲染？
    * 有关年会抽奖项目的高频操作DOM问题 
        * 从timeline图中，还可以挖掘很多页面性能指标，包括但不限于fps
        * 每个点对应的优化点是什么？在本项目中体现在哪些地方？具体怎么解决？效果怎么样？
    * 顶部导航条相关及scroll滚动优化 
        * 技术文档应抽出技术细节，不要将业务逻辑放出来（如：谁也不知道业务为什么非iphone时不采用优化方案）
        * 此处缺少问题描述和现象反映，实际应优化的animationShow并没有看到
* 操作DOM的性能瓶颈及影响性能的原因 
    * 页面的加载过程及浏览器的渲染原理
        * 贴了一下页面加载过程和字段说明，并没有讨论指标和如何分析页面性能优劣
    * DOM操作影响页面性能的核心问题 
        * 补充了第一部分没有交代的影响 重绘 和 重构 的行为，列举出来，但没有清晰的分类
* 针对操作DOM的性能优化方法 
    * 减少在循环内进行DOM操作
        * 需要数据/实操说明优化效果如何
    * 进行DOM缓存
        * 其实1、2点的优化手段相同
    * 只控制DOM节点的显示或隐藏，而不是直接去改变DOM结构 
        * 没有代码对比，可以讨论一下优化前后的思路对比（主要在这一过程中讨论问题点）
        * 相对使用框架，如果使用原生的方法实现这一思路，对比如何？性能低下会不会有框架问题？
        * 错别字
    * 操作DOM前，先把DOM节点删除或隐藏 
        *  需要数据/实操说明优化效果如何
    * 一次性修改样式和属性，不要每次只修改一个
        * 并没有理解优化后代码的优化点，如何验证此处可以提升性能？
* 其他的影响页面性能的因素及解决办法
    * 网页(加载)性能优化，其实可以铺开写很多东西，此处罗列方法和思路并没有达到想要的目的
    * 建议拆出新命题专门研究，本文着重细致研究DOM操作的性能最好(不论是否高频)
        * DNS寻址时间 
        * 首字节加载时间 
        * request请求耗时 
        * 解析DOM树结构时间 

希望文档进一步沉淀，逐步完善内容。
其他意见或建议可以直接跟本邮件回复~

----
----

> 以下是新同学 @雪婷 在吸收导师 @鹏飞 点评后，修正版的正文

## 一、高频操作DOM会导致的问题

DOM的修改会导致`重绘`和`重构`，重绘意味着网页样式的改变比如背景颜色、字体颜色等，重构意味着结构的改变，消耗性能要大于重绘，浏览器不会在js执行的时候更新dom，而是会把这些dom操作存放在一个队列中，在js执行完之后按顺序一次性执行完毕，因此在js执行过程中用户一直在被阻塞。

### 1.1 年会抽奖项目的高频操作DOM问题

在最近做的年会抽奖项目中，就遇到了这样的高频操作DOM，严重影响页面性能的问题，在经历几轮抽奖后，文字滚动速度越来越慢，肉眼能感受到与第一次抽奖时文字滚动速度的明显差别，如持续时间过长或轮次过多，还会造成浏览器假死现象。

实现demo: [https://gxt19940130.github.io/demo/dom.html](https://gxt19940130.github.io/demo/dom.html)

衡量页面性能一个重要的指标是fps，即帧率（每秒帧数），帧率越高，页面运行越流畅。
由下图demo的timeline可以看出，fps显示为红色的占多数，这个demo中的帧率多数在20~45fps之间，页面会出现严重的掉帧的情况，当帧率低于24fps时，肉眼就会感觉到页面存在卡顿现象，所以用这种频繁操作DOM来实现文字滚动效果的方法写出的页面性能很差。

![enter image description here](http://img002.qufenqi.com/products/ef/eb/efeb57b2d53bf06f4e4cfd6ec0b7e034.png)

![enter image description here](http://img002.qufenqi.com/products/1a/a4/1aa4afc11fff82f5f5563caa812a8013.png)


针对该项目中的问题，采取的解决方法是：

* 一次性生成全部`<li>`，并且隐藏这些`<li>`，随机生成一组随机数数组，只有index与数组里面的随机数相等时，才显示该位置的`<li>`。
* 用`requestAnimationFrame`取代`setTimeout`不断生成随机数。

> requestAnimationFrame与setTimeout和setInterval类似，都是通过递归调用同一个方法不断更新页面。但是setTimeout和setInterval都存在性能上的问题，而requestAnimationFrame在运行时，浏览器会自动优化方法的调用，并且如果页面不是激活状态下的话，动画会自动暂停，有效节省了CPU开销。

在采用上面的方法进行优化后，在经历多轮抽奖后，文字滚动速度依旧正常，网页性能良好，不会出现文字滚动速度越来越慢，最后导致浏览器假死的现象。

### 1.2 顶部导航条相关及scroll滚动优化

顶部导航条要求当页面滚动到某个区域时，对应该区域的导航条在设置的显示范围内吸顶显示，因此需要监听页面的scroll事件，并在页面滚动时进行计算和DOM操作。

```javascript
// 在页面滚动时对显示范围进行计算
// 延迟到整个dom加载完后再调用，并且异步到所有事件后执行
$(function(){
//animationShow优化滚动效果，scrollShow为实际计算显示范围及操作DOM的函数
 setTimeout( function() {
     window.Scroller.on('scrollend', animationShow);
        window.Scroller.on('scrollmove', animationShow);
    })
});
function animationShow(){
   return window.requestAnimationFrame ?window.requestAnimationFrame(scrollShow) : scrollShow();
}
```

scroll事件被触发的频率高、间隔近，如果此时进行DOM操作或计算并且这些DOM操作和计算无法在下一次scroll事件发生前完成，就会造成掉帧、页面卡顿，影响用户体验。

针对该项目中的问题，采取的解决方法是：

* 尽量控制DOM的显示或隐藏，而不是删除或添加。页面加载时根据当前页面中吸顶导航的数量复制对应的DOM，并且隐藏这些导航。当页面滚动到指定区域后，显示对应的导航。
* 一次性操作DOM，将复制的DOM存储到数组中，将该数组append到对应的父节点下，而不是根据复制得到DOM的数量依次循环插入到父节点下。
* 多做缓存，如果某个节点将在后续进行多次操作，可以将该节点利用变量存储起来，而不是每次进行操作时都去查找一遍该节点。

## 二、DOM操作影响页面性能的核心问题

页面加载时，浏览器会根据HTML构建DOM树，再根据CSS和DOM树构建渲染树。如前面所说，`DOM操作影响页面性能的核心问题主要是页面的重绘和重排`。

* 重绘是指一些样式的修改，元素的位置和大小都没有改变；
* 重排是指元素的位置或尺寸发生了变化，浏览器需要重新计算渲染树，而新的渲染树建立后，浏览器会重新绘制受影响的元素。因此页面重绘的速度要比页面重排的速度快，在页面交互中要尽量避免页面的重排操作。

导致页面重排的一些操作：

* DOM元素的几何属性的变化
	* 例如改变DOM元素的宽高值时，原渲染树中的相关节点会失效，浏览器会根据变化后的DOM重新构建渲染树中的相关节点。如果父节点的几何属性变化时，还会使其子节点及后续兄弟节点重新计算位置等，造成一系列的重排。
* DOM树的结构变化
	* 添加DOM节点、修改DOM节点位置及删除某个节点都是对DOM树的更改，会造成页面的重排。浏览器布局是从上到下的过程，修改当前元素不会对其前边已经遍历过的元素造成影响，但是如果在所有的节点前添加一个新的元素，则后续的所有元素都要进行重排。
* 获取某些属性
	* 除了渲染树的直接变化，当获取一些属性值时，浏览器为取得正确的值也会发生重排，这些属性包括：`offsetTop`、`offsetLeft`、 `offsetWidth`、`offsetHeight`、`scrollTop`、`scrollLeft`、`scrollWidth`、`scrollHeight`、 `clientTop`、`clientLeft`、`clientWidth`、`clientHeight`、`getComputedStyle()`。
* 浏览器窗口尺寸改变
	* 窗口尺寸的改变会影响整个网页内元素的尺寸的改变，即DOM元素的集合属性变化，因此会造成重排。

导致页面重绘的操作

* 应用新的样式或者修改任何影响元素外观的属性
	* 只改变了元素的样式，并未改变元素大小、位置，此时只涉及到重绘操作。
* 重排一定会导致重绘
	* 一个元素的重排一定会影响到渲染树的变化，因此也一定会涉及到页面的重绘。

## 三、针对操作DOM的性能优化方法

### 3.1 减少在循环内进行DOM操作，在循环外部进行DOM缓存

```javascript
//优化前代码
var _li = $("<li>"),
    _dom = $("<div>"),
    timer = null;
for (var i = 0; i < 50; i++) {
 //随机生成50个li，插入到ul列表中
    $(".list-ul").append(_li.clone());
}
```

```javascript
//优化后代码
var _li = $("<li>"),
    _dom = $("<div>"),
    _lis = document.getElementsByTagName("li"),
    timer = null,
    _arr = [];
for (var i = 0; i < 50; i++) {
 //随机生成50个li，存入到数组中
    _arr.push(_li.clone());
}
//将生成好的全部li一次性append到ul中
$(".list-ul").append(_arr);
```

优化前的代码中，对于`$(".list-ul")`元素进行了50次的append，即进行了50次的DOM操作。而对于优化后的代码，在append操作前，先将所有`<li>`存入数组中，最后只进行了一次append，因此性能会有所提高。

### 3.2 只控制DOM节点的显示或隐藏，而不是直接去改变DOM结构

在年会抽奖项目中频繁操作DOM来控制文字滚动的方法（demo： https://gxt19940130.github.io/demo/dom.html ），导致页面性能很差，最后修改为如下代码。

```javascript
<div class="staff-list" :class="list">
   <ul class="staff-list-ul">
       <li v-for="item in staffList" v-show="isShow($index)">
           <div>{ item.staff_name | addSpace } </div>
           <div class="staff_phone">{ item.phone_no } </div>
       </li>
   </ul>
</div>
```
上面代码的优化原理即先生成所有DOM节点，但是所有节点均不显示出来，利用vue.js中的`v-show`，根据计算的随机数来控制显示某个`<li>`，来达到文字滚动效果。

如果采用jquery，则需要将生成的所有`<li>`全部存放在`<ul>`下，并且隐藏它们，在根据生成的随机数组，利用jquery查找index与生成的随机数对应的`<li>`并显示，达到文字滚动效果。

### 3.3 操作DOM前，先把DOM节点删除或隐藏

```javascript
list.style.display = "none";  
for (var i=0; i < items.length; i++){  
    var item = document.createElement("li");  
    item.appendChild(document.createTextNode("Option " + i);  
    list.appendChild(item);  
}  
list.style.display = "";
```

display属性值为none的元素不在渲染树中，因此对隐藏的元素操作不会引发其他元素的重排。如果要对一个元素进行多次DOM操作，可以先将其隐藏，操作完成后再显示。这样只在隐藏和显示时触发2次重排，而不会是在每次进行操作时都出发一次重排。

### 3.4 一次性修改样式和属性，不要每次只修改一个
```javascript
//优化前代码
element.style.backgroundColor = "blue";  
element.style.color = "red";  
element.style.fontSize = "20px";
```

```javascript
//优化后代码
//js操作
.newStyle {  
    background-color: blue;  
    color: red;  
    font-size: 20px;  
}  
element.className = "newStyle";
//jquery操作
$(element).css({
	background-color: blue;  
    color: red;  
    font-size: 20px; 
})
```
优化前的代码每一次更改样式都会查找一次该元素进行一次DOM操作，而优化后的代码，对于要修改的几个样式，都是只进行一次查找操作，因此只进行了一次DOM操作，避免了多次重绘或者重排。
