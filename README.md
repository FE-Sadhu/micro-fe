# 深入浅出微前端

- 背景？
- 什么是微前端？
- 对比各方案优缺点？
- systemjs是什么？
- single-spa使用和原理？
- qiankun使用和原理？
- umi-qiankun如何工作？
- webpack5 module federation是什么？
- emp实践
- 智慧案场微前端方案
  - 架构设计？umi-qiankun
  - 子产品接入指南？
  - 如何借助@focus/cli更好的工作？
  - @focus/pro-com

## 背景

在微前端出现之前，一个系统的前端开发模式基本都是单仓库，包含了所有的功能、代码...

很多企业也基本在物理上进行了应用代码隔离，实行单个应用单个库，闭环部署更新测试环境和正式环境。

比如我们公司的权限管理后台，首页中罗列了各个系统的入口，每个系统由单独仓库管理，点击具体系统，打开新窗口进行访问。

![admin-panel](./assets/admin-panel.jpg)

由于多个应用一级域名一致，使用不同二级域名区分。`cookie`存放在一级域名下，所以各应用可以借此实现用户信息的一致性。但是对于**头部、左侧菜单**通用的模块，以及多个应用之间如何实现资源共享？

我们尝试采用**npm包形式**对**头部、左侧菜单**抽离成npm包的形式进行管理和使用。但是却带来了**发布效率低下**的问题；

> 如果需要迭代npm包内的逻辑业务，需要先发布npm包之后，再每个使用了该npm包的应用都更新一次npm包版本，再各自构建发布一次，过程繁琐。如果涉及到的应用更多的话，花费的人力和精力就更多了。

不仅如此，我们可能还有下面几个诉求：

- 不同团队间开发同一个应用技术栈不同怎么办？
- 希望每个团队都可以独立开发，独立部署怎么办？（上述方式虽然可以解决，但是体验不好）
- 项目中还需要老的应用代码怎么办？

## 什么是微前端

在2016年，微前端的概念诞生。[micro-frontends](https://micro-frontends.org/)中定义`Techniques, strategies and recipes for building a modern web app with multiple teams that can ship features independently.`翻译成中文为`用来构建能够让 多个团队 独立交付项目代码的 现代web app 技术，策略以及实践方法`。

![micro-service](./assets/micro-service.jpg)

微前端也是借鉴后端微服务的思想。微前端就是将不同的功能按照不同的纬度拆分成多个子应用。通过主应用来加载这些子应用。

微前端的核心在于**先拆后合**。

### 微前端优势

- 同步更新
- 增量升级
- 简单、解耦的代码库
- 独立开发、部署

### 微前端解决方案

- 基座模式：通过搭建基座、配置中心来管理子应用。如基于`single spa`的`qiankun`方案。
- 自组织模式：通过约定进行互相调用，但会遇到处理第三方依赖的问题。
- 去中心模式：脱离基座模式，每个应用之间都可以批次分享资源。如基于`webpack5 module federation`实现的`EMP微前端方案`，可以实现多个应用彼此共享资源。


## 为什么不是TA
### 为什么不是 iframe

`qiankun技术圆桌`中有一篇关于微前端[Why Not Iframe](https://www.yuque.com/kuitos/gky7yw/gesexv)的思考，下面贴一下`iframe`的优缺点

- iframe 提供了浏览器原生的硬隔离方案，不论是样式隔离、 js 隔离这类问题统统都能被完美解决。
- url 不同步。浏览器刷新 iframe url 状态丢失、后退前进按钮无法使用。
- UI 不同步，DOM 结构不共享。想象一下屏幕右下角 1/4 的 iframe 里来一个带遮罩层的弹框，同时我们要求这个弹框要浏览器居中显示，还要浏览器 resize 时自动居中..
- 全局上下文完全隔离，内存变量不共享。iframe 内外系统的通信、数据同步等需求，主应用的 cookie 要透传到根域名都不同的子应用中实现免登效果。
- 慢。每次子应用进入都是一次浏览器上下文重建、资源重新加载的过程。

因为这些原因，最终大家都舍弃了 iframe 方案。

### 为什么不是 Web Component

[MDN Web Components](https://developer.mozilla.org/zh-CN/docs/Web/Web_Components)由三项主要技术组成，它们可以一起使用来创建封装功能的定制元素，可以在你喜欢的任何地方重用，不必担心代码冲突。

- **Custom elements（自定义元素）**：一组JavaScript API，允许您定义custom elements及其行为，然后可以在您的用户界面中按照需要使用它们。

- **Shadow DOM（影子DOM）**：一组JavaScript API，用于将封装的“影子”DOM树附加到元素（与主文档DOM分开呈现）并控制其关联的功能。通过这种方式，您可以保持元素的功能私有，这样它们就可以被脚本化和样式化，而不用担心与文档的其他部分发生冲突。

- **HTML templates（HTML模板）**： `<template> 和 <slot> `元素使您可以编写不在呈现页面中显示的标记模板。然后它们可以作为自定义元素结构的基础被多次重用。

官方提供的示例[web-components-examples](https://github.com/mdn/web-components-examples)。

但是兼容性很差，查看[can i use WebComponents](https://caniuse.com/?search=WebComponents)。

![web-component](./assets/web-component.png)

### 为什么不是ESM

`ESM`即`ES Module`，是一种前端模块化手段。他能做到微前端的几个核心点

- **无技术栈限制**： ESM加载的只是js内容，无论哪个框架，最终都要编译成js，因此，无论哪种框架，ESM都能加载。
- **应用单独开发**： ESM只是js的一种规范，不会影响应用的开发模式。
- **多应用整合**： 只要将微应用以ESM的方式暴露出来，就能正常加载。
- **远程加载模块**: ESM能够直接请求cdn资源，这是它与生俱来的能力。

但是可惜的是兼容性不好，查看[can i use import](https://caniuse.com/mdn-javascript_statements_import)。

![es-module](./assets/es-module.png)

## single spa

查看`single-spa`配置文件[rollup.config.js](https://github.com/single-spa/single-spa/blob/master/rollup.config.js#L44)可得知，使用了`rollup`做打包工具，并采用的`system`模块规范做输出。

> 感兴趣可查看对[@careteen/rollup](https://github.com/careteenL/rollup)的简易实现。

那我们就很有必要先介绍下`SystemJS`的相关知识。

### SystemJS使用

`SystemJS` 是一个通用的模块加载器，它能在浏览器上动态加载模块。微前端的核心就是加载微应用，我们将应用打包成模块，在浏览器中通过 `SystemJS` 来加载模块。

> 下方示例存放在[@careteen/micro-fe/system.js](https://github.com/careteenL/micro-fe/tree/master/system.js)，感兴趣可以前往调试。

#### 新建项目并配置

安装依赖

```shell
$ mkdir system.js
$ yarn init
$ yarn add webpack webpack-cli webpack-dev-server babel-loader @babel/core @babel/preset-env @babel/preset-react html-webpack-plugin -D
$ yarn add react react-dom
```

配置`webpack.config.js`文件，采用`system.js`模块规范作为`output.libraryTarget`，并不打包`react/react-dom`。

```js
const path = require("path");
const HtmlWebpackPlugin = require("html-webpack-plugin");
module.exports = (env) => {
  return {
    mode: "development",
    output: {
      filename: "index.js",
      path: path.resolve(__dirname, "dest"),
      libraryTarget: env.production ? "system" : "",
    },
    module: {
      rules: [
        {
          test: /\.js$/,
          use: { loader: "babel-loader" },
          exclude: /node_modules/,
        },
      ],
    },
    plugins: [
      !env.production &&
        new HtmlWebpackPlugin({
          template: "./public/index.html",
        }),
    ].filter(Boolean),
    externals: env.production ? ["react", "react-dom"] : [],
  };
};
```

配置`.babelrc`文件

```json
{
  "presets":[
    "@babel/preset-env",
    "@babel/preset-react"
  ]
}
```

配置`package.json`文件

```json
"scripts": {
  "dev": "webpack serve",
  "build": "webpack --env production"
},
```

#### 编写js、html代码

新建`src/index.js`入口文件

```js
import React from 'react';
import ReactDOM from 'react-dom';

ReactDOM.render(
  <h1>hello system.js</h1>,
  document.getElementById('root')
)
```

新建`public/index.html`文件，以cdn的形式引入`system.js`，并且将`react/react-dom`作为前置依赖配置到`systemjs-importmap`中。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>system.js demo</title>
  </head>

  <body>
    <script type="systemjs-importmap">
      {
        "imports": {
          "react": "https://cdn.bootcdn.net/ajax/libs/react/17.0.2/umd/react.production.min.js",
          "react-dom": "https://cdn.bootcdn.net/ajax/libs/react-dom/17.0.2/umd/react-dom.production.min.js"
        }
      }
    </script>
    <div id="root"></div>
    <script src="https://cdn.bootcdn.net/ajax/libs/systemjs/6.10.1/system.min.js"></script>
    <script>
      System.import("./index.js").then(() => {});
    </script>
  </body>
</html>
```

然后命令行运行

```shell
$ npm run dev # or build
```

打开浏览器访问，可正常显示文本。

#### 查看dest目录

观察`dest/index.js`文件，可发现通过`system.js`打包后会根据`webpack`配置而先`register`预加载`react/react-dom`然后返回`execute`执行函数。

```js
System.register(["react","react-dom"], function(__WEBPACK_DYNAMIC_EXPORT__, __system_context__) {
  return {
    setters: [
      // ...
    ],
    execute: function() {
      // ...
    }
  };
});
```

并且我们在使用时是通过`System.import("./index.js").then(() => {});`这个形式。

基于上述观察，我们了解到`system.js`两个核心`api`

- System.import ：加载入口文件
- System.register ：预加载

下面将做个简易实现。

### SystemJS原理

> 下方实现原理代码存放在[@careteen/micro-fe/system.js/dest/index.html](https://github.com/careteenL/micro-fe/blob/master/system.js/dest/index.html)，感兴趣可以前往调试。

首先提供构造函数，并将`window`的属性存一份，目的是查找对`window`属性进行的修改。

```js
function SystemJS() {}
let set = new Set();
const saveGlobalPro = () => {
  for (let p in window) {
    set.add(p);
  }
};
const getGlobalLastPro = () => {
  let result;
  for (let p in window) {
    if (set.has(p)) continue;
    result = window[p];
    result.default = result;
  }
  return result;
};

saveGlobalPro();
```

实现`register`方法，主要是对前置依赖做存储，方便后面加载文件时取值加载。

```js
let lastRegister;
SystemJS.prototype.register = function (deps, declare) {
  // 将本次注册的依赖和声明 暴露到外部
  lastRegister = [deps, declare];
};
```

使用`JSONP`提供`load`创建`script`脚本函数。

```js
function load(id) {
  return new Promise((resolve, reject) => {
    const script = document.createElement("script");
    script.src = id;
    script.async = true;
    document.head.appendChild(script);
    script.addEventListener("load", function () {
      // 加载后会拿到 依赖 和 回调
      let _lastRegister = lastRegister;
      lastRegister = undefined;

      if (!_lastRegister) {
        resolve([[], function () {}]); // 表示没有其他依赖了
      }
      resolve(_lastRegister);
    });
  });
}
```

实现`import`方法，传参为`id`即入口文件，加载入口文件后，解析[查看dest目录](#查看dest目录)中的`setters和execute`。

由于`react` 和 `react-dom` 会给全局增添属性 `window.React`,`window.ReactDOM`属性，所以可以通过`getGlobalLastPro`获取到这些新增的依赖库。

```js
SystemJS.prototype.import = function (id) {
  return new Promise((resolve, reject) => {
    const lastSepIndex = window.location.href.lastIndexOf("/");
    const baseURL = location.href.slice(0, lastSepIndex + 1);
    if (id.startsWith("./")) {
      resolve(baseURL + id.slice(2));
    }
  }).then((id) => {
    let exec;
    // 可以实现system模块递归加载
    return load(id)
      .then((registerition) => {
        let declared = registerition[1](() => {});
        // 加载 react 和 react-dom  加载完毕后调用setters
        // 调用执行函数
        exec = declared.execute;
        return [registerition[0], declared.setters];
        // {setters:[],execute:function(){}}
      })
      .then((info) => {
        return Promise.all(
          info[0].map((dep, i) => {
            var setter = info[1][i];
            // react 和 react-dom 会给全局增添属性 window.React,window.ReactDOM
            return load(dep).then((r) => {
              // console.log(r);
              let p = getGlobalLastPro();
              // 这里如何获取 react和react-dom?
              setter(p); // 传入加载后的文件
            });
          })
        );
      })
      .then(() => {
        exec();
      });
  });
};
```

上述简单实现了`system.js`的核心方法，可注释掉cdn引入形式，使用自己实现的进行测试，可正常展示。

```js
let System = new SystemJS();
System.import("./index.js").then(() => {});
```

### single spa使用

> 下方示例代码存放在[@careteen/micro-fe/single-spa](https://github.com/careteenL/micro-fe/tree/master/single-spa)，感兴趣可以前往调试。

安装脚手架，方便快速创建应用。
```shell
$ npm i -g create-single-spa
```
#### 创建基座

```shell
$ create-single-spa base
```

![create-single-spa-base](./assets/create-single-spa-base.png)

在`src/careteen-root-config.js`文件中新增下面子应用配置

```js
registerApplication({
  name: "@careteen/vue", // 应用名字
  app: () => System.import("@careteen/vue"), // 加载的应用
  activeWhen: ["/vue"], // 路径匹配
  customProps: {
    name: 'single-spa-base',
  },
});

registerApplication({
  name: "@careteen/react",
  app: () => System.import("@careteen/react"),
  activeWhen: ["/react"],
  customProps: {
    name: 'single-spa-base',
  },
});
start({
  urlRerouteOnly: true, // 全部使用SingleSpa中的reroute管理路由
});
```

提供`registerApplication`方法注册并加载应用，`start`方法启动应用

查看`src/index.ejs`文件

```html
<script type="systemjs-importmap">
  {
    "imports": {
      "single-spa": "https://cdn.jsdelivr.net/npm/single-spa@5.9.0/lib/system/single-spa.min.js"
    }
  }
</script>
<link rel="preload" href="https://cdn.jsdelivr.net/npm/single-spa@5.9.0/lib/system/single-spa.min.js" as="script">

<script>
  System.import('@careteen/root-config');
</script>
```

可得知需要`single-spa`作为前置依赖，并且实现`preload`预加载，最后加载基座应用`System.import('@careteen/root-config');`。

下面继续使用脚手架创建子应用

#### 创建vue项目

```shell
$ create-single-spa slave-vue
```

![create-single-spa-vue](./assets/create-single-spa-vue.png)

此处选择`vue3.x`版本。新建`vue.config.js`配置文件，配置开发端口号为`3000`

```js
module.exports = {
  devServer: {
    port: 3000,
  },
}
```

还需要修改`src/router/index.js`

```js
const router = createRouter({
  history: createWebHistory('/vue'),
  routes,
});
```

在基座中配置

```html
<script type="systemjs-importmap">
  {
    "imports": {
      "@careteen/root-config": "//localhost:9000/careteen-root-config.js",
      "@careteen/slave-vue": "//localhost:3000/js/app.js"
    }
  }
</script>
```

#### 创建react项目

```shell
$ create-single-spa slave-react
```

![create-single-spa-react](./assets/create-single-spa-react.png)

修改开发端口号为`4000`

```json
"scripts": {
  "start": "webpack serve --port 4000",
}
```

创建下面路由

```js
import { BrowserRouter as Router, Route, Link, Switch, Redirect } from 'react-router-dom'
import Home from './components/Home.js'
import About from './components/About.js'

export default function Root(props) {
  return <Router basename="/react">
    <div>
      <Link to="/">Home React</Link>
      <Link to="/about">About React</Link>
    </div>
    <Switch>
      <Route path="/"  exact={true} component={Home}></Route>
      <Route path="/about" component={About}></Route>
      <Redirect to="/"></Redirect>
    </Switch>
  </Router>
}
```

在基座中配置`react/react-dom`以及`@careteen/react`

```html
<script type="systemjs-importmap">
  {
    "imports": {
      "single-spa": "https://cdn.jsdelivr.net/npm/single-spa@5.9.0/lib/system/single-spa.min.js",
      "react":"https://cdn.bootcdn.net/ajax/libs/react/17.0.2/umd/react.production.min.js",
      "react-dom":"https://cdn.bootcdn.net/ajax/libs/react-dom/17.0.2/umd/react-dom.production.min.js"        
    }
  }
</script>
<script type="systemjs-importmap">
  {
    "imports": {
      "@careteen/root-config": "//localhost:9000/careteen-root-config.js",
      "@careteen/slave-vue": "//localhost:3000/js/app.js",
      "@careteen/react": "//localhost:4000/careteen-react.js"
    }
  }
</script>
```

#### 启动项目

```shell
$ cd base && yarn start
$ cd ../slave-vue && yarn start
$ cd ../slave-react && yarn start
```

浏览器打开 http://localhost:9000/

![single-spa-base](./assets/single-spa-base.png)

手动输入 http://localhost:9000/vue/ 并可以切换路由

![single-spa-vue](./assets/single-spa-vue.png)

手动输入 http://localhost:9000/react/ 并可以切换路由

![single-spa-react](./assets/single-spa-react.png)

### single spa原理

从`single spa`使用中，可以发现主要是两个方法`registerApplication`和`start`。

先新建`single-spa/example/index.html`文件，使用cdn的形式使用`single-spa`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>my single spa demo</title>
    <script src="https://cdn.bootcdn.net/ajax/libs/single-
spa/5.9.3/umd/single-spa.min.js"></script>
  </head>

  <body>
    <!-- 切换导航加载不同的应用 -->
    <a href="#/a">a应用</a>
    <a href="#/b">b应用</a>
    <!-- 源码中single-spa 是用rollup打包的 -->
    <script type="module">
      const { registerApplication, start } = singleSpa;
      // 接入协议
      let app1 = {
        bootstrap: [
          // 这东西只执行一次 ，加载完应用，不需要每次都重复加载
          async (customProps) => {
            // koa中的中间件 vueRouter4 中间件
            console.log("app1 启动~1", customProps);
          },
          async () => {
            console.log("app1 启动~2");
          },
        ],
        mount: async (customProps) => {
          console.log("app1 mount");
        },
        unmount: async (customProps) => {
          console.log("app1 unmount");
        },
      };
      let app2 = {
        bootstrap: [
          async () => {
            console.log("app2 启动~1");
          },
          async () => {
            console.log("app2 启动~2");
          },
        ],
        mount: async () => {
          console.log("app2 mount");
        },
        unmount: async () => {
          console.log("app2 unmount");
        },
      };

      const customProps = { name: "single spa" };
      // 注册微应用
      registerApplication(
        "app1", // 这个名字可以用于过滤防止加载重复的应用
        async () => {
          return app1;
        },
        (location) => location.hash == "#/a",
        customProps
      );
      registerApplication(
        "app2", // 这个名字可以用于过滤防止加载重复的应用
        async () => {
          return app2;
        },
        (location) => location.hash == "#/b",
        customProps
      );

      start();
    </script>
  </body>
</html>

```

接着去实现核心方法

新建`single-spa/src/single-spa.js`

```js
export { registerApplication } from './applications/apps.js';
export { start } from './start.js';
```

新建`single-spa/src/applications/app.js`

```js
import { reroute } from "../navigation/reroute.js";
import { NOT_LOADED } from "./app.helpers.js";

export const apps = [];
export function registerApplication(appName, loadApp, activeWhen, customProps) {
  const registeration = {
    name: appName,
    loadApp,
    activeWhen,
    customProps,
    status: NOT_LOADED,
  };
  apps.push(registeration);
  reroute();
}
```

维护数组`apps`存放所有的子应用，每个子应用需要的传参如下

- appName: 应用名称
- loadApp: 应用的加载函数 此函数会返回 bootstrap  mount unmount
- activeWhen: 当前什么时候激活 location => location.hash == '#/a'
- customProps: 用户的自定义参数
- status: 应用状态

将子应用保存到`apps`中，后续可以在数组里晒选需要的app是加载 还是 卸载 还是挂载

还需要调用`reroute`，重写路径， 后续切换路由要再次做这些事 ，这也是`single-spa`的核心。

`NOT_LOADED(未加载)`为应用的默认状态，那应用还存在哪些状态呢？

![single-spa-status](./assets/single-spa-status.png)

新建`single-spa/src/applications/app.helpers.js`存放所有状态

```js
export const NOT_LOADED = "NOT_LOADED"; // 应用默认状态是未加载状态
export const LOADING_SOURCE_CODE = "LOADING_SOURCE_CODE"; // 正在加载文件资源
export const NOT_BOOTSTRAPPED = "NOT_BOOTSTRAPPED"; // 此时没有调用bootstrap
export const BOOTSTRAPPING = "BOOTSTRAPPING"; // 正在启动中,此时bootstrap调用完毕后，需要表示成没有挂载
export const NOT_MOUNTED = "NOT_MOUNTED"; // 调用了mount方法
export const MOUNTED = "MOUNTED"; // 表示挂载成功
export const UNMOUNTING = "UNMOUNTING"; // 卸载中， 卸载后回到NOT_MOUNTED

// 当前应用是否被挂载了 状态是不是MOUNTED
export function isActive(app) {
  return app.status == MOUNTED;
}

// 路径匹配到才会加载应用
export function shouldBeActive(app) {
  // 如果返回的是true 就要进行加载
  return app.activeWhen(window.location);
}
```

于此同时还是提供几个方法判断当前应用所处状态。

然后再提供根据`app`状态对所有注册的app进行分类

```js
// `single-spa/src/applications/app.helpers.js`
export function getAppChanges() {
  // 拿不到所有app的？
  const appsToLoad = []; // 需要加载的列表
  const appsToMount = []; // 需要挂载的列表
  const appsToUnmount = []; // 需要移除的列表
  apps.forEach((app) => {
    const appShouldBeActive = shouldBeActive(app); // 看一下这个app是否要加载
    switch (app.status) {
      case NOT_LOADED:
      case LOADING_SOURCE_CODE:
        if (appShouldBeActive) {
          appsToLoad.push(app); // 没有被加载就是要去加载的app，如果正在加载资源 说明也没有加载过
        }
        break;
      case NOT_BOOTSTRAPPED:
      case NOT_MOUNTED:
        if (appShouldBeActive) {
          appsToMount.push(app); // 没启动柜， 并且没挂载过 说明等会要挂载他
        }
        break;
      case MOUNTED:
        if (!appShouldBeActive) {
          appsToUnmount.push(app); // 正在挂载中但是路径不匹配了 就是要卸载的
        }
      default:
        break;
    }
  });
  return { appsToLoad, appsToMount, appsToUnmount };
}

```

然后开始实现`single-spa/src/navigation/reroute.js`的核心方法

```js
import {
  getAppChanges,
} from "../applications/app.helpers.js";
export function reroute() {
  // 所有的核心逻辑都在这里
  const { appsToLoad, appsToMount, appsToUnmount } = getAppChanges();
  return loadApps();
  function loadApps() {
  // 获取所有需要加载的app,调用加载逻辑
  const loadPromises = appsToLoad.map(toLoadPromise); // 调用加载逻辑
    return Promise.all(loadPromises)
  }
}
```

于此同时再提供工具方法，方便处理传参进来的生命周期钩子是数组的场景

```js
function flattenFnArray(fns) {
  fns = Array.isArray(fns) ? fns : [fns];
  return function (customProps) {
    return fns.reduce(
      (resultPromise, fn) => resultPromise.then(() => fn(customProps),
      Promise.resolve()
    );
  };
}
```

实现原理类似于`koa中的中间件`，将多个promise组合成一个promise链。

再提供`toLoadPromise`， 只有当子应用是`NOT_LOADED` 的时候才需要加载，并使用`flattenFnArray`将各个生命周期进行处理

```js
function toLoadPromise(app) {
  return Promise.resolve().then(() => {
    if (app.status !== NOT_LOADED) {
      return app;
    }
    app.status = LOADING_SOURCE_CODE;
    return app.loadApp().then((val) => {
      let { bootstrap, mount, unmount } = val; // 获取应用的接入协议，子应用暴露的方法
      app.status = NOT_BOOTSTRAPPED;
      app.bootstrap = flattenFnArray(bootstrap);
      app.mount = flattenFnArray(mount);
      app.unmount = flattenFnArray(unmount);

      return app;
    });
  });
}
```

然后实现`single-spa/src/start.js`

```js
import { reroute } from "./navigation/reroute.js";
export let started = false;
export function start() {
  started = true; // 开始启动了
  reroute();
}
```

接着需要对`reroute`方法进行完善，将不需要的组件全部卸载，将需要加载的组件去`加载-> 启动 -> 挂载`，如果已经加载完毕，那么直接启动和挂载。

```js
export function reroute() {
  const { appsToLoad, appsToMount, appsToUnmount } = getAppChanges();
  if (started) { // 启动应用
    return performAppChanges();
  }
  function performAppChanges() { 
    appsToUnmount.map(toUnmountPromise);
    appsToLoad.map(app => toLoadPromise(app).then((app) => tryBootstrapAndMount(app)))
    appsToMount.map(appToMount => tryBootstrapAndMount(appToMount))
  }
}
```

其核心就是**卸载需要卸载的应用-> 加载应用 -> 启动应用 -> 挂载应用**


然后提供`toUnmountPromise`，标记成正在卸载，调用卸载逻辑 ， 并且标记成 未挂载。

```js
function toUnmountPromise(app) {
  return Promise.resolve().then(() => {
    // 如果不是挂载状态 直接跳出
    if (app.status !== MOUNTED) {
      return app;
    }
    app.status = UNMOUNTING;
    return app.unmount(app.customProps).then(() => {
      app.status = NOT_MOUNTED;
    });
  });
}
```

以及`tryBootstrapAndMount`，提供`a/b`应用的切换

```js
// a -> b b->a a->b
function tryBootstrapAndMount(app, unmountPromises) {
  return Promise.resolve().then(() => {
    if (shouldBeActive(app)) {
      return toBootStrapPromise(app).then((app) =>
        unmountPromises.then(() => {
          capturedEventListeners.hashchange.forEach((item) => item());
          return toMountPromise(app);
        })
      );
    }
  });
}
```

实现`toBootStrapPromise`启动应用

```js
function toBootStrapPromise(app) {
  return Promise.resolve().then(() => {
    if (app.status !== NOT_BOOTSTRAPPED) {
      return app;
    }
    app.status = BOOTSTRAPPING;
    return app.bootstrap(app.customProps).then(() => {
      app.status = NOT_MOUNTED;
      return app;
    });
  });
}
```

实现`toMountPromise`加载应用

```js
function toMountPromise(app) {
  return Promise.resolve().then(() => {
    if (app.status !== NOT_MOUNTED) {
      return app;
    }
    return app.mount(app.customProps).then(() => {
      app.status = MOUNTED;
      return app;
    });
  });
}
```

上述实现了子应用各个状态的切换逻辑，下面还需要将路由进行重写。

新建`single-spa/src/navigation/navigation-events.js`，监听hashchange和popstate，路径变化时重新初始化应用。

```js
import { reroute } from "./reroute.js";

function urlRoute() {
  reroute();
}
window.addEventListener("hashchange", urlRoute);
window.addEventListener("popstate", urlRoute);
```

需要对浏览器的事件进行拦截，其实现方式和`vue-router`类似，使用`AOP`的思想实现的。

因为子应用里面也可能会有路由系统，需要先加载父应用的事件，再去调用子应用。

```js
const routerEventsListeningTo = ["hashchange", "popstate"];
export const capturedEventListeners = {
  hashchange: [],
  popstate: [],
};
const originalAddEventListener = window.addEventListener;
const originalRemoveEventLister = window.removeEventListener;

window.addEventListener = function (eventName, fn) {
  if (
    routerEventsListeningTo.includes(eventName) &&
    !capturedEventListeners[eventName].some((l) => fn == l)
  ) {
    return capturedEventListeners[eventName].push(fn);
  }
  return originalAddEventListener.apply(this, arguments);
};

window.removeEventListener = function (eventName, fn) {
  if (routerEventsListeningTo.includes(eventName)) {
    return (capturedEventListeners[eventName] = capturedEventListeners[
      eventName
    ].filter((l) => fn != l));
  }
  return originalRemoveEventLister.apply(this, arguments);
};
```

需要对跳转方法进行拦截，例如 vue-router内部会通过pushState() 不改路径改状态，所以还是要处理下。如果路径不一样，也需要重启应用。


```js
function patchedUpdateState(updateState, methodName) {
  return function() {
    const urlBefore = window.location.href;
    const result = updateState.apply(this, arguments);
    const urlAfter = window.location.href;
    if (urlBefore !== urlAfter) {
      window.dispatchEvent(new PopStateEvent("popstate"));
    }
    return result;
  }
}
window.history.pushState = patchedUpdateState(window.history.pushState, 'pushState');
window.history.replaceState = patchedUpdateState(window.history.replaceState, 'replaceState')
```

提供触发事件的方法

```js
export function callCapturedEventListeners(eventArguments) { // 触发捕获的事件
  if (eventArguments) {
    const eventType = eventArguments[0].type;
    // 触发缓存中的方法
    if (routingEventsListeningTo.includes(eventType)) {
      capturedEventListeners[eventType].forEach(listener => {
        listener.apply(this, eventArguments);
      })
    }
  } 
}
```

改动`reroute`逻辑，启动完成需要调用`callAllEventListeners`，应用卸载完毕也需要调用`callAllEventListeners`。

```js
export function reroute() {
  const { appsToLoad, appsToMount, appsToUnmount } = getAppChanges();
  if (started) {
    return performAppChanges();
  }
  return loadApps();

  function loadApps() {
    const loadPromises = appsToLoad.map(toLoadPromise);
    return Promise.all(loadPromises).then(callAllEventListeners); // ++
  }
  function performAppChanges() {
    let unmountPromises = Promise.all(appsToUnmount.map(toUnmountPromise)).then(callAllEventListeners); // ++

    appsToLoad.map((app) =>
      toLoadPromise(app).then((app) =>
        tryBootstrapAndMount(app, unmountPromises)
      )
    );
    appsToMount.map((app) => tryBootstrapAndMount(app, unmountPromises));
  }
}
```

上述代码已经实现了基本功能

```shell
$ cd single-spa
$ yarn
$ yarn dev
```

打开 http://127.0.0.1:5000/example 点击切换a b应用查看打印结果

![my-single-spa-result](./assets/my-single-spa-result.png)


## qiankun


