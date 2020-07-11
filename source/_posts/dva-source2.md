---
title: DVA 源码研究（二）
date: 2020-07-11 23:55:45
tags:
---

# dva-core

## 视图与数据(下)

处理 React 的 model 层问题有很多种办法，比如状态管理就不一定要用 Redux，也可以使用 Mobx(写法会更有 MVX 框架的感觉)；异步数据流也未必使用 redux-saga，redux-thunk 或者 redux-promise 的解决方式也可以(不过目前看来 saga 是相对更优雅的)。
<!-- more -->
放两篇个人感觉比较全面的技术文档：

- 阮一峰前辈的 [redux 三部曲](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_one_basic_usages.html)。
- redux-saga 的[中文文档](http://leonshi.com/redux-saga-in-chinese/docs/api/index.html)。

以及两者的 github：

- [redux](https://github.com/reactjs/redux)
- [redux-saga](https://github.com/redux-saga/redux-saga)

然后继续深扒 `dva-core`，还是先从 `package.json` 扒起。

## package.json

`dva-core` 的 `package.json` 中依赖包如下：

```
    "babel-runtime": "^6.26.0",  // 一个编译后文件引用的公共库，可以有效减少编译后的文件体积
    "flatten": "^1.0.2", // 一个将多个数组值合并成一个数组的库
    "global": "^4.3.2",// 用于提供全局函数比如 document 的引用
    "invariant": "^2.2.1",// 一个有趣的断言库
    "is-plain-object": "^2.0.3", // 判断是否是一个对象
    "redux": "^3.7.1", // redux ，管理 react 状态的库
    "redux-saga": "^0.15.4", // 处理异步数据流
    "warning": "^3.0.0" // 同样是个断言库，不过输出的是警告
```

当然因为打包还是用的 `ruban`，script 里没有什么太多有用的东西。继续依循惯例，去翻 `src/index.js`。

## `src/index.js`

`src/index` 的源码在[这里](https://github.com/dvajs/dva/blob/master/packages/dva-core/src/index.js)

在 `dva` 的 `src/index.js` 里，通过传递 2 个变量 `opts` 和 `createOpts` 并调用 `core.create`，`dva` 创建了一个 app 对象。其中 `opts` 是使用者添加的控制选项，`createOpts` 则是初始化了 reducer 与 redux 的中间件。

`dva-core` 的 `src/index.js` 里便是这个 app 对象的具体创建过程以及包含的方法：

```
export function create(hooksAndOpts = {}, createOpts = {}) {
  const {
    initialReducer,
    setupApp = noop,
  } = createOpts;

  const plugin = new Plugin();
  plugin.use(filterHooks(hooksAndOpts));

  const app = {
    _models: [
      prefixNamespace({ ...dvaModel }),
    ],
    _store: null,
    _plugin: plugin,
    use: plugin.use.bind(plugin),
    model,
    start,
  };
  return app;
  	// .... 方法的实现
	
	function model(){
		// model 方法
	}
	
	functoin start(){
		// Start 方法
	}
  }
  ```

> 我最开始很不习惯 JavaScript 就是因为 JavaScript 还是一个函数向的编程语言，也就是函数里可以定义函数，返回值也可以是函数，class 最后也是被解释成函数。在 dva-core 里创建了 app 对象，但是把 model 和 start 的定义放在了后面。一开始对这种简写没看懂，后来熟悉了以后发现确实好理解。一眼就可以看到 app 所包含的方法，如果需要研究具体方法的话才需要向后看。

[Plugin](https://github.com/dvajs/dva/blob/master/packages/dva-core/src/Plugin.js) 是作者设置的一堆**钩子**性监听函数——即是在符合某些条件的情况下下(dva 作者)进行手动调用。这样使用者只要按照作者设定过的关键词传递回调函数，在这些条件下便会自动触发。

> 有趣的是，我最初理解**钩子**的概念是在 Angular 里。为了能像 React 一样优雅的控制组件的生命周期，Angular 设置了一堆接口(因为使用的是 ts，所以 Angular 里有类和接口的区分)。只要组件实现(implements)对应的接口————或者称生命周期钩子，在对应的条件下就会运行接口的方法。 

#### Plugin 与 plugin.use

Plugin 与 plugin.use 都有使用数组的 reduce 方法的行为：
```
const hooks = [
  'onError',
  'onStateChange',
  'onAction',
  'onHmr',
  'onReducer',
  'onEffect',
  'extraReducers',
  'extraEnhancers',
];

export function filterHooks(obj) {
  return Object.keys(obj).reduce((memo, key) => {
  // 如果对象的 key 在 hooks 数组中
  // 为 memo 对象添加新的 key，值为 obj 对应 key 的值
    if (hooks.indexOf(key) > -1) {
      memo[key] = obj[key];
    }
    return memo;
  }, {});
}

export default class Plugin {
  constructor() {
    this.hooks = hooks.reduce((memo, key) => {
      memo[key] = [];
      return memo;
    }, {});
	/*
		等同于
		
		this.hooks = {
			onError: [],
			onStateChange:[],
			....
			extraEnhancers: []
		}
	*/
  }

  use(plugin) {
    invariant(isPlainObject(plugin), 'plugin.use: plugin should be plain object');
    const hooks = this.hooks;
    for (const key in plugin) {
      if (Object.prototype.hasOwnProperty.call(plugin, key)) {
        invariant(hooks[key], `plugin.use: unknown plugin property: ${key}`);
        if (key === 'extraEnhancers') {
          hooks[key] = plugin[key];
        } else {
          hooks[key].push(plugin[key]);
        }
      }
    }
  }

  // 其他方法
}
```
- 构造器中的 `reduce` 初始化了一个以 `hooks` 数组所有元素为 key，值为空数组的对象，并赋给了 class 的私有变量 `this.hooks`。

- `filterHooks` 通过 `reduce` 过滤了 `hooks` 数组以外的钩子。

- `use` 中使用 `hasOwnProperty` 判断 `key` 是 `plugin` 的自身属性还是继承属性，使用原型链调用而不是 `plugin.hasOwnProperty()` 是防止使用者故意捣乱在 `plugin` 自己写一个 `hasOwnProperty = () => false // 这样无论如何调用 plugin.hasOwnProperty() 返回值都是 false`。

- `use` 中使用 `reduce` 为 `this.hooks` 添加了 `plugin[key]` 。 

## model 方法

`model` 是 app 添加 model 的方法，在** dva 项目**的 index.js 是这么用的。

> app.model(require('./models/example'));

在 `dva` 中没对 model 做任何处理，所以 `dva-core` 中的 model 就是 ** dva 项目**里调用的 model。

```
  function model(m) {
    if (process.env.NODE_ENV !== 'production') {
      checkModel(m, app._models);
    }
    app._models.push(prefixNamespace(m));
  }
  
```

- `checkModel` 主要是用 `invariant` 对传入的 model 进行了合法性检查。

- `prefixNamespace` 又使用 reduce 对每一个 model 做处理，为 model 的 reducers 和 effects 中的方法添加了 `${namespace}/` 的前缀。

> Ever wonder why we dispatch the action like this in dva ? `dispatch({type: 'example/loadDashboard'` 

## start 方法

`start` 方法是 `dva-core` 的核心，在 `start` 方法里，dva 完成了** `store` 初始化** 以及 **`redux-saga` 的调用**。比起 `dva` 的 `start`，它引入了更多的调用方式。

一步一步分析：

### `onError`

```
    const onError = (err) => {
      if (err) {
        if (typeof err === 'string') err = new Error(err);
        err.preventDefault = () => {
          err._dontReject = true;
        };
        plugin.apply('onError', (err) => {
          throw new Error(err.stack || err);
        })(err, app._store.dispatch);
      }
    };
```
这是一个全局错误处理，返回了一个接收错误并处理的函数，并以 `err` 和 `app._store.dispatch` 为参数执行调用。

看一下 `plugin.apply` 的实现：

```
  apply(key, defaultHandler) {
    const hooks = this.hooks;
	/* 通过 validApplyHooks 进行过滤， apply 方法只能应用在全局报错或者热更替上 */ 
    const  validApplyHooks = ['onError', 'onHmr'];
    invariant(validApplyHooks.indexOf(key) > -1, `plugin.apply: hook ${key} cannot be applied`);
	/* 从钩子中拿出挂载的回调函数 ，挂载动作见 use 部分*/
    const fns = hooks[key];

    return (...args) => {
		// 如果有回调执行回调
      if (fns.length) {
        for (const fn of fns) {
          fn(...args);
        }
		// 没有回调直接抛出错误
      } else if (defaultHandler) {
        defaultHandler(...args);
		
		/*
		这里 defaultHandler 为 (err) => {
          throw new Error(err.stack || err);
        }
		*/
      }
    };
  }
  ```

###  `sagaMiddleware`

下一行代码是：    

> `const sagaMiddleware = createSagaMiddleware();`

和 `redux-sagas` 的入门教程有点差异，因为正统的教程上添加 sagas 中间件的方法是： `createSagaMiddleware(...sagas)`

> sagas 为含有 saga 方法的 generator 函数数组。

但是 api 里确实还提到，还有一~~~招从天而降的掌法~~~种动态调用的方式：

>  `const task = sagaMiddleware.run(dynamicSaga)`

于是：

```
	  const sagaMiddleware = createSagaMiddleware();
	  // ...
      const sagas = [];
      const reducers = {...initialReducer
      };
      for (const m of app._models) {
      	reducers[m.namespace] = getReducer(m.reducers, m.state);
      	if (m.effects) sagas.push(app._getSaga(m.effects, m, onError, plugin.get('onEffect')));
      }
      // ....

      store.runSaga = sagaMiddleware.run;
      // Run sagas
      sagas.forEach(sagaMiddleware.run);
```

### `sagas`

那么 sagas 是什么呢？

```
    const {
      middleware: promiseMiddleware,
      resolve,
      reject,
    } = createPromiseMiddleware(app);
    app._getSaga = getSaga.bind(null, resolve, reject);

    const sagas = [];
 
    for (const m of app._models) {
      if (m.effects) sagas.push(app._getSaga(m.effects, m, onError, plugin.get('onEffect')));
    }
```

显然，sagas 是一个数组，里面的元素是用 `app._getSaga` 处理后的返回结果，而 `app._getSaga` 又和上面 createPromiseMiddleware 代理 app 后返回的对象有很大关系。

#### `createPromiseMiddleware`

createPromiseMiddleware 的代码[在此](https://github.com/dvajs/dva/blob/master/packages/dva-core/src/createPromiseMiddleware.js)。

如果看着觉得眼熟，那肯定不是因为看过 redux-promise 源码的缘故，:-p。

##### `middleware`

`middleware` 是一个 redux 的中间件，即在不影响 redux 本身功能的情况下为其添加了新特性的代码。redux 的中间件通过拦截 action 来实现其作用的。

```
  const middleware = () => next => (action) => {
    const { type } = action;
    if (isEffect(type)) {
      return new Promise((resolve, reject) => {
		// .... resolve ,reject
      });
    } else {
      return next(action);
    }
  };
  
    function isEffect(type) {
		// dva 里 action 的 type 有固定格式： model.namespace/model.effects
		// const [namespace] = type.split(NAMESPACE_SEP); 是 es6 结构的写法
		// 等同于 const namespace = type.split(NAMESPACE_SEP)[0];
		// NAMESPACE_SEP 的值是 `/`
    	const [namespace] = type.split(NAMESPACE_SEP);
		// 根据 namespace 过滤出对应的 model
    	const model = app._models.filter(m => m.namespace === namespace)[0];
		// 如果 model 存在并且 model.effects[type] 也存在，那必然是 effects
    	if (model) {
    		if (model.effects && model.effects[type]) {
    			return true;
    		}
    	}

    	return false;
    }
  ```

>  const middleware = ({dispatch}) => next => (action) => {... return next(action)} 基本上是一个标准的中间件写法。在 return next(action) 之前可以对 action 做各种各样的操作。因为此中间件没用到 dispatch 方法，所以省略了。

本段代码的意思是，如果 dispatch 的 action 指向的是 model 里的 effects，那么返回一个 Promise 对象。此 Promise 的对象的解决( resolve )或者驳回方法 ( reject ) 放在 map 对象中。如果是非 effects (那就是 action 了)，放行。

换句话说，middleware 拦截了指向 effects 的 action。

##### 神奇的 bind

bind 的作用是绑定新的对象，生成新函数是大家都知道概念。但是 bind 也可以提前设定好函数的某些参数生成新函数，等到最后一个参数确定时直接调用。

> JavaScript 的参数是怎么被调用的？[JavaScript 专题之函数柯里化](https://juejin.im/post/598d0b7ff265da3e1727c491)。作者：[冴羽](https://juejin.im/user/58e4b9b261ff4b006b3227f4)。文章来源：[掘金](https://juejin.im/timeline)

这段代码恰好就是 bind 的一种实践方式。

```
  const map = {};

  const middleware = () => next => (action) => {
    const { type } = action;
    // ...
      return new Promise((resolve, reject) => {
        map[type] = {
          resolve: wrapped.bind(null, type, resolve),
          reject: wrapped.bind(null, type, reject),
        };
      });
	// ....
  };
  
  function wrapped(type, fn, args) {
    if (map[type]) delete map[type];
    fn(args);
  }

  function resolve(type, args) {
    if (map[type]) {
      map[type].resolve(args);
    }
  }

  function reject(type, args) {
    if (map[type]) {
      map[type].reject(args);
    }
  }
  
   return {
    middleware,
    resolve,
    reject,
  };
```
分析这段代码，dva 是这样做的：

1. 通过 `wrapped.bind(null, type, resolve)` 产生了一个新函数，并且赋值给匿名对象的 resolve 属性(reject 同理)。

> 1.1 wrap 接收三个参数，通过 bind 已经设定好了两个。`wrapped.bind(null, type, resolve)` 等同于 `wrap(type, resolve, xxx)`（**此处  `resolve` 是 Promise 对象中的**）。 

> 1.2 通过 bind 赋给匿名对象的 resolve 属性后，匿名对象.resolve(xxxx) 等同于 wrap(type, resolve, xxx)，即 reslove(xxx)。

2. 使用 type 在 map 对象中保存此匿名对象，而 type 是 action 的 type，即 namespace/effects 的形式，方便之后进行调用。

3. return 出的 resolve 接收 type 和 args 两个参数。type 用来在 map 中寻找 1 里的匿名函数，args 用来像 1.2 里那样执行。

> 这样做的作用是：分离了 promise 与 promise 的执行。在函数的作用域外依然可以访问到函数的内部变量，换言之：闭包。

#### `getSaga`

导出的 `resolve` 与 `reject` 方法，通过 bind 先设置进了 `getSaga` (同时也赋给了 `app._getSaga`)，sagas 最终也将 `getSaga` 的返回值放入了数组。

[getSaga 源码](https://github.com/dvajs/dva/blob/master/packages/dva-core/src/getSaga.js)

```
export default function getSaga(resolve, reject, effects, model, onError, onEffect) {
  return function *() {
    for (const key in effects) {
      if (Object.prototype.hasOwnProperty.call(effects, key)) {
        const watcher = getWatcher(resolve, reject, key, effects[key], model, onError, onEffect);
		// 将 watcher 分离到另一个线程去执行
        const task = yield sagaEffects.fork(watcher);
		// 同时 fork 了一个线程，用于在 model 卸载后取消正在进行中的 task
		// `${model.namespace}/@@CANCEL_EFFECTS` 的发出动作在 index.js 的 start 方法中，unmodel 方法里。
        yield sagaEffects.fork(function *() {
          yield sagaEffects.take(`${model.namespace}/@@CANCEL_EFFECTS`);
          yield sagaEffects.cancel(task);
        });
      }
    }
  };
}
```
可以看到，`getSaga` 最终返回了一个 [generator 函数](http://www.ruanyifeng.com/blog/2015/04/generator.html)。

在该函数遍历了** model 中 effects 属性**的所有方法（注：同样是 generator 函数）。结合 `index.js` 里的 ` for (const m of app._models)`，该遍历针对所有的 model。

对于每一个 effect，getSaga 生成了一个 watcher ，并使用 saga 函数的 **fork** 将该函数切分到另一个单独的线程中去（生成了一个 task 对象）。同时为了方便对该线程进行控制，在此 fork 了一个 generator 函数。在该函数中拦截了取消 effect 的 action（事实上，应该是卸载effect 所在 model 的 action），一旦监听到则立刻取消分出去的 task 线程。

##### getWatcher

```
function getWatcher(resolve, reject, key, _effect, model, onError, onEffect) {
  let effect = _effect;
  let type = 'takeEvery';
  let ms;

  if (Array.isArray(_effect)) {
	// effect 是数组而不是函数的情况下暂不考虑
  }

  function *sagaWithCatch(...args) {
    try {
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@start` });
      const ret = yield effect(...args.concat(createEffects(model)));
      yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@end` });
      resolve(key, ret);
    } catch (e) {
      onError(e);
      if (!e._dontReject) {
        reject(key, e);
      }
    }
  }

  const sagaWithOnEffect = applyOnEffect(onEffect, sagaWithCatch, model, key);

  switch (type) {
    case 'watcher':
      return sagaWithCatch;
    case 'takeLatest':
      return function*() {
        yield takeLatest(key, sagaWithOnEffect);
      };
    case 'throttle':
      return function*() {
        yield throttle(ms, key, sagaWithOnEffect);
      };
    default:
      return function*() {
        yield takeEvery(key, sagaWithOnEffect);
      };
  }
}

function createEffects(model) {
  function assertAction(type, name) {
	// 合法性判断
  }
  function put(action) {
    const { type } = action;
    assertAction(type, 'sagaEffects.put');
    return sagaEffects.put({ ...action, type: prefixType(type, model) });
  }
  function take(type) {
    if (typeof type === 'string') {
      assertAction(type, 'sagaEffects.take');
      return sagaEffects.take(prefixType(type, model));
    } else {
      return sagaEffects.take(type);
    }
  }
  return { ...sagaEffects, put, take };
}

function applyOnEffect(fns, effect, model, key) {
  for (const fn of fns) {
    effect = fn(effect, sagaEffects, model, key);
  }
  return effect;
}
```

先不考虑 effect 的属性是数组而不是方法的情况。

`getWatcher` 接收六个参数：
- `resolve/reject`: 中间件 `middleware` 的 res 和 rej 方法。
- `key`:经过 prefixNamespace 转义后的 effect 方法名，namespace/effect（也是调用 action 时的 type）。
-` _effect`:effects 中 key 属性所指向的 generator 函数。
- `model`： model
- `onError`： 之前定义过的捕获全局错误的方法
- `onEffect`：plugin.use 中传入的在触发 effect 时执行的回调函数（钩子函数）


`applyOnEffect` 对 effect 进行了动态代理，在保证 effect （即 `_effect`）正常调用的情况下，为期添加了 fns 的回调函数数组(即 `onEffect`)。使得在 effect 执行时， `onEffect` 内的每一个回调函数都可以被触发。





# Dva 爱你哟

- [ ] Todo

