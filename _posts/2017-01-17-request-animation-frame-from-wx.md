---
layout: post
title: 从微信小程序重力感应API到requestAnimationFrame探索实现
author: 沈琦耀
tag: 微信小程序,重力感应,requestAnimationFrame
digest: 最近做微信小程序的开发时，想做一个靠感知手机方向，使页面上节点跟随移动的动画（即重力感应视差效果）功能。结果发现微信小程序有一些坑，本文主要通过这些坑展开，聊聊相关的一些技术知识。
---

最近做微信小程序的开发时，想做一个靠感知手机方向，使页面上节点跟随移动的动画（即重力感应视差效果）功能。结果发现微信小程序有一些坑：
- 微信小程序不支持html5的[DeviceOrientationEvent](https://developer.mozilla.org/zh-CN/docs/Web/API/Detecting_device_orientation)重力感应API，而是自己实现的[wx.onAccelerometerChange](https://mp.weixin.qq.com/debug/wxadoc/dev/api/accelerometer.html?t=2017112#wxonaccelerometerchangecallback)
- 这个API回调实现，频率为5次/s 

在这个背景下，要实现平滑的重力感应的视差体验就那么优雅了，因为人对至少60帧每秒的动画才会感觉流畅。最终实现的效果会有卡顿现象。

实现期间，想起好像有`requestAnimationFrame`这个跟动画相关的API，其功能表现与`setTimeout`类似，即隔一段时间调用一个回调函数。对于这个API，之前了解不深，这次拿起来产生了一个疑问：既生`setTimeout`，何生`requestAnimationFrame`？带着疑问，开始调研。

## 与setTimeout的不同
在MDN上，关于`requestAnimationFrame`的定义是：
> window.requestAnimationFrame()这个方法是用来在页面重绘之前，通知浏览器调用一个指定的函数，以满足开发者操作动画的需求。这个方法接受一个函数为参，该函数会在重绘前调用。

在这里，我产生一个疑问，所谓的“**在页面重绘之前**”，指的是，这个指定函数（以下简称cb）会在底层机制的运行下，在页面重绘之前调用；还是人为地设置一个间隔时间，去调用cb，导致重绘？

之前大概了解过页面的重排和重绘。动画能用重排和重绘来实现（这里指的是能用这两种途径来达到动画目的，而不是两者都适合用来实现动画），而定义里只提到重绘，没提到重排的原因，虽然我没去细究，但很重要一点肯定是因为，重绘性能远高于重排。所以动画不要通过left、margin等来实现，应该通过translate属性来实现。

既然说到translate，稍微延伸一下，动画如果要用translate，最好用tranlate3d。因为**较于tranlate，tranlate3d能得到更完整的GPU加速的支持**，使得性能更优。

言归正传，继续解决刚刚的疑问。往下阅读，发现这么一段解释：
> 如果你想做逐帧动画的时候，你应该用这个方法。这就要求你的动画函数执行会先于浏览器重绘动作。通常来说，被调用的频率是每秒60次，但是一般会遵循W3C标准规定的频率。如果是后台标签页面，重绘频率则会大大降低。

从这段话可以看出，在用了这个方法后，浏览器会根据自己的重绘频率，而每次重绘前会调用cb。用以下代码验证：

```javascript
var laststart
function test () {
  laststart && console.log(Date.now() - laststart)
  laststart = Date.now()
  requestAnimationFrame(test)
}
requestAnimationFrame(test)
```

得到结果，我所在的浏览器环境（mac + chrome[55.0.2883.95]）的重绘频率约为60次每秒：

![](http://img002.qufenqi.com/products/18/21/18211bd4ca1925b87e680390b65cfe76.png@250h)

对此我产生几个疑问：
- 问题1：如果我干扰了重绘的频率，是否还会是一个几乎保持在每秒60帧的频率呢（只针对提高频率进行探究）？
- 问题2：是否调用了requestAnimationFrame就会产生一定频率的重绘？
- 问题3：如果不调用requestAnimationFrame，在无其他代码去重绘页面的话，页面就不会重绘吗？

为了验证问题1，我加了这么一段代码：

```javascript
var x = 1
var style = document.querySelectorAll('.test')[0].style
function interference () {
  x *= -1
  style.transform = `translate3d(${x * 20}px, 0 ,0)`
  setTimeout(interference, 5)
}
interference()
```

得到的结果与上一个结果一致。

由此得出结论：cb的调用频率在人为干扰重绘频率的情况下，依旧我行我素。

等等，人为干扰重绘频率成功了吗？会不会虽然`interference`的调用频率为5ms一次，但浏览器的重绘频率依旧是约等于60次每秒，即`interference`虽然试图去触发浏览器5ms重绘一次，但浏览器只会阻塞住，等下一次浏览器默认频率重绘时再一起重绘？

为了探究这个问题，我将`interference`的setTimeout时间分别设置为16ms（简称为i16）、10ms（简称为i10）、8ms（简称为i8）、5ms（简称为i5），如果浏览器重绘频率无法人为干扰，因以下两个原因：
- `interference`的函数对`$('.test')`的改变为水平位移正负20px交替出现
- 浏览器默认重绘频率接近16ms一次

i8会因16ms中被调用2次，使得`$('.test')`回归原位而导致肉眼看到的`$('.test')`闪动频率最慢；而i16的调用频率和重绘频率最为接近，在这种情况下，肉眼看到的`$('.test')`闪动频率会是最快。

结果肉眼看到的闪动频率从高到低依次是：`i16 > i10、i5 > i8`。

故得出结论，用上面的方法，**无法人为干扰浏览器默认的重绘频率**。

那么是否有办法设置浏览器的重绘频率呢？没有查到直接答案，但在阮一峰的<a href="http://www.ruanyifeng.com/blog/2015/09/web-page-performance-in-depth.html">网页性能管理详解</a>中，有提到：
> 大多数显示器的刷新频率是60Hz，为了与系统一致，以及节省电力，浏览器会自动按照这个频率，刷新动画

证明浏览器的重绘频率和显示器的刷新频率相等。所以应该没有直接设置浏览器重绘频率的方法，毕竟页面重绘了，但显示器没刷新，影响了性能却没效果。

其实通过chrome自带的开发者工具里的timeline功能，就能清晰看到，上面的方法的间隔时间无论怎么设置，都不会改变重绘频率：

![](http://img002.qufenqi.com/products/fe/a9/fea9ade41a468176c806520299ee8973.png@450w)

有了这个工具，其余两个问题也迎刃而解：
- 问题2的答案：调用了requestAnimationFrame就会产生一定频率的重绘，只是这种情况下的重绘会因并无实质重绘内容，而历时极短。

![](http://img002.qufenqi.com/products/d9/43/d94342754a05580ff4bd955bd375e835.png@350w)

- 问题3的答案：如果不调用requestAnimationFrame，在无其他代码去重绘页面的话，页面就不会重绘

![D436903A-740E-42B9-A817-7FEB7BF84C1B.png](http://img002.qufenqi.com/products/b2/4f/b24f2a33e30f870bce34827989e15026.png@400w)

调研到这一步，发现requestAnimationFrame和setTimeout根本不是一回事。requestAnimationFrame是一个根据浏览器重绘频率来调用的方法，setTimeout则是一个计时器。定义不同，适用的场景也完全不同，也没有性能高低之分。

## 兼容性
MDN给出的requestAnimationFrame的兼容性如下：

![](http://img002.qufenqi.com/products/ce/90/ce9084c0ccd9adee452dbda3955ebcab.png@400w)

也就是说requestAnimationFrame肯定有兼容性问题。所以降级处理也是必须的。以下是降级代码：

```javascript
;(function() {

  var lastTime = 0;
  
  // 兼容各种浏览器
  var vendors = ['ms', 'moz', 'webkit', 'o'];
  for(var x = 0; x < vendors.length && !window.requestAnimationFrame; ++x) {
    window.requestAnimationFrame = window[vendors[x]+'RequestAnimationFrame'];
    window.cancelAnimationFrame = window[vendors[x]+'CancelAnimationFrame'] || window[vendors[x]+'CancelRequestAnimationFrame'];
  }

  // 降级处理
  if (!window.requestAnimationFrame) {
    window.requestAnimationFrame = function(callback, element) {
      // 保证如果重复执行callback的话，callback的执行起始时间相隔16ms
      var currTime = new Date().getTime();
      var timeToCall = Math.max(0, 16 - (currTime - lastTime));
      var id = window.setTimeout(function() { callback(currTime + timeToCall); },
        timeToCall);
      lastTime = currTime + timeToCall;
      return id;
    };
  }

  if (!window.cancelAnimationFrame) {
    window.cancelAnimationFrame = function(id) {
      clearTimeout(id);
    };
  }

}());
```

## requestAnimationFrame在微信小程序里的表现
- 微信iOS版小程序完全不支持requestAnimationFrame

## 结论
- requestAnimationFrame和setTimeout根本不是一回事，根据其定义，可以在不同场景下使用。
- 较于tranlate，tranlate3d能得到更完整的GPU加速的支持。
- 浏览器对页面的重绘有一个默认的最大频率，最大频率无法人为设置，也没有设置的必要。
- 调用了requestAnimationFrame就会产生一定频率的重绘，只是这种情况下的重绘会因并无实质重绘内容，而历时极短。
- 如果不调用requestAnimationFrame，在无其他代码去重绘页面的话，页面就不会重绘。


最后，附上我们趣店集团的小程序二维码：

![](http://img002.qufenqi.com/products/d2/c8/d2c8cd6b2bef586d726bcd7fa106416b.jpg@200w)