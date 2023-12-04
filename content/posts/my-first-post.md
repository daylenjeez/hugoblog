+++
title = 'React Hooks 细粒度更新'
date = 2023-11-20T22:36:39+08:00
tags = ["React"]
+++

## 前言
&nbsp;&nbsp;什么是细粒度更新？简单来说就是只更新需要更新的部分，而不是整个组件都更新。我们熟知的`Vue`就是细粒度更新，你无需关心你的组件是如何更新的，只需要关心你的数据是如何变化的，`Vue`会“自动追踪依赖”。
&nbsp;&nbsp;而`React`则不同，`React`的更新是粗粒度的，也就是说，当你的组件的任何一个`props`或者`state`发生变化时，整个组件都会重新渲染，这也是在使用`React`时令人头疼的地方；接下来我们自己实现一个简单的细粒度更新的 `React Hooks`；

## 实现

### 1. 简单实现useState
```javascript
function useState(state){
  const getter = () => state;
  const setter = (newState) => state = newState;
  return [getter,setter]
}
```

1. `useState`接收一个value为参数，形成闭包（因为`getter`和`setter`都引用了`state`）；
2. 用`getter`会返回闭包中`state`的值；`setter`会修改闭包中`state`的值；

使用方式如下：
```javascript
const [count,setCount] = useState(0);
console.log(count()); // 0
setCount(1);
console.log(count()); // 1
```

### 2. 简单实现useEffect

我们期望`useEffect`的行为是：
1. **立即执行**：当`useEffect`执行后，立即执行回调函数；（与`React Hooks`的`useEffect`一致）；
2. **依赖更新时执行**：当依赖更新时，执行回调函数；（与`React Hooks`的`useEffect`一致）；
3. **不需要显示指明依赖**：当依赖更新时，自动执行回调函数；（与`React Hooks`不一致）；

期望具体行为如下：
```javascript
const [count,setCount] = useState(0);

//effect 1
useEffect(()=>{
  console.log(`count is ${count()}`); //1.打印 count is 0
});

//effect 2
useEffect(()=>{
  console.log('nothing'); //2.打印 nothing
});

setCount(1);//3. 执行effect 1，打印 count is 1
```

要实现如上效果我们要做的核心就是让`useEffect`和`useState`建立关系 ，我们可以通过**发布订阅**来实现：
1. **订阅**：在`useEffect`回调中执行`useState`的`getter`时，该effect就订阅了该`getter`；
2. **发布**：当`useState`的`setter`执行时，就通知所有 订阅了该`getter`的`useEffect`回调函数 执行；

而要建立关系，我们需要分别在`useState`和`useEffect` 中各自维护一个订阅列表和发布列表：

1. `useState`的`subs`：用来存储订阅该`state`的`effect`；
2. `useEffect`的`effect.deps`：用来存储`effect`订阅的`state`所对应的`subs`集合；

![useState 与 useEffect 的订阅发布关系](image-1.png)

```js
function useEffect(callback){
    //...
    const effect = {
      //用于执行 useEffect 的回调函数
      execute,
      //保存该 useEffect 依赖的 state 对应 subs 集合
      deps:new Set()
    }
    //...
}

function useState(value){
  // 保存订阅该 state 变化的 effect
  const subs = new Set();
  //...
}
```

完整的``useEffect``实现如下

```js
function useEffect(callback){
  const execute = () => {
    //重制依赖
    cleanup(effect);
    //将当前 effect 推入栈顶
    effectStack.push(effect);

    try{
      //执行回调
      callback();
    } finally{
      // effect 出栈
      effectStack.pop();
    }
  }

  const effect = {
    execute,
    deps:new Set()
  }

  // 立即执行一次，建立发布订阅关系
  execute();
}
```

或许你看不懂上面的``useEffect``实现，我们可以先关注三个重要的细节：
1. 在``callback``执行前调用``cleanup``清除所有“与该``effect``相关的订阅发布关系”，在执行时，会重建发布订阅关系；
  ```js
  function cleanup(effect){
    //从该effect订阅的所有state对应subs中移除该effect
    for(const subs of effect.deps){
      subs.delete(effect)
    }
    // 从该effect依赖的多有 state 对应 subs 移除
    effect.deps.clear();
  }
  ```
  2. 在调用``state``的``getter``时，需要了解该``state``当前所处的的是哪个``effect``上下文（用于建立该``state``与``effect``的联系），因此在 callback 执行前将当前``effect``保存在栈 ``effectStack``的顶端，在``callback``执行后``effect``出栈。在 ``useState``的``getter``内部获取``effectStack``的栈顶``effect``即为“当前所处``effect``上下文”；
  3. 在``useEffect``执行后内部会执行``excute``，首次建立订阅发布关系。这也是“**自动收集依赖**”的关键。

## 结论

通过上面的代码我们可以很容易的实现一个细粒度更新的`React Hooks`，或许你会疑问为什么`React`自己不实现一个细粒度更新的`Hooks`呢？因为`React`是一个“应用级的框架” ，选择使用 Hooks 的设计方式是为了提供一种更简洁、可组合和易于理解的方式来处理组件的状态和副作用，而不是过于关注细粒度的更新；同时它具有很强的灵活性，你可以根据你的项目需求来选择是否实现细粒度更新，而不是强制性的实现细粒度更新，这也是`React`的灵活性所在。
