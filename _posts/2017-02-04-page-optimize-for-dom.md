---
layout: post
title: 高频dom操作和页面性能优化探索
author: 高雪亭
tag: DOM操作,性能优化,导师点评
digest: 新同学完成一个年会抽奖项目后，对项目中遇到的些许技术问题进行深度思考和总结；对于前端老司机来说，这些并不是什么困难的技术点，但对于新同学来说，能有这样的思考和总结，是非常值得赞许的。这里尤其值得关注的，是文末导师对新人总结的细心点评，及其负责任的帮助新同学成长。
---

> 新同学完成一个年会抽奖项目后，对项目中遇到的些许技术问题进行深度思考和总结；对于前端老司机来说，这些并不是什么困难的技术点，但对于新同学来说，能有这样的思考和总结，是非常值得赞许的。这里尤其值得关注的，是`文章末尾`导师对新人总结的细心点评，及其负责任的帮助新同学成长。


> 以下是新同学 @雪亭 在吸收导师 @鹏飞 点评后，修正版的正文

## 一、DOM操作影响页面性能的核心问题


-----
通过js操作DOM的代价很高，影响页面性能的主要问题有如下几点：

* 访问和修改DOM元素
* 修改DOM元素的样式，导致`重绘`或`重排`
* 通过对DOM元素的事件处理，完成与用户的交互功能

-----
> DOM的修改会导致`重绘`和`重排`。
>
* 重绘是指一些样式的修改，元素的位置和大小都没有改变；
* 重排是指元素的位置或尺寸发生了变化，浏览器需要重新计算渲染树，而新的渲染树建立后，浏览器会重新绘制受影响的元素。

页面重绘的速度要比页面重排的速度快，在页面交互中要尽量避免页面的重排操作。浏览器不会在js执行的时候更新DOM，而是会把这些DOM操作存放在一个队列中，在js执行完之后按顺序一次性执行完毕，因此在js执行过程中用户一直在被阻塞。

### 1.页面渲染过程

一个页面更新时，渲染过程大致如下：

![enter image description here](http://img002.qufenqi.com/products/1b/88/1b882c131d28eb11a0cbc9d5f0173720.jpg)

* JavaScript: 通过js来制作动画效果或操作DOM实现交互效果
* Style: 计算样式，如果元素的样式有改变，在这一步重新计算样式，并匹配到对应的DOM上
* Layout: 根据上一步的DOM样式规则，重新进行布局（`重排`）
* Paint: 在多个渲染层上，对新的布局重新绘制（`重绘`）
* Composite: 将绘制好的多个渲染层合并，显示到屏幕上

在网页生成的时候，至少会进行一次布局和渲染，在后面用户的操作时，不断的进行重绘或重排，因此如果在js中存在很多DOM操作，就会不断地出发重绘或重排，影响页面性能。

### 2.DOM操作对页面性能的影响

如前面所说，DOM操作影响页面性能的核心问题主要在于DOM操作导致了页面的`重绘`或`重排`，为了减少由于重绘和重排对网页性能的影响，我们要知道都有哪些操作会导致页面的重绘或者重排。

#### 2.1 导致页面重排的一些操作：
* 内容改变
	* 文本改变或图片尺寸改变
* DOM元素的几何属性的变化
	* 例如改变DOM元素的宽高值时，原渲染树中的相关节点会失效，浏览器会根据变化后的DOM重新排建渲染树中的相关节点。如果父节点的几何属性变化时，还会使其子节点及后续兄弟节点重新计算位置等，造成一系列的重排。
* DOM树的结构变化
	* 添加DOM节点、修改DOM节点位置及删除某个节点都是对DOM树的更改，会造成页面的重排。浏览器布局是从上到下的过程，修改当前元素不会对其前边已经遍历过的元素造成影响，但是如果在所有的节点前添加一个新的元素，则后续的所有元素都要进行重排。
* 获取某些属性
	* 除了渲染树的直接变化，当获取一些属性值时，浏览器为取得正确的值也会发生重排，这些属性包括：`offsetTop`、`offsetLeft`、 `offsetWidth`、`offsetHeight`、`scrollTop`、`scrollLeft`、`scrollWidth`、`scrollHeight`、 `clientTop`、`clientLeft`、`clientWidth`、`clientHeight`、`getComputedStyle()`。
* 浏览器窗口尺寸改变
	* 窗口尺寸的改变会影响整个网页内元素的尺寸的改变，即DOM元素的集合属性变化，因此会造成重排。

#### 2.2 导致页面重绘的操作
* 应用新的样式或者修改任何影响元素外观的属性
	* 只改变了元素的样式，并未改变元素大小、位置，此时只涉及到重绘操作。
* 重排一定会导致重绘
	* 一个元素的重排一定会影响到渲染树的变化，因此也一定会涉及到页面的重绘。

## 二、高频操作DOM会导致的问题

接下来会分享一下在平时项目中由于高频操作DOM影响网页性能的问题。

### 1. 抽奖项目的高频操作DOM问题

#### 1.1 存在的问题

在最近做的抽奖项目中，就遇到了这样的由于高频操作DOM，导致页面性能变差的问题。在经历几轮抽奖后，文字滚动速度越来越慢，肉眼能感受到与第一次抽奖时文字滚动速度的明显差别，如持续时间过长或轮次过多，还会造成浏览器假死现象。

>实现demo: [https://gxt19940130.github.io/demo/dom.html](https://gxt19940130.github.io/demo/dom.html)

#### 1.2 问题分析

下图为抽奖时文字滚动过程中的timeline记录。
![enter image description here](http://img002.qufenqi.com/products/b9/92/b9924f828168e8f9ae67ae665c6eb620.png)

> timeline分析：
>
> 1、FPS:最上面一栏为绿色柱形为帧率(FPS)，顶点值为60fps，上方红色方块表示长帧，这些长帧被Chrome称为jank(卡顿)。
>
> 2、CPU:第二栏为CPU，蓝色表示`loading`（网络通信和HTML解析），黄色表示`scripting`（js执行时间），紫色表示`rendering`（样式计算和布局，即`重排`）， 绿色为`painting`（即`重绘`）。
> 
> PS：更多timeline使用方法可参考：[如何使用Chrome Timeline 工具（译）](http://www.jianshu.com/p/4da0f0bda768)

由上图可以看出，在文字滚动过程中红色方块出现频繁，页面中存在的卡顿过多。帧率的值越低，人眼感受到的效果越差。
参考文章：[脑洞大开：为啥帧率达到 60 fps 就流畅？](http://www.jianshu.com/p/71cba1711de0)。

-----
接下来选择一段长帧区域放大来看
![enter image description here](http://img002.qufenqi.com/products/f1/6e/f16ecdf14067cd5e49ba71dd2435d8ff.png)

在这段区域内最大一帧达到了49.7ms，帧率只有20fps，接下来看看这一帧里是什么因素耗时过长

-----
![enter image description here](http://img002.qufenqi.com/products/a6/82/a6828cda64e64788f446a344bb5a0762.png)

由上图可以看出，耗时最大的在scripting，js的执行时间达到了44.9ms，占总时间的93.2%，因为主要靠js计算控制DOM的显示内容，所以js运行时间过长。

-----
选取一段FPS值很低的部分查看造成这段值低的原因
![enter image description here](http://img002.qufenqi.com/products/cb/77/cb7733afac5fbce131637473bbaa405f.png)

由下图可看出主要为dom.html中的js执行占用时间。
![enter image description here](http://img002.qufenqi.com/products/4b/c9/4bc93fab71242e6378898245d294fcca.png)

点进dom.html文件，即可定位到该函数
![enter image description here](http://img002.qufenqi.com/products/2d/24/2d24c4d01143b8025d6156e2805199bb.png)

由此可知，主要是`rolling`这个函数执行时间过长，对该部分失帧影响较大。而这个函数的主要作用就是实现文字的滚动效果，也可以从代码中看出，这个函数利用的setTimeout来反复执行，并且在这个函数中存在着循环以及大量的DOM操作，造成了页面的失帧等问题。

#### 1.3 优化方案
针对该项目中的问题，采取的解决方法是：

* 一次性生成全部`<li>`，并且隐藏这些`<li>`，随机生成一组随机数数组，只有index与数组里面的随机数相等时，才显示该位置的`<li>`，虽然也会触发重排和重绘，但是性能要远远高于直接操作DOM的添加和删除。
* 用`requestAnimationFrame`取代`setTimeout`不断生成随机数。

> requestAnimationFrame与setTimeout和setInterval类似，都是通过递归调用同一个方法不断更新页面。
>
> * setTimeout()：在特定的时间后执行函数，而且只执行一次，如果在特定时间前想取消执行函数，可以用clearTimeout立即取消执行。但是并不是每次执行setTimeout都会在特定的时间后执行，页面加载后js会按照主线程中的顺序按序执行那个，如果在延迟时间内主线程不空闲，setTimeout里面的函数是不会执行的，它会延迟到主线程空闲时才执行。
> * setInterval()：在特定的时间间隔内重复执行函数，除非主动清除它，不然会一直执行下去，清除函数可以使用clearInterval。setInterval也会等到主线程空闲了再执行，但是setInterval去排队时，如果发现自己还在队列中未执行，就会被drop掉，所以可能会造成某段时间的函数未被执行。
> * requestAnimationFrame()：它不需要设置时间间隔，它会在浏览器每次刷新之前执行回调函数的任务。这样我们动画的更新就能和浏览器的刷新频率保持一致。requestAnimationFrame在运行时，浏览器会自动优化方法的调用，并且如果页面不是激活状态下的话，动画会自动暂停，有效节省了CPU开销。

在采用上面的方法进行优化后，在经历多轮抽奖后，文字滚动速度依旧正常，网页性能良好，不会出现文字滚动速度越来越慢，最后导致浏览器假死的现象。

> 实现demo: [https://gxt19940130.github.io/demo/demo_gxt/dom_by_vue.html](https://gxt19940130.github.io/demo/demo_gxt/dom_by_vue.html)

#### 1.4 优化前后FPS对比
优化前文字滚动时的timeline
![enter image description here](http://img002.qufenqi.com/products/32/17/3217ffb2474300c6fd3c025210074128.png)

-----
优化后文字滚动时的timeline
![优化后的timeline](http://img002.qufenqi.com/products/b3/1a/b31a72c274182c6f2a85233c412b5ef7.png)

优化前的代码对DOM操作很频繁，因此FPS值普遍偏低，而优化后可以看出红色方块明显减少，FPS值一直处于高值。
#### 1.5 优化前后CPU占用对比

优化前文字滚动时的timeline
![enter image description here](http://img002.qufenqi.com/products/b9/92/b9924f828168e8f9ae67ae665c6eb620.png)

-----
优化后文字滚动时的timeline
![优化后的timeline](http://img002.qufenqi.com/products/fe/d5/fed57e02c481d1355352dbda835f0e64.png)

-----
优化前js的CPU占用率较高，而优化后占用CPU的主要为渲染时间，因为优化后的代码只是控制了节点的显示和隐藏，所以在js上消耗较少，在渲染上消耗较大。

### 2.吸顶导航条相关及scroll滚动优化

#### 2.1 存在的问题

吸顶导航条要求当页面滚动到某个区域时，对应该区域的导航条在设置的显示范围内保持吸顶显示。涉及到的操作：

* 监听页面的scroll事件
* 在页面滚动时进行计算和DOM操作
	* 计算：计算当前所在位置是否为对应导航条的显示范围
	*  DOM操作：显示在范围内的导航条并且隐藏其他导航条

由于scroll事件被触发的频率高、间隔近，如果此时进行DOM操作或计算并且这些DOM操作和计算无法在下一次scroll事件发生前完成，就会造成掉帧、页面卡顿，影响用户体验。

#### 2.2 优化方案

针对该项目中的问题，采取的解决方法是：

* 尽量控制DOM的显示或隐藏，而不是删除或添加：
	>页面加载时根据当前页面中吸顶导航的数量复制对应的DOM，并且隐藏这些导航。当页面滚动到指定区域后，显示对应的导航。
	
* 一次性操作DOM:
	>将复制的DOM存储到数组中，将该数组append到对应的父节点下，而不是根据复制得到DOM的数量依次循环插入到父节点下。

* 多做缓存:
	>如果某个节点将在后续进行多次操作，可以将该节点利用变量存储起来，而不是每次进行操作时都去查找一遍该节点。

* 使用 requestAnimationFrame优化页面滚动

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

> 对于scroll的滚动优化还可以采用防抖（Debouncing）和节流（Throttling）的方式，但是防抖和节流的方式还是要借助于setTimeout，因此和requestAnimationFrame相比，还是requestAnimationFrame实现效果好一些。
>
> 参考文章：[高性能滚动 scroll 及页面渲染优化](http://web.jobbole.com/86158/)

## 三、针对操作DOM的性能优化方法总结


为了减少DOM操作对页面性能产生的影响，在实现页面的交互效果时一定要注意一下几点：

-----

### 1.减少在循环内进行DOM操作，在循环外部进行DOM缓存

```javascript
//优化前代码
function Loop() {
   console.time("loop1");
   for (var count = 0; count < 15000; count++) {
       document.getElementById('text').innerHTML += 'dom';
   }
   console.timeEnd("loop1");
}
```

```javascript
//优化后代码
function Loop2() {
    console.time("loop2");
    var content = '';
    for (var count = 0; count < 15000; count++) {
        content += 'dom';
    }
    document.getElementById('text2').innerHTML += content;
    console.timeEnd("loop2");
}
```
-----
两个函数的执行时间对比：

![enter image description here](http://img002.qufenqi.com/products/65/fd/65fd413b5fb38de9bfe9dbb85bfde402.png)

>优化前的代码中，每进行一次循环，都会读取一次`div`的`innerHtml`属性，并且对这个属性进行了重新赋值，即每循环一次就会操作两次DOM，因此执行时间很长，页面性能差。

>在优化后的代码中，将要更新的DOM内容进行缓存，在循环时只操作字符串，循环结束后字符串的值写入到`div`中，只进行了一次查找`innerHtml`属性和一次对该属性重新赋值的操作，因此同样的循环次数先，优化后的方法执行时间远远少于优化前。

### 2.只控制DOM节点的显示或隐藏，而不是直接去改变DOM结构

在抽奖项目中频繁操作DOM来控制文字滚动的方法（demo:https://gxt19940130.github.io/demo/dom.html 导致页面性能很差，最后修改为如下代码。

```html
<div class="staff-list" :class="list">
   <ul class="staff-list-ul">
       <li v-for="item in staffList" v-show="isShow($index)">
           <div>{item.staff_name | addSpace} </div>
           <div class="staff_phone">{item.phone_no} </div>
       </li>
   </ul>
</div>
```
上面代码的优化原理即先生成所有DOM节点，但是所有节点均不显示出来，利用vue.js中的`v-show`，根据计算的随机数来控制显示某个`<li>`，来达到文字滚动效果。

如果采用jquery，则需要将生成的所有`<li>`全部存放在`<ul>`下，并且隐藏它们，在根据生成的随机数组，利用jquery查找index与生成的随机数对应的`<li>`并显示，达到文字滚动效果。
优化后demo: https://gxt19940130.github.io/demo/demo_gxt/dom_by_vue.html

对比结果可查看`2.4`

### 3.操作DOM前，先把DOM节点删除或隐藏

```javascript
var list1 = $(".list1");
list1.hide();
for (var i = 0; i < 15000; i++) {
    var item = document.createElement("li");
    item.append(document.createTextNode('0'));
    list1.append(item);
}
list1.show();
```

display属性值为none的元素不在渲染树中，因此对隐藏的元素操作不会引发其他元素的重排。如果要对一个元素进行多次DOM操作，可以先将其隐藏，操作完成后再显示。这样只在隐藏和显示时触发2次重排，而不会是在每次进行操作时都出发一次重排。

----
页面rendering时间对比：
下图为同样的循环次数下未隐藏节点直接进行DOM操作的rendering时间（图一）和隐藏节点再进行DOM操作的rendering时间（图二）

![enter image description here](http://img002.qufenqi.com/products/2d/4e/2d4e94b4978467f1cab5c7b0f9640340.png)

![enter image description here](http://img002.qufenqi.com/products/c9/04/c904ad494ced5a0e1660ee95a0c7d5d3.png)

由对比图可以看出，总时间、js执行时间以及rendering时间都明显减少，并且避免了painting以及其他的一些操作。

### 4. 最小化重绘和重排

```javascript
//优化前代码
var element = document.getElementById('mydiv');
element.style.height = "100px";  
element.style.borderLeft = "1px";  
element.style.padding = "20px";
```
在上面的代码中，每对element进行一次样式更改都会影响该元素的集合结构，最糟糕情况下会触发三次重排。
优化方式：利用js或juqery对该元素的class重新赋值，获得新的样式，这样减少了多次的DOM操作。

```css
.newStyle {  
    height: 100px;  
    border-left: 1px;  
    padding: 20px;  
}  
```

```javascript
//js操作
element.className = "newStyle";
//jquery操作
$(element).css({
	height: 100px;  
    border-left: 1px;  
    padding: 20px; 
})
```

到此本文结束，如果对于问题分析存在不正确的地方，还请及时指出，多多交流。

> 参考文章：
>
> * [高性能JS-DOM](http://mp.weixin.qq.com/s/Khr9u1cacbBgWHvveL7DPA)
> * [Effective前端6：避免页面卡顿](https://zhuanlan.zhihu.com/p/25166666?refer=dreawer)
> * [前端性能优化](https://mp.weixin.qq.com/s?__biz=MjM5MTA1MjAxMQ==&mid=2651225651&idx=1&sn=63b8e8d654b76fd0fe5c5b10b1eaeaca&chksm=bd49a7b78a3e2ea1e20540faded15c199cec67133236697c9313a3984f47b583f038fb68a80a&mpshare=1&scene=1&srcid=0215KDxZbhyStawfAWsQEMTG&key=793f4c1da962693ce265a22ea3dc30b4d52108b8e31df5b241f574c47bc74725251ad2fd181eacfccc6236fac1ec8147749854bbe26c1e292af4ce38c97045f4f2a7548c2d48430b6a461f77ab8fafb6&ascene=0&uin=MTI2MDEwMjk4MQ%3D%3D)
> * [高性能滚动 scroll 及页面渲染优化](http://web.jobbole.com/86158/)
> * [脑洞大开：为啥帧率达到 60 fps 就流畅？](http://www.jianshu.com/p/71cba1711de0)
> * [如何使用Chrome Timeline 工具（译）](http://www.jianshu.com/p/4da0f0bda768)


----
----

## 附录：导师点评

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
> 导师`鹏飞`点评2017.02.17更新

本次的文章质量相对上篇提升了很多👍，但仍有些许地方值得进一步细化，以下是一些建议~

### 1、重绘与重排部分：可以调查并列举明确的导致原因，在后面的对比中可以有理有据，如果有文档的引用更有力
* 1.1、可以使用问题更明显的demo（最开始卡爆的那个），来体现出明显的问题
* 1.2、“当帧率低于24fps时，肉眼就会感觉到页面的卡顿现象”：指标等值，需要 明确的计算方式 或者 权威的说明性文档
* 1.2、“由上图可以看出，耗时最大的在scripting”：这个例子可以展开，比如：深入找到具体导致此处耗时过长的根源，举例中可以看到是一个匿名函数，进而追踪到
* 1.3、解决方法，“...并且隐藏这些<li>”：
    * 1、此处怎么隐藏，会不会导致重排、重绘？为什么要提出此解决方法，可以结合开始的调查分析原因
    * 2、对比 “requestAnimationFrame与setTimeout和setInterval类似” ，没有优缺点就没法空口下谁更好的结论
    * 3、“而requestAnimationFrame在运行时，浏览器会自动优化方法的调用”，从实现原理讲一下，不要用自动优化方法的调用这样模糊的语言带过去
* 1.4、优化前后对比： 此处4个图，没有说明性分析
    * 1、先全面对比，宏观分析大块优化优势
    * 2、可以摘出具体点，优化比较好的地方，进行详细对比（可以结合上述找到的地方，如何进行优化提升性能的）
* 1.5 ”顶部导航条”，叫做“吸顶导航”更好理解
    * 1、“尽量控制DOM的显示或隐藏，而不是删除或添加”：  同上面的问题，为什么这样做是解决方法？ 还有position定位等属性对display改变是否有影响（比如脱离文档流、提高层级等）
    * 2、“使用 requestAnimationFrame优化页面滚动”： 其实从参考文章找不到哪篇讲节流的了（比如:http://www.cnblogs.com/zztt/p/4098657.html）里，这种方案和 节流方案 的对比怎么样

### 2、这部分宜扔到开始的地方，先放出来理论调研，才能发现并解决问题

### 3、这部分宜做成总结的部分，质量不错

### 4、总结放在三里，每个小标写清楚就够了，没必要再换写法把三的标题写一遍 + 注意错别字

基本上这次的各点再完善一下，就可以作为一篇质量比较高的文章发布了~
