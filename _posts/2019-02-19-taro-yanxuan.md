---
layout: post
title: Taro 多端开发的正确姿势：打造三端统一的网易严选（小程序、H5、React Native）
author: 蔡珉星
tag: 多端统一开发,taro,架构
digest: 结合趣店 FED 在过去小半年的实践经验，我们开发了首个 Taro 三端统一应用：taro-yanxuan（高仿网易严选微信小程序），用以探讨 Taro 多端开发的正确姿势。
---

![overview](/public/images/taro-yanxuan/overview.png)

## 前言

趣店 FED 早在去年 10 月份就已全面使用 [Taro](https://github.com/NervJS/taro) 框架开发小程序（当时版本为 1.1.0-beta.4），至今也上线了 2 个微信小程序、2 个支付宝小程序。

之所以选用 Taro，解决微信小程序原生开发的痛点是一方面，另一方面团队也有多端统一开发的诉求，Taro 无疑是当时支持最好的。另外 React 也符合团队的整体技术栈，可显著降低团队学习成本。

可以说，Taro 在小程序端、H5 端支持程度已经不错，也有不少上线实例可以查看，但在 React Native 的支持上，Github 中公开的项目在 RN 这块均未适配：

![Github 相关项目](/public/images/taro-yanxuan/github-project.png)

这种现况可以理解，毕竟要做到多端统一是有一定难度的，需准确把握各端差异，并做出合理取舍，而 Taro 虽以多端为设计目标，可重心在小程序端，没有对多端做出一定的开发约束，无从下手也便正常。笔者曾在 2018 iWeb 峰会 - 厦门站做过[《多端统一开发实践》](http://cdn.jsnewbee.com/files/%E5%A4%9A%E7%AB%AF%E7%BB%9F%E4%B8%80%E5%BC%80%E5%8F%91%E5%AE%9E%E8%B7%B5.pdf)的分享，提到用 Taro 开发 RN 端的坑与大体思路，并加以实践。

结合趣店 FED 在过去小半年的实践经验，我们开发了首个 Taro 三端统一应用：taro-yanxuan（高仿网易严选微信小程序），用以探讨本文的重点：Taro 开发多端应用的正确姿势。

相关代码已开源：<https://github.com/js-newbee/taro-yanxuan>。

## 在线预览

可在线预览 H5、RN 端（直接调用了网易严选接口，若要体验登录、购物车功能，请使用网易邮箱账号登录）：

| 微信小程序 | H5 - [访问链接](http://jsnewbee.com/taro-yanxuan/) | React Native |
| :--------: | :--------:| :--------: |
| 请 clone 代码本地运行 | ![H5](/public/images/taro-yanxuan/h5-qr-code.png) | [Expo Snacks](https://snack.expo.io/@caiminxing/taro-yanxuan) |

如下是 **React Native** 的运行截图：

| 首页、分类 | 二级分类、详情 | 购物车、个人 |
| :--------: | :--------:| :--------: |
| ![首页、分类](/public/images/taro-yanxuan/video-01.gif) | ![二级分类、详情](/public/images/taro-yanxuan/video-02.gif) | ![购物车、个人](/public/images/taro-yanxuan/video-03.gif) |

## 样式管理

样式管理是多端开发的首要挑战，因为 React Native 与一般 Web 样式支持度差异较大，上述几个未适配 RN 的多端项目多数已栽在样式上了，用到了大量 RN 不支持的样式，这种情况再要去兼容 RN 无异于重写页面，想必也是有心无力了。这也是本文所强调的，需把握正确的多端开发姿势。

样式上 H5 最为灵活，小程序次之，RN 最弱，统一多端样式即是对齐短板，**也就是要以 RN 的约束来管理样式，同时兼顾小程序的限制**，核心可以用三点来概括：

* 使用 Flex 布局
* 基于 BEM 写样式
* 采用 style 属性覆盖组件样式

### 使用 Flex 布局

在进一步阐述之前，需先了解 RN 端几个影响样式方案的主要差异：

* `display` 只有 `flex / none`，`position` 只有 `relative / absolute`；
* 不支持标签选择器、子代选择器、伪元素，不支持 `background: url()` 等；
* 文本要用 `Text` 标签包裹，文本的样式不能加在 `View` 标签上，只能加在 `Text` 标签上。

使用 Flex 布局，不单单是因为 RN 的 `View` 标签有默认样式 `display: flex; flex-direction: column`，更重要的是 Flex 可以解决幽灵空白问题：

``` jsx
// View 标签高度不会是 100px，图片下方会有几像素空白，称为幽灵空白
<View>
  <Image src={...} style={{ height: '100px' }}
</View>
```

常规解决方案是在 View 标签上设置 `font-size / line-height: 0`, 或 Image 标签 `display: inline-block` 等，但这些在 RN 中都不支持，给 View 标签设置 `display: flex` 算是唯一可靠方案了。

何况 Flex 布局能力强大，为啥不用呢？只需要注意一点，RN 中 View 标签默认主轴方向是 `column`，如果不将其他端改成与 RN 一致，就需要在所有用到 `display: flex` 的地方都显式声明主轴方向。

### 基于 BEM 写样式

RN 实际上只支持一种样式声明方式，即声明 style 属性：

``` js
<View style={{ height: '100%' }}
```

这也导致 Taro 在 RN 端基本只支持 class 选择器这一种写法（最终编译成对象字面量），BEM（Block Element Modifier）在此处就恰如其分的发挥了作用：

1. 避免样式冲突（RN、小程序样式独立，但 H5 不是）
2. 自解释、语义化

例如每行 2 个元素的列表，每行最后 1 个元素有特定样式，用伪元素选择器 `:nth-child(even)` 很容易实现，在 RN 中就需要自行计算了：

``` jsx
{list.map((item, index) => (
  <View className={classNames('block__element',
    index % 2 === 1 && 'block__element--even' 
  )} />
)}
```

基于 BEM 写 class 样式，不依赖其他选择器，虽然会让代码稍显繁琐，但也能保证多端都是行得通的，不存在支持问题。

### 采用 style 属性覆盖组件样式

小程序、RN 在页面、组件间传递样式时均有问题：

``` jsx
// 目前 Taro RN 端还未实现往组件传递 className 对应样式
<CompA compClass='my-style' />

// CompA，样式不生效
<View className={this.props.compClass} />
```

上述场景小程序虽可通过组件外部样式 externalClasses 实现，但官网文档有强调 "在同一个节点上使用普通样式类和外部样式类时，两个类的优先级是未定义的，因此最好避免这种情况"；用全局样式倒是可以，但这样样式就不好维护了。

那么，通过 style 传递、覆盖组件样式也就成了唯一可选方案了。需要注意一点，样式文件是会经过编译处理兼容多端的，但 style 方式需要运行时兼容：

``` jsx
<Comp style={postcss({ background: '#fff' })} />

// 简单演示，如 RN 不支持 background，需改成 background-color
function postcss(style) {
  const { background, ...restStyle } = style
  const newStyle = {}
  if (background) {
    newStyle.backgroundColor = background
  }
  return { ...newStyle, ...restStyle }
}
```

从这个角度看，styled-components 或许是多端开发的最佳样式方案，然而 Taro 还不支持，且微信小程序官方文档中也提到 "尽量避免将静态的样式写进 style 中，以免影响渲染速度"，全部样式都用写到 style 中恐怕就不靠谱了，但只用来覆盖少量样式不见得会有太大影响。

### 样式兼容

即便是把握了如上样式管理思路，多端样式差异的问题依然存在，例如 `white-space: nowrap` 这个样式在 RN 端会报错，Taro 有提供解决方案：

``` css
.text {
  /*postcss-pxtransform rn eject enable*/
  white-space: nowrap;
  /*postcss-pxtransform rn eject disable*/
}
```

但项目中不止一处会有这个问题，都这样写实在不太美观，可以用 Sass mixins 稍微封装下：

``` sass
@mixin eject($attr, $value) {
  /*postcss-pxtransform rn eject enable*/
  #{$attr}: $value;
  /*postcss-pxtransform rn eject disable*/
}

.text {
  @includes eject(white-soace, nowrap);
}
```

Sass mixins 并不能解决差异，但对于部分各端不兼容的样式，通过 Sass mixins 统一处理是比较合理的方式，代码相对美观也方便维护。

## 端能力差异

相较于样式，端能力的差异倒是还好，各端差异是客观存在的，更不用说 RN 在 iOS 与 Android 上就已存在大量差异。

应对端能力差异，要么改变实现思路，例如 RN 端还不支持 `Taro.(get/set)StorageSync`，那就改用 `async / await` + `Taro.(get/set)Storage` 实现，要么就得使用环境判断方式了。

Taro 提供 `process.env.TARO_ENV` 用于环境判断，多数小的差异都可以用这种方式来解决： 

``` js
function foo() {
  if (process.env.TARO_ENV === 'weapp') {
    // 微信小程序逻辑
  }
  if (process.env.TARO_ENV === 'h5') {
    // H5 逻辑
  }
  if (process.env.TARO_ENV === 'rn') {
    // RN 逻辑
  }
}
```

这个时候也比较考验开发者的封装能力了，一般是建议将这些差异逻辑的判断统一起来，例如在 src/utils 中进行封装，对外提供一致的接口，尽量不要在业务页面中杂糅太多的判断。

而对于简单的环境判断处理不了的问题，就只能动用原生开发了，例如 Taro 还不支持 RN 端的 WebView 组件，就需要自己用原生 RN 实现：

``` jsx
// Taro 页面，根据环境引入 RN 原生页面
import { WebView } from '@tarojs/components'
const WebViewRN = process.env.TARO_ENV === 'rn' ? require('./rn').default : null

export default class extends Component {
  render() {
    return process.env.TARO_ENV === 'rn' ?
      <WebViewRN src={this.url} /> :
      <WebView src={this.url} />
  }
}

// 原生 RN 页面，从 react-native 引入 WebView
import Taro, { Component } from '@tarojs/taro'
import { WebView } from 'react-native'

export default class WebViewRN extends Component {
  render() {
    return <WebView source={{ uri: this.props.src }} />
  }
}
```

`process.env.TARO_ENV` 的处理是编译时而不是运行时，也就是说若不是编译 RN，上述用原生写的 RN 页面不会被打包，保证了编译成其他端时不会引入不支持的内容。

原生页面能够引入，多端问题也就有了基本的实现保障。

## Taro RN 端的坑

Taro RN 端目前小问题还是不少的，本项目开发过程中也顺带解了几个 bug：

![给 Taro 提的 pr](/public/images/taro-yanxuan/pr.png)

除此之外还有好几个问题，时间关系还未提 pr 解决，暂且先绕过，但其中有两个坑还是值得一说的。

### onClick

RN 的 View 标签不支持 onClick ，但这又是很通常的需求，原生解决方式是套一层 Touchable 组件，如：

``` jsx
<TouchableOpacity onPress={this.handlePress}>
  <View>{...}</View>
</TouchableOpacity>
```

而 Taro 是引入 `PanResponder` 响应用户操作：

``` jsx
<View
    {...PanResponder.carete({ ...})}
    style={wrapperStyle}
>
    <WrappedComponent style={innerStyle} />
</View>
```

问题在于这样多嵌套了一层 View，并把样式拆分成 wrapperStyle、innerStyle 分别应用，但样式拆分有问题，导致绑定 onClick 之后元素的样式错乱了，这点在开发过程中还是相当坑的。

### 宽高自适应

onClick 的问题也还好，改改样式能绕过去，宽高自适应的坑就比较尴尬了。

小程序、H5 可用 `rpx / em` 实现自适应，而 RN 的自适应方案麻烦些，一般需通过 `Dimensions` 获取宽高再进行换算。Taro.pxTransform() 可解决该问题，但编译 RN 端样式文件时并没有考虑这点，即 `width: 100px` 会被编译成 `width: 50`，而不是 `width: Taro.pxTransform(100)`，无法适配屏幕不同的屏幕尺寸。

因此，目前 Taro RN 端还不好做到自适应，要么非百分比的宽高都用 style + Taro.pxTransform()，要么就得自己写个脚本去处理编译后的样式文件。

这两个问题都提了 issue [2204](https://github.com/NervJS/taro/issues/2204) [2205](https://github.com/NervJS/taro/issues/2205)，有需要的可以关注下解决进度

## 其他

要做到多端统一，能说的细节点实在太多，上述实现思路虽然简单，但背后也都是隐含着对各端差异的斗争与取舍，本文也仅是列出最基本的几点，用于阐述 Taro 多端开发的核心思路。

本项目代码没有做过多封装，方便阅读，也实现了足够多的样式细节进行踩坑，具体涉及的踩坑点、注意事项都在代码中以注释 `// TODO`（Taro 还未支持的）、`// NOTE`（开发技巧、注意事项）注明了，更多内容就有待各位去实践、体会了。

![注释](/public/images/taro-yanxuan/comment.png)

## 总结

如前言所说，Taro 虽然是以多端为设计目标，但重心是小程序端，RN 端目前的支持情况不算特别理想。但充分理解多端差异、掌握正确的多端开发姿势（特别是样式管理方面，避免项目成型后再去兼容需要大动刀斧）之后，在简单的项目上是完全可以一展拳脚的。

若说 2 个礼拜开发一个小程序，是稀疏平常的事，但 2 个礼拜即搞定了小程序端（微信、支付宝、百度等等），还搞定了 H5、React Native 端，后续更新也只要改一处地方，这产出、维护效率就实在太惊人了，这大抵也就是 "Write once, run anywhere" 的魅力所在（虽然在前端领域极容易发展成 "Write once, **debug** everywhere" 😂）

相信随着小程序热度不断上升，还会有更多优秀的开源框架、解决方案涌现。而我们不倾向于造轮子，更关注基于现有方案如何更好地去开发多端应用。若有兴趣的前端小伙伴，不妨加入我们，一起搞事 caiminxing#qudian.com 😁

项目开源地址：<https://github.com/js-newbee/taro-yanxuan>
