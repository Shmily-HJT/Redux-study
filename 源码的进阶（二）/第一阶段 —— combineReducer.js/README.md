# Redux从入门到放弃 —— 源码的进阶（二）
</br>



## combineReducers.js
</br>

通过之前 [Redux从入门到放弃 —— 理解与使用（一）](https://github.com/Shmily-HJT/Redux-study/tree/master/%E7%90%86%E8%A7%A3%E4%B8%8E%E4%BD%BF%E7%94%A8%EF%BC%88%E4%B8%80%EF%BC%89)的学习，createStore他接收的第一个参数便是**combineReducers(reducer)**,源码分析的第一步，我们便先从**combineReducers.js**这个文件开刀。
</br>
</br>

> #### 使用回顾

首先我们先来回顾一下之前的操作流程：
```javascript
const Name = (state = {name:"shmily"}, action) => {
    switch (action.type) {
        case 'name':
            return {name:action.name}
        default:
            return state;
    }
}
const Age = (state = {age:22}, action) => {
    switch (action.type) {
        case 'age':
            return {age:action.age}
        default:
            return state;
    }
}
/*使用combineReducers将所有Reducer进行合并*/
combineReducers({Name ,Age})
```
</br>
</br>

> #### 代码架构

```javascript
import ActionTypes from './utils/actionTypes';
import warning from './utils/warning';
import isPlainObject from './utils/isPlainObject';

function getUndefinedStateErrorMessage(key, action){
	/*从命名分析，它的作用很明显：就是去检测某一个state是否被定义，然后进行相关错误处理*/
}

function getUnexpectedStateShapeWarningMessage(inputState,reducers,action,unexpectedKeyCache){
	/*这个方法从命名来看，也同样是用来处理state相关的一些错误信息的*/
}

function assertReducerShape(reducers){
	/*该方法主要将reducers传入，然后检测最后的返回值是否符合要求（结合使用分析，reducer最后会返回）一个state*/
}

function combineReducers(reducers){
	/*今天的主角*/
}
```
</br>
</br>



> #### 头部的三个引用文件

```javascript
/*
* **************actionTypes.js文件**************
* 它主要声明了三种actionTypes类型
* 都是用一串字符串+随机数组合而成
*/
const randomString = () =>
  Math.random()
    .toString(36)
    .substring(7)
    .split('')
    .join('.')

const ActionTypes = {
  INIT: `@@redux/INIT${randomString()}`,
  REPLACE: `@@redux/REPLACE${randomString()}`,
  PROBE_UNKNOWN_ACTION: () => `@@redux/PROBE_UNKNOWN_ACTION${randomString()}`
}
export default ActionTypes


/*
* **************worning.js文件**************
* 用来处理错误信息
*/
function warning(message) {
  if (typeof console !== 'undefined' && typeof console.error === 'function') {
    console.error(message)
  }
  try {
    throw new Error(message)
  } catch (e) {}
}


/*
* **************isPlainObject.js文件**************
* 判断是否为一个对象，且他的原型不能继承null，确保对象的正确性
*/
function isPlainObject(obj) {
  if (typeof obj !== 'object' || obj === null) return false
  let proto = obj
  /*getPrototypeOf拿到传入obj继承原型*/
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto)
  }
  return Object.getPrototypeOf(obj) === proto
}
```
</br>
</br>



> #### getUnexpectedStateShapeWarningMessage

```javascript
/*
* 如果你的reducer最终返回为undefined，就会调用该方法，
* 他会根据你的传入reducer的action，帮你找哪一个reducer没有返回值
*/
function getUndefinedStateErrorMessage(key, action) {
  /*如果action不是null、undefined等字段，将默认拿到action的type值*/
  const actionType = action && action.type
  /*如果actionType存在，拿到具体出问题的actionType，否则只能笼统提示某一处的action有问题*/
  const actionDescription = (actionType && `action "${String(actionType)}"`) || 'an action'
  /*返回报错信息*/
  return (
    `Given ${actionDescription}, reducer "${key}" returned undefined. ` +
    `To ignore an action, you must explicitly return the previous state. ` +
    `If you want this reducer to hold no value, you can return null instead of undefined.`
  )
}
```
</br>
</br>


> #### getUnexpectedStateShapeWarningMessage

```javascript
/*
*  这个方法代码量有点大，但是别怕，他的主要功能就是错误检测，判断是否有一些无效的state，如果有则会被新的state进行替换
*  他接受四个参数：
*  第一个参数代表store中保存的state
*  第二个参数代表以对象的形式组合而成的所有reducers
*  第三个参数就是一个普通的action
*  第四个参数就是一个空对象，用来记录下那些无效的state的键值
*  （如果对以上描述有些懵逼，请继续往下看~~~）
*/
function getUnexpectedStateShapeWarningMessage(inputState,reducers,action,unexpectedKeyCache){
  /*遍历reducers对象（回想一下传入combineReducers时那个reducers对象），并拿到他的key键值*/
  const reducerKeys = Object.keys(reducers);
  
  /*判断传入的action是否代是Redux内部的一个初始化action，从而收集到一些错误信息*/
  const argumentName = action && action.type === ActionTypes.INIT ?
    			'preloadedState argument passed to createStore'
      			: 'previous state received by the reducer'；
                
  /*如果传入的reducers是一个空对象，从返回错误信息*/              
  if (reducerKeys.length === 0) {
    return (
      'Store does not have a valid reducer. Make sure the argument passed ' +
      'to combineReducers is an object whose values are reducers.'
    )
  }
  
  /*判断传入state是否为一个对象，且他的原型不能继承null，以确保该对象的正确性*/
  if (!isPlainObject(inputState)) {
    return (
      `The ${argumentName} has unexpected type of ` +
      {}.toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] +
      `. Expected argument to be an object with the following ` +
      `keys: ${reducerKeys.join('", "')}`
    )
  }
  
  /*
  * 判断state的键名是否在reducers中都存在着该键名，也就是需要一一对应
  *（简言之，就是state有的键名，对应着reducers中也应该有）
  * 你可能会疑问，为什么store中的全局state会和reducer的key值有一一对应关系？
  * 这个疑问你将在combination方法中得到解决.....继续耐心往下看吧~~~
  */
  const unexpectedKeys = Object.keys(inputState).filter(
    key => !reducers.hasOwnProperty(key) && !unexpectedKeyCache[key]
  )
  
  /*
  * 记录下那些无效的键名，但是为什么设置为true，实在摸不透......
  * 因为后面也没有unexpectedKeys的使用了
  */
  unexpectedKeys.forEach(key => {
    unexpectedKeyCache[key] = true
  })
  
  /*如果当前正在进行state的替换操作，则忽略掉*/
  if (action && action.type === ActionTypes.REPLACE) return
  
  /*抛出警告信息，告诉我们将自动替换或抹掉那些无用state*/
  if (unexpectedKeys.length > 0) {
    return (
      `Unexpected ${unexpectedKeys.length > 1 ? 'keys' : 'key'} ` +
      `"${unexpectedKeys.join('", "')}" found in ${argumentName}. ` +
      `Expected to find one of the known reducer keys instead: ` +
      `"${reducerKeys.join('", "')}". Unexpected keys will be ignored.`
    )
  }
}
```
</br>
</br>

> #### assertReducerShape

```javascript
/*这个方法主要用于检测每一个reducer最终是否有返回值，不能没有返回值！！！*/
function assertReducerShape(reducers) {
  Object.keys(reducers).forEach(key => {
    const reducer = reducers[key]
    const initialState = reducer(undefined, { type: ActionTypes.INIT })
    /*检测你的reducer中最后是否会返回一个新的state对象，如果没有返回值，会报错*/
    if (typeof initialState === 'undefined') {
      throw new Error(
		"balabala......(这里我省略了报错信息)"
      )
    }
    if (
      typeof reducer(undefined, {type: ActionTypes.PROBE_UNKNOWN_ACTION()}) === 'undefined'
    ) {
      throw new Error(
        "balabala......(这里我省略了报错信息)"
      )
    }
  })
}
```
</br>
</br>

> #### combineReducers

combineReducers.js文件中的主角登场，通过对他的剖析，你将消除之前的种种疑惑:
```javascript
/*
*  根据combineReducers的用法，我们可以发现reducer其实是一个function，最终会返回一个全新的state
*  因此combineReducers方法中的第一步就是：去识别过滤那些不是function的reducer
*/
const reducerKeys = Object.keys(reducers)
//finalReducers用于保存符合条件的正确reducer
const finalReducers = {}
for (let i = 0; i < reducerKeys.length; i++) {
    //拿到当前的每一个reducer
    const key = reducerKeys[i]
    //判断当前环境，如果是开发环境，会对程序员的一些开发操作进行警告提示
    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }
    //去筛选符合条件的reducer
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
}
  
/*这里即将做第二次检测，判断reducer中传入的state参数是否合法*/
const finalReducerKeys = Object.keys(finalReducers)
//这个空对象参数就是在非生产环境下调用getUnexpectedStateShapeWarningMessage函数的第四个参数
let unexpectedKeyCache
if (process.env.NODE_ENV !== 'production') {
	unexpectedKeyCache = {}
}
//shapeAssertionError用来保存错误的一个变量
let shapeAssertionError
try {
	//assertReducerShape这个方法就是用来判断reducer中最终返回来出来的全新的state是否合法
	assertReducerShape(finalReducers)
} catch (e) {
	shapeAssertionError = e
}

/*
*  最后，combineReducers这个方法其实返回的就是一个函数
*  而该函数又是一个reducer，他接受一个state和action
*/
return function combination(state = {}, action) {
    //如果上面捕获到了错误，就抛出这个错误
    if (shapeAssertionError) {
      throw shapeAssertionError
    }

    //判断当前所处的环境，利用getUnexpectedStateShapeWarningMessage去处理一些警告信息
    if (process.env.NODE_ENV !== 'production') {
      const warningMessage = getUnexpectedStateShapeWarningMessage(
        state,
        finalReducers,
        action,
        unexpectedKeyCache
      )
      if (warningMessage) {
        warning(warningMessage)
      }
    }

    //这两个变量代表着是否有更新，和下一版的新的state
    let hasChanged = false
    const nextState = {}

    //循环遍历每一个reducer
    for (let i = 0; i < finalReducerKeys.length; i++) {
      //拿到每一个reducer的键名
      const key = finalReducerKeys[i]
      //拿到当前键名的reducer函数，其实跟finalReducers[i]没啥区别
      const reducer = finalReducers[key]
      
      /* 
      * 前方高能！请注意！这里即将揭开为什么每一个reducer的键名都将跟state的键名一一对应真正原因
      */
      
      //通过key值，找到对应的state
      const previousStateForKey = state[key]
      //执行该reducer，拿到最终的返回值
      const nextStateForKey = reducer(previousStateForKey, action)
      //如果返回值是undefined，抛出错误
      if (typeof nextStateForKey === 'undefined') {
        const errorMessage = getUndefinedStateErrorMessage(key, action)
        throw new Error(errorMessage)
      }
      /*
      * !!!关键之处!!!他将之前的空对象nextState的键名设置为了reducer的键名
      * 且键值为执行reducer后的最终返回值
      */
      nextState[key] = nextStateForKey
      //比较nextStateForKey和previousStateForKey是否发生改变
      //此处有个点需要注意！！！如果我们的state是引用类型值
      //那么nextStateForKey !== previousStateForKey永远为true
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
    //遍历完所有state，最终根据hasChanged来决定是否更新当前state
    return hasChanged ? nextState : state
}
```




## 总结
可能读完后的你还是有点蒙蔽状态，没关系，我们再通过一个流程图看看他到底干了什么事情：
</br>
![](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-two-one-1.jpg)
</br>
</br>
总的来说，combineReducer最重要的作用就是返回了一个函数，且这个函数会根据我们传入的Reducer去得到一个nextState，最后根据这个nextState和原本的preState进行对比，决定是否更新当前的全局状态。
</br>
</br>
下一篇，将带来 [creatStore.js的源码分析](https://github.com/Shmily-HJT/Redux-study/tree/master/%E6%BA%90%E7%A0%81%E7%9A%84%E8%BF%9B%E9%98%B6%EF%BC%88%E4%BA%8C%EF%BC%89/creatStore.js%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90)，只要你掌握了这两个js文件的源码，Redux也就手握大片天下了~~~
