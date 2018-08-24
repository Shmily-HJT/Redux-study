# Redux从入门到放弃 —— 源码的进阶（三）
</br>

## applyMiddlewrap.js
在前面的文章中，我一直避开**createStore**的第三个参数**enhancer**不谈，因为个人感觉这是Redux中最难理解的一部分，而当初在初探Redux时，就试图想一口吃个大胖子，后来得不偿失，所以我将其放在了源码进阶的第三部分进行阐述，只有真正懂了这部分，redux中间件等操作你将融会贯通。
</br>

> ### compose

在初探**applyMiddlewrap.js**文件之前，我们先来看看**compose**这个方法，虽然这个方法只有短短8行代码，却也够我们喝一壶了
```javascript
/*
*	该方法接收的参数是一个function数组
*	reducer这个API接收一个function当做规则，然后会根据此规则去遍历调用他的数组，最后根据该规则返回相应值
*	最后compose函数的返回值将是 (...args)=>fn1(fn2(fn3(...args)))这么一个函数，再多函数也以此类内
*/
function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}


//还是有点蒙没关系，我们将结合一个例子进行分析
var fn1 = (value) => "fn1-" + value
var fn2 = (value) => "fn2-" + value
var fn3 = (value) => "fn3-" + value
//输出结果为：fn1-fn2-fn3-done
compose(fn1,fn2,fn3)('done')


//接下来，我们再一步一步分析这个结果的原因
/******第一步******/
a = fun1;  b = fun2;
a(b(...args)) = (...args) => fn1(fn2(...args));
返回值为：(...args) => fn1(fn2(...args))

/******第二步******/
a = (...args) => fn1(fn2(...args)); b = fn3;
a(b(...args)) = fn1(fn2(fn3(...args)));
返回值为：(...args) => fn1(fn2(fn3(...args)));

//因此compose函数最终的返回值就是(...args) => fn1(fn2(fn3(...args)))这样一个函数
//带入'done'参数，fn1(fn2(fn3('done')))
```
</br>
</br>

> ### applyMiddlewrap

接下里走进本文的主角**applyMiddlewrap**方法，首先我们来看看我们一般是如何使用他的：
```javascript
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';    //这是一个异步中间件，你就把他理解为一个函数就行
import reducer from './reducers';   //不再阐述，参考前文

const store = createStore(
  reducer,
  applyMiddleware(thunk)
);
```
然后我们再回到creatStore方法中，看看**enhancer这个参数**（也就是applyMiddleware(thunk)）如何被调用的：
```javascript
/*
*  由此可见applyMiddleware(thunk)最终会返回一个函数并接收当前的createStore方法最为参数
*  然后生成一个全新的createStore方法
*  最后把默认传入的reducer和preloadedState参数作为新的createStore方法中的两个参数
*/
enhancer(createStore)(reducer, preloadedState)
```

上面的代码我们在applyMiddleware方法中传入了一个thunk的中间件（其实就是一个函数），但是applyMiddleware方法跟上面的compose方法一样，同样可以接收多个函数，接下来我们来看看applyMiddleware的源码：
```javascript
import compose from './compose'

/*
* 由此可见，applyMiddleware方法最终会返回一个createStore方法
* 这也就是为什么我之前会说createStore方法enhancer这个参数就是改造强化createStore方法的原因
*/
export default function applyMiddleware(...middlewares) {
  //基于上述使用和分析，其实...args就是我们初始传入的reducer, preloadedState两个参数
  //而这里的createStore就是我们所传入的createStore的初始方法
  //伪代码就是这样：(createStore) => (reducer,preloadedState) => {......}
  return createStore => (...args) => {
    //生成一个store
    const store = createStore(...args)
    //错误检测处理
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    
    //拿到强化前的store的`getState`和`dispatch`两个方法
    //这两个API非常有用，第一个可以拿到当前的state，第二个可以改变state
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }

    //将store中可供外部改造的API传给每一个中间件
    //可以推测这里的chain也会是一个函数组成的数组
    const chain = middlewares.map(middleware => middleware(middlewareAPI))

    /*
    * 结合之前的compose函数分析,这里最终将返回(...args) => fn1(fn2(fn3(...args)))这么一个函数
    * 对应此处，dispatch应该为：(store.dispatch) => chain1(chain2(chain3(store.dispatch)))
    * 最终返回了一个dispatch方法，因此这些中间件只是去改造了dispatch这个方法执行过程
    */
    dispatch = compose(...chain)(store.dispatch)
	
    //返回原本store的那行API，以及改造后的dispatch函数
    return {
      ...store,
      dispatch
    }
  }
}
```
</br>
</br>

> ### 中间件的编写

现在我们终于知道了，**enhancer**这第三个参数其实就是用来改造createStore函数的，而最终动手脚的部分还是在dispatch方法上。目前我们只是知道了，chain是一个函数数组，compose(...chain)(store.dispatch)会返回一个新的dispatch函数，但是这其中的猫腻，还未戳破，这些猫腻便是我们接下来将学习的 —— 如何编写一个Redux中间件
</br>
</br>
首先我们来观摩一下最简单的一个中间件长什么样子：
```javascript
//该中间件就只复制打印action和当前store中的数据源
const logger = store => next => action => {
    console.log('dispatch', action);
    console.log('nextState', store.getState());
    //下面这段代码必须有
    return next(action);
};
//将该中间件放入applyMiddleware中
applyMiddleware(logger)
```
咋一看，这代码通过箭头函数显得很简洁，但是想要理解透他在干什么还真是一个复杂的过程：
```javascript
/*******第一步：变形logger函数*******/
function logger(store){
	return function(next){
    	return function(action){
        	console.log('dispatch', action);
            console.log('nextState', store.getState());
            return next(action);
        }
    }
}


/*******第二步：传入middlewareAPI*******/
function logger({getState: store.getState,dispatch: (...args) => dispatch(...args)}){
	return function(next){
    	return function(action){
        	console.log('dispatch', action);
            console.log('nextState', store.getState());
            return next(action);
        }
    }
}


/*******第三步：传入store.dispatch*******/
function logger({getState: store.getState,dispatch: (...args) => dispatch(...args)}){
	return function(store.dispatch){
    	return function(action){
        	console.log('dispatch', action);
            console.log('nextState', store.getState());
            return store.dispatch(action);
        }
    }
}

/*
*	dispatch函数本身就是需要接受一个action，因此我们可以发现最终返回的一个函数就是一个原本的dispatch函数
*	在'return next(action)'之前我们便可以自由添加一些逻辑（例如异步）
*	现在我们回头来看看为什么每个中间件必须有'return next(action)'这段代码
*	还记得compose函数,最终将返回(...args) => fn1(fn2(fn3(...args)))这么一个函数吗？
*	如果我们有多个中间件也将变成这样一个形式：(store.dispatch) => chain1(chain2(store.dispatch))
*	chain1跟chain2便是两个中间件，他们都需要接受一个next的参数，而这个next便是store.dispatch这个函数
*	倘若chain2这个中间件没有'return next(action)'这段代码，那chain1这个中间件的参数将不再是store.dispatch
*	最终dispatch = compose(...chain)(store.dispatch)这段代码的dispatch方法也将为undefined
*	因此对于Redux中间件而言基本的格式如下：
*/
const XXX = store => next => action => {
	//这里可以写你的中间件的逻辑代码，而这个XXX便是你的中间件
    return next(action);
};
```
</br>
</br>
##### Redux-thunk中间件
之前我们说到了这个Redux-thunk中间件，他可以异步处理我们的数据，通过上面的分析，我们再来剖析一下redux-thunk这个中间件到底如何使dipatch可以异步改变数据的。
</br>
</br>
首先我们来看看使用redux-thunk后，到底如何通过异步来改变数据源:
```javascript
//正常的action是一个纯对象
function action(name) {
    return {
        type: 'actionType',
        name: name
    }
}

//使用redux-thunk后，异步改变数据源的action返回的是一个方法
//但是我们可以惊奇发现，该action其实在3s之后还是会传入一个对象给dispatch函数
function action() {
    return (dispatch, getState) => {
		setTimeout(()=>{
        	dispatch({
                type: 'actionType',
                name: "shmily"
            });
        },3000)
    }
}
```
</br>

知道了如何异步改变store后，我们再来看看github有9681个start的**redux-thunk源码**
```javascript
//一段10行的代码，有一大半是按照中间件模板来编写，算上大括号只有3行代码是自己的加入的......
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
  	/*
    *	action本来应该是一个对象，这里处理异步的关键就在于我们的action变成了一个函数
    *	如果action是一个函数就将dispatch, getState参数传入
    *	在我们使用时便可以在异步拿到数据后，直接利用dispatch去改变store
    */
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    
    
    /*
    * 最后你会发现，之前不是说'next(action)'这个代码不执行就会阻塞后面的中间件了吗？
    * 的确如此，因此你会发现这些中间件在官方其实有一个先后顺序
    * 如果使用redux-thunk的话，必须将其当做applyMiddlewrap的第一个参数
    * 他将成为最后一个中间件，因此就算最后没有返回next(action)，也不会造成影响
    */
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
```
</br>
</br>

看到这，如果我们心心相映，相信你已经基本参透redux的原理了，老样子，如果此时的你还是有点蒙蔽，我们将结合前面两章的源码分析，以图示之整个中间件流程：
</br>
![流程图](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-two-three-1.jpg)
</br>
</br>


***通过redux的基本用法和源码的进阶，下一章我们将继续介绍Redux在大型应用中的实战~~***


