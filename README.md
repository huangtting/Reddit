# Reddit
```
npm install
npm start
```
利用Reddit API得到相关文章的标题

# 中间件
```
import thunk from 'redux-thunk'
import { createLogger } from 'redux-logger'

const middleware = [ thunk ]
if (process.env.NODE_ENV !== 'production') {
  middleware.push(createLogger())
}

const store = createStore(
  reducer,
  applyMiddleware(...middleware)
)
```
中间件的原理就是通过改造原生的dispatch，使得dispatch具有更多的功能。

# redux-thunk
redux-thunk改造dispatch，使得dispatch可以dispatch一个函数，而不仅仅是一个plain Object.

redux-thunk源码：
```
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;

```
applyMiddleware源码：
```
export default function applyMiddleware(...middlewares) {
  return (createStore) => (...args) => {
    const store = createStore(...args)
    let dispatch = store.dispatch
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
可以看出applyMiddleware对每个中间件middleware都会传入middlewareAPI（包含getState和dispatch两项），所以redux-thunk的dispatch和getState由applyMiddleware传入。

redux-thunk中的next则来自dispatch = compose(...chain)(store.dispatch)这一句中的store.dispatch。

来看compose的源码：
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
compose的简单例子：
```
var fn1 = val => 'fn1-' + val
var fn2 = val => 'fn2-' + val 
var fn3 = val => 'fn3-' + val 

compose(fn1,fn2,fn3)('测试')

输出结果： fn1-fn2-fn3-测试
```
第一步时 a = fn1 ,b = fn2 
a(b(...args)) = fn1(fn2(...args))

第二步时
a = fn1(fn2(...args))
b = fn3

a(b(...args)) = fn1(fn2(fn3(...args))) 
b(...args)作为a函数的参数传入，也就是 ...arg = fn3(...args)

可以看到compose的作用是：将 compose(f, g, h)(...args)变成(...args) => f(g(h(...args))).

compose将各个中间件改造成嵌套调用，每一个中间件都会改变store.dispatch，redux-thunk的next就是经过redux-thunk改造前的store.dispatch,然后将改造后的store.dispatch传递给下一个中间件作为这个中间件的next。

比如下面这个例子可以看到next的作用：
```
let next = store.dispatch
store.dispatch = function dispatchAndLog(action) {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}
```

然后回到例子中的thunk action
```
const fetchPosts = subreddit => dispatch => {
  dispatch(requestPosts(subreddit))
  return fetch(`https://www.reddit.com/r/${subreddit}.json`)
    .then(response => response.json())
    .then(json => dispatch(receivePosts(subreddit, json)))
}
```
通过dispatch(fetchPosts(subreddit))进行调用。
subreddit是redux-thunk源码中的action参数，redux-thunk先判断action是否是函数，如果是函数，那么我们就传递dispatch, getState, extraArgument进行函数调用。

# redux-thunk的异步操作？
redux-thunk使得dispatch可以调用函数，在函数中又可以重新发起dispatch去调用另外的函数，直到返回值被一层一层返回。
这样就完成了异步操作。

# 参考
[Redux源码分析之compose](https://www.cnblogs.com/cloud-/p/7282188.html)

[浅析`redux-thunk`中间件源码](https://www.jianshu.com/p/a3b9b0958aeb)


