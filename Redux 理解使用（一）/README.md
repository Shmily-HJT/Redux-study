# Redux 理解使用(一)


## 一、理解
</br>
### 1、我眼中的Redux
它好比一个容器，装载着整个项目的数据源，充当着一个 **全局"状态"管理者** 的一个角色。例如在React中，组件2想要跟组件3进行通信，则必须依赖组件1的存在：</br>
![image](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-one-1.jpg)
</br>

但是当Redux的引入后，整个组件的通信将会变得如此简单（在没有学习reaxt-redux之前，你可以这样理解）：</br>
![image](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-one-2.jpg)
</br>
</br>

### 2、工作原理
在Redux中，它是由Store、Reducer、Action这三兄弟构成，整个Redux运作也全靠这三兄弟的齐心协力。简单来理解，当平时我们修改一个对象时，可能就是直接在该对象上进行操作，而redux只是把修改对象这个过程变得更流程化，更可控化。
</br>
</br>
> **Store：** 三兄弟的老大哥，他负责记录三兄弟的财务，任何一笔收入和支出都将由老大哥记录。（Store相当于一个容器，所有的数据都将保存在这个容器中）

</br>
</br>

> **Reducer：** 三兄弟的老二，他负责跟踪三兄弟财务的变化，一有变化，就会计算出这个财务的变化结果，并通知大哥最终结果。（Reducer相当于一个载体，一旦状态发生改变，他就会返回一个新的状态，并让store做出相应改变）

</br>
</br>


> **Action：** 三兄弟的老三，他负责外勤，与各大商人打交道，一旦谈判成功或失败，他就会通知老二财务有更新（Action相当于一个改变状态的规则）

</br>
</br>
结合React，下面这幅图将很好地诠释Redux的工作流程：</br>
![image](https://github.com/Shmily-HJT/Redux-study/blob/master/image/redux-one-3.jpg)
</br>


## 二、使用
#### 1、安装
```javascript
npm install redux
```

</br>

#### 2、创建store(store.js文件)
```javascript
//这两个API时从redux中模块中引入的
import { createStore ,combinereducers } from 'redux';
//这里的reducers是从reducer文件引入的，后续会说明
import Reducers from "./reducers";

/*
*  creatStore一共接收三个参数，
*  第一个参数：combinereducers这个方法你可以简单理解为整合你的Reducer就行，将所有的Reducer整合成一个对象Reducers传入即可
*  第二个参数：代表初始化的State
*  第三个参数：是用于强化并改造createStore的一个方法，此处暂不说明，后期通过源码分析你将有深刻体会
*/
const initState = {name:"Shmily"}
const store = createStore(combinereducers(Reducers),initState);

export default store;
```
</br>
</br>

#### 3、reducers.js文件
```javascript
/*
*	每个Reducer都将接收两个参数
*	第一个参数代表初始值state
*	第二个参数是一个action，Reducer会根据action.type的类型最终返回一个新的state去覆盖以前的state
*	由此可见，action.type跟state是一一对应关系，且每一个action.type都是独一无二的
*	最后会将所有的reducer以一个对象的形式传出去，当作combinereducers这个方法的参数
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

#### 4、actions.js文件
```javascript
/*
* Action其实就是一个对象
* 每一个Action都必须有个type字段，且每个type字段的值是唯一的，不能重复
*/
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

#### 5、View层去触发改变Store
```javascript
import {set_Name ,set_Age} from "./actions"
import store from "./store.js"

/*
*  从store.js文件中引入我们的所创建的store
*  store有一个dispath方法，他接收一个action，这个API是redux中可以去更新唯一可以更新state的API
*  看到这里，你可能会有种种疑惑，不是reducer才是告诉store去更新store的吗？
*  请记录下你的疑点，我们继续往后看，很多疑惑点将慢慢解开......
*/
store.dispatch(set_Name("Bob"));
store.dispatch(set_Age(24));
```
</br>
</br>


**以上便是Redux的理解和基本用法，相信于你而言并没有什么难度，后面我们将继续我们的： [Redux从入门到放弃 —— 源码的进阶（二）](https://github.com/Shmily-HJT/Redux-study/tree/master/%E6%BA%90%E7%A0%81%E7%9A%84%E8%BF%9B%E9%98%B6%EF%BC%88%E4%BA%8C%EF%BC%89)**


