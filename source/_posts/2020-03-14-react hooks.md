---
title: React Hook 不完全指南
date: 2020-03-14 16:45:31
tags:
---

# 前言
`React Hook`是`React16.8.0`版本之后提出的新增特性，由于之前的项目都不怎么用到`React`，因此也就匆匆了解一下，最近因为换工作，主要技术栈变为`React`了，所以需要着重研究一下`React`的一些特性以更好地应用到项目开发中和更好地进行知识沉淀。

# Hook是什么

在解释这个问题之前，可以先看一段代码：
```javascript
import React, { useState } from 'react'
function Example() {
    // 声明一个叫 "count" 的 state 变量
    const [count, setCount] = useState(0)
    // 与 componentDidMount and componentDidUpdate效果类似
    useEffect(() => {
        // Update the document title using the browser API
        document.title = `You clicked ${count} times`
    })

    return (
        <div>
        <p>You clicked {count} times</p>
        <button onClick={() => setCount(count + 1)}>
            Click me
        </button>
        </div>
    );
}
```

`Hook`是一个特殊的函数，它可以让你“钩入”`React`的特性。例如，`useState`是允许你在`React`函数组件中添加`state`的`Hook`。如果你在编写函数组件并意识到需要向其添加一些`state`，以前的做法是必须将其它转化为`class`。现在你可以在现有的函数组件中使用`Hook`;又例如`useEffect Hook`可以告诉`React`组件需要在渲染后执行某些操作，`React`会保存你传递的函数（我们将它称之为 “effect”），并且在执行 DOM 更新之后调用它，可以把它看做`componentDidMount`，`componentDidUpdate`和`componentWillUnmount`这三个函数的组合。

> 官方解释：Hook 是 React 16.8 的新增特性。它可以让你在不编写 class 的情况下使用 state 以及其他的 React 特性。


# 为什么要提出React Hook

### 在组件之间复用状态逻辑很难

> `React`没有提供将可复用性行为“附加”到组件的途径（例如，把组件连接到`store`）。如果你使用过`React`一段时间，你也许会熟悉一些解决此类问题的方案，比如`render props`和`高阶组件`。但是这类方案需要重新组织你的组件结构，这可能会很麻烦，使你的代码难以理解。如果你在`React DevTools`中观察过`React`应用，你会发现由 `providers，consumers，高阶组件，render props`等其他抽象层组成的组件会形成“嵌套地狱”。尽管我们可以在`DevTools`过滤掉它们，但这说明了一个更深层次的问题：`React`需要为共享状态逻辑提供更好的原生途径。

`Hook`可以在无需修改组件结构的情况下复用状态逻辑，这使得在组件间或社区内共享`Hook`变得更便捷

### 复杂组件变得难以理解

我们经常维护一些组件，组件起初很简单，但随着业务复杂度的提升，组件逐渐会变得比较复杂，使得每个生命周期常常包含一些不相关的逻辑。例如，组件常常在`componentDidMount`和`componentDidUpdate`中获取数据。但是，同一个`componentDidMount`中可能也包含很多其它的逻辑，如设置事件监听，而之后需在`componentWillUnmount`中清除。相互关联且需要对照修改的代码被进行了拆分，而完全不相关的代码却在同一个方法中组合在一起,如此很容易产生`bug`，并且导致逻辑不一致,维护起来也会显得比较吃力。

为了解决这个问题，`Hook`将组件中相互关联的部分拆分成更小的函数（比如设置订阅或请求数据），而并非强制按照生命周期划分。你还可以使用`reducer`来管理组件的内部状态，使其更加可预测。

### 难以理解的 class

引用官方的话：

> 除了代码复用和代码管理会遇到困难外，我们还发现`class`是学习`React`的一大屏障。你必须去理解`JavaScript`中`this`的工作方式，这与其他语言存在巨大差异。还不能忘记绑定事件处理器。没有稳定的语法提案，这些代码非常冗余。大家可以很好地理解`props`，`state`和自顶向下的数据流，但对`class`却一筹莫展。即便在有经验的`React`开发者之间，对于函数组件与`class`组件的差异也存在分歧，甚至还要区分两种组件的使用场景。

> 另外，`React`已经发布五年了，我们希望它能在下一个五年也与时俱进。就像`Svelte`，`Angular`，`Glimmer`等其它的库展示的那样，组件预编译会带来巨大的潜力。尤其是在它不局限于模板的时候。最近，我们一直在使用`Prepack`来试验`component folding`，也取得了初步成效。但是我们发现使用`class`组件会无意中鼓励开发者使用一些让优化措施无效的方案。`class`也给目前的工具带来了一些问题。例如，`class`不能很好的压缩，并且会使热重载出现不稳定的情况。因此，我们想提供一个使代码更易于优化的`API`。

`Hook`能够在非`class`的情况下使用更多的`React`特性。 其实, `React`组件一直更像是函数。而`Hook`则拥抱了函数，同时也没有牺牲`React`的精神原则。`Hook`提供了问题的解决方案，无需学习复杂的函数式或响应式编程技术。

**最重要的是，`Hook`是向下兼容的，它和现有代码可以同时工作，你可以渐进式地使用他们，不用急着迁移到`Hook`。**

# 内置常用`HOOK`概览

## React中内置的Hook API

* 基础Hook
    * useState
    * useEffect
    * useContext

* 额外的Hook
    * useReducer
    * useCallback
    * useMemo
    * useRef
    * useImperativeHandle
    * useLayoutEffect
    * useDebugValue

## State Hook

可以看下面的代码：
```javascript
import React, { useState } from "react";
import "./styles.css";

export default function App() {
  const [count, setCount] = useState(0);
  return (
    <div className="App">
      <h1>这是一个示例</h1>
      <div>点击了{count}次</div>
      <button
        onClick={() => {
          setCount(count + 1);
        }}
      >
        点击
      </button>
      <button
        onClick={() => {
          setCount(0);
        }}
      >
        清除
      </button>
    </div>
  );
}
```

上述代码中，`useState`就是一个`Hook`。通过在函数组件里调用它来给组件添加一些内部`state`。`React`会在重复渲染时保留这个`state`。`useState`会返回一对值：当前状态和一个让你更新它的函数，你可以在事件处理函数中或其他一些地方调用这个函数。它类似`class`组件的`this.setState`，但是它不会把新的`state`和旧的`state`进行合并。

`useState`唯一的参数就是初始`state`。在上面的例子中，我们的计数器是从零开始的，所以初始`state`就是`0`。值得注意的是，不同于`this.state`，这里的`state`不一定要是一个对象，但如果你有需要，它也可以是。这个初始`state`参数只有在第一次渲染时会被用到。

你也可以在函数组件中多次使用`state Hook`。

### 调用 useState 方法的时候做了什么? 

它定义一个 “state 变量”。在上面的示例中该变量叫`count`， 但它可以是任意的变量名，比如`banana`。这是一种在函数调用时保存变量的方式，`useState`是一种新方法，它与`class`里面的`this.state`提供的功能完全相同。一般来说，在函数退出后变量就会”消失”，而`state`中的变量会被`React`保留。

### useState 需要哪些参数？

`useState()`方法里面唯一的参数就是初始`state`。不同于`class`的是，我们可以按照需要使用数字或字符串对其进行赋值，而不一定是对象。在示例中，只需使用数字来记录用户点击次数，所以我们传了`0` 作为变量的初始`state`。（如果我们想要在`state`中存储两个不同的变量，只需调用`useState()`两次即可。）


### useState 方法的返回值是什么？

返回值为：当前`state`以及更新`state`的函数。这就是我们写 `const [count, setCount] = useState()` 的原因。这与`class`里面`this.state.count`和`this.setState`类似，唯一区别就是你需要成对的获取它们。

### 继续深入

#### 每一次渲染都有它自己的props和state

我们直接看代码来方便理解：

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

`<p>You clicked {count} times</p>`看到这段代码，有一定经验的人可能会想，这其中的原理是不是通过`watcher`，或者是`data binding`或者是`proxy`来实现的呢？都不是，count仅仅只是一个数字类型的变量而已，不是上述中的任何一个，就像下面的普通的变量赋值一样：

```javascript
const count = 42;
// ...
<p>You clicked {count} times</p>
```

组件在第一次渲染的时候，从`useState()`拿到`count`的初始值`0`。当我们调用`setCount(1)`，`React`会再次渲染组件，这一次`count`是`1`。就如同下面示例的一样：

```javascript
// During first render
function Counter() {
  const count = 0; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>
  // ...
}

// After a click, our function is called again
function Counter() {
  const count = 1; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>
  // ...
}

// After another click, our function is called again
function Counter() {
  const count = 2; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>
  // ...
}
```

**当我们更新状态的时候，`React`会重新渲染组件。每一次渲染都能拿到独立的`count`状态，这个状态值是函数中的一个常量**

所以下面的这行代码没有做任何特殊的数据绑定：

```javascript
<p>You clicked {count} times</p>
```

**它仅仅只是在渲染输出中插入了`count`这个数字**。这个数字由`React`提供。当`setCount`的时候，`React`会带着一个不同的`count`值再次调用组件。然后，`React`会更新`DOM`以保持和渲染输出一致。

这里关键的点在于任意一次渲染中的`count`常量都不会随着时间改变。渲染输出会变是因为我们的组件被一次次调用，而每一次调用引起的渲染中，它包含的`count`值独立于其他渲染。


## Effect Hook

什么是副作用？`React`官网是这么定义的：

> 你之前可能已经在`React`组件中执行过数据获取、订阅或者手动修改过`DOM`。我们统一把这些操作称为“副作用”，或者简称为“作用”。

`useEffect`就是一个`Effect Hook`，给函数组件增加了操作副作用的能力。它跟`class`组件中的 `componentDidMount`、`componentDidUpdate`和`componentWillUnmount`具有相同的用途，只不过被合并成了一个`API`。

例如，下面这个组件在`React`更新`DOM`后会设置一个页面标题：

```javascript
import React, { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  // 相当于 componentDidMount 和 componentDidUpdate:
  useEffect(() => {
    // 使用浏览器的 API 更新页面标题
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

当你调用`useEffect`时，就是在告诉`React`在完成对`DOM`的更改后运行你的“副作用”函数。由于副作用函数是在组件内声明的，所以它们可以访问到组件的`props`和`state`。默认情况下，`React`会在每次渲染后调用副作用函数，包括第一次渲染的时候。

副作用函数还可以通过返回一个函数来指定如何“清除”副作用。例如，在下面的组件中使用副作用函数来订阅好友的在线状态，并通过取消订阅来进行清除操作：

```javascript
import React, { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);

    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```

在这个示例中，`React`会在组件销毁时取消对`ChatAPI`的订阅，然后在后续渲染时重新执行副作用函数。

跟`useState`一样，你可以在组件中多次使用`useEffect`。通过使用`Hook`，你可以把组件内相关的副作用组织在一起（例如创建订阅及取消订阅），而不要把它们拆分到不同的生命周期函数里。这样就有利于你对代码的维护。也再一次说明了`React`官方为什么会使用`Hook`。


### 深入

#### 每次渲染都有它自己的Effects

再次看到官网文档中的例子：

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

那么：**effect是如何读取到最新的`count`状态值的呢？**

也许，是某种`data binding`或`watching`机制使得`count`能够在`effect`函数内更新？也或许`count`是一个可变的值，`React`会在我们组件内部修改它以使我们的`effect`函数总能拿到最新的值？

都不是。

我们已经知道count是某个特定渲染中的常量。事件处理函数“看到”的是属于它那次特定渲染中的`count`状态值。对于`effects`也同样如此：

**并不是count的值在“不变”的`effect`中发生了改变，而是`effect`函数本身在每一次渲染中都不相同。**

每一个`effect`版本“看到”的`count`值都来自于它属于的那次渲染：

```javascript
// During first render
function Counter() {
  // ...
  useEffect(
    // Effect function from first render
    () => {
      document.title = `You clicked ${0} times`;
    }
  );
  // ...
}

// After a click, our function is called again
function Counter() {
  // ...
  useEffect(
    // Effect function from second render
    () => {
      document.title = `You clicked ${1} times`;
    }
  );
  // ...
}

// After another click, our function is called again
function Counter() {
  // ...
  useEffect(
    // Effect function from third render
    () => {
      document.title = `You clicked ${2} times`;
    }
  );
  // ..
}
```

`React`会记住你提供的`effect`函数，并且会在每次更改作用于`DOM`并让浏览器绘制屏幕后去调用它。

所以虽然我们说的是一个`effect`（这里指更新`document`的`title`），但其实每次渲染都是一个不同的函数 — 并且每个`effect`函数“看到”的`props`和`state`都来自于它属于的那次特定渲染。




# Hook使用规则

Hook 就是 JavaScript 函数，但是使用它们会有两个额外的规则：

* 只能在**函数最外层**调用 `Hook`。**不要**在循环、条件判断或者子函数中调用。
* 只能在`React`的函数中调用调用`Hook`。不要在普通的`JavaScript`函数中调用`Hook`，你可以：
    * 在`React`的函数组件中调用`Hook`
    * 在自定义`Hook`中调用其他`Hook`

为了更好地执行这个规则，react提供了eslint插件帮助你去检测和强制执行上述规则：[eslint-plugin-react-hooks](https://www.npmjs.com/package/eslint-plugin-react-hooks)。

## 为什么是这样的规则呢？

这要从`React`内部执行`Hook`的机制说起：

`React`函数组件中，可以使用多个`useState`或者`useEffect`，那么`React`怎么知道哪个`state`对应哪个`useState`？答案是`React`靠的是`Hook`调用的顺序。只要`Hook`的调用顺序在多次渲染之间保持一致，`React`就能正确地将内部`state`和对应的`Hook`进行关联。如果我们将一个`Hook`调用放在了条件语句中，就有可能会扰乱`Hook`的调用的顺序，导致内部错误的对应state和useState，进而导致bug的产生。

# 自定义Hook

**自定义`Hook`是一个函数，其名称以 “use” 开头，函数内部可以调用其他的`Hook`**。
当我们想在两个函数之间共享逻辑时，我们会把它提取到第三个函数中。而组件和`Hook`都是函数，所以也同样适用这种方式。

可以直接看下面的例子：

```javascript
import { useState, useEffect } from 'react';

function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.subscribeToFriendStatus(friendID, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendID, handleStatusChange);
    };
  });

  return isOnline;
}
```

与`React`组件不同的是，自定义`Hook`不需要具有特殊的标识。我们可以自由的决定它的参数是什么，以及它应该返回什么。换句话说，它就像一个正常的函数,但是它的名字应该始终以`use`开头，这样可以一眼看出其符合`Hook`的规则。

此处`useFriendStatus`的`Hook`目的是订阅某个好友的在线状态。这就是我们需要将`friendID`作为参数，并返回这位好友的在线状态的原因。

## 使用自定义Hook

```javascript
function FriendStatus(props) {
  const isOnline = useFriendStatus(props.friend.id);

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}

function FriendListItem(props) {
  const isOnline = useFriendStatus(props.friend.id);

  return (
    <li style={{ color: isOnline ? 'green' : 'black' }}>
      {props.friend.name}
    </li>
  );
}
```

自定义`Hook`是一种自然遵循`Hook`设计的约定，而并不是`React`的特性。

**自定义`Hook`必须以 “use” 开头吗**？必须如此。这个约定非常重要。不遵循的话，由于无法判断某个函数是否包含对其内部`Hook`的调用，`React`将无法自动检查你的`Hook`是否违反了`Hook`的规则。

**在两个组件中使用相同的`Hook`会共享`state`吗**？不会。自定义`Hook`是一种重用状态逻辑的机制(例如设置为订阅并存储当前值)，所以每次使用自定义`Hook`时，其中的所有`state`和副作用都是完全隔离的。

**自定义`Hook`如何获取独立的`state`**？每次调用`Hook`，它都会获取独立的`state`。由于我们直接调用了`useFriendStatus`，从`React`的角度来看，我们的组件只是调用了`useState`和`useEffect`。正如我们在之前章节中了解到的一样，我们可以在一个组件中多次调用`useState`和`useEffect`，它们是完全独立的。

# 总结

零零碎碎写了这么多，作为一个入门参考，看了这篇文章，应该会对React Hook有了大致的了解，文章中也有深入其内部机制剖析的地方，但是仅仅对state和effect部分做了简要的深入，而实际上React Hook中间还有很多的点值得去深入推敲，由于实际项目工作中用到的不多，因此也没法抓住某个坑做深入的研究，准备后续认真研读一下react的源码，对其内部机制做深入的研究。好好静下心来沉淀。


#参考文档
* [react 官方中文](https://zh-hans.reactjs.org/docs/hooks-intro.html)
* [useEffect 完整指南](https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/)
* [React Hook 不完全指南](https://segmentfault.com/a/1190000019223106#item-7-8)