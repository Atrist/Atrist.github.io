# Vue 动画
CSS3 的出现，使得前端动画直接上了一个层次。我们可以抛掉落后的`.animate()`方法，通过 `transition`、`animation` 等样式属性，只需要配置起点终点、时间、加速度曲线，甚至还可以配置关键帧、循环播放等，就可以实现超级高大上的动效，同时还不用小心翼翼地避开性能问题。

而 Vue 则是再一次优化了动画的使用，提供了多组件过渡、多元素过渡、CSS 动画、Javascript 动画等各种方式，开发者可以很方便地实现一些常用的列表过渡、页面切换等动画效果。不知道大家平时用的多不多，但如果你也有仔细研究的话，会发现还是挺有意思的。

## 1. transition 组件
`<transition>`元素作为单个元素/组件的过渡效果，它只会把过渡效果应用到其包裹的内容上，而不会额外渲染 DOM 元素，也不会出现在检测过的组件层级中。
### 1. Vue 过渡常用方式
Vue 过渡主要分为两类: **CSS 动画**和**Javascript 动画**。

根据官方描述，Vue 在插入、更新或者移除 DOM 时，提供多种不同方式的应用过渡效果，举例如下：

- 在 CSS 过渡和动画中自动应用 class（CSS 动画）
- 可以配合使用第三方 CSS 动画库，如 Animate.css（CSS 动画）
- 在过渡钩子函数中使用 JavaScript 直接操作 DOM（Javascript 动画）
- 可以配合使用第三方 JavaScript 动画库，如 Velocity.js（Javascript 动画）

这些我们不一定全部会用到，常用的可能包括列表过渡、路由过渡、组件过渡一些方式。我们先来看看动画过渡的一些方法和原理。

### 2. 动画过渡的方法和原理
以前我写动画的日子里，通常是使用 `jQuery`，各种计算然后调用`.animate()`等。后面也接触了 `CSS3`，`transform` 到 `animation`，省去了不少的力气，CSS 的动画能力简直要上天了。而关于 `requestAnimationFrame`，如果说没怎么写过动画的，或许几乎没听过，但如果你做动画，大概都会需要知道这么一个方法。

#### 强大的 requestAnimationFrame
一般来说，大家常用的方式主要是来解决传说中浏览器刷新频率的问题。我们来看看 MDN 上是怎么介绍的：`window.requestAnimationFrame()`方法告诉浏览器您希望执行动画，并请求浏览器在下一次重绘之前调用指定的函数来更新动画。该方法使用一个回调函数作为参数，这个回调函数会在浏览器重绘之前调用。

解决什么问题呢？例如常用浏览器的绘制间隔是 `16.7ms`，如果开发者强制使用 `10ms` 的间隔进行`setInterval()`或者`setTimeout()`来绘制动画，就会出现掉帧的情况，动画会变得卡顿而不流畅。

使用 `requestAnimationFrame`，就是跟着浏览器的绘制频率走，如果浏览设备绘制间隔是 16.7ms, 就 16.7ms 绘制。这样就不会存在过度绘制的问题，动画不会掉帧。`requestAnimationFrame` 做的事情很简单，浏览器页面每次要重绘，就会通知 `requestAnimationFrame`，然后把需要绘制的内容一起带上进行绘制，这是资源非常高效的一种利用方式。

##### 使用方式
```js
window.requestAnimationFrame(callback);
```
##### callback
一个在每次需要重新绘制动画时调用的包含指定函数的参数。这个回调函数有一个传参，`DOMHighResTimeStamp`，指示从触发 `requestAnimationFrame` 回调到现在（重新渲染页面内容之前）的时间。
##### 返回值
一个 long 整数，请求 ID，也是回调列表中唯一的标识。可以传此值到`window.cancelAnimationFrame()`以取消回调函数。

总之，我们先来看看 Vue 里面的相关代码：
```js
// binding to window is necessary to make hot reload work in IE in strict mode
// 此处在不兼容时使用setTimeout进行向下兼容
const raf =
  inBrowser && window.requestAnimationFrame
    ? window.requestAnimationFrame.bind(window)
    : setTimeout;

export function nextFrame(fn: Function) {
  raf(() => {
    raf(fn);
  });
}
```
`nextFrame`就是 Vue 中用来 CSS 动画过渡的方法，可以在下一次重绘之前进行一些处理。后面讲 CSS 动画过渡的时候我们会再详细点分析。

#### 关键的 transitionend 和 animationend
这两个分别是 CSS 动画中的结束事件，transitionend 是 CSStransition（如 transform/scale 等）结束的触发事件，animationend 则是 CSS 动画（keyframe）结束的触发事件。
##### transitionend
transitionend 事件会在 CSS transition 结束后触发。以下两种情况不会触发：
1. 当 transition 完成前移除 transition 时，比如移除 css 的 transition-property 属性。
2. 在 transition 完成前设置 display 为"none"。

使用方式如下：
```js
// 在指定的元素上监听transitionend事件
element.addEventListener("transitionend", callback, false);
```
##### animationend
`animationend` 事件会在一个 CSS 动画完成时触发。与 `transitionend` 相似，`animationend` 的触发不包括完成前就已终止的情况，例如元素变得不可见或者动画从元素中移除。

##### 使用方式如下：
```js
// 在指定的元素上监听animationend事件
element.addEventListener("animationend", callback, false);
```
Vue 的 CSS 过渡中会使用到这两个事件，后续会讲到。相比 CSS transition，CSS 动画还有相关的事件，包括：
- **animationstart**: animationstart 事件会在 CSS 动画开始时触发。如果有 animation-delay 延时，事件会在延迟时效过后立即触发
- **animationiteration**: 循环动画中，在每次循环结束时触发
- **animationcancel**: animationcancel 事件会在 CSS 异常终止时触发（即在未触发 animationend 事件的情况下停止运行）
### 3. transition 组件
Vue 提供了 `transition` 的封装组件，可以给元素和组件添加 `entering/leaving` 过渡。

#### transition 的使用
我们看看有哪些情况下可以使用 transition：

- 条件渲染 (使用`v-if`)
- 条件展示 (使用`v-show`)
- 动态组件
- 组件根节点

`transition` 的封装组件，主要是结合组件生命周期，在一些特殊逻辑（如`v-if`、`v-show`、`v-for`）里增加钩子，来触发动画的实现。我们知道，Vue 里面实现动画过渡有 CSS 和 Javascript 两种，而两种又是可以结合的，当然这是通过预埋的钩子以及上面讲到的 CSS 事件的方式来实现的。至于具体的实现，后面分篇来讲一下，这里就不过多讲述了。

#### 来个 demo

我们直接来看看一个特别简单的例子：
```html
<!-- transition的使用 -->
<div id="demo">
  <button v-on:click="show = !show">
    Toggle
  </button>
  <transition name="fade">
    <p v-if="show">hello</p>
  </transition>
</div>
```
```css
/* 简单的css transition实现动画 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
  opacity: 0;
}
```
这里是使用的 CSS transition 方式来实现动画过渡的，我们看看在切换`v-if`的时候都发生了什么:
1. v-if绑定值的变更，导致插入或删除包含在 `transition` 组件中的元素。
2. 自动嗅探目标元素是否应用了 CSS 过渡或动画。这里的确使用 CSS 过渡，于是会在元素添加时添加 CSS 类名，并判断动画加载完之后删除 CSS 类名。
3.  如果过渡组件提供了 JavaScript 钩子函数，这些钩子函数将在恰当的时机被调用。（这里由于没有使用到，故不会执行）
4.  如果没有找到 JavaScript 钩子并且也没有检测到 CSS 过渡/动画，DOM 操作 (插入/删除) 在下一帧中立即执行。（当然在我们的例子中，这一步也不会生效）

## 2. CSS 过渡与动画
下面我们来认识下 CSS 过渡。
### 1. CSS transition
CSS transitions 提供了一种在更改 CSS 属性时控制动画速度的方法。 其可以让属性变化成为一个持续一段时间的过程，而不是立即生效的。比如，将一个元素的颜色从白色改为黑色，通常这个改变是立即生效的，使用 CSS transitions 后该元素的颜色将逐渐从白色变为黑色，按照一定的曲线速率变化。这个过程可以自定义。

通常将两个状态之间的过渡称为隐式过渡（implicit transitions），因为开始与结束之间的状态由浏览器决定。
#### transition 使用
CSS transitions 可以决定哪些属性发生动画效果 (明确地列出这些属性)，何时开始 (设置 delay），持续多久 (设置 duration) 以及如何动画 (定义 timing funtion，比如匀速地或先快后慢)。

CSS 过渡由简写属性 transition 定义是最好的方式：
```css
div {
  transition: <property> <duration> <timing-function> <delay>;
}
```
表 6-1 transition 属性说明

属性名	|说明
---|---
property|	指定哪个或哪些 CSS 属性用于过渡。只有指定的属性才会在过渡中发生动画，其它属性仍如通常那样瞬间变化。all 则为全部属性
duration|	指定过渡的时长。或者为所有属性指定一个值，或者指定多个值，为每个属性指定不同的时长
timing-function|	指定一个函数，定义属性值怎么变化。常用如 linear、ease
delay|	指定延迟，即属性开始变化时与过渡开始发生时之间的时长

下面是简写和不简写的例子：
```css
.short-for-transition {
  transition: margin-left 4s ease 2s;
}

.transition-detail {
  transition-property: margin-left;
  transition-duration: 4s;
  transition-timing-function: ease;
  transition-delay: 2s;
}
```
### 2. Vue 与 CSS transition
上面我们已经大概讲了 CSS transition 的使用方式，这里与 Vue 的结合也变得很简单，我们翻回刚刚用到的这个例子：
```html
<template>
  <!-- transition的使用 -->
  <div id="demo">
    <button v-on:click="show = !show">
      Toggle
    </button>
    <transition name="fade">
      <p v-if="show">hello</p>
    </transition>
  </div>
</template>

<script>
  new Vue({
    el: "#demo",
    data: {
      show: true
    }
  });
</script>

<style>
  /* 简单的css transition实现动画 */
  .fade-enter-active,
  .fade-leave-active {
    transition: opacity 0.5s;
  }
  .fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
    opacity: 0;
  }
</style>
```
我们在 transition 上设置了 name 的值为 fade，然后效果是 Vue 匹配对应的 fade-status 的类名。下面我们来讲解下上述代码都做了什么。
#### 过渡的类名
我们先来看看 Vue 里面提供了哪些类名：

表 6-2 Vue 中过渡类名

类名	|定义|	说明
---|---|---
`v-enter`	|定义进入过渡的开始状态|	在元素被插入时生效，在下一个帧移除
`v-enter-active`|	定义过渡的状态	|在元素整个过渡过程中作用，在元素被插入时生效，在 transition/animation 完成之后移除 这个类可以被用来定义过渡的过程时间，延迟和曲线函数
`v-enter-to`	|(2.1.8 版及以上)定义进入过渡的结束状态	|在元素被插入一帧后生效 (与此同时`v-enter`被删除)，在 `transition/animation` 完成之后移除
v-leave	定义离开过渡的开始状态	在离开过渡被触发时生效，在下一个帧移除
`v-leave-active`	|定义过渡的状态	|在元素整个过渡过程中作用，在离开过渡被触发后立即生效，在 transition/animation 完成之后移除 这个类可以被用来定义过渡的过程时间，延迟和曲线函数
`v-leave-to`	|(2.1.8 版及以上)定义离开过渡的结束状态	|在离开过渡被触发一帧后生效 (与此同时v-leave被删除)，在 transition/animation 完成之后移除

文字很多，读起来有点费力气，是时候贴上官网的这个图了：

![](./images/transition.png)

图 6-1 2.1.8 版本前 Vue 过渡类名说明

![](./images/transition2.1.8.png)

图 6-2 2.1.8 版本及以上 Vue 过渡类名说明

#### 实现逻辑
前面我们简单介绍了一些 transition 组件的实现原理相关的函数和事件，这里我们再详细分析下。根据前面内容，我们可以获得以下的信息：

- 可以使用 requestAnimationFrame 来请求浏览器在下一次重绘之前调用指定的函数来更新动画
- 当 CSS transition 结束时，会触发 transitionend 事件

所以结合之前的猜想和 Vue 的源码，能大概得到这里的实现方式：
1. transition 组件关注子元素是否展示，包括使用`v-if/v-else/v-for`等指令绑定数据生成的元素。
2. 当元素状态变更（display -> none 或 none -> display）时，预埋的钩子检测是否应用了 CSS 过渡。
3. 若使用了 CSS 过渡，则分两种情况讨论。

#### 进入动画
当新元素插入时，我们按照以下方式实现进入动画：
1. 当元素插入完成后（mounted），给元素添加对应 `v-enter` 和 `v-enter-active` 类名，此时元素开始动画过渡。
2. 同时通过 `requestAnimationFrame` 来指定下一帧绘制前，给元素添加 `v-enter-to` 类名，同时移除 `v-enter` 类名。
3. 设置动画结束时间，或者通过 `transitionend` 事件监听，在过渡结束后，移除 `v-enter-to` 和 `v-enter-active` 类名。

#### 离开动画
当新元素被删除时，我们按照以下方式实现离开动画：
1. 当元素删除前（beforeDestroy），给元素添加对应 v-leave 和 v-leave-active 类名，此时元素开始动画过渡。
2. 同时通过 requestAnimationFrame 来指定下一帧绘制前，给元素添加 v-leave-to 类名，同时移除 v-leave 类名。
3. 设置动画结束时间，或者通过 transitionend 事件监听，在过渡结束后，移除 v-leave-to 和 v-leave-active 类名，同时执行元素删除操作。

以上是整体实现方式，可见在我们使用 CSS 过渡的时候，最关键的是 v-enter-active 和 v-leave-active 两个样式。我们来看看 Vue 里面的部分代码：
```js
// start enter transition
if (expectsCSS) {
  // 添加v-enter和v-enter-active类名
  addTransitionClass(el, startClass);
  addTransitionClass(el, activeClass);
  // nextFrame为下一帧前的执行函数
  // 使用setTimeout向下兼容requestAnimationFrame
  nextFrame(() => {
    // 添加v-enter-to类名，移除v-enter类名
    addTransitionClass(el, toClass);
    removeTransitionClass(el, startClass);
    if (!cb.cancelled && !userWantsControl) {
      if (isValidDuration(explicitEnterDuration)) {
        // 若获取过渡的duration成功，则通过定时器来触发结束后逻辑
        setTimeout(cb, explicitEnterDuration);
      } else {
        // 若获取过渡的duration失败，则通过监听transitionend事件来触发结束后逻辑
        whenTransitionEnds(el, type, cb);
      }
    }
  });
}
```
当然，完整的 CSS 过渡、相关的钩子等会有更多的考虑和处理，这里只展示核心的实现逻辑部分。

### 3. CSS animation
CSS animations 使得可以将从一个 CSS 样式配置转换到另一个 CSS 样式配置。动画包括两个部分:描述动画的样式规则和用于指定动画开始、结束以及中间点样式的关键帧。

创建动画序列，需要使用 `animation` 属性或其子属性，该属性允许配置动画时间、时长以及其他动画细节。和 `transition` 不一样的是，该属性不能配置动画的实际表现，动画的实际表现是由`@keyframes`规则实现。

#### 使用 animation
我们来看看 `animation` 的使用方式，其实跟 `transition` 很相像的，子属性如表 6-3：

表 6-3 animation 属性说明
属性名	|说明
---|---
`animation-delay`	|设置延时，即从元素加载完成之后到动画序列开始执行的这段时间
`animation-direction`	|设置动画在每次运行完后是反向运行还是重新回到开始位置重复运行
`animation-duration`	|设置动画一个周期的时长
`animation-iteration-count`	|设置动画重复次数， 可以指定 infinite 无限次重复动画
`animation-name`	|指定由@keyframes 描述的关键帧名称
`animation-play-state`	|允许暂停和恢复动画
`animation-timing-function`	|设置动画速度， 即通过建立加速度曲线，设置动画在关键帧之间是如何变化
`animation-fill-mode`	|指定动画执行前后如何为目标元素应用样式

同样的，CSS animation 属性也是一个简写属性形式:
```css
/* @keyframes duration | timing-function | delay |
   iteration-count | direction | fill-mode | play-state | name */
animation: 3s ease-in 1s 2 reverse both paused slidein;

/* @keyframes duration | timing-function | delay | name */
animation: 3s linear 1s slidein;

/* @keyframes duration | name */
animation: 3s slidein;
```
我们也可以用详细的子属性方式来写，这里就不详细描述啦。

#### 关键帧动画
一旦完成动画的时间设置， 接下来就需要定义动画的表现。通过使用`@keyframes`建立两个或两个以上关键帧来实现。每一个关键帧都描述了动画元素在给定的时间点上应该如何渲染。

因为动画的时间设置是通过 CSS 样式定义的， **关键帧使用 percentage 来指定动画发生的时间点。0%表示动画的第一时刻，100%表示动画的最终时刻。因为这两个时间点十分重要，所以还有特殊的别名：from和to。** 当然，也可包含额外可选的关键帧，描述动画开始和结束之间的状态。来看个云在天上飘来飘去的 demo：
```css
.clouldmove {
  position: absolute;
  width: 20%;
  margin: 5%;
  float: left;
  margin-left: 70%;
  top: 0;
  animation-name: ClouldMove;
  animation-duration: 8s;
  animation-timing-function: linear;
  animation-iteration-count: infinite;
  animation-direction: alternate;
  animation-play-state: running;
}
@keyframes ClouldMove {
  0% {
    left: 0;
  }
  25% {
    left: 5%;
  }
  50% {
    left: 0;
  }
  75% {
    left: 5%;
  }
  100% {
    left: 0;
  }
}
```
还有很多的动画效果，包括轮回播放、来回运动等，这里不详细说明，大家感兴趣可以去 MDN 搜一下对应的内容。总之要记住，CSS animation 是个超级变态的动画效果实现。
#####  animation 事件监听器
上面我们讲到了 `transitionend`，`animation` 比它还多了些事件。
```js
// animationstart事件在动画一开始时就被触发
e.addEventListener("animationstart", listener, false);
// 每个周期完成后（除了最后一个周期），会触发animationiteration事件
e.addEventListener("animationend", listener, false);
// 最后一个周期完成后，不会触发animationiteration事件，而触发animationend事件
e.addEventListener("animationiteration", listener, false);
```
对于 Vue 来说，不管是 transition 还是 animation，都是通过 CSS 和类名来实现动画的，所以基本逻辑原理相似，除去动画结束监听的事件不一样而已，所以这里也不过多讲解了。

## 3. Javascript 钩子
除了 CSS 过渡和动画，在 Vue 中还可以使用 Javascript 钩子的方式来实现动画效果。

### 1. 完整的钩子信息
我们来看看 Vue 动画一共有多少的 Javascript 钩子：
```html
<transition
  v-on:before-enter="beforeEnter"
  v-on:enter="enter"
  v-on:after-enter="afterEnter"
  v-on:enter-cancelled="enterCancelled"
  v-on:before-leave="beforeLeave"
  v-on:leave="leave"
  v-on:after-leave="afterLeave"
  v-on:leave-cancelled="leaveCancelled"
>
  <!-- ... -->
</transition>
```
```js
// ...
methods: {
  // --------
  // 进入中
  // --------

  beforeEnter: function (el) {
    // ...
  },
  // 此回调函数是可选项的设置
  // 与 CSS 结合时使用
  enter: function (el, done) {
    // ...
    done()
  },
  afterEnter: function (el) {
    // ...
  },
  enterCancelled: function (el) {
    // ...
  },

  // --------
  // 离开时
  // --------

  beforeLeave: function (el) {
    // ...
  },
  // 此回调函数是可选项的设置
  // 与 CSS 结合时使用
  leave: function (el, done) {
    // ...
    done()
  },
  afterLeave: function (el) {
    // ...
  },
  // leaveCancelled 只用于 v-show 中
  leaveCancelled: function (el) {
    // ...
  }
}
```
这里对进入和离开的动画钩子说明也比较清晰了，我们来看看 demo。

##### 使用 jQuery 的例子
官方提供一个使用 `Velocity.js` 的例子，我们这里就直接拿个最简单的 jQuery 动画来写。
```html
<div id="app">
  <button @click="show = !show">
    Toggle
  </button>
  <transition v-on:enter="enter" v-on:leave="leave" v-bind:css="false">
    <p v-if="show">
      Demo
    </p>
  </transition>
</div>
```
```js
new Vue({
  el: "#app",
  data: {
    show: false
  },
  methods: {
    enter: (el, done) => {
      // 元素已被插入 DOM
      // 在动画结束后调用 done
      $(el)
        .css("opacity", 0)
        .animate({ opacity: 1, fontSize: "100px" }, 200, done);
    },
    leave: (el, done) => {
      // 与 enter 相同
      $(el).animate({ opacity: 0, fontSize: "0px" }, 200, done);
    }
  }
});
```
#### 初始渲染
Vue 的动画提供了一个初始渲染的开关，指的是第一次展示（而不是第一次切换）的时候是否需要动画效果。具体可以看看上面的例子，当我们把 show 的默认值设置为 true 的时候，初次展示并不会加载进入动画。但是当我们在 transition 组件中添加 appear 属性时，则在初次展示时也会加载进入动画：
```html
<div id="example-4">
  <button @click="show = !show">
    Toggle
  </button>
  <transition appear v-on:enter="enter" v-on:leave="leave" v-bind:css="false">
    <p v-if="show">
      Demo
    </p>
  </transition>
</div>
```
当然，你也可以绑定 appear 专属的动画效果：
```html
<transition
  appear
  v-on:before-appear="customBeforeAppearHook"
  v-on:appear="customAppearHook"
  v-on:after-appear="customAfterAppearHook"
  v-on:appear-cancelled="customAppearCancelledHook"
>
  <!-- ... -->
</transition>
```
#### 实现逻辑
前面我们详细描述了在使用 CSS 过渡和动画的时候，其中的具体实现原理。而 Javascript 钩子的实现逻辑其实很简单：
1. 检测组件是否使用了 Javascript 钩子。
2. 若使用了 Javascript 钩子，则这些钩子函数将在恰当的时机被调用。

当然这些钩子分别穿插在各个地方，顺序的话也很明了。这里以离开动画为例子，我们来看看这段被我简化后的代码：
```js
export function leave(vnode: VNodeWithData, rm: Function) {
  // el是被动画的元素
  const el: any = vnode.elm;

  // 当然如果进入动画还没结束，就取消吧~
  // call enter callback now
  if (isDef(el._enterCb)) {
    el._enterCb.cancelled = true;
    el._enterCb();
  }

  // 嗯，这里获取该获取的东西，包括Javascript钩子和CSS类名
  const data = resolveTransition(vnode.data.transition);
  const {
    css,
    type,
    leaveClass,
    leaveToClass,
    leaveActiveClass,
    beforeLeave,
    leave,
    afterLeave,
    leaveCancelled,
    delayLeave,
    duration
  } = data;

  // 这个cb比较重要啦，将
  const cb = (el._leaveCb = once(() => {
    // 移除v-leave-to和v-leave-active类名
    if (expectsCSS) {
      removeTransitionClass(el, leaveToClass);
      removeTransitionClass(el, leaveActiveClass);
    }

    if (cb.cancelled) {
      // 如果离开被取消，则移除v-leave类名
      if (expectsCSS) {
        removeTransitionClass(el, leaveClass);
      }
      // 看到了吧，这里埋下了leaveCancelled的Javascript钩子
      leaveCancelled && leaveCancelled(el);
    } else {
      rm();
      // 这里埋下了afterLeave的Javascript钩子
      afterLeave && afterLeave(el);
    }
    // 这个离开动画的函数只调用一次啦
    el._leaveCb = null;
  }));

  // 看看要不要延迟执行啦
  if (delayLeave) {
    delayLeave(performLeave);
  } else {
    performLeave();
  }

  // 嗯，这个才是真正执行的函数
  function performLeave() {
    // the delayed leave may have already been cancelled
    if (cb.cancelled) {
      return;
    }

    // 看，这里埋下了beforeLeave的Javascript钩子
    beforeLeave && beforeLeave(el);
    // 下面这是上一节讲到的逻辑，执行CSS过渡动画的
    // cb是上面定义的回调啦，里面包括afterLeave钩子
    if (expectsCSS) {
      addTransitionClass(el, leaveClass);
      addTransitionClass(el, leaveActiveClass);
      nextFrame(() => {
        addTransitionClass(el, leaveToClass);
        removeTransitionClass(el, leaveClass);
        if (!cb.cancelled && !userWantsControl) {
          if (isValidDuration(explicitLeaveDuration)) {
            setTimeout(cb, explicitLeaveDuration);
          } else {
            whenTransitionEnds(el, type, cb);
          }
        }
      });
    }

    // 看，这里还埋下了leave的Javascript钩子
    // 和其他钩子不同的是，这里传进去cb了
    leave && leave(el, cb);
    if (!expectsCSS && !userWantsControl) {
      cb();
    }
  }
}
```
逻辑大概是这样了，现在大家明白了`leave: function (el, done)`里的 done 是什么了。

## 4. 多元素/组件过渡
对于列表、状态模块来说，常常会需要用到一些过渡动画，包括多元素、多组件之间的过渡方式。

### 1. 多元素过渡
这里的多元素过渡，通常指的是 `v-if` 和 `v-else` 等指令绑定元素状态切换的动画效果。Vue 官网里面提到一个需要注意的地方：当有相同标签名的元素切换时，需要通过 `key` 特性设置唯一的值来标记以让 `Vue` 区分它们，否则 `Vue` 为了效率只会替换相同标签内部的内容。

这个就跟`v-for`中绑定key是差不多原理的，总之是追踪元素，然后来进行操作或是优化更新算法。使用方式如下：
```html
<transition>
  <button v-if="docState === 'saved'" key="saved">
    Edit
  </button>
  <button v-if="docState === 'edited'" key="edited">
    Save
  </button>
  <button v-if="docState === 'editing'" key="editing">
    Cancel
  </button>
</transition>
```
### 2. 多组件过渡
多个组件的过渡要使用动态组件。什么是动态组件呢？通过使用保留的 component 元素，并对其 is 特性进行动态绑定，你可以在同一个挂载点动态切换多个组件：
```html
<component v-bind:is="currentView">
  <!-- 组件在 vm.currentview 变化时改变！ -->
</component>
```
```js
var vm = new Vue({
  el: "#example",
  data: {
    currentView: "home"
  },
  components: {
    home: {
      /* ... */
    },
    posts: {
      /* ... */
    },
    archive: {
      /* ... */
    }
  }
});
```
这里要加过渡的话，就是酱紫：
```html
<transition name="component-fade">
  <component v-bind:is="currentView"></component>
</transition>
```
路由组件的过渡，大概也是属于这种方式。

### 3. 过渡模式
多元素和多组件的过渡中，会有上一个的离开以及下一个的进入的情况，于是便有了这个过渡模式。Vue 提供了以下过渡模式：
- **in-out**：新元素先进行过渡，完成之后当前元素过渡离开。
- **out-in**：当前元素先进行过渡，完成之后新元素过渡进入。

```html
<transition name="fade" mode="out-in">
  <!-- ... the buttons ... -->
</transition>
```
当然，在不做任何设置的时候，transition 的默认行为是：进入和离开同时发生。

### 4. 实现逻辑
其实多元素和组件的过渡，大致总体是跟单个元素的过渡相似的，大概是多了切换的过程，也就是一个结束另外一个开始的过渡把。所以这里可以看看以下的源码：
```js
// replace old child transition data with fresh one
// important for dynamic transitions!
const oldData: Object =
  oldChild && (oldChild.data.transition = extend({}, data));
// handle transition mode
if (mode === "out-in") {
  // return placeholder node and queue update when leave finishes
  this._leaving = true;
  mergeVNodeHook(oldData, "afterLeave", () => {
    this._leaving = false;
    this.$forceUpdate();
  });
  return placeholder(h, rawChild);
} else if (mode === "in-out") {
  if (isAsyncPlaceholder(child)) {
    return oldRawChild;
  }
  let delayedLeave;
  const performLeave = () => {
    delayedLeave();
  };
  mergeVNodeHook(data, "afterEnter", performLeave);
  mergeVNodeHook(data, "enterCancelled", performLeave);
  mergeVNodeHook(oldData, "delayLeave", leave => {
    delayedLeave = leave;
  });
}
```
这里大概的操作是：
1. 当 mode 为'out-in'时，将下一个元素的插入时间延后，等待上一个元素的离开动画完成后，再触发下一个元素的插入和进入动画。
2. 当 mode 为'in-out'时，将上一个元素的移除时间延后，等待下一个元素的进入动画完成后，再触发上一个元素的移除和离开动画。

因为前面我们也分析过，进入动画和元素插入捆绑，而离开动画和元素移除捆绑，所以这里只需要控制元素的延时变更，就能得到动画控制的效果了。

## 5. FLIP 与列表过渡
关于 FLIP，在了解之后会有“这世界上竟然还有这种骚操作”的感觉。

### 1.什么是 FLIP
FLIP 代表 First、Last、Invert、Play，见表 6-4：

表 6-4 FLIP 说明

字母代表	|说明
---|---
`First`	|元素参与 transtion 的初始状态
`Last`	|元素的最终状态
`Invert`	|清楚了元素从开始到结束是如何改变的，如它的 width、height、opacity。下一步，你应用 transform 和 opacity 的变化来扭转或反转它们。如果元素已经向下移动到初始和结束状态之间的 90px，然后在 Y 轴上应用 transform -90px。这让元素看起来仍然在初始的位置，所以元素并没有达到最终的位置
`Play`	|为你之前改变的属性开启过渡，然后移除反转的改变。因为当移除 transform 和 opacity 时，元素都在它们的最终位置。这会缓解从伪造的初始位置到最终位置的计算量

### 2. 如何理解 FLIP
最初看到这些解释的时候，也是不是很明白在说啥。后面仔细看了下，真的是太厉害了。这里用我自己的方式来讲一下。
#### 动画的性能
故事要从很多年前说起，当年 CSS3 还没普遍兼容，我们使用着最原始的 jQuery 来做动画。那时候怎么做的呢？通常是通过定时器和直接改变元素的 style 来实现动画的，jQuery 的话，一般就是用`.animate()`来做的。

但是这样做会有什么问题呢？这要从浏览器渲染说起，这里描述下两个概念：
- **Repaint**：屏幕的一部分要重画，比如某个 CSS 的背景色变了。但是元素的几何尺寸没有变。
- **Reflow**：意味着元件的几何尺寸变了，我们需要重新验证并计算 Render Tree。是 Render Tree 的一部分或全部发生了变化。

简单说就是，Reflow 的成本比 Repaint 的成本高得多的多。而我们手动设置这些动画的时候，则可能会中了死穴。但是不是所有属性动画消耗的性能都一样，其中消耗最低的是 transform 和 opacity 两个属性（当然还有会触发 Composite 的其他 CSS 属性），其次是 Paint 相关属性。所以从某种角度来说，我们使用 CSS3 的 transform 这些属性，是可以优化动画性能的。

#### 元素的样式规整
更多的情况是，我们在计算动画时，通常很难计算最终状态，例如一些元素的重新排列，而我们又希望能有个过渡动画的时候。

通常来说，我们如果真的需要完整地计算整个样式过渡的动画，很多时候只能将元素设置`position: absolute/fixed`这样实现，但是这样存在很多性能、兼容和响应式问题，不好维护同时也不够灵活。

#### FLIP 过渡
FLIP，本质上它是一个准则，而不是一个框架或库。这是对动画的一种思考，试图在浏览器中能更轻易地让动画达到 60 fps（关于后者，这里先不做过多讨论哈）

**FLIP 将动画翻转**，可以这样理解：
1. 正常动画过渡，会计算最终 B 状态，再减去初始 A 状态，然后根据想要的过渡时间，定时器设置每段时间的变化，应用到元素上。
2. FLIP 的做法是，A 状态到 B 状态，不用手动计算每一帧的改变，而是计算从 B 到 A 的反向动画，然后下一帧直接切换 B 状态，在把反向动画应用在 B 上。

其实真实元素是直接从 `A->B`，但是由于在 `B` 状态上加了反向动画，所以用户看起来像是过渡。还是看不懂？来看看代码：
```js
// 获取初始位置A
var first = el.getBoundingClientRect();

// 将其移动到最终位置B
el.classList.add("totes-at-the-end");

// 获取最终位置B
var last = el.getBoundingClientRect();

// 翻转，计算B->A需要的移动
var invert = first.top - last.top;

// 从翻转的位置到最终位置
var player = el.animate(
  [
    { transform: "translateY(" + invert + "px)" },
    { transform: "translateY(0)" }
  ],
  {
    duration: 300,
    easing: "cubic-bezier(0,0,0.32,1)"
  }
);

// 动画结束后做一些处理，如移除样式什么的。
```
### 3. 列表过渡
#### transition-group 组件
前面我们讲到 `transition` 组件，多半用于单个节点，或者是同一时间中只渲染单个节点的情况。

这里我们要渲染多个节点的过渡效果，需要用到 `transition-group` 组件：
- 不同于 `transition`（`transition` 不会转换为真实元素），它会以一个真实元素呈现：默认为一个 `<span>`。你也可以通过 tag 特性更换为其他元素
- 内部元素总是需要提供唯一的 key 属性值（transition 是当有相同标签名的元素切换时，需要通过 key 特性设置唯一的值来标记）

##### demo
我们来看个简单的例子：
```html
<div id="list-demo" class="demo">
  <button v-on:click="add">Add</button>
  <button v-on:click="remove">Remove</button>
  <transition-group name="list" tag="p">
    <span v-for="item in items" v-bind:key="item" class="list-item">
      {{ item }}
    </span>
  </transition-group>
</div>
```
我们每次随机添加两个、删除两个：
```js
new Vue({
  el: "#list-demo",
  data: {
    items: [1, 2, 3, 4, 5, 6, 7, 8, 9],
    nextNum: 10
  },
  methods: {
    randomIndex: function() {
      return Math.floor(Math.random() * this.items.length);
    },
    add: function() {
      this.items.splice(this.randomIndex(), 0, this.nextNum++);
      this.items.splice(this.randomIndex(), 0, this.nextNum++);
    },
    remove: function() {
      this.items.splice(this.randomIndex(), 1);
      this.items.splice(this.randomIndex(), 1);
    }
  }
});
```
```css
.list-item {
  display: inline-block;
  margin-right: 10px;
}
.list-enter-active, .list-leave-active {
  transition: all 1s;
}
.list-enter, .list-leave-to
/* .list-leave-active for below version 2.1.8 */ {
  opacity: 0;
  transform: translateY(30px);
}
```
这里我们能看到，只有新加和删除的元素才会有过渡效果，其他的元素会瞬间移动，没有过渡效果。为了解决这个问题，我们需要用到 FLIP。

### 4. 排序过渡
什么是排序过渡？大概是：
1. 新增的元素有进入动画。
2. 移除的元素有离开动画。
3. 位置移动的元素有移动动画。

前面两点我们已经可以通过之前的方式实现了，关于第三点，我们使用 FLIP。比较关键的，我们看看 Vue 中的实现代码：
```js
// we divide the work into three loops to avoid mixing DOM reads and writes
// in each iteration - which helps prevent layout thrashing.
children.forEach(callPendingCbs); // 1. 之前如果有没结束的过渡，先结束掉
children.forEach(recordPosition); // 2. 记录新的位置
children.forEach(applyTranslation); // 3. 计算反向动画，生成个CSS动画

// force reflow to put everything in position
const body: any = document.body;
const f: number = body.offsetHeight; // eslint-disable-line

children.forEach((c: VNode) => {
  if (c.data.moved) {
    var el: any = c.elm;
    var s: any = el.style;
    // 添加反向CSS动画
    addTransitionClass(el, moveClass);
    s.transform = s.WebkitTransform = s.transitionDuration = "";
    // 监听动画结束，结束之后要移除CSS动画类名啦
    el.addEventListener(
      transitionEndEvent,
      (el._moveCb = function cb(e) {
        if (!e || /transform$/.test(e.propertyName)) {
          el.removeEventListener(transitionEndEvent, cb);
          el._moveCb = null;
          removeTransitionClass(el, moveClass);
        }
      })
    );
  }
});
```
这里还有个反向动画的计算可能大家会稍微感兴趣：
```js
function applyTranslation(c: VNode) {
  const oldPos = c.data.pos;
  const newPos = c.data.newPos;
  // 计算原位置和最终位置位移
  const dx = oldPos.left - newPos.left;
  const dy = oldPos.top - newPos.top;
  if (dx || dy) {
    c.data.moved = true;
    const s = c.elm.style;
    // 设置反位移
    s.transform = s.WebkitTransform = `translate(${dx}px,${dy}px)`;
    s.transitionDuration = "0s";
  }
}
```
其实也不难，跟上面的 FLIP demo 逻辑相似。排序过渡中，官方文档提到一个`v-move`特性，它会在元素的改变定位的过程中应用。像前面的类名一样，可以通过 `name` 属性来自定义前缀，也可以通过move-class属性手动设置，像上面的。
```js
.list-move {
  transition: transform 1s;
}
```
但其实这只是一个设置 transition 的过渡时间和过渡曲线而已，真正的亮点是藏在 FLIP 里面。

### 5. 列表交错过渡
交错过渡听起来好厉害的样子，其实说白了就是：设置延时，给元素设置顺序延时的动画效果，通过 data 属性与 JavaScript 通信就可以实现啦。大概就是获取元素的序号，计算延时，然后设置定时器动画。我们来实现一个延时动画效果的下拉菜单：
```html
<template>
  <div>
    <!-- 这是基于 bootstrap 常见的下拉菜单样式 -->
    <div class="row" style="margin-left: 20px;">
      <label class="mr5" style="font-size: 14px;">下拉菜单</label>
      <!-- v-click-outside 绑定方法名 -->
      <div class="btn-group" v-click-outside="closeMenu">
        <!-- 这里点击会切换菜单是否可见 -->
        <button
          type="button"
          class="btn btn-default dropdown-toggle"
          @click="isMenuShown = !isMenuShown"
        >
          点击 <span class="caret"></span>
        </button>
        <transition-group
          class="dropdown-menu"
          style="display:block;"
          name="staggered-fade"
          tag="ul"
          :css="false"
          v-on:before-enter="beforeEnter"
          v-on:enter="enter"
          v-on:leave="leave"
        >
          <li
            v-for="(item, index) in itemsShow"
            :key="item"
            :data-index="index"
          >
            <a>{{item}}</a>
          </li>
        </transition-group>
      </div>
    </div>
  </div>
</template>

<script>
  export default {
    el: "#app",
    data: {
      items: [
        "Action",
        "Another action",
        "Something else here",
        "Separated link"
      ],
      isMenuShown: false
    },
    computed: {
      itemsShow() {
        return this.items.filter(x => this.isMenuShown);
      }
    },
    methods: {
      beforeEnter: function(el) {
        el.style.opacity = 0;
        el.style.height = 0;
      },
      enter: function(el, done) {
        // 看看这里的定时器延时
        var delay = el.dataset.index * 800;
        setTimeout(function() {
          Velocity(el, { opacity: 1, height: "1.6em" }, { complete: done });
        }, delay);
      },
      leave: function(el, done) {
        // 还有这里的定时器延时
        var delay = el.dataset.index * 800;
        setTimeout(function() {
          Velocity(el, { opacity: 0, height: 0 }, { complete: done });
        }, delay);
      }
    }
  };
</script>
```
Vue 动画设计得真心好用，配合 CSS3 原有的一些动画能力，我们能用很简单的方法来实现一些比较复杂和酷炫的过渡效果。虽然很多时候动画都是比较受忽视的地方，但就跟 PPT 过渡一样，一些简单的过渡效果和动画，能让受众前后连贯起来，是一种很舒服的体验，也常常是让人眼前一亮的东西。