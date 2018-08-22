# Redux从入门到放弃 —— 理解与使用(一)


### 理解
##### 1、我眼中的Redux
<font size="2">它好比一个容器，装载着整个项目的数据源，充当着一个**全局"状态"管理者**的一个角色。</font>
    
</br>
例如在Ract中，组件2想要跟组件3进行通信，则必须依赖组件1的存在：</br>
![image](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-one-1.jpg)
</br>

但是当Redux的引入后，整个组件的通信将会变得如此简单：</br>
![image](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-one-2.jpg)
</br>
</br>

##### 2、工作原理
在Redux中，它是由Store、Reducer、Action这三兄弟构成，整个Redux运作也全靠这三兄弟的齐心协力。
</br>
</br>
> **Store:** 

三兄弟的老大哥，他负责管账，所有的财务都将由大哥负责。（Store相当于一个state容器，所有的数据都将保存在整个容器中）
</br>
</br>
> **Reducer:** 

三兄弟的老二，他负责记账，一旦账单所有变化就会立即通报给大哥，然后大哥会根据账单做出相应财产调整。（Reducer相当于一个载体，一旦state发生改变，他就会将此信息传递给store并更新其中的数据）
</br>
</br>
> **Action：** 

三兄弟的老三，负责对外社交，行贿之事都交由他负责，一旦谈判成功，老二就会记下这笔账并通报大哥。（Action相当于一个改变state的操作，用于定义Store中的state应该如何修改）
</br>
</br>
结合React，下面这幅图将很好地诠释Redux的工作流程：</br>
![image](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-one-3.jpg)



### 使用
##### 1、安装
> npm install redux

</br>

##### 2、创建store(store.js文件)
```javascript
import { createStore ,combinereducers } from 'redux';
/*这里的reducers是从reducer文件引入的，后续会说明*/
import Reducers from "./reducers";

/*
*  creatStore一共接收三个参数，
*  第一个参数：combinereducers这个方法是用来合并你所有的Reducer，最终整合成一个大的Reducer传入Store
*  第二个参数：代表初始化的State
*  第三个参数：是用于强化改造createStore的一个方法，此处暂不说明，后期通过源码分析你将有深刻体会
*/
const initState = {name:"Shmily"}
const store = createStore(combinereducers(Reducers),initState);

export default store;
```
</br>
</br>

##### 3、reducers.js文件
```javascript
/*
*	每个Reducer都将接收两个参数
*	第一个参数代表初始值state
*	第二个参数是一个action，Reducer会根据action.type的类型最终返回一个新的state去覆盖以前的state
*	在reducer文件中，我们最终输出的将是一个对象，里面包含所有的reducer
*/
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
export default {Name,Age}
```
</br>
</br>

##### 4、actions.js文件
```javascript
/*Action其实就是一个对象*/
export function set_Name(parm) {
    return {
        type: 'name',
        name:parm
    }
}
export function set_Age(parm) {
    return {
        type: 'age',
        age:parm
    }
}
```
</br>
</br>

##### 5、View层去触发改变Store
```javascript
import {set_Name ,set_Age} from "./actions"
import store from "./store.js"

/*
*  从store.js文件中引入我们的所创建的store
*  该store有一个dispath方法，他接收一个action，你可以理解为他搭载着action去告知reducer应该如何更新state
*/
store.dispatch(set_Name("Bob"));
store.dispatch(set_Age(24));
```
</br>
</br>


**以上便是Redux的理解和基本用法，相信于你而言并没有什么难度，后面我们将继续我们的： Redux从入门到放弃 —— 源码的进阶（二）**


