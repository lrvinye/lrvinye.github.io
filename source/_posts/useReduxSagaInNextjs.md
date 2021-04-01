---
date: 2021-01-22 14:21:00
title: 在 Nextjs 中使用 Redux-saga
tags: [React, Redux-saga, Redux, Nextjs]
category: 前端
---

# Package.json 版本环境

```json
  "dependencies": {
    "next": "10.0.5",
    "next-redux-wrapper": "^6.0.2",
    "react": "17.0.1",
    "react-dom": "17.0.1",
    "react-redux": "^7.2.2",
    "redux": "4.0.5",
    "redux-logger": "^3.0.6",
    "redux-saga": "1.1.3"
  },
  "devDependencies": {
    "redux-devtools-extension": "^2.13.8"
  }
```

# 实现 Redux

## 创建 store 数据仓库

```js
//store.js
import { applyMiddleware, createStore } from "redux";

//saga中间件创建工具
import createSagaMiddleware from "redux-saga";

//nextjs与redux的集成
import { createWrapper } from "next-redux-wrapper";

//导入reducer
import rootReducer from "./reducer";
//导入saga
import rootSaga from "./saga";

//绑定saga等中间件
const bindMiddleware = (middleware) => {
  //非生产环境开启调试工具
  if (process.env.NODE_ENV !== "production") {
    const { composeWithDevTools } = require("redux-devtools-extension");
    return composeWithDevTools(applyMiddleware(...middleware));
  }
  return applyMiddleware(...middleware);
};

//makeStore should return a new instance of redux store each time when called, no memoization needed here.
//创建store函数，每次调用时返回一个新的redux store的实例
export const makeStore = (context) => {
  //实例化saga
  const sagaMiddleware = createSagaMiddleware();

  //绑定saga
  const store = createStore(rootReducer, bindMiddleware([sagaMiddleware]));

  //运行并监控各个action
  store.sagaTask = sagaMiddleware.run(rootSaga);

  return store;
};

//暴露出一个wrapper函数，用于包裹整个App
export const wrapper = createWrapper(makeStore, { debug: true });
```

> 上述步骤中，对 store 的创建操作，考虑了服务端渲染

## 定义 Action

> 在此文件中统一管理 action 的命名等

```js
//actions.js
//定义并暴露出 所有的action
export const actionTypes = {
  //切换夜间模式
  TOGGLE_DARK_MODE: "TOGGLE_DARK_MODE",
  SET_DARK_MODE: "SET_DARK_MODE",
};

/**  下面定义具体每个action的传值方式 */

export const toggle_dark_mode = () => {
  return {
    type: actionTypes.TOGGLE_DARK_MODE,
  };
};
export const set_dark_mode = (bool) => {
  return {
    type: actionTypes.SET_DARK_MODE,
    bool,
  };
};
```

## 编写 reducer

> 为了更好的管理项目的多个 reducer，这里将拆分多个子 reducer；拆分 reducer 后，读取子 reducer 中的 state，需要使用 state.common.darkMode 的形式读取

- 子 reducer

```js
// reducers/common.js

//导入所有的action
import { actionTypes } from "../actions";
//初始化的默认数据
const initialState = {
  darkMode: false,
};

function reducer(state = initialState, action) {
  switch (action.type) {
    //设置夜间模式
    case actionTypes.SET_DARK_MODE:
      return {
        ...state,
        ...{ darkMode: action.bool },
      };

    default:
      return state;
  }
}

export default reducer;
```

- 在下面的根 reducer 中整合所有的子 reducer，并置顶 **HYDRATE** reducer

```js
// reducers/index.js

import { combineReducers } from "redux";
import common from "./common";

//获取nextjs的混合state支持
import { HYDRATE } from "next-redux-wrapper";

//合并子reducer
const combinedReducer = combineReducers({
  common,
});

const rootReducer = (state, action) => {
  if (action.type === HYDRATE) {
    //这个action必须放在最顶级，用于捕获初始dispatch
    //页面获取初始数据时会dispatch这个action，其中payload会携带渲染时的state，因此reducer必须将其与客户端现有的state合并。
    const nextState = {
      ...state, // use previous state
      ...action.payload, // apply delta from hydration
    };
    //这样会覆盖客户端的state
    //如果需要保留某些客户端的state，需要在此处执行下面的用法
    //if (state.count) nextState.count = state.count // 用来保留客户端的state
    return nextState;
  } else {
    //普通的dispatch会走自定义的reducer
    return combinedReducer(state, action);
  }
};

export default rootReducer;
```

## Saga 的副作用管理

> Saga 在 redux 的数据流中是作为中间件存在的，其实中间件就是在 dispatch -> reducer -> 中间件 -> state 这个流程中起作用的

- 同样的，我们的 saga 也会进行拆分管理

```js
// sagas/common.js

import { put, call, select, takeLatest } from "redux-saga/effects";
import { actionTypes, set_dark_mode } from "../actions";

export function* toggleDarkMode() {
  yield takeLatest(actionTypes.TOGGLE_DARK_MODE, function* () {
    try {
      const isDarkModeNow = yield select((state) => state.common.darkMode);
      yield put(set_dark_mode(!isDarkModeNow));
    } catch (error) {
      console.log(error);
    }
  });
}
```

> 注意！generator 函数不支持箭头函数写法

- 根 saga 函数，也就是一个合并的监听器，用来监听需要

```js
// sagas/index.js

import { takeLatest, all } from "redux-saga/effects";
//导入actions
import { actionTypes, toggle_dark_mode } from "../actions";

//导入子saga
import { toggleDarkMode } from "./common";

// 同时启动多个Sagas  并暴露监听action动作的函数
export function* appWatcher() {
  yield all([call(toggleDarkMode)]);
}
```

### Saga API

> 上述 saga 提供的具体 API 用法如下

| 类型          | API                               | 用法                                                                                                                                                                                      |
| ------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 辅助函数      | takeEvery(pattern,saga,args...)   | 用于捕获符合 pattern 的**每一个**action，并派发给 saga 函数，是构建在 take 和 fork 上面的高阶 api                                                                                         |
|               | takeLatest(pattern,saga,args...)  | 用于捕获**最新**的符合 pattern 的 action，并派发给 saga 函数，同时会取消上一个相同已经启动但未执行完成的 saga 任务                                                                        |
|               | takeLeading(pattern,saga,args...) | 用于捕获的符合 pattern 的 action 并派发给 saga 函数，同时会**阻塞不再接受**相同的 saga 任务，**直到当前的 saga 任务完成**，再开始继续接受                                                 |
|               | throttle(ms,pattern,saga,args...) | 用于捕获的符合 pattern 的 action 并派发给 saga 函数,并实现**节流 ms 毫秒**，在派送一次 saga 后，最多再接受（最近的）一个相同的 saga 任务，同时在 ms 毫秒内停止接受相同的 saga 任务        |
| Effect 创建器 | take(pattern)                     | 用于阻塞捕获**一次**符合 pattern 的 action，才向下继续执行后面的语句，take 返回的是监听到的 action 对象，action 可以读取 payload 用于下一步处理                                           |
|               | put(action)                       | 向 store 发起一次 action，非阻塞                                                                                                                                                          |
|               | put.resolve(action)               | 向 store 发起一次 action，阻塞,如果 dispatch 返回了 promise，会等待其结果，put 方法与 redux 原生的 dispatch 类似，通常用于处理完数据后，最终提交给 reducer。ex: yield put({type:'login'}) |
|               | call(fn,args...)                  | 以参数 args，调用 fn 函数执行， 主要用于异步请求 ， fn 可以是普通函数，也可以是 generator                                                                                                 |
|               | fork(fn,args...)                  | 以参数 args，调用 fn 函数执行，非阻塞，fork 在非阻塞调用中十分有用。可以在循环遍历中调用                                                                                                  |
|               | join(task)                        | 用来命令等待之前的一个交叉任务的结果                                                                                                                                                      |
|               | cancel(task)                      | 用来命令取消之前的一个交叉任务，如果不带参数则就是**自取消**的意思                                                                                                                        |
|               | select(selector,args...)          | 获取 redux 中的 state，不带参数则获取整个 state，ex: const state= yield select() or const name= yield select((state)=>state.name)                                                         |
| Effect 组合器 | race(effects)                     | 在多个 effect 中进行竞赛，其中有一个完成了之后，其余的都取消                                                                                                                              |
|               | all(effects)                      | 运行多个 effect，并等待完成，用于同时调用多个 saga 的 generator 函数 all([...多个函数])                                                                                                   |

## Store 的创建

> 由于使用了服务端渲染，因此对于 store 的创建需要考虑到客户端与服务端的不同实例

```js
// store.js

import { applyMiddleware, createStore } from "redux";
//saga中间件创建工具
import createSagaMiddleware from "redux-saga";
//nextjs与redux的集成
import { createWrapper } from "next-redux-wrapper";

//导入reducer,及初始化的state数据
import rootReducer from "./reducers";
//导入saga的监听函数
import { appWatcher } from "./sagas";

//绑定saga等中间件
const bindMiddleware = (middleware) => {
  //非生产环境开启调试工具
  if (process.env.NODE_ENV !== "production") {
    const { composeWithDevTools } = require("redux-devtools-extension");
    // 开发模式打印redux信息
    const { logger } = require("redux-logger");
    middleware.push(logger);
    return composeWithDevTools(applyMiddleware(...middleware));
  }
  return applyMiddleware(...middleware);
};

//makeStore should return a new instance of redux store each time when called, no memoization needed here.
//创建store函数，每次调用时返回一个新的redux store的实例，为不同的客户端创建新实例
export const makeStore = (context) => {
  //实例化saga,这一步由于服务端渲染的原因，必须放在此函数内执行
  const sagaMiddleware = createSagaMiddleware();

  //导入reducer、初始化state，绑定中间件
  const store = createStore(rootReducer, bindMiddleware([sagaMiddleware]));

  //saga常驻进程 运行并监控各个action
  store.sagaTask = sagaMiddleware.run(appWatcher);

  return store;
};

//暴露出一个wrapper函数，用于包裹整个App
export const wrapper = createWrapper(makeStore, {
  debug: process.env.NODE_ENV !== "production",
});
```

# 与 Nextjs 服务端渲染整合

## 包装 App

```js
// pages/_app.js

function App({ Component, pageProps }) {
  return <Component {...pageProps} />;
}

import { END } from "redux-saga";
App.getInitialProps = async ({ Component, ctx }) => {
  // 1. 等待所有页面的初始化流程中的dispatch执行完毕
  const pageProps = {
    ...(Component.getInitialProps ? await Component.getInitialProps(ctx) : {}),
  };

  // 2. 如果在服务器上则停止saga
  if (ctx.req) {
    ctx.store.dispatch(END);
    await ctx.store.sagaTask.toPromise();
  }

  // 3. 返回子组件（页面）所需的props
  return {
    pageProps,
  };
};

import { wrapper } from "../store";
//使用此方法包装整个App，用于在所有页面中正常调用wrapper
export default wrapper.withRedux(App);
```

## 在页面中的使用

```js
import { useEffect } from 'react'
import {useSelector, useDispatch } from 'react-redux'
import { wrapper } from '../store'
import { toggle_dark_mode } from "_redux/actions";

const Index = () => {
  const isDarkMode = useSelector((state) => {
    return state.common.darkMode;
  });
  const dispatch = useDispatch()

return (
  ...
)
}
export const getStaticProps = wrapper.getStaticProps(async ({ store }) => {
  //可以在这里执行初始化的dispatch，以获取初始数据
  //  store.dispatch(toggle_dark_mode(false))
  // 如果在初始化周期中（也就是这个函数里），调用了dispatch，则需要执行下面的语句来到等待dispatch中的异步完成，
  //未调用则不能使用下面的语句，否则会阻塞
  //await store.sagaTask.toPromise();
});

export default Index
```

> 要调用 action，可以使用 useDispatch

```js
//这个hook 用于共享状态,从Redux的store中提取数据（state）
import { useDispatch } from "react-redux";
//导入action
import { toggle_dark_mode } from "_redux/actions";

dispatch(toggle_dark_mode());
```

> 要读取 state 中的数据，可以使用 useSelector

```js
//这个hook 用于共享状态,从Redux的store中提取数据（state）
import { useSelector } from "react-redux";

const num = useSelector((state) => state.num);
```
