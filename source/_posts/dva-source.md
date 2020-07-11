---
title: DVA 源码研究
date: 2020-07-06 09:27:35
tags:
---
# npm run start

## 隐藏在 package.json 里的秘密

随便哪个 dva 的项目，只要敲入 npm start 就可以运行启动。之前敲了无数次我都没有在意，直到我准备研究源码的时候才意识到：**在敲下这行命令的时候，到底发生了什么呢？**

答案要去 package.json 里去寻找。
<!-- more -->
>有位技术大牛曾经告诉过我：看源码之前，先去看 package.son 。看看项目的入口文件，翻翻它用了哪些依赖，对项目便有了大致的概念。

package.json 里是这么写的：

```
 "scripts": {
    "start": "roadhog server"
  },
```

翻翻依赖，`"roadhog": "^0.5.2"`。


既然能在 devDependencies 找到，那么肯定也能在 [npm](https://www.npmjs.com/package/roadhog) 上找到。原来是个和 webpack 相似的库，而且作者看着有点眼熟...

如果说 dva 是亲女儿，那 [roadhog](https://github.com/sorrycc/roadhog.git) 就是亲哥哥了，起的是 webpack 自动打包和热更替的作用。

在 roadhog 的默认配置里有这么一条信息：

```
{
  "entry": "src/index.js",
}
```

后转了一圈，启动的入口回到了 `src/index.js`。

## `src/index.js`

在 `src/index.js` 里，dva 一共做了这么几件事：

0. 从 'dva' 依赖中引入 dva ：`import dva from 'dva'`; 

1. 通过函数生成一个 app 对象：`const app = dva()`; 

2. 加载插件：`app.use({})`;

3. 注入 model：`app.model(require('./models/example'))`;

4. 添加路由：`app.router(require('./routes/indexAnother'))`;

5. 启动：app.start('#root');

在这 6 步当中，dva 完成了 `使用 React 解决 view 层`、`redux 管理 model `、`saga 解决异步`的主要功能。事实上在我查阅资料以及回忆用过的脚手架时，发现目前端框架之所以被称为“框架”也就是解决了这些事情。前端工程师至今所做的事情都是在**分离动态的 data 和静态的 view **，只不过侧重点和实现方式也不同。

至今为止出了这么多框架，但是前端 MVX 的思想一直都没有改变。

# dva 

## 寻找 “dva”

既然 dva 是来自于 `dva`，那么 dva 是什么这个问题自然要去 dva 的[源码](https://github.com/dvajs/dva)中寻找了。

> 剧透：dva 是个函数，返回一了个 app 的对象。

> 剧透2：目前 dva 的源码核心部分包含两部分，`dva` 和 `dva-core`。前者用高阶组件 React-redux 实现了 view 层，后者是用 redux-saga 解决了 model 层。

老规矩，还是先翻 package.json 。

引用依赖很好的说明了 dva 的功能：统一 view 层。

```
// dva 使用的依赖如下：

    "babel-runtime": "^6.26.0", // 一个编译后文件引用的公共库，可以有效减少编译后的文件体积
    "dva-core": "^1.1.0", // dva 另一个核心，用于处理数据层
    "global": "^4.3.2", // 用于提供全局函数的引用
    "history": "^4.6.3", // browserHistory 或者 hashHistory
    "invariant": "^2.2.2", // 一个有趣的断言库
    "isomorphic-fetch": "^2.2.1", // 方便请求异步的函数，dva 中的 fetch 来源
    "react-async-component": "^1.0.0-beta.3", // 组件懒加载
    "react-redux": "^5.0.5", // 提供了一个高阶组件，方便在各处调用 store
    "react-router-dom": "^4.1.2", // router4，终于可以像写组件一样写 router 了
    "react-router-redux": "5.0.0-alpha.6",// redux 的中间件，在 provider 里可以嵌套 router
    "redux": "^3.7.2" // 提供了 store、dispatch、reducer 
	
```
不过 script 没有给太多有用的信息，因为 `ruban build` 中的 `ruban` 显然是个私人库(虽然在 tnpm 上可以查到但是也是私人库)。但根据惯例，应该是 dva 包下的 `index.js` 文件提供了对外调用：
```
Object.defineProperty(exports, "__esModule", {
  value: true
});

exports.default = require('./lib');
exports.connect = require('react-redux').connect;
```

显然这个 `exports.default` 就是我们要找的 dva，但是源码中没有 `./lib` 文件夹。当然直接看也应该看不懂，因为一般都是使用 babel 的命令 `babel src -d libs` 进行编译后生成的，所以直接去看 `src/index.js` 文件。


## `src/index.js`

`src/index.js`[在此](https://github.com/dvajs/dva/blob/master/packages/dva/src/index.js) ：

在这里，dva 做了三件比较重要的事情：

1. 使用 call 给 dva-core 实例化的 app(这个时候还只有数据层) 的 start 方法增加了一些新功能（或者说，通过代理模式给 model 层增加了 view 层）。
2. 使用 react-redux 完成了 react 到 redux 的连接。
3. 添加了 redux 的中间件 react-redux-router，强化了 history 对象的功能。

### 使用 call 方法实现代理模式

dva 中实现代理模式的方式如下：

**1. 新建 function ，函数内实例化一个 app 对象。**
**2. 新建变量指向该对象希望代理的方法， `oldStart = app.start`。**
**3. 新建同名方法 start，在其中使用 call，指定 oldStart 的调用者为 app。**
**4. 令 app.start = start，完成对 app 对象的 start 方法的代理。**

上代码:

```
export default function(opts = {}) {

  // ...初始化 route ，和添加 route 中间件的方法。

  /**
   * 1. 新建 function ，函数内实例化一个 app 对象。
   * 
   */
  const app = core.create(opts, createOpts);
  /**
   * 2. 新建变量指向该对象希望代理的方法
   * 
   */
  const oldAppStart = app.start;
  app.router = router;
  /**
   * 4. 令 app.start = start，完成对 app 对象的 start 方法的代理。
   * @type {[type]}
   */
  app.start = start;
  return app;

  // router 赋值

  /**
   * 3.1 新建同名方法 start，
   * 
   */
  function start(container) {
    // 合法性检测代码

    /**
     * 3.2 在其中使用 call，指定 oldStart 的调用者为 app。
     */
    oldAppStart.call(app);
	
	// 因为有 3.2 的执行才有现在的 store
    const store = app._store;

	// 使用高阶组件创建视图
  }
}
```  

> 为什么不直接在 start 方式中 oldAppStart ?
- 因为 dva-code 的 start 方法里有用到 this，不用 call 指定调用者为 app 的话，oldAppStart() 会找错对象。

> 实现代理模式一定要用到 call 吗？
- 不一定，看有没有 使用 this 或者代理的函数是不是箭头函数。从另一个角度来说，如果使用了 function 关键字又在内部使用了 this，那么一定要用 call/apply/bind 指定 this。

> 前端还有那里会用到 call ？
- 就实际开发来讲，因为已经使用了 es6 标准，基本和 this 没什么打交道的机会。使用 class 类型的组件中偶尔还会用到 this.xxx.bind(this)，stateless 组件就洗洗睡吧(因为压根没有 this)。如果实现代理，可以使用继承/反向继承的方法 —— 比如高阶组件。


### 使用 react-redux 的高阶组件传递 store

经过 call 代理后的 start 方法的主要作用，便是使用 react-redux 的 provider 组件将数据与视图联系了起来，生成 React 元素呈现给使用者。

不多说，上代码。

```
// 使用 querySelector 获得 dom
if (isString(container)) {
  container = document.querySelector(container);
  invariant(
    container,
    `[app.start] container ${container} not found`,
  );
}

// 其他代码

// 实例化 store
oldAppStart.call(app); 
const store = app._store;

// export _getProvider for HMR
// ref: https://github.com/dvajs/dva/issues/469
app._getProvider = getProvider.bind(null, store, app);

// If has container, render; else, return react component
// 如果有真实的 dom 对象就把 react 拍进去
if (container) {
  render(container, store, app, app._router);
  // 热加载在这里
  app._plugin.apply('onHmr')(render.bind(null, container, store, app));
} else {
  // 否则就生成一个 react ，供外界调用
  return getProvider(store, this, this._router);
}
  
 // 使用高阶组件包裹组件
function getProvider(store, app, router) {
  return extraProps => (
    <Provider store={store}>
      { router({ app, history: app._history, ...extraProps }) }
    </Provider>
  );
}

// 真正的 react 在这里
function render(container, store, app, router) {
  const ReactDOM = require('react-dom');  // eslint-disable-line
  ReactDOM.render(React.createElement(getProvider(store, app, router)), container);
}
```

> React.createElement(getProvider(store, app, router)) 怎么理解？
- getProvider 实际上返回的不单纯是函数，而是一个无状态的 React 组件。从这个角度理解的话，ReactElement.createElement(string/ReactClass type,[object props],[children ...]) 是可以这么写的。

> 怎么理解 React 的 stateless 组件和 class 组件？
- 你猜猜？
```
JavaScript 并不存在 class 这个东西，即便是 es6 引入了以后经过 babale 编译也会转换成函数。因此直接使用无状态组件，省去了将 class 实例化再调用 render 函数的过程，有效的加快了渲染速度。

即便是 class 组件，React.createElement 最终调用的也是 render 函数。不过这个目前只是我的推论，没有代码证据的证明。
```

#### react-redux 与 provider 

> provider 是个什么东西？

本质上是个高阶组件，也是代理模式的一种实践方式。接收 redux 生成的 store 做参数后，通过上下文 context 将 store 传递进被代理组件。在保留原组件的功能不变的同时，增加了 store 的 dispatch 等方法。

> connect 是个什么东西？

connect 也是一个代理模式实现的高阶组件，为被代理的组件实现了从 context 中获得 store 的方法。

> connect()(MyComponent) 时发生了什么？

只放关键部分代码，因为我也只看懂了关键部分(捂脸跑)：

```
import connectAdvanced from '../components/connectAdvanced' 
export function createConnect({
  connectHOC = connectAdvanced,
.... 其他初始值
} = {}) {
	
  return function connect( { // 0 号 connnect
    mapStateToProps,
    mapDispatchToProps,
   	... 其他初始值
    } = {}
  ) {
	....其他逻辑
    return connectHOC(selectorFactory, {//  1号 connect
		.... 默认参数
		selectorFactory 也是个默认参数
      })
  }
}

export default createConnect() // 这是 connect 的本体，导出时即生成 connect 0

```
```
// hoist-non-react-statics，会自动把所有绑定在对象上的非React方法都绑定到新的对象上
import hoistStatics from 'hoist-non-react-statics'
// 1号 connect 的本体
export default function connectAdvanced() {
	// 逻辑处理

	// 1 号 connect 调用时生成 2 号 connect
  return function wrapWithConnect(WrappedComponent) {
   	// ... 逻辑处理

	// 在函数内定义了一个可以拿到上下文对象中 store 的组件
    class Connect extends Component {
      
      getChildContext() {
		// 上下文对象中获得 store
        const subscription = this.propsMode ? null : this.subscription
        return { [subscriptionKey]: subscription || this.context[subscriptionKey] }
      }
		
		// 逻辑处理

      render() {

		  	// 	最终生成了新的 react 元素，并添加了新属性
          return createElement(WrappedComponent, this.addExtraProps(selector.props))

      }
    }

	// 逻辑处理
	
	// 最后用定义的 class 和 被代理的组件生成新的 react 组件
    return hoistStatics(Connect, WrappedComponent)  // 2 号函数调用后生成的对象是组件
  }
}


```
结论：对于 connect()(MyComponent)

1. connect 调用时生成 0 号 connect
2. connect()  0 号 connect 调用，返回 1 号 connect 的调用 `connectHOC()` ，生成 2 号 connect(也是个函数) 。
3. connect()(MyComponent) 等价于 connect2(MyComponent)，返回值是一个新的组件


### redux 与 router

redux 是状态管理的库，router 是(唯一)控制页面跳转的库。两者都很美好，但是不美好的是两者无法协同工作。换句话说，当路由变化以后，store 无法感知到。

于是便有了 `react-router-redux`。

`react-router-redux` 是 redux 的一个中间件(中间件：JavaScript 代理模式的另一种实践 针对 dispatch 实现了方法的代理，在 dispatch action 的时候增加或者修改) ，主要作用是：

> 加强了React Router库中history这个实例，以允许将history中接受到的变化反应到stae中去。

[github 在此](https://github.com/reactjs/react-router-redux)

从代码上讲，主要是监听了 history 的变化：

`history.listen(location => analyticsService.track(location.pathname))`

dva 在此基础上又进行了一层代理，把代理后的对象当作初始值传递给了 dva-core，方便其在 model 的 
subscriptions 中监听 router 变化。

看看 `index.js` 里 router 的实现：

1.在 createOpts 中初始化了添加 react-router-redux 中间件的方法和其 reducer ，方便 dva-core 在创建 store 的时候直接调用。

2. 使用 patchHistory 函数代理 history.linsten，增加了一个回调函数的做参数(也就是订阅)。

> subscriptions 的东西可以放在 dva-core 里再说，

```
import createHashHistory from 'history/createHashHistory';
import {
  routerMiddleware,
  routerReducer as routing,
} from 'react-router-redux';
import * as core from 'dva-core';

export default function (opts = {}) {
  const history = opts.history || createHashHistory();
  const createOpts = {
  	// 	初始化 react-router-redux 的 router
    initialReducer: {
      routing,
    },
	// 初始化 react-router-redux 添加中间件的方法，放在所有中间件最前面
    setupMiddlewares(middlewares) {
      return [
        routerMiddleware(history),
        ...middlewares,
      ];
    },
	// 使用代理模式为 history 对象增加新功能，并赋给 app
    setupApp(app) {
      app._history = patchHistory(history);
    },
  };

  const app = core.create(opts, createOpts);
  const oldAppStart = app.start;
  app.router = router;
  app.start = start;
  return app;

  function router(router) {
    invariant(
      isFunction(router),
      `[app.router] router should be function, but got ${typeof router}`,
    );
    app._router = router;
  }


}

// 使用代理模式扩展 history 对象的 listen 方法，添加了一个回调函数做参数并在路由变化是主动调用
function patchHistory(history) {
  const oldListen = history.listen;
  history.listen = (callback) => {
    callback(history.location);
    return oldListen.call(history, callback);
  };
  return history;
}
```

> 剧透：redux 中创建 store 的方法为：

```
// combineReducers 接收的参数是对象
// 所以 initialReducer 的类型是对象
// 作用：将对象中所有的 reducer 组合成一个大的 reducer
const reducers = {}; 
// applyMiddleware 接收的参数是可变参数
// 所以 middleware 是数组
// 作用：将所有中间件组成一个数组，依次执行
const middleware = []; 
const store = createStore(
  combineReducers(reducers),
  initial_state, // 设置 state 的初始值
  applyMiddleware(...middleware)
);
```

## 视图与数据

`src/index.js` 主要实现了 dva 的 view 层，同时传递了一些初始化数据到 dva-core 所实现的 model 层。当然，还提供了一些 dva 中常用的方法函数：

- `dynamic` 动态加载(2.0 以后官方提供 1.x 自己手动实现吧)
- `fetch` 请求方法(其实 dva 只是做了一把搬运工)
- `saga`(数据层处理异步的方法)。

这么看 dva 真的是很薄的一层封装。

而 dva-core 主要解决了 model 的问题，包括 state 管理、数据的异步加载、订阅-发布模式的实现，可以作为数据层在别处使用(看 2.0 更新也确实是作者的意图)。使用的状体啊管理库还是 redux，异步加载的解决方案是 saga。当然，一切也都写在 index.js 和 package.json 里。

如果我没被懒癌打倒的话，会继续写完的^_^y




