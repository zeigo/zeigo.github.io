---
title: React-Redux总结
date: 2017-12-04 17:52:19
tags: [React, JavaScript, Redux]
category: JavaScript
description: react-redux学习的总结和源码的分析
---
Redux的文档里[提到参考了Elm的架构](https://redux.js.org/docs/introduction/PriorArt.html#elm)，之前一直写[Elm](elm-lang.org)，对Redux的东西接受的也比较快，这里主要讨论`react-redux`里的`Provider`，`connect`实现，探讨其中性能优化的部分。
<!-- more -->

## 基础

```js
import { Provider, connect } from "react-redux"
```

 Component 组件| Presentational展示 | Container 容器
 ---|---|---
数据来源 | props | store, ownProps
修改数据| props提供callback | dispatch action

## Provider

传入store，只需渲染根组件时用`<Provider />`，使所有容器组件可访问store，不必层层传递。

### 使用

```js
let store = createStore(todoApp)

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
```

### 原理

父组件通过`getChildContext()`向context中传值{ store: this.props.store }，而通过了contextType验证的子组件可以通过`this.context.store`拿到store。

```js
class Provider extends Component {
  getChildContext() {
    return { [storeKey]: this[storeKey], [subscriptionKey]: null }
  }

  constructor(props, context) {
    super(props, context)
    this[storeKey] = props.store
  }

  render() {
    return Children.only(this.props.children)
  }
}

Provider.childContextTypes = {
  [storeKey]: storeShape.isRequired,
  [subscriptionKey]: subscriptionShape,
}
```

### createProvider

允许控制store在context中的key值，存在多store才需要，同时`connect`的options也要设置相应的`storeKey`。但不推荐多store，比较鸡肋。。

```js
import { createProvider } from "react-redux"

const MY_KEY = "mystore"
const Provider = createProvider(MY_KEY)

<Provider store={store} />

connect(.., .., .., { storeKey: MY_KEY })(App)
```

## connect

### 简易实现

不通过`connect`，直接context，contextTypes并注册监听。

```js
class App extends Component {
  componentDidMount() {
    const { store } = this.context;
    this.unsubscribe = store.subscribe(() =>
      this.forceUpdate()
    );
  }

  render() {
    const props = this.props;
    const { store } = this.context;
    const state = store.getState();
    // ...
  }
}

App.contextTypes = {
  store: PropTypes.object
}
```

### 使用

形成容器组件使用`connnect()`而不是原生，连接React组件和store，以获得优化。主要两点作用：

1. 从context拿到store，进而得到`getState` `dispatch`;

1. 监听store的state和容器组件ownProps，判定是否组合并重渲染，并实现优化。

```js
const AppCtn = connect(
  mapStateToProps,
  mapDispatchToProps
)(App)

export default AppCtn
```

### 参数

`connect([mapStateToProps], [mapDispatchToProps], [mergeProps], [options])`

1. mapStateToProps(state, [ownProps]): stateProp: 设置该参数时，使该组件可以监听redux store。

1. mapDispatchToProps(dispatch, [ownProps]): dispatchProps: action creator，若省略，默认注入`dispatch`。

1. mergeProps(stateProps, dispatchProps, ownProps): props: 默认返回`Object.assign({}, ownProps, stateProps, dispatchProps)` 的结果，可用于将props绑定到action creator等，结果作为被包裹组件的props。

1. options(object)

- pure(Boolean)

如果为 true，当相关的state/props可能变化时，若经浅比较没变，不会重渲染或调用`mapStateToProps`，`mapDispatchToProps`，`mergeProps`等。默认值为 true。

- areStatesEqual(next, prev): bool(Function)

用来比较store state的方法，默认值`strictEqual(===)`，当`mapStateToProps`操作代价大，而只与state一部分有关时可以使用。

- areOwnPropsEqual(Function)

比较ownProps，默认`shallowEqual`。

- areStatePropsEqual(Function)

比较`mapStateToProps`结果，默认`shallowEqual`。

- areMergedPropsEqual(Function)

比较`mergeProps`结果，默认`shallowEqual`。

- storeKey(String)

## shallowEqual实现

```js
const hasOwn = Object.prototype.hasOwnProperty

//比较基本类型或同一个对象，Object.is()的polyfill
function is(x, y) {
  if (x === y) { //is(+0, -0) === false
    return x !== 0 || y !== 0 || 1 / x === 1 / y
  } else { //is(NaN, NaN) === true
    return x !== x && y !== y
  }
}

export default function shallowEqual(objA, objB) {
  if (is(objA, objB)) return true

  //过滤不都是对象的情况
  if (typeof objA !== 'object' || objA === null ||
      typeof objB !== 'object' || objB === null) {
    return false
  }

  const keysA = Object.keys(objA)
  const keysB = Object.keys(objB)

  //长度不同直接false
  if (keysA.length !== keysB.length) return false

  //若B自身不含Akey的key，或都有该key，但is(va, vb)返回false时，返回false
  for (let i = 0; i < keysA.length; i++) {
    if (!hasOwn.call(objB, keysA[i]) ||
        !is(objA[keysA[i]], objB[keysA[i]])) {
      return false
    }
  }

  return true
}
```

可见对象比较最多只比较一层，无法精确比较嵌套对象。

## 简单的connect实现

```js
import React, { Component } from "react"
import PropTypes from "prop-types"

const connect = (mapStateToProps, mapDispatchToProps, mergeProps, options) => (App) => {
  const mergeProps = mergeProps || ((...args) => Object.assign({}, ...args))

  class Ctn extends Component {
    static contextPtops = {
      store: PropTypes.object
    }

    constructor() {
      super()
      this.state = {
        allProps: {}
      }
    }

    updateAllProps() {
      const { store } = this.context
      let stateProps = mapStateToProps
        ? mapStateToProps(store.getState(), this.props)
        : {} // 防止 mapStateToProps 没有传入
      let dispatchProps = mapDispatchToProps
        ? mapDispatchToProps(store.dispatch, this.props)
        : {} // 防止 mapDispatchToProps 没有传入
      let newAllProps = {
        ...stateProps,
        ...dispatchProps,
        ...this.props
      }
      this.setState({ allProps: newAllProps })
    }

    componentWillMount() {
      const { store } = this.context
      updateAllProps()
      if (mapStateToProps) //传入mapStateToProps时才监听
        this.unsubscribe = store.subscribe(() => this.updateAllProps())
    }

    componentWillReceiveProps(nextProps) {
      shallowEqual(nextProps, this.props) || this.updateAllProps() //ownProps改变才setState
    }

    shouldComponentUpdate(nextProps, nextState) {
      return !shallowEqual(nextState.allProps, this.state.allProps)
      //简单的优化，当state.allProps改变才update，才更新到子组件
    }

    componentWillUnmount() {
      this.unsubscribe && this.unsubscribe()
    }

    render() {
      return <App {...this.state.allProps} />
    }
  }

  return Ctn
}
```

## 总结

总的来说Redux的原理还是很简单的，它只是提供了一个状态管理的工具，而如何根据业务组织数据，做好性能优化才是重点。它借鉴了Elm的架构，但Elm这门函数式语言自带的数据不变性(Immutability)，纯函数，静态类型，模式匹配以及强大的类型组合是Redux实现不了的，需要自己实现。