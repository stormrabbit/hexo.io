---
title: redux 源码分析：中间件的写法与排列顺序研究
date: 2020-07-03 15:47:08

---
## 设计模式与 redux 中间件

中间件是代理/装饰模式的一种的实践方式，通过改造 store.dispatch 方法，可以拦截 action（代理）或添加额外功能（装饰）。
<!-- more -->
> 突然发现 Javascript 里的代理/装饰模式的写法蛮通用的....

对于创建的 store 对象，如果希望代理/装饰 dispatch 函数，基本的写法如下：

```
const applyMyMiddlware = (store) => {
  // 1. 新建一个变量指向 store.dispatch
  const oldDispatch = store.dispatch;
  
  // 2. 新建 dispach，接收参数为 action
  const dispatch = (action) => {
  
    // 3. 编写额外逻辑
    /* 
      ........
    */
	
    // 3.1 所谓的代理就是拦截参数 action，根据 action 来进行自己的操作
    // 3.2 所谓的装饰就是不拦截 action，但是在这之前进行自己的逻辑处理
    // 3.3 注意对象中 this（如果有） 的指向问题

    // 4. 在 dispach 内部执行 oldDispatch，并返回。
    return oldDispatch(action);
	// 4.1 store.dispatch 是有返回值的，返回值类型是 action
  };
  //5 令 store.dispatch 指向新的 dispatch ，返回新的 store
  store.dispatch = dispatch;

  return store
  // 或者也可以这样写
  //  returen {
  //    ...store,
  //    dispatch
  //  } 
}

```

执行 `store = applyMyMiddlware(store)` 后， 调用 store.dispatch(action) 的结果便为代理/装饰后的结果。

## applyMiddleware 源码研究

redux 提供了官方加载中间件的函数 applyMiddleware，同时规定了中间件的写法必须是：

```
({dispatch, getState}) => next => action => {
  // .... 中间件自己的逻辑

  return next(action);
}
```

> 直到看源码之前，我只是单纯的记住了这么一长串的多重调用，并不理解为什么。


而这种多重返回的原因，就在 applyMiddleware 的源码里

```
import compose from './compose'

export default function applyMiddleware(...middlewares) {
  return (createStore) => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
        `Other middleware would not be applied to this dispatch.`
      )
    }
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}

```

### 1. ({dispatch, getState})


分析一下 applyMiddleware 源码很容易找到 {dispatch, getState} 的来源：

- 首先用传入的 createStore 方法创建了 store 对象。此时 store 中有 store.dispatch 以及 store.getState 方法（subscribe 暂时不考虑）。

- 初始化了一个 dispatch，但是中间塞了一个断言。如果直接调用，就会报错。

- 定义了一个 chain 数组和

```
  const middlewareAPI = {
    getState: store.getState,
    dispatch: (...args) => dispatch(...args)
  }
```
> getState 可以获得当前 state 状态，dispatch 则是经过处理添加了新功能

- 通过 map 函数，把每个中间件执行了一遍，传入的参数就是 middlewareAPI。

```
 chain = middlewares.map(middleware => middleware(middlewareAPI))
```

因此，对于中间件：

```
const middleware = ({dispatch, getState}) => next => action => {
  // .... 中间件自己的逻辑

  return next(action);
}
```
第一个参数 {dispatch, getState} 显然是 (middlewareAPI)，返回值为 

```
 next => action => {
  // .... 中间件自己的逻辑

  return next(action);
} 
```

这么调用的好处是，在返回值（也是个函数）内部依然可以调用到 store.getState 方法~~闭包~~。

### 2. next 是什么（上）：丧心病狂的 compose

经过 map 遍历，chain 数组此刻的值为：
```
[
  next => action => {
    // .... 中间件自己的逻辑

    return next(action);
  },
  next => action => {
    // .... 中间件自己的逻辑

    return next(action);
  } 
  // ...其他中间件
]
```
这么一种形式。

`dispatch = compose(...chain)(store.dispatch)`，是整段代码中最不(~~丧~~)好(~~心~~)理(~~病~~)解(~~狂~~)的部分。

贴一下 compose 函数源码：

```
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}

```

- 对于 `dispatch = compose(...chain)(store.dispatch)`，如果 chian 的长度是 0（也就是未传入中间件），等价于 dispatch = (args=> args)(store.dispatch) ，即 dispatch = store.dispatch (args => args 为 function(arg) {return arg)，执行结果为参数自身)。


- 如果 chian 的长度为 1，也就是说为 
```
  [
    next => action => {
      console.log('0号中间件')
      return next(action)
    }
  ];
```

令 
```
  const [temp0] = [
    next => action => {
      console.log('0号中间件')
      return next(action)
    }
  ];

  /*
  temp0 = next => action => {
    console.log('0号中间件');
    return next(action);
  }
  */

```
`compose(...chain)(store.dispatch)` 等同于 `temp0(next = store.disptach)`。**此时 next 的地址指向了 store.dispatch 的地址，next(action) 便等同于 store.dispatch(action)。**

因此 `temp0(next = store.disptach)` 的返回值应为：

```
  action => {
    console.log('0号中间件');
    return store.dispatch(action);
  }

```

即：

```
  dispatch = action => {
    console.log('0号中间件');
    return store.dispatch(action);
  }
```


结论：
**在只有一个中间件的情况下，next 的值是 `store.dispatch`**。把  `console.log('0号中间件');` 换成其他的逻辑，就可以在保证原本 `store.dispatch` 功能的情况下，加入自己的东西。

### 3. next 是什么（下）：庶民推理

- 假设 chain 的长度 为 2 。

即：

```
  chain = [next => action => {
      console.log('0 号中间件');
      return next(action);
    }, next => action => {
      console.log('1 号中间件')
      return next(action);
    }] // 只有 2 个元素的情况下

```

脑补 compose 的执行过程。

第 1 步. `return funcs.reduce((a, b) => (...args) => a(b(...args)))`

因为没有初始值，所以 a b 为最开始的两个元素。即 

`return (...args) => a(b(...args)));`

即 `compose(...chian) = (...args) => a(b(...args)));`。

第 2 步. `b(...args)`

b 为

```
next=> action => {
    console.log('1 号中间件')
    return next(action);
  }
```

所以 b(...args) 的执行结果为

```
action => {
    console.log('1 号中间件')
    return (...args)(action);
  }
```

第 3 步. `a(b(...args))`

a 为：

```
next => action => {
    console.log('0 号中间件');
    return next(action);
  }
```

 a(b(...args)) 就等同于

```
  action => {
      console.log('0 号中间件');
      return (
        // 用 b(...args) 的返回值代替 next
        action => {
          console.log('1 号中间件')
          return (...args)(action);
        }
      )(action)
}

```

即 `compose(...chian)` 为

```  
  (...args) => action => {
      console.log('0 号中间件');
      return (
        // 用 b(...args) 的值代替 next
        action => {
          console.log('1 号中间件')
          return (...args)(action);
        }
      )(action)
```

第 4 步：dispatch

`dispatch` 等价于 `compose(...chian)(store.dispatch)` 等价于 
```
  // 因为 compose(...chian)(store.dispatch) 的参数 ...args 等于 store.dispatch
  // 去掉 (...args)=> 
  dispatch = action => {
      console.log('0 号中间件');
      return (action => {
        console.log('1 号中间件')
          // 使用 store.dispatch 代替 ...args
        return (store.dispatch)(action);
      })(action);
    
```

换个写法：

```
dispatch = action => {
  console.log('0 号中间件');

  const next = action => {
    console.log('1 号中间件');
    return store.dispatch(action);
  }

  reutrn next(action);
}
```

对于长度为 2 的中间件数组 chain：

**中间件 chain[1] 的 next 指向 `store.dispatch`。
中间件 chain[0] 的 next 指向 chain[1] 的 `action => {... return next(action)}` 部分。**

## 4. 递归与中间件调用

现在考虑 chain 的数组多于 2 个元素的情况，例如 `chain = [a, b, c]`。

由 3 得知 , b c 的执行结果是 

```
dispatchBC = action => {
  console.log('b');

  const next = action => {
    console.log('c');
    return store.dispatch(action);
  }

  reutrn next(action);
}
```

因此 `compose(...[a, b, c])` 的执行结果等同与 

```
dispatch = action => {
  console.log(a);

  const next = dispatchBC;

  return next(action);

}
```

> 递归

结论：**当多于 2 个的元素的时候，action 从每个中间件内走了一遍，最后一步执行 store 原本的 dispatch (当然前提是此 action 没有被中途拦截)。**

推论：
中间件的执行顺序是中间件数组的顺序（这也是为什么中间件的添加顺序有讲究）。除最后一位中间件调用 store.dispatch 发出 action 之外，其余中间件都是调用下一位中间件处理 action 的函数。


## 后记：函数与 JavaScript 

我刚开始用 js 的时候有位高人对我说：**JavaScript 其实并不是正统的 OOP 函数。**

确实，直到 ES6 里面才有了 extends 关键字进行继承，ES6 之前只有 proTotype。而所谓的 class 也不过是转成函数，进行调用。虽然 JavaScript 经过 es6 的革新和 es7 的强化后写法不再那么反人类，但是离纯 OOP 的语言比如 Java 还有不小的差距。

研究过 dva 和 redux 的部分源码之后，我发现 JavaScript 框架的作者在解决通用性问题的方式，都是通过提供了组合的函数而不是一个组合过的类( dva 处理异步调用的时候是返回了一个 takeEvery 的函数)。

**没有什么不是一个函数可以解决的问题，如果有就再来一个**。

这个就和目前的 OOP 思想差别相当大了，瞄准的是功能而不是对象。

对于 Java，虽然可以使用反射实现动态调用，但是类必须真实存在的；
对于 JavaScript，有没有类无所谓，没有就自己造一个。只要产生的对象能嘎嘎叫并像鸭子一样走路，那就是鸭子(著名的鸭式辨型)。
现在前端推广 stateless 组件和高阶组件，写来写去也是函数。

OOP 用多了，有的时候是有思维盲区存在的；换个角度从函数出发，说不定真的有惊喜。


