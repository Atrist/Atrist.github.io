# Vue Router 路由搭建应用
页面的跳转、局部内容的刷新是 web 应用中使用最多的场景。想象一下，如果我们只刷新了页面的内容，但是 URL 并没有改变，当用户刷新当前页面的时候，原先的内容会丢失，需要重新操作进入到对应的页面中，这是比较糟糕的一种体验。所以我们可以把页面的内容匹配到对应的路由信息中，即使是 ctrl+F5 这样的强制刷新，URL 信息也不会丢，用户依然可以快速恢复原先的页面浏览信息，这也是我们要设计和使用路由的很重要的原因。

本章主要介绍如何使用 Vue Router 来管理路由，包括路由的安装和配置、跳转、路由鉴权等。了解一个工具如何使用的同时，更应该了解工具的实现、为什么需要这样来使用。关于 Vue Router 的实现，其实可以从前端路由的出现和常见的实现方式来说起。

## 1. 前端路由
在过去，服务端处理来自浏览器的请求时，要根据不同的 URL 路由拼接出对应的视图页面，通过 Http 返回给浏览器进行解析渲染。使用这种方式加载页面，整个页面都需要重新加载，导致体验不够友好。随着几年前 AngularJS、React、Ember 等这些框架的出现，Web 应用也随之出现。第1章中我们讲了页面的局部刷新，Web 应用则是使用了这种局部刷新的能力，在路由信息变更的时候进行局部页面内容的刷新（而不是重新加载一个完整的页面），从而获取更好的体验。

### 1. 路由实现
路由相关的功能，大多数都是基于基本的 History API、Location API 这些浏览器提供的 API 能力封装的，所以这里我们先介绍和路由有关的一些 API。前端路由的实现，一般包括两种模式：History 模式和 Hash 模式。
#### History 模式
History 的路由模式，依赖了一个关键的属性**window.history**

`window.history`是一个只读属性，用来获取 `History` 对象的引用。History 对象提供了操作浏览器会话历史（浏览器地址栏中访问的页面，以及当前页面中通过框架加载的页面）的接口，使用`window.history`我们可以实现以下与路由相关的重要能力：

1. 在 `history` 中跳转。
    
    使用`window.history.back()`、`window.history.forward()`和`window.history.go()`方法来完成在用户历史记录中向后和向前的跳转。
2. 添加和修改历史记录中的条目。

    HTML5 引入了`history.pushState()`和`history.replaceState()`方法，它们分别可以添加和修改历史记录条目。这两个 API 都会操作浏览器的历史栈，而不会引起页面的刷新。区别在于，`pushState()`会增加一条新的历史记录，而`replaceState()`则会替换当前的历史记录：

    ```js
    /**
     * parameters
     * @state {object} 状态对象 state 是一个 JavaScript 对象，一般是JSON格式的对象字面量
     * @title {string} 可以理解为 document.title，在这里是作为新页面传入参数的
     * @url {string} 该参数定义了增加或改变的历史 URL 记录，可以是相对路径或者绝对路径，url的具体格式可以自定
     */
    history.pushState(state, title, url); // 向浏览器历史栈中增加一条记录
    history.replaceState(state, title, url); // 替换历史栈中的当前记录
    ```
3. 监听 popstate 事件。

    当同一个页面在历史记录间切换时，就会产生`popstate`事件，`popstate`事件会被传递给`window`对象，所以页面路由切换通常会与`window.onpopstate`配合使用。

上面介绍的`history.pushState()`或者`history.replaceState()`调用不会触发`popstate`事件，`popstate`事件只会在浏览器某些行为下触发, 比如点击后退、前进按钮(或者在 JavaScript 中调用`history.back()`、`history.forward()`、`history.go()`方法)。所以我们可以结合`popstate`事件、`pushState()`和`replaceState()`来完成完整的路由监听和修改能力。

如果当前处于激活状态的历史记录条目是由`history.pushState()`方法创建，或者由`history.replaceState()`方法修改过的, 则`popstate`事件对象的`state`属性包含了这个历史记录条目的`state`对象的一个拷贝。我们来看看示例：
```js
// 假如当前网页地址为http://example.com/example.html
window.onpopstate = function(event) {
  alert(
    "location: " + document.location + ", state: " + JSON.stringify(event.state)
  );
};
//绑定事件处理函数
//添加并激活一个历史记录条目 http://example.com/example.html?page=1,条目索引为1
history.pushState({ page: 1 }, "title 1", "?page=1");
//添加并激活一个历史记录条目 http://example.com/example.html?page=2,条目索引为2
history.pushState({ page: 2 }, "title 2", "?page=2");
//修改当前激活的历史记录条目 http://ex..?page=2 变为 http://ex..?page=3,条目索引为3
history.replaceState({ page: 3 }, "title 3", "?page=3");

history.back(); // 弹出 "location: http://example.com/example.html?page=1, state: {"page":1}"
history.back(); // 弹出 "location: http://example.com/example.html, state: null
history.go(2); // 弹出 "location: http://example.com/example.html?page=3, state: {"page":3}
```

#### Hash 模式
History 模式需要依赖 HTML5 History API（IE10 以上），以及服务器的配置来支持，所以也有不少的开发者会使用 Hash 模式来管理 Web 应用的路由。而 Hash 模式主要依赖 `Location` 对象的 `hash` 属性（l`ocation.hash`）和`hashchange`事件，我们来分别看一下。
1. Location 对象。

    `window.location`用来获取 `Location` 对象的引用。`Location` 对象存储在 `Window` 对象的 `Location` 属性中，表示当前 Web 地址。Location 对象提供的属性如表 7-1：

    表 7-1 Location 属性

    Location 属性|	描述|	示例，`https://www.test.com/en-US/search?q=URL#search-results`
    ---|---|---
    `hash`|	设置或返回从井号(#)开始的 URL（锚）|	#search-results
    `host`	|设置或返回主机名和当前 URL 的端口号|	`www.test.com`
    `hostname`|	设置或返回当前 URL 的主机名	|`www.test.com`
    `href`	|设置或返回完整的 URL	|`https://www.test.com/en-US/search?q=URL#search-results`
    `pathname`|	设置或返回当前 URL 的路径部分	|`/en-US/search`
    `port`	|设置或返回当前 URL 的端口号	|默认 80 端口
    `protocol`|	设置或返回当前 URL 的协议	|`https:`
    `search`|	设置或返回从问号(?)开始的 URL（查询部分）	|`?q=URL`

    `location.hash`的设置和获取，并不会造成页面重新加载，利用这一点，我们可以记录页面关键信息的同时，提升页面体验。

2. hashchange 事件。

    当一个窗口的 `hash` 改变时就会触发`hashchange`事件。`hashchange`事件对象有两个属性，`newURL`为当前页面新的 URL，`oldURL`为当前页面旧的 URL。

    `Hash` 模式通过`window.onhashchange`监听基于 `hash` 的路由变化，来进行页面更新处理的。部分浏览器不支持`onhashchange`事件，我们可以自行使用定时器检测和触发的方式来进行兼容：
    ```js
    (function(window) {
    // 如果浏览器原生支持该事件,则退出
    if ("onhashchange" in window.document.body) {
        return;
    }
    var location = window.location,
        oldURL = location.href,
        oldHash = location.hash;
    // 每隔100ms检测一下location.hash是否发生变化
    setInterval(function() {
        var newURL = location.href,
        newHash = location.hash;
        // 如果hash发生了变化,且绑定了处理函数...
        if (newHash != oldHash && typeof window.onhashchange === "function") {
        // 执行事件触发
        window.onhashchange({
            type: "hashchange",
            oldURL: oldURL,
            newURL: newURL
        });
        oldURL = newURL;
        oldHash = newHash;
        }
    }, 100);
    })(window);
    ```
### 2. 路由的实现原理
路由现在也成为前端框架里一个最基本的能力了，虽然实现上并没有太大的难度，但却是单页应用不可或缺的部分。因为实现的简单和相似，所以大家基本都直接使用框架自带的 router，像 `ngRouter/vue-router/react-router` 等等。


#### 路由实现步骤
一般来说，路由都是通过 History API、Location API 和相关的事件监听实现。我们以 Hash 模式为例，如果要实现路由变化的时候加载相应的内容，步骤大概分为三步：
1. 设置监听器，监听popstate或者hashchange事件。
2. 通过 hash（`location.href.hash`）获取当前的路由位置。
3. 根据当前匹配路径，判断后加载对应模块。

以上是简单的实现，像 Vue Router 这种工具库则是将上面的实现封装成通用的能力，同时提供了简单的事件监听和通知之外的一些更高级的能力，例如`<router-link>`、`<router-view>`这种与渲染层结合解析和处理的能力，以及路由的钩子、匹配和映射、懒加载打包等这种能力。我们来从使用的角度看看 Vue Router 的能力吧。
#### #Vue Router 实现
前面介绍了 History 模式和 Hash 模式两种，我们常见的路由库也都同样会提供这两种方式，Vue Router 也不例外。Vue Router 甚至兼容了 Node.js 服务端的情况，它提供的路由模式包括三种：

表 7-2 Vue Router 路由模式
路由模式	|说明|	示例
---|---|---
hash	|使用 URL hash 值来作路由（支持所有浏览器，包括不支持 HTML5 History Api 的浏览器）	|a.com/#/pageone
history	|充分利用history.pushState API 来完成 URL 跳转而无须重新加载页面	|a.com/pageone
abstract |	支持所有 JavaScript 运行环境，如 Node.js 服务器端（在 Node.js 端会自动设置该模式为默认值 如果发现没有浏览器的 API，路由会自动强制进入这个模式	| -


需要注意的地方是，History 模式需要依赖 HTML5 History API（IE10 以上）和服务器配置。你要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个`index.html`页面，这个页面就是你 `app` 依赖的页面。所以如果觉得 `hash` 片段可以接受（有些人会觉得太丑了无法接受），可以优先选择 `Hash` 模式来进行开发，简单方便。

我们看看 Vue Router 中的 History 对象：
```js
// HTML5 History
export class HTML5History extends History {
  constructor(router: Router, base?: string) {
    super(router, base);

    const expectScroll = router.options.scrollBehavior;
    const supportsScroll = supportsPushState && expectScroll;

    if (supportsScroll) {
      setupScroll();
    }

    const initLocation = getLocation(this.base);
    // 添加事件监听
    window.addEventListener("popstate", e => {
      const current = this.current; // History路由因为异步防护，不会更新

      // 首先避免在某些浏览器中调度第一个`popstate`事件
      const location = getLocation(this.base);
      if (this.current === START && location === initLocation) {
        return;
      }

      // 该方法进行路由的更新、对应钩子逻辑的执行、处理完毕的回调
      this.transitionTo(location, route => {
        if (supportsScroll) {
          handleScroll(router, route, current, true);
        }
      });
    });
  }
  // 前进到某个位置
  go(n: number) {
    window.history.go(n);
  }
  // 添加到路由
  push(location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this;
    // 前置逻辑执行
    this.transitionTo(
      location,
      route => {
        // 添加到历史记录中
        pushState(cleanPath(this.base + route.fullPath));
        handleScroll(this.router, route, fromRoute, false);
        // 回调
        onComplete && onComplete(route);
      },
      onAbort
    );
  }
  // 更新当前路由
  replace(location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this;
    this.transitionTo(
      location,
      route => {
        replaceState(cleanPath(this.base + route.fullPath));
        handleScroll(this.router, route, fromRoute, false);
        onComplete && onComplete(route);
      },
      onAbort
    );
  }
  // 确认是否某个路由
  ensureURL(push?: boolean) {
    if (getLocation(this.base) !== this.current.fullPath) {
      const current = cleanPath(this.base + this.current.fullPath);
      push ? pushState(current) : replaceState(current);
    }
  }
  // 获取当前路由
  getCurrentLocation(): string {
    return getLocation(this.base);
  }
}
```
我们再来看看 Hash 对象：
```js
export class HashHistory extends History {
  constructor(router: Router, base: ?string, fallback: boolean) {
    super(router, base);
    // check history fallback deeplinking
    if (fallback && checkFallback(this.base)) {
      return;
    }
    ensureSlash();
  } // 以避免过早地触发hashchange侦听器

  // 这会延迟到应用程序安装完毕
  setupListeners() {
    const router = this.router;
    const expectScroll = router.options.scrollBehavior;
    const supportsScroll = supportsPushState && expectScroll;

    if (supportsScroll) {
      setupScroll();
    }
    // 添加事件监听
    // 优先使用 popstate，同时使用 hashchange 兜底
    window.addEventListener(
      supportsPushState ? "popstate" : "hashchange",
      () => {
        const current = this.current;
        if (!ensureSlash()) {
          return;
        }
        // 该方法进行路由的更新、对应钩子逻辑的执行、处理完毕的回调
        this.transitionTo(getHash(), route => {
          if (supportsScroll) {
            handleScroll(this.router, route, current, true);
          }
          if (!supportsPushState) {
            replaceHash(route.fullPath);
          }
        });
      }
    );
  }
  // 前进到某个位置
  go(n: number) {
    window.history.go(n);
  }
  // 添加到路由
  push(location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this;
    this.transitionTo(
      location,
      route => {
        pushHash(route.fullPath);
        handleScroll(this.router, route, fromRoute, false);
        onComplete && onComplete(route);
      },
      onAbort
    );
  }
  // 更新当前路由
  replace(location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this;
    this.transitionTo(
      location,
      route => {
        replaceHash(route.fullPath);
        handleScroll(this.router, route, fromRoute, false);
        onComplete && onComplete(route);
      },
      onAbort
    );
  }
  // 确认是否某个路由
  ensureURL(push?: boolean) {
    const current = this.current.fullPath;
    if (getHash() !== current) {
      push ? pushHash(current) : replaceHash(current);
    }
  }
  // 获取当前路由
  getCurrentLocation() {
    return getHash();
  }
}
```
我们能看到，在 Vue Router 中路由模式的使用方式被抹平了，不管是 History 模式还是 Hash 模式，最终暴露出来的都是通用的`go()`、`push()`、`replace()`、`ensureURL()`、`getCurrentLocation()`这几种方法。而即使在 Hash 模式下，也会优先检测浏览器是否支持 History 模式而使用带"#"的 History 模式，而真正的 Hash 模式则是用来兜底。

除此之外，我们能看到还有 Scroll 相关的逻辑，这是 Vue Router 提供的自定义路由切换时页面如何滚动的能力。使用前端路由，当切换到新路由时，想要页面滚到顶部，或者是保持原先的滚动位置，就像重新加载页面那样，可以通过 Router 实例的`scrollBehavior`方法来定义，这里不详细介绍。我们来看看要怎么使用 Vue Router。

## 2. 快速接入 Vue Router
前面讲了关于前端路由和实现相关的介绍，大家目前也能大概理解 Vue Router 的功能。从本节内容开始，我们要正式地进入 Vue Router 安装和使用的讲解了。

### 1. 安装和使用
跟 Vue 一样，Vue Router 也提供了 CDN 的在线资源。虽然我们也可以使用引入 CDN 资源的方式来进行开发，但是一般来说要用到 Vue Router 的应用都会稍微复杂点，也不适合在一个文件里维护了。所以我们直接介绍依赖安装的方式，同时 Vue CLI 也提供了 vue + vue-router 的快速安装模板，大家也可以去试试看。

#### 1. 安装依赖
我们通过 npm 来安装 Vue Router（当然也可以使用 yarn）：
```bash
# 安装依赖
npm install vue-router
```
#### 2. 引入 vue-router
在 Vue 中，我们可以通过插件的方式来使用一些工具库、组件系统等。我们需要通过全局方法`Vue.use()`使用插件（要在调用`new Vue()`启动应用之前完成）。Vue Router 同样通过插件的方式提供，所以我们可以这样来使用：

```js
// main.js
import Vue from "vue";
// 引入 vue-router
import VueRouter from "vue-router";
Vue.use(VueRouter); // 使用 vue-router
// 后面是 new Vue() 相关逻辑
```
事实上，Vue.js 官方提供的一些插件(例如这里的 vue-router)，在检测到 Vue 是可访问的全局变量时会自动调用`Vue.use()`，所以这里其实也可以省略的。

### 2. 路由设计与配置
我们实际开发中，也常常只更新某块内容。像页面顶部的导航条、侧边的菜单列表等等，一般都是固定不变的内容，剩余的部分则需要根据具体的路由情况来渲染对应的页面内容。要实现这样的能力，可以通过嵌套路由或的方式来进行。

我们在使用路由功能前，需要先设计一下我们的应用里有哪些页面，例如这是一个简单的路由和嵌套关系：
```bash
/home/page1                /home/page2
+------------------+       +-----------------+
| Home             |       | Home            |
| +--------------+ |       | +-------------+ |
| | Page1        | |       | | Page2       | |
| |              | |       | |             | |
| |              | |  +--) | |             | |
| |              | |       | |             | |
| |              | |       | |             | |
| +--------------+ |       | +-------------+ |
+------------------+       +-----------------+
```
为了配对上面的这样一段路由关系，我们需要给 Vue Router 配上路由关系（插件接入的部分已经在前面讲过了）：
```js
// main.js
// 前面是使用 Vue Router 插件的部分，省略

// 配置路由信息
const routes = [
  // 以 / 开头的嵌套路径会被当作根路径
  {
    path: "/home",
    component: Home,
    name: "Home",
    children: [
      { path: "page1", component: Page1, name: "Page1" },
      { path: "page2", component: Page2, name: "Page2" }
    ]
  },
  // 通配符 * 会匹配所有路径
  // 路由 { path: '*' } 通常用于客户端 404 错误
  // 含有通配符的路由应该放在最后
  { path: "*", redirect: { name: "Home" } }
];

const router = new VueRouter({
  routes // （缩写）相当于 routes: routes
});

// 启动一个 Vue 应用
new Vue({
  el: "#app",
  router, // 传入路由能力
  render: h => h(App)
});
```
我们需要在 App 组件中，使用`<router-view>`添加最顶层的出口，渲染最高级路由匹配到的组件：
```html
<!-- App.vue -->
<template>
  <!-- 使用 <router-view></router-view> 来渲染最高级路由匹配到的组件 -->
  <router-view></router-view>
</template>
```
同时，我们需要在 Home 页面中使用`<router-view>`来展示子路由界面：
```html
<!-- Home.vue -->
<template>
  <div>
    <div>Home</div>
    <div>
      <router-link :to="{ name: 'Page1'}" tag="button">goto Page1</router-link>
      <router-link :to="{ name: 'Page2'}" tag="button">goto Page2</router-link>
    </div>
    <!-- 子路由界面 -->
    <!-- 如果路由为 /home/page1，此处是 Page1 组件 -->
    <!-- 如果路由为 /home/page2，此处是 Page2 组件 -->
    <router-view></router-view>
  </div>
</template>
```
`<router-view>`其实可以理解为占位符，而占位的内容是匹配到的路由信息对应的组件，这个组件会替代`<router-view>`进行渲染。一个被渲染组件同样可以包含自己的嵌套`<router-view>`，我们只需要保证在`routes`中的路由配置信息和我们的嵌套关系保持一致就可以。

这里还出现了`<router-link>`元素，它是 Vue Router 中的一种导航方式，可以理解为加了路由能力的`<a>`标签。

#### 路由配置参数
匹配到对应路由的时候该加载什么组件、是否使用了路由钩子、是否要重定向等，都可以通过`routes`来配置。我们来看看路由配置`routes`的参数都包括哪些：

```js
interface RouteConfig = {
  path: string, // 匹配的路径
  component?: Component, // 命中的组件，会加载在父组件/根页面下的 <router-view> 所在位置
  name?: string, // 命名路由，相当于给路由起个名字
  components?: { [name: string]: Component }, // 命名视图组件，需要渲染多个组件的时候可以使用
  redirect?: string | Location | Function, // 重定向到某个路由
  props?: boolean | Object | Function,
  alias?: string | Array<string>,
  children?: Array<RouteConfig>, // 嵌套路由，像 routes 配置一样的路由配置数组
  beforeEnter?: (to: Route, from: Route, next: Function) => void, // 进入路由的钩子
  meta?: any,

  // 2.6.0+
  caseSensitive?: boolean, // 匹配规则是否大小写敏感？(默认值：false)
  pathToRegexpOptions?: Object // 编译正则的选项
}
```
我们经常会使用的配置主要包括`path`、`component`、`name`、`redirect`、`children`等，更多的使用方式可以参考官网的说明。

### 3. 导航方式
在 Vue Router 中，有两种导航方式：
1. router编程式导航。
2. `<router-link>`声明式导航。

我们分别来看一下。

#### 编程式导航
Vue Router 提供了`router`的实例方法，通过编写代码来实现导航功能。在 Vue 实例内部，你可以通过`$router`访问路由实例，我们来看看该实例提供的一些方法。

##### router.push
想要导航到不同的 URL，则使用`router.push`方法。这个方法会向 history 栈添加一个新的记录，所以，当用户点击浏览器后退按钮时，则回到之前的 URL。使用方式如下：
```js
// 字符串
router.push("/home");
// 对象
router.push({ path: "/home" });
// 命名的路由
router.push({ name: "Home" });
```
##### router.replace
跟`router.push`很像，唯一的不同就是，它不会向 history 添加新记录，而会替换掉当前的 history 记录。
##### router.go(n)
这个方法的参数是一个整数，意思是在 history 记录中向前（正整数）或者后退（负整数）多少步，类似`window.history.go(n)`:
```js
// 在浏览器记录中前进一步，等同于 history.forward()
router.go(1);
// 后退一步记录，等同于 history.back()
router.go(-1);
```
前面我们有讲到，Hash 模式和 History 模式的路由方法差异被封装和抹平了，所以 Vue Router 的导航方法(`push`、`replace`、`go`)在各类路由模式 mode(`history`、`hash`和`abstract`)下表现一致。

##### 声明式导航
`<router-link>`组件支持用户在具有路由功能的应用中 (点击) 导航。通过`to`属性指定目标地址，默认渲染成带有正确链接的`<a>`标签，可以通过配置`tag`属性生成别的标签。另外，当目标路由成功激活时，链接元素自动设置一个表示激活的 CSS 类名。

表 7-3 `<router-link>`Props 说明

`<router-link>`Props	|说明|	类型
---|---|---
to	|表示目标路由的链接。当被点击后，内部会立刻把`to`的值传到`router.push()`，所以这个值可以是一个字符串或者是描述目标位置的对象|	`string/Location`
replace	|当点击时，会调用`router.replace()`而不是`router.push()`，于是导航后不会留下 history 记录|	`boolean`（默认false）
tag	|指定`<router-link>`渲染成某种标签。同样它还是会监听点击，触发导航|	`string`（默认"`a`"）
`active-class`	|设置链接激活时使用的 CSS 类名|	`string`（默认`"router-link-active"`）
`event`	|声明可以用来触发导航的事件	|`string/Array<string>`（默认"`click`"）

以上这些属性是比较常用的，我们可以看一些代码来体验一下：
```html
<!-- 字符串 -->
<router-link to="/home">Home</router-link>
<!-- 使用 v-bind 的 JS 表达式 -->
<router-link v-bind:to="'/home'">Home</router-link>
<!-- 不写 v-bind 也可以，就像绑定别的属性一样 -->
<router-link :to="'/home'">Home</router-link>
<!-- 可使用 router.push() 的参数 -->
<router-link :to="{ name: 'Home' }">Home</router-link>

<!-- router-link 替换元素为 button -->
<router-link :to="{ name: 'Home' }" tag="button">Home</router-link>
<!-- 渲染结果 -->
<!-- 不会有 herf 内容，但会监听点击，触发导航 -->
<button>Home</button>

<router-link :to="{ name: 'Page1' }">Page1</router-link>
<!-- 当当前的路由命中时，router-link-exact-active 为精确匹配时的样式 -->
<a href="#/home/page1" class="router-link-exact-active router-link-active"
  >Page1</a
>
```
### 4. 路由传参
有些时候，我们需要在 URL 上带一些参数来标识当前内容。因为页面逻辑是通用的，只有内容不一样，这些内容的数据常常是根据某个 ID 来从后台获取，例如根据 ID 从后台获取某本书的详细信息。如果我们希望刷新页面的时候该 ID 不会丢失，则需要把这个标识带到我们的 URL 里。通常我们有两种方式来携带：
1. `/page/detail?id=123`，在 Vue Router 中用`query`表示。
2. `/page/detail/123`，在 Vue Router 中用`params`表示。

通常来说，我们需要解析 URL 来获取对应的`query`或者`params`参数。我们来看看在 Vue Router 中，是怎样解析`query`的：
```js
export function parsePath(
  path: string
): {
  path: string,
  query: string,
  hash: string
} {
  let hash = "";
  let query = "";

  // 解析 hash 片段
  const hashIndex = path.indexOf("#");
  if (hashIndex >= 0) {
    hash = path.slice(hashIndex);
    path = path.slice(0, hashIndex);
  }

  // 解析 query 片段
  const queryIndex = path.indexOf("?");
  if (queryIndex >= 0) {
    query = path.slice(queryIndex + 1);
    path = path.slice(0, queryIndex);
  }

  return { path, query, hash };
}
```
我们能看到，通过?来分割出 URL 中的`query`片段，然后再进行处理就能获得所有的query参数和传参内容，而其实`params`的实现方式也是类似的。我们来分别看看两种方式要怎么传参和获取参数值。

#### params 传参
params 的传参模式，我们需要首先在路由配置中进行特殊的配置（冒号":"标记）：`
```js
const router = new VueRouter({
  routes: [
    {
      path: "/home",
      component: Home,
      name: "Home",
      children: [
        // 动态路径参数以冒号 ":" 开头
        { path: "page1/:id", component: Page1, name: "Page1" },
        { path: "page2", component: Page2, name: "Page2" }
      ]
    }
  ]
});
```
一个“路径参数”(param)使用冒号`:`标记。当匹配到一个路由时，参数值会被设置到`$route.params`，可以在每个组件内使用：
```html
<!-- Page1.vue -->
<template>
  <!-- $route 可直接注入到模板 -->
  <div>{{ $route.params.id }}</div>
</template>
<script>
  export default {
    data() {
      return {};
    },
    mounted() {
      // 每个组件中都可以通过 this.$route 获取路由信息
      console.log(this.$route.params);
    }
  };
</script>
```
而我们在跳转的时候，可以通过编程式导航或声明式导航来传参：
```js
// 编程式导航传参
this.$router.push({ name: "Page1", params: { id: 123 } });
// 会跳转到 /home/page1/123
```
```html
<!-- 声明式导航传参 -->
<router-link :to="{ name: 'Page1', params: {id: 1234}}">goto Page1</router-link>
<!-- 点击会跳转到 /home/page1/1234 -->
```

![](./images/vue-7-1.jpg)

#### #query 传参
params 传参有一个不方便的地方，即我们必须要传入一个动态路径参数才能匹配到对应的页面。但是有些时候，我们的页面也允许某些可选参数的缺失，例如我们想要`/edit`表示新建，`/edit/123`表示修改`id=123`的内容，但是当我们输入`/edit`的时候却匹配不到相同的页面。这种时候我们可以使用 query 传参，`/edit`表示新建、`/edit?id=123`表示修改。

query 的传参模式，我们不需要像 params 传参一样使用`/xxxx/:xxx`这样更改路由配置，只需要在导航的时候传参：
```js
// 编程式导航传参
this.$router.push({ name: "Page2", query: { id: 123 } });
// 会跳转到 /home/page2?id=123
```
```html
<!-- 声明式导航传参 -->
<router-link :to="{ name: 'Page2', query: {id: 1234}}">goto Page2</router-link>
<!-- 点击会跳转到 /home/page2?id=1234 -->
```
在使用这样的方式导航之后，参数值会被设置到`$route.query`，同样支持在每个组件内使用：
```html
<!-- Page2.vue -->
<template>
  <!-- $route 可直接注入到模板 -->
  <div>{{ $route.query.id }}</div>
</template>
<script>
  export default {
    data() {
      return {};
    },
    mounted() {
      // 每个组件中都可以通过 this.$route 获取路由信息
      console.log(this.$route.query);
    }
  };
</script>
```

![](./images/vue-7-2.jpg)

以上，我们介绍了 Vue Router 的安装、注入、嵌套路由、路由导航和传参，基本上已经覆盖了大多数基础的 web 应用路由能力。但是在一些特殊的场景下，我们还需要更多的能力如路由拦截、监听、懒加载等，还需要进一步地掌握更多的使用方法。

## 3. Vue Router 进阶
Vue Router 进阶主要讲述一些稍微偏门但是也经常需要使用到的路由能力，包括路由监听、鉴权、路由懒加载等。

### 1. 监听路由
当我们使用单个组件绑定多个路由（或者多个路由参数）的时候，路由的跳转并不能再次触发组件的重建（因此无法重新触发一些生命周期钩子），此时我们需要对路由进行监视。例如，我们对某个页面进行传参，来控制是需要新建某个内容，还是需要修改某项内容。

现在，我们有一个修改的路由`/edit`和一个新建的路由`/add`，但两个路由其实是匹配到同一个组件，因为页面内容是相似的。一般来说我们要对某个表单内容进行修改，需要根据当前的 id（这个 id 通常保存在 URL 路由中）从后台拉取已有的内容填充，然后再对这些内容进行修改。

如果我们在修改某个内容的时候（此时在`/edit`路由下），又通过`<router-link>`或别的方式触发到新增（跳转到`/add`路由）。由于两个路由匹配到了同一个组件，组件不会重新进行渲染，原先已填充的内容依然存在，即使它是需要清空或更新的。

这个时候，我们可以通过`watch`(监测变化)`$route`对象，来对路由参数的变化作出响应：

```html
<template>
  <div>
    <div>Detail</div>
    <div>{{$route.query.id ? '修改' : '新建'}}</div>
    <div>name: <input v-model="detail.name" /></div>
    <div>text: <input v-model="detail.text" /></div>
  </div>
</template>

<script>
  // 下面是 Vue 组件
  export default {
    data() {
      return {
        detail: {
          name: "",
          text: ""
        }
      };
    },
    watch: {
      $route(to, from) {
        // 对路由变化作出响应，更新参数
        this.updateDetail();
      }
    },
    methods: {
      updateDetail() {
        const id = this.$route.query.id;
        if (id) {
          // 传入 id 则意味着修改，需要获取并录入原先内容
          this.detail = {
            name: `name-${id}`,
            text: `text-${id}`
          };
        } else {
          // 未传入 id 则意味着新建，需要重置原有内容
          this.detail = {
            name: "",
            text: ""
          };
        }
      }
    }
  };
</script>
```

![](./images/vue-7-5.jpg)

如图 7-3，我们可以通过 watch `$route`，在每次路由更新之后，重新获取 id 然后更新对应的内容。但上面这种做法依然存在问题，如果我们的路由从`detail?id=123`变成了`detail?id=123&test=hahaha`，`$route`会出发侦听器，但是我们的 id 其实并没有变更，而这个时候由于重新获取内容，会覆盖掉我们正在编辑的内容。为了避免这种情况，我们可以通过参数(`to, from`)来检测是否不一致：
```js
export default {
  watch: {
    $route(to, from) {
      // 对路由变化作出响应
      // 只有 id 值变更的时候，才进行更新
      if (to.query.id !== from.query.id) {
        this.updateDetail();
      }
    }
  }
};
```
以上，便是 Vue Router 提供的路由监听能力，也是很实用的。

### 2. 路由鉴权
Vue Router 提供了导航守卫的功能，主要用来通过跳转或取消的方式守卫导航。守卫导航可以理解为，我们需要进入某个路由页面，而守卫导航的职责是负责检查我们有没有通行证，有就放行，没有就进行拦截。导航守卫经常用在权限和条件控制中，例如管理者的某个操作界面/路由，只有拥有管理者权限的人才能进入。这个过程我们简单称作路由鉴权。

常见的，例如我们的管理系统需要登录才能进入，否则默认跳转到登录页面：
```js
const router = new VueRouter({ ... })

// 这里我们注册一个前置守卫
router.beforeEach((to, from, next) => {
  if (to.name !== "Login") {
    // 非 login 页面，检查是否登录
    // 这里简单前端模拟是否填写了用户名，真实环境需要走后台登录，缓存到本地
    if (!isLogin) {
      // 未登录则需要跳转到登录页面
      next({ name: "Login" })
    }
  }
  // 其他情况正常执行
  next()
});
```
我们能看到，当用户去往的页面并非登录页面、同时该用户属于未登录状态的情况下，守卫会将用户转到登录页面，其他情况下正常访问。

在这里，`beforeEach`是 Vue Router 的全局前置守卫，而在 Vue Router 中还有全局解析守卫`beforeResolve`、全局后置守卫`afterEach`、路由配置的`beforeEnter`前置守卫、每个组件都拥有的守卫（`beforeRouteEnter`、`beforeRouteUpdate`、`beforeRouteLeave`），每个守卫（除了`afterEach`）的参数都是一样的(`to`, `from`, `next`)：

表 7-4 Vue Router 守卫参数名

参数名	|说明
---|---
to	|即将要进入的目标 Route 路由对象
from	|当前导航正要离开的 Route 路由对象
next	|一定要调用该方法来 resolve 这个钩子

`next`的使用方式如下：
```js
// 进行管道中的下一个钩子
// 如果全部钩子执行完了，则导航的状态就是confirmed(确认的)
next()

// 中断当前的导航
// 如果浏览器的 URL 改变了，那么 URL 地址会重置到 from 路由对应的地址
next(false)

// 当前的导航被中断，然后进行一个新的导航，跳转到一个不同的地址
// 可以向 next 传递任意位置 router.push 中的选项
next('/')
next({ path: '/' })

// (2.4.0+) 终止导航
// 传入一个 Error 实例，导航会被终止且该错误会被传递给 router.onError() 注册过的回调
next(error):
```
守卫是异步解析执行，此时导航在所有守卫`resolve`完之前一直处于等待中。关于不同的守卫注意事项这里不详细介绍了，一般来说我们只需要大概有哪些能力，需要用的时候再去查。大家可以去官网上看，但是完整的导航解析流程可以简单了解下：

表 7-5 完整的导航解析流程

序号	|流程	|说明
---|---|---
1	|导航被触发	|-
2	|在失活的组件里调用`beforeRouteLeave`离开守卫	|`beforeRouteLeave`: 导航离开该组件的对应路由时调用
3	|调用全局的`beforeEach`守卫	|`beforeEach`: 当一个导航触发时，全局前置守卫按照创建顺序调用
4	|在重用的组件里调用`beforeRouteUpdate`守卫|	`beforeRouteUpdate`: 在当前路由改变，但是该组件被复用时调用
5	|在路由配置里调用`beforeEnter`	|如果路由配置`route`里有设置，则会调用
6	|解析异步路由组件	|-
7	|在被激活的组件里调用`beforeRouteEnter`|`	beforeRouteEnter:` 在渲染该组件的对应路由被 confirm 前调用
8	|调用全局的`beforeResolve`守卫	|`beforeResolve:` 在导航被确认之前，同时在所有组件内守卫和异步路由组件被解析之后，解析守卫就被调用
9	|导航被确认，导航状态为 confirmed	|-
10	|调用全局的`afterEach`钩子	|`afterEach`钩子不会接受next函数也不会改变导航本身
11	|触发 DOM 更新	| -
12	|用创建好的实例调用`beforeRouteEnter`守卫中传给`next`的回调	| -

需要注意的是，参数`params`或查询`query`的改变并不会触发进入/离开的导航守卫，两种传参的改变只能通过前面介绍的 watch `$route`来获取。

### 3. 路由懒加载
懒加载是前端性能优化中很重要的一个能力。懒加载的出现，是因为我们使用工具进行代码打包，而打包过程会把整个应用的逻辑都打包到一起，这种情况下 JavaScript 包会变得非常大，影响页面加载。

Vue Router 提供了很简单的配置方式，来允许我们把不同路由对应的组件分割成不同的代码块。当对应的路由被访问的时候，Vue Router 才会加载对应组件，这样就能大大减小首页的代码包大小，加快加载速度。使用方式是：
1. 将异步组件定义为返回一个 Promise 的工厂函数，该函数返回的 Promise 需要 resolve 组件本身。
2. 使用动态`import`语法来定义代码分块点（依赖了 Webpack 的代码分割功能）。

默认官方脚手架 Vue CLI 使用的是 Babel，我们需要添加`syntax-dynamic-import`插件，才能使 Babel 可以正确地解析语法，进行代码分割。简单来说，就是这样：
```js
// 实现 Page2 页面懒加载
// 不会被打包到主包中，当匹配到对应的路由时候，才会被加载
const Page2 = () => import("./Page2.vue");
```
如果我们需要把几个组件都打包到一起，使用相同的`webpackChunkName`就可以实现：
```js
const Page1 = () => import(/* webpackChunkName: "page" */ "./Page1.vue");
const Page2 = () => import(/* webpackChunkName: "page" */ "./Page2.vue");
```
到这里，我们本章的内容也结束了。从路由的出现、实现方法，到 Vue Router 的常用能力介绍，相信大家对 Web 应用的页面渲染过程、URL 路由解析也多了不少的了解，而掌握了 Vue Router 的大致实现和功能，我们也能很快地找到一些问题的解决办法。