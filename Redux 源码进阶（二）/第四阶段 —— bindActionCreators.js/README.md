# 源码的进阶（4） —— bindActionCreator.js

</br>

如果你前三章都掌握了，我相信 redux 于你而言，已经揭开了它的神秘面纱。但是当我们在实战开发中想要更灵活把玩 Redux，我们还得学学他最后一个方法： **bindActionCreator**

```javascript
/*
 * 还记得以前我们必须store.dispatch（action）这样去改变状态
 * 现在通过bindActionCreators我们可以直接func（）去调用，他能自动dispatch对应的action，
 * 因此bindActionCreators是redux提供的一个辅助方法，能够让我们以方法的形式来调用action
 */

/*
 * 这个函数主要就是返回一个dispatch函数，然后当我们通过func()调用去改变state时
 * 其实就会自动调用这个返回函数，因此也就dispatch对应的action
 * 这段代码这样写会更明了：return function(...args) {return dispatch(actionCreator.apply(this, args))}
 */
function bindActionCreator(actionCreator, dispatch) {
    return function () {
        return dispatch(actionCreator.apply(this, arguments));
    };
}

//第一个参数actionCreator是一个以action组合而成的一个对象，第二个参数就是store中的dispatch
export default function bindActionCreators(actionCreators, dispatch) {
    //还记得之前说过的redux-thunk异步处理吗？这就是检测该action是否是一个函数，并直接返回这个dispatch函数
    if (typeof actionCreators === "function") {
        return bindActionCreator(actionCreators, dispatch);
    }
    //如果传入的actionCreators不是一个对象或者是null值，抛出错误
    if (typeof actionCreators !== "object" || actionCreators === null) {
        throw new Error(
            `bindActionCreators expected an object or a function, instead received ${
                actionCreators === null ? "null" : typeof actionCreators
            }. ` + `Did you write "import ActionCreators from" instead of "import * as ActionCreators from"?`
        );
    }
    //获取所有action create函数的名字（因为我们想通过func直接调用改变状态的嘛，所以这里要拿到他的方法名）
    const keys = Object.keys(actionCreators);
    //保存bindActionCreator返回的函数集合，键名为方法名，键值就为bindActionCreator返回的函数
    //因此通过XXX.func（）相当于dispatch(action)
    const boundActionCreators = {};
    //遍历keys
    for (let i = 0; i < keys.length; i++) {
        const key = keys[i];
        const actionCreator = actionCreators[key];
        // 排除值不是函数的actionCreator
        if (typeof actionCreator === "function") {
            boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
        }
    }

    /*
     * 返回这个对象，而boundActionCreators的基本形式就是：
     * {
     *   funcName: function() {dispatch(actionCreator.apply(this, arguments))},
     *   ......
     * }
     * 因此我们在使用时直接funcName()即可
     */
    return boundActionCreators;
}
```

</br>
</br>
如果你没有过实战使用，我相信你看完应该不会有太大收获，不过老样子，我们最后还是以图示之，希望能帮助你理解它的作用：
</br>

![展示](../image/redux-two-four-1.jpg)

</br>
</br>
我们在后续的文章将会介绍**Redux应用实战**,然后你会彻底理解这个方法的作用
