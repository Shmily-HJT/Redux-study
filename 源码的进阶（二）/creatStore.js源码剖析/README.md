# Redux从入门到放弃 —— 源码的进阶（二）
</br>


## createStore.js
</br>

> #### 代码架构

```javascript
import $$observable from 'symbol-observable'
import ActionTypes from './utils/actionTypes'
import isPlainObject from './utils/isPlainObject'

function createStore(reducer, preloadedState, enhancer) {

	/*
    *  用于保证nextListeners不会污染currentListeners
    *  因为currentListeners和nextListeners都是引用类型，防止互相指向同一引用地址
    */
	ensureCanMutateNextListeners(){}

	//拿到当前state的一个方法，会返回当前的currentState
	getState(){}
    
    //消息订阅，传入一个litsener，当我们调用dispatch方法的时候，会触发每一个litsener去执行
    subscribe(listener){}
    
    /*
    * 接受一个action对象，去更新全局state
    * 还记得combineReducer最后返回的方法吗，这个方法会最终返回一个新的state
    * dispatch去更新全局状态就依赖于combineReducer最后返回的该方法
    */
    dispatch(action){}
    
    //替换reducer的方法
    replaceReducer(nextReducer) {}
    
    //这个方法暂时不详，平时没怎么使用过，后续了解了再补充
    observable(){}
}
```
</br>
</br>

> #### 源码解读


```javascript
/*
* 该方法接收三个参数
* 1、第一个reducer经过我们之前的分析，已得知就是combineReducer最后返回出来的那个方法
* 2、第二个参数代表初始化的state，一般不会用到，因为你在传入reducer后就会默认根据你的reducer返回值初始化全局状态
* 3、第三次参数代表强化redux的一个函数，本篇暂不讨论，下一篇将进行深入分析
*/
export default function createStore(reducer, preloadedState, enhancer) {

  //判断用户使用时，如果只传了两个参数，但是第二个参数却是一个function，则代表第二个参数传入的便是一个enhacer
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }


  //判断如果传入了enhancer，但这个enhacer却不是一个function，则报错
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
    //去使用enhancer改造你的createStore函数
    return enhancer(createStore)(reducer, preloadedState)
  }

  //判断传入的reducer是否是一个function，如果不是，则报错
  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  //combineReducer后返回的combination(state = {}, action)这个函数，这个函数最终会返回一个state
  let currentReducer = reducer
  //传入的initState，初始化state
  let currentState = preloadedState        
  //当前监听器
  let currentListeners = []
  //初始化下一版的监听器
  let nextListeners = currentListeners
  //是否正在更新全局状态
  let isDispatching = false



  //slice()方法用于浅拷贝一个数组，这里主要是保证nextListeners和currentListeners不会指向同一个引用
  function ensureCanMutateNextListeners() {
  	//currentListeners只有一层，因此浅拷贝即可
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }


  //拿到当前state的一个方法，会返回当前的currentState
  function getState() {
    //如果当前正在执行redcer去修改state，则抛出错误
    if (isDispatching) {
      throw new Error(
        'You may not call store.getState() while the reducer is executing. ' +
          'The reducer has already received the state as an argument. ' +
          'Pass it down from the top reducer instead of reading it from the store.'
      )
    }
    return currentState
  }


  //消息订阅，传入一个litsener，当我们调用dispatch方法的时候，会去触发每一个listenr的执行
  function subscribe(listener) {
    //如果传入的listener不是一个function，则抛出错误
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.')
    }
    //如果当前正在通过reducer去更新state，也抛出错误
    if (isDispatching) {
      throw new Error(
        'You may not call store.subscribe() while the reducer is executing. ' +
          'If you would like to be notified after the store has been updated, subscribe from a ' +
          'component and invoke store.getState() in the callback to access the latest state. ' +
          'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
      )
    }
    //声明一个变量代表，当前正在订阅消息
    let isSubscribed = true

    //去保证nextListeners和currentListeners不会指向同一个引用,不会互相影响
    ensureCanMutateNextListeners()
    //将新来的listener传入到nextListeners
    nextListeners.push(listener)


    //返回一个取消订阅的方法
    return function unsubscribe() {
      //判断当前是否正在订阅消息
      if (!isSubscribed) {
        return
      }
      //如果当前正在通过reducer去更新state，则不允许取消订阅
      if (isDispatching) {
        throw new Error(
          'You may not unsubscribe from a store listener while the reducer is executing. ' +
            'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
        )
      }
      //取消了订阅状态
      isSubscribed = false
      //从nextListeners中删除之前加入的listener
      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }


  //接受一个action对象
  function dispatch(action) {

    //检查是否是一个对象
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
          'Use custom middleware for async actions.'
      )
    }
    //且该action对象必须要有他的type属性
    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
          'Have you misspelled a constant?'
      )
    }
    //如果当前redecer正在更新state，也不允许执行
    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }
    //改变当前dispatching的状态，将当前的currentState和action传给combination，最终会返回一个全新的state
    try {
      isDispatching = true
      //前方高能！！！这个地方才是真正改变state的地方！！！
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    //将nextListeners赋值给listeners和currentListeners
    const listeners = (currentListeners = nextListeners)
    //去执行这些listeners
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
    //最后返回这个action
    return action
  }



  //替换当前Reducer的方法
  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }
    currentReducer = nextReducer
    dispatch({ type: ActionTypes.REPLACE })
  }


  //这个方法不详，后期用到了的话，再去深究（感觉react-redux应该会用到）
  function observable() {
    const outerSubscribe = subscribe
    return {
      subscribe(observer) {
        if (typeof observer !== 'object' || observer === null) {
          throw new TypeError('Expected the observer to be an object.')
        }
        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }
        observeState()
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },
      [$$observable]() {
        return this
      }
    }
  }

  /*
  * 初始化你的state（你的每一个reducer会传入一个state，该state将作为你的初始值）
  * 还记得createStore的第二个参数initState吗？个人感觉第二个参数很累赘
  * 因为这里dispatch会初始化一次我们的state，且如果在开发环境下，还会去调用combineReducer文件中的getUnexpectedStateShapeWarningMessage这个方法，如果传入的initState的键名跟Reducers的键名不匹配，传入的initState将会被替换掉，还会报警告信息，因此个人感觉第二个参数initState很累赘（可能是我阅历太浅，分析不到位，仅是个人观点，毕竟Redux这个开源项目有600多个贡献者）
  */
  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}

```
</br>
</br>


> #### 总结

如果看完分析还是一脸懵逼，我们将以一幅图来简单理解**creatStore.js和combineReducer.js**这两个方法如何打造大片Redux江山的：
</br>
![](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-two-one-2.jpg)
</br>
下一篇我们将剖析Redux最难以理解的一部分，也就是我们从一开始一直避讳的createStore方法中的第三个参数**enhancer**

