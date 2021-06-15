---
title: 理解 useEffect
date: "2021-06-08"
template: "post"
draft: false
toc: true
description: "A Complete Guide to useEffect"
---

原文: [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)

TLDR Part

### 每次渲染都有自己的 Props 和 State

开始聊 effect 之前, 我们需要先聊一聊组件的渲染.

以下是一个计时器组件. 关注其中高亮的部分:

```jsx
function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      // highlight-next-line
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  )
}
```

这代表这什么呢? `count` 是否在时刻关注我们的状态(state)变化, 然后自动更新呢? 作为一个十分符合直觉的答案, 这样的想法可能会在你刚开始学习 React 的时候给你带来比较大的帮助. 但是实际上, 这个理解并不准确. 我写过一篇关于这个话题的文章, 这才是[准确的心智模型](https://overreacted.io/react-as-a-ui-runtime/). <mark>加上译文</mark>

**在上面的例子中, `count` 仅仅是个数字而已.** 内部并没有数据绑定, "观察者", "代理"等等的逻辑. 就像下面的代码示例一样, 它就是一个数字.

```jsx
const count = 42
// ...
;<p>You clicked {count} times</p>
// ...
```

当组件第一次渲染的时候, 从 `useState()` 中读取到的 `count` 是 `0`. 当我们调用了 `setCount(1)` 之后, React 会重新调用我们的组件. 这一次 `count` 值将会是 `1`, 以此类推:

```jsx
// 首次渲染
function Counter() {
  // highlight-next-line
  const count = 0 // useState() 返回的内容
  // ...
  ;<p>You clicked {count} times</p>
  // ...
}

// 点击事件之后, 函数再次被调用
function Counter() {
  // highlight-next-line
  const count = 1 // useState() 返回的内容
  // ...
  ;<p>You clicked {count} times</p>
  // ...
}

// 第二次点击事件之后, 函数被调用
function Counter() {
  // highlight-next-line
  const count = 2 // useState() 返回的内容
  // ...
  ;<p>You clicked {count} times</p>
  // ...
}
```

**无论何时, 状态更新之后, React 都会重新调用我们的组件. 每一次渲染的结果都会"看到"组件内部的 `counter` 状态值, 而这个值在函数中实际上是一个固定值.**

所以说以下这一行的代码并没有任何特殊的数据绑定逻辑:

```jsx
<p>You clicked {count} times</p>
```

**它仅仅只是将数字的值嵌入到渲染结果中而已.** 这个数字是由 React 提供的. 当我们调用 `setCount` 的时候, React 会用一个最新的 `count` 值来重新调用我们的组件. 然后 React 更新 DOM, 以匹配最新的渲染结果.

这里的关键点是: `count` 值在每次渲染的调用中, 都是固定的. 是我们的组件传递了最新的 `count` 值, 每一次渲染, 渲染结果都输出了这个值. 每一次渲染的 `count` 都互不关联.

(想要更深入地了解具体的渲染流程, 可以查看这篇文章: [React 作为 UI 运行时](https://overreacted.io/react-as-a-ui-runtime/))

### 每次渲染都有自己的事件处理器

查看下面的示例. 我们设置了一个 3 秒的定时器, 之后弹出 `count` 值:

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  // highlight-start
  function handleAlertClick() {
    setTimeout(() => {
      alert("You clicked on: " + count)
    }, 3000)
  }
  // highlight-end

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
      // highlight-start
      <button onClick={handleAlertClick}>Show alert</button>
      // highlight-end
    </div>
  )
}
```

我按顺序做了以下几件事情:

- 将计时器的值**增加**到 3
- **点击**"Show alert"
- 在定时器的回调被执行之前将计时器的值**增加**到 5

![counter](./counter.gif)

你认为 `alert` 的结果是什么呢? 是 `alert` 时的 state 值 5, 还是点击时的 state 值 3?

---

剧透:

---

可以在[这里](https://codesandbox.io/s/w2wxl3yo0l)自己试一试!

如果这个示例对你来说很费解, 可以想象一个更实际的例子: 一个聊天应用, 将 count 类比为当前接受的消息对应的 ID, 存储在局部状态中, 将按钮类比为发送消息的按钮. [这篇文章](https://overreacted.io/how-are-function-components-different-from-classes/)详细解释了出现上述结果的原因, 正确答案是 3.

`alert` 会"捕捉"点击当时 state 的值.

<mark>(There are ways to implement the other behavior too but I’ll be focusing on the default case for now. When building a mental model, it’s important that we distinguish the “path of least resistance” from the opt-in escape hatches.)</mark>

---

但是其中的原理是怎样的呢?

我们先前提到过, `count` 值在每次调用时都是一个固定值. 有必要重点指出的是 -- **我们的函数会被调用多次(每次渲染被调用一次), 每一次被调用时, 这个 count 值都会是由 useState 控制的一个特定值.**

这种特性并非 React 独有 -- 普通的函数也是这样工作的:

```js
function sayHi(person) {
  // highlight-next-line
  const name = person.name
  setTimeout(() => {
    alert("Hello, " + name)
  }, 3000)
}

let someone = { name: "Dan" }
sayHi(someone)

someone = { name: "Yuzhi" }
sayHi(someone)

someone = { name: "Dominic" }
sayHi(someone)
```

在这个[示例](https://codesandbox.io/s/mm6ww11lk8)中, 外部的 `someone` 变量被多次重新赋值. (就像在 React 中, 当前组件的 state 不断变化一样. ) **在 `sayHi` 内部, 有一个变量 `name`, 它会读取 `person` 的 `name` 属性.** 这个变量在函数内部, 每次调用之间都是独立的. 因此, 当定时器的回调被调用的时候, 每次 alert 都会"记住"自己的 `name`.

这也解释了为什么我们的事件处理器在点击的当下"捕捉"了 `count` 值. 同样的规则下, 每次渲染都能"看到"自己的 `count`:

```jsx
// 首次渲染
function Counter() {
  // highlight-next-line
  const count = 0 // useState() 返回的内容
  // ...
  function handleAlertClick() {
    setTimeout(() => {
      alert("You clicked on: " + count)
    }, 3000)
  }
  // ...
}

// 点击事件之后函数被重新调用
function Counter() {
  // highlight-next-line
  const count = 1 // useState() 返回的内容
  // ...
  function handleAlertClick() {
    setTimeout(() => {
      alert("You clicked on: " + count)
    }, 3000)
  }
  // ...
}

// 又一次点击事件, 函数被重新调用
function Counter() {
  // highlight-next-line
  const count = 2 // useState() 返回的内容
  // ...
  function handleAlertClick() {
    setTimeout(() => {
      alert("You clicked on: " + count)
    }, 3000)
  }
  // ...
}
```

每一次渲染, 都会返回各自版本的 `handleAlertClick`, 并且有各自版本的 `count`:

```jsx
// 首次渲染
function Counter() {
  // ...
  function handleAlertClick() {
    setTimeout(() => {
      // highlight-next-line
      alert("You clicked on: " + 0)
    }, 3000)
  }
  // ...
  // highlight-next-line
  ;<button onClick={handleAlertClick} /> // 内部值为 0 的版本
  // ...
}

// 点击事件之后函数被重新调用
function Counter() {
  // ...
  function handleAlertClick() {
    setTimeout(() => {
      // highlight-next-line
      alert("You clicked on: " + 1)
    }, 3000)
  }
  // ...
  // highlight-next-line
  ;<button onClick={handleAlertClick} /> // 内部值为 1 的版本
  // ...
}

// 又一次点击事件, 函数被重新调用
function Counter() {
  // ...
  function handleAlertClick() {
    setTimeout(() => {
      // highlight-next-line
      alert("You clicked on: " + 2)
    }, 3000)
  }
  // ...
  // highlight-next-line
  ;<button onClick={handleAlertClick} /> // 内部值为 2 的版本
  // ...
}
```

这也是为什么, 在这里的[示例](https://codesandbox.io/s/w2wxl3yo0l)中, 事件处理器"属于"各自的渲染, 当用户点击的时候, 会使用那次渲染的 `counter` 值.

**对于每一次渲染, 内部的 props 和 state 始终都是一致的, 并且每次渲染都是独立的.** 既然每次渲染的 props 和 state 各自独立, 消费它们的部分(包括事件处理器), 在渲染间也各自独立, 都从属于特定的渲染. 因此事件处理器内部的异步函数也会"看到"同样的 `count` 值.

_边注: 我在 `handleAlertClick` 中直接使用了 `count` 值. 这样的替换是安全的, 因为在一次渲染流程中, `count` 值不可能有变化. 它被声明为 `const`, 且是一个不可变的数字. 同样的原则在对象中仍然适用, 不过我们必须要确保避免改变(mutate)状态的值. 用一个全新创建的对象去调用 `setSomething(newObj)`, 而不改变这个对象就可以了. 这样的话, 前一次渲染的状态(state)值就能够保持不被下一次渲染修改._

### 每次渲染都有自己的副作用

这篇文章的主要内容本该是副作用(effect)的, 现在我们会开始详细介绍它. 其实, 副作用的和以上两部分是一样的, 每次渲染都有自己的副作用.

现在我们回到 React [文档](https://reactjs.org/docs/hooks-effect.html)中的一个例子:

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  // highlight-start
  useEffect(() => {
    document.title = `You clicked ${count} times`
  })
  // highlight-end

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  )
}
```

**我想提出一个问题: effect 是如何读取到 `count` 的最新值的?**

在 effect 函数的内部有一些数据绑定或者订阅的逻辑, 使得 effect 函数每次都能读取到 `count` 的最新值? 还是 `count` 是一个可变的值, React 在组件的内部维护了这个值, 因此我们的 effect 函数能够读取到最新值?

都不是的.

先前我们已经知道, `count` 在每次特定的组件渲染流程中, 都是一个常量. 事件处理器从它所属的渲染流程中读取到了对应的 `count` 值, 因为 `count` 是处于对应作用域的变量. 在 effect 函数中也是这样的!

**并不是在一个"不会变化"的副作用方法中, `count` 变量值时刻变化. 而是每一次渲染, 副作用函数本身都是不一样的.**

同样地, 每个版本的副作用方法读取到的都是当次渲染所传入的 `count` 值:

```jsx
// 首次渲染
function Counter() {
  // ...
  useEffect(
    // highlight-start
    // 首次渲染的副作用函数
    () => {
      document.title = `You clicked ${0} times`
    }
    // highlight-end
  )
  // ...
}

// 点击事件之后函数被重新调用
function Counter() {
  // ...
  useEffect(
    // highlight-start
    // 第二次渲染的副作用函数
    () => {
      document.title = `You clicked ${1} times`
    }
    // highlight-end
  )
  // ...
}

// 又一次点击事件, 函数被重新调用
function Counter() {
  // ...
  // highlight-start
  useEffect(
    // 第三次渲染的副作用函数
    () => {
      document.title = `You clicked ${2} times`
    }
    // highlight-end
  )
  // ..
}
```

React 会记住每一次你所提供的副作用方法, 在当次渲染结束, UI 呈现出对应变化之后调用这个副作用方法.

尽管这个副作用方法看起来是一样的(功能: 更新文档的标题), 但是实际上每一次渲染这个方法都是不一样的 -- 并且每一个副作用方法都会看到"渲染"当次的 props 和 state.

**在概念上, 你可以把副作用方法理解为每次渲染的结果.**

严格来说, 其实它们并不是渲染结果(为了能够通过简单的语法[实现 Hooks 的组合](https://overreacted.io/why-do-hooks-rely-on-call-order/), 减少运行时开销). 但是基于我们目前想要建立的心智模型, 我们可以在概念上认为副作用函数是某一次渲染的结果.

---

现在来巩固一下以上的内容, 首先回顾首次渲染:

- **React:** 当 state 值为 `0` 的时候, 给我你希望渲染的 UI.
- **组件:**
  - 这是我要渲染的内容: `<p>You clicked 0 times</p>`.
  - 渲染结束之后请执行这个副作用方法: `() => { document.title = 'You clicked 0 times' }`.
- **React:** 好的, 更新 UI. Hey, 浏览器, 我想要在 DOM 中添加一些内容.
- **浏览器:** 好的, 我把它们绘制到屏幕中.
- **React:** 好的, 我准备开始执行副作用方法了.
  - 执行 `() => { document.title = 'You clicked 0 times' }`.

---

现在回顾点击之后的渲染流程:

- **组件:** Hey, React, 把我的状态(state)值设置为 `1`.
- **React:** 当状态更新为 `1` 的时候, 把对应的 UI 返回给我吧.
- **组件:**
  - 这是渲染的结果: `<p>You clicked 1 times</p>`.
  - 记得执行这个副作用方法: `() => { document.title = 'You clicked 1 times' }`.
- **React:** 好的, 更新 UI. Hey, 浏览器, 我已经修改了 DOM 了.
- **浏览器:** 好的, 我把它们绘制到屏幕中.
- **React:** OK, 我现在开始执行副作用方法.
  - 执行 `() => { document.title = 'You clicked 1 times' }`.

---

### 每次渲染的所有值都属于当次渲染

**我们知道了, 副作用方法在每次渲染之后都会执行, 从概念上可以将副作用方法理解为组件输出内容的一部分, 并且副作用方法能够"看到"当次渲染的 props 和 state.**

现在我们来做一次实验. 查看下面的代码:

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  // highlight-start
  useEffect(() => {
    setTimeout(() => {
      console.log(`You clicked ${count} times`)
    }, 3000)
  })
  // highlight-end
  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  )
}
```

如果我在每次延迟的时间段内点击多次, 最终输出的值会是怎样的呢?

---

剧透

---

你可能会认为这是个糊弄你的题目, 最终的结果会很出人意料. 其实并不是! 我们来看这一系列的输出 -- 每一个输出都属于特定的渲染, 因此每一次输出都是自己的 `count` 值. [CodeSandbox 代码示例](https://codesandbox.io/s/lyx20m1ol).

![timeout_counter](./timeout_counter.gif)

你可能会觉得: "当然是这样输出的, 否则会怎样输出呢?"

不过, `this.state` 在类组件中并不是这样工作的. 很多人会把 `useEffect` 看作是和[类中](https://codesandbox.io/s/kkymzwjqz3)的`componentDidUpdate` 类似的方法:

```jsx
  componentDidUpdate() {
    setTimeout(() => {
      console.log(`You clicked ${this.state.count} times`);
    }, 3000);
  }
```

实际上并不是这样的, `this.state.count` 始终是最新的 `count` 值, 但是 `useEffect` 中的则是当次渲染的值, 因此以上代码的结果, 输出的会是 `5`:

![timeout_counter_class](./timeout_counter_class.gif)

我觉得有点讽刺的是, Hooks 的实现十分依赖 JavaScript 中的闭包,
I think it’s ironic that Hooks rely so much on JavaScript closures, and yet it’s the class implementation that suffers from the canonical wrong-value-in-a-timeout confusion that’s often associated with closures. This is because the actual source of the confusion in this example is the mutation (React mutates this.state in classes to point to the latest state) and not closures themselves.

**当我们需要锁定一个永远不会变化的值的时候, 使用闭包是最合适的手段. 这使得我们很容易能够推出正确答案, 因为归根结底你正在读取的值始终是一个常量.** 既然我们现在已经知道了如何维持渲染时的 props 和 state, 可以开始尝试[使用闭包](https://codesandbox.io/s/w7vjo07055)对 class 版本的代码进行改造.

### 逆流而上

现在我们已经得到了一个共识: 函数式组件在渲染时, 内部的每一项(包括事件处理器, 副作用方法, 超时时间, API 调用等)都会"捕捉"渲染当时的 props 和 state.

因此以下两个例子其实是一致的:

```jsx
function Example(props) {
  useEffect(() => {
    setTimeout(() => {
      // highlight-next-line
      console.log(props.counter)
    }, 1000)
  })
  // ...
}
```

```jsx
function Example(props) {
  // highlight-next-line
  const counter = props.counter
  useEffect(() => {
    setTimeout(() => {
      // highlight-next-line
      console.log(counter)
    }, 1000)
  })
  // ...
}
```

**从上面的代码可以看出, 不管是不是在组件中提前读取 state 或者 props 的值, 对副作用函数中读取到的结果其实都没有影响.** 在单次渲染的作用域内, props 和 state 始终会保持不变. (将 props 解构能够使得这个概念更容易理解.)

在某些场景下, 我们可能会希望能够副作用函数的回调中读取到最新的值而不是渲染当时的值. 最简单的方式是使用 `ref`, 在这篇[文章](https://overreacted.io/how-are-function-components-different-from-classes/)的最后一部分我们有提及到相关的概念.

有一点需要注意的是, 当我们希望在过去的渲染中读取到未来的 props 和 state 时, 我们实际上在逆流前进. 这当然没有错(在某些情况下甚至是必要的), 不过这样的做法看起来比较"不干净", 违反了现有的模式. 其实 React 团队是刻意把函数式组件的行为设计成这样的, 如此依赖, 用户就能够很显而易见地发现代码的缺陷. 在类式组件中, 发现这类问题就比较困难.

这里有[另一个版本计时器的例子](https://codesandbox.io/s/rm7z22qnlp), 模拟了类式组件的行为:

```jsx
function Example() {
  const [count, setCount] = useState(0);
  // highlight-next-line
  const latestCount = useRef(count);

  useEffect(() => {
  // highlight-start
    // Set the mutable latest value
    latestCount.current = count;
  // highlight-end
    setTimeout(() => {
    // highlight-start
      // Read the mutable latest value
      console.log(`You clicked ${latestCount.current} times`);
    // highlight-end
    }, 3000);
  });
  // ...
```

![timeout_counter_refs](./timeout_counter_refs.gif)

在 React 中修改(mutate)某些值或许看起来会很奇怪. 但是其实在类式组件中, React 就是这样为 `this.state` 赋值的. 与获取渲染当时的 props 和 state 不同的是, 读取 `latestCount.current` 的值, 获取到的结果无法得到任何的保证, 在某一次回调中是这样的结果, 到了下一次或许又不一样了. 因为它的值是可变的. 这也是为什么, 我们不愿意将这样的行为设置成默认行为.

### Cleanup 函数

[文档中提到](https://reactjs.org/docs/hooks-effect.html#effects-with-cleanup), 某些副作用还需要存在对应的清除副作用的阶段. 本质上来说, 在这个阶段中我们做的事情, 就是撤销副作用(比如取消订阅事件.)

查看下面的代码:

```jsx
useEffect(() => {
  ChatAPI.subscribeToFriendStatus(props.id, handleStatusChange)
  return () => {
    ChatAPI.unsubscribeFromFriendStatus(props.id, handleStatusChange)
  }
})
```

假如在首次渲染时, `props` 的值为 `{id: 10}`, 第二次渲染时, 值为 `{id: 20}`. 你或许会认为渲染流程是这样发生的:

- React 清除 props 为 `{id: 10}` 时的副作用.
- React 为 `{id: 20}` 渲染对应的 UI.
- React 为 `{id: 20}` 执行对应的副作用方法.

(但是实际情况并不是这样的.)

如果你建立以上的心智模型, 就会认为清除副作用的函数"看到了"旧的 props, 因为这个函数是在重新渲染之前执行的, 然后新的副作用函数"看到了"新的 props. 这个心智模型对于类式组件来说, 是完全正确的, 但是对于函数式组件, 情况并不是这样. 我们来看看原因是什么.

React 执行副作用方法的时机是[浏览器绘制需要渲染的内容之后](https://medium.com/@dan_abramov/this-benchmark-is-indeed-flawed-c3d6b5b6f97f). 这样能够使得我们的应用体验更佳, 不阻塞浏览器的渲染. 清除副作用的方法同时也会延迟. **前一次渲染的副作用直到使用新的 props 开始重新渲染, 才会被清除:**

- React 为 `{id: 20}` 渲染对应的 UI.
- 浏览器绘制内容, 我们看到 `{id: 20}` 对应的 UI.
- React 清除 props 为 `{id: 10}` 时的副作用.
- React 为 `{id: 20}` 执行对应的副作用方法.

你或许会觉得奇怪, 为什么前一次渲染的副作用方法读取到的是前一次渲染的 props 值, 而不是此刻的 `{id: 20}`?

其实我们之前已经提及过这个话题...... 🤔

先前章节的摘录:

> 组件渲染时内部的每一个函数(包括事件处理器, 副作用函数, timeout 回调, API 调用等), 都会"捕捉"当时的 props 和 state.

这样一来, 答案就很明显了. 清除副作用的函数没有读取最新的 props, 它始终会读取渲染当时所定义的 props 和 state:

```jsx
// 首次渲染 props 是 {id: 10}
function Example() {
  // ...
  useEffect(
    // 首次渲染的副作用函数
    () => {
      ChatAPI.subscribeToFriendStatus(10, handleStatusChange)
      // highlight-start
      // 首次渲染的清除副作用函数
      return () => {
        ChatAPI.unsubscribeFromFriendStatus(10, handleStatusChange)
      }
      // highlight-end
    }
  )
  // ...
}
// 第二次渲染 props 是 {id: 20}
function Example() {
  // ...
  useEffect(
    // 第二次渲染的副作用函数
    () => {
      ChatAPI.subscribeToFriendStatus(20, handleStatusChange)
      // 第二次渲染的清除副作用函数
      return () => {
        ChatAPI.unsubscribeFromFriendStatus(20, handleStatusChange)
      }
    }
  )
  // ...
}
```

Kingdoms will rise and turn into ashes, the Sun will shed its outer layers to be a white dwarf, and the last civilization will end. But nothing will make the props “seen” by the first render effect’s cleanup anything other than {id: 10}.

That’s what allows React to deal with effects right after painting — and make your apps faster by default. The old props are still there if our code needs them.

### 同步 忽略生命周期

我喜欢 React 的原因之一是, 它统一描述了首次渲染的结果和对应的更新. 这种模式[减少了程序的熵](https://overreacted.io/the-bug-o-notation/). <mark>?</mark>

举个例子, 如果我的组件是下面这样的:

```jsx
function Greeting({ name }) {
  return <h1 className="Greeting">Hello, {name}</h1>
}
```

不管是先渲染 `<Greeting name="Dan" />`, 然后渲染 `<Greeting name="Yuzhi" />`, 还是直接渲染 `<Greeting name="Yuzhi" />`. 最后的结果都是看到屏幕中呈现: "Hello, Yuzhi".

人们常说: 重要的是旅程而不是目的地. 但是在 React 中, 情况就不是这样了. **我们只关注结果, 而非过程.** 举个例子, 在 jQuery 中, 我们关注过程, 比如说使用 `$.addClass` 和 `$.removeClass` 来操控标签的 `className`, 而在 React 中, 我们会直接声明 `className` 应该是什么样子的(关注结果).

**React 会根据当前的 props 和 state 同步 DOM 的内容.** 在渲染过程中, 挂载(mount)和更新(update)其实没有什么区别.

我们可以以同样的角度思考副作用函数. **`useEffect` 函数使得我们能够根据 props 和 state 同步 React 树之外的内容.**

```jsx
function Greeting({ name }) {
  // highlight-start
  useEffect(() => {
    document.title = "Hello, " + name
  })
  // highlight-end
  return <h1 className="Greeting">Hello, {name}</h1>
}
```

这样的心智模型和我们所熟悉的 _挂载/更新/卸载_ 有一点区别. 了解这个模型很重要. **如果你希望实现的副作用函数, 会根据是第一次渲染, 还是重新渲染, 有不同的表现形式, 那么你就是在逆流前进!** 如果我们的结果依赖的是"过程"而非"目的地", 同步的过程就会失败.

不管我们是按照属性(A, B, C)的顺序进行渲染, 还是直接用 C 进行渲染, 其实基本上没有什么差别. 唯一的差别可能是在请求数据的过程中存在的. 但是最终的结果始终是相同的.

有一个毋庸置疑的点是: 每一次渲染都执行所有的副作用其实很影响效率. (同时在某些情况下, 会导致无限循环.)

那么我们怎么修复它呢?

### 告诉 React 副作用函数何时会产生变化

我们已经学习了 React 如何 diff DOM. React 并没有在每次渲染的时候更新所有的 DOM, 而是更新有修改的部分.

当你更新以下组件

```jsx
<h1 className="Greeting">Hello, Dan</h1>
```

为:

```jsx
<h1 className="Greeting">Hello, Yuzhi</h1>
```

的时候, React 看到的是这样两个对象:

```jsx
const oldProps = { className: "Greeting", children: "Hello, Dan" }
const newProps = { className: "Greeting", children: "Hello, Yuzhi" }
```

React 会查看每一个 props, 然后判断出 `children` 有变化, 再更新 DOM, `className` 没有变化, 因此不需要更新. 所以 React 只需要这样处理即可:

```jsx
domNode.innerText = "Hello, Yuzhi"
// No need to touch domNode.className
```

**那么针对副作用函数, 是否也能够只在必要的时候更新呢?**

举个例子, 在以下函数中, 组件会因为 state 的更新而更新:

```jsx
function Greeting({ name }) {
  const [counter, setCounter] = useState(0)

  useEffect(() => {
    document.title = "Hello, " + name
  })

  return (
    <h1 className="Greeting">
      Hello, {name}
      // highlight-start
      <button onClick={() => setCounter(count + 1)}>Increment</button>
      // highlight-end
    </h1>
  )
}
```

但是我们的副作用函数并没有用到 `counter` 的值. **副作用函数只会根据 `name` 的属性值更新 `document.title`, 但是 `name` 属性值始终是一致的.** 在每次 `counter` 值变化的时候给 `document.title` 重新赋值是完全没有必要的.

那么 React 如何分辨副作用函数的更新与否呢?

```jsx
let oldEffect = () => {
  document.title = "Hello, Dan"
}
let newEffect = () => {
  document.title = "Hello, Dan"
}
// React 能够看出这两个函数做了同样的事情吗?
```

并不能, React 在调用函数之前, 是无法分析出这个函数所做的事情的. <mark>(The source doesn’t really contain specific values, it just closes over the name prop.)</mark>

因此, 我们需要找到方法避免在不必要的情况下执行副作用函数, 我们可以传入一个依赖数组参数到 `useEffect` 方法中, 只有当依赖数组中的值变化的时候, 副作用函数才会重新执行.

```jsx
useEffect(() => {
  document.title = "Hello, " + name
  // highlight-next-line
}, [name]) // 依赖数组参数
```

**以上的场景就好像是我们在告诉 React: "Hey React, 我知道你不能看到函数内部的内容, 但是我可以确保函数内部使用的渲染相关参数只有 `name`. "**

如果前一次副作用函数执行时 `name` 的值与这一次执行时一致的话, 那么就可以跳过此次副作用函数的执行:

```jsx
const oldEffect = () => {
  document.title = "Hello, Dan"
}
const oldDeps = ["Dan"]

const newEffect = () => {
  document.title = "Hello, Dan"
}
const newDeps = ["Dan"]

// React 无法看到韩式内部的情况, 但是它可以对比这些依赖参数
// 由于依赖参数的值始终前后是一致的, 那么就不需要执行新的副作用函数
```

如果依赖数组中的某个值在渲染前后有所差异, 那么这个副作用函数就必须执行. 同步所有这些值.

### 要确保依赖数组中的参数的准确性 不要欺骗 React

欺骗 React 关于依赖数组值的内容会带来比较不好的影响. <mark>Intuitively, this makes sense, but I’ve seen pretty much everyone who tries useEffect with a mental model from classes try to cheat the rules. (And I did that too at first!)</mark>

```jsx
function SearchResults() {
  async function fetchData() {
    // ...
  }

  useEffect(() => {
    fetchData()
  }, []) //  这样的写法是合理的吗? 在某些场景下不合理, 我们可以有更好的方式来编写这部分代码

  // ...
}
```

(关于 [Hooks 的常见问题](https://reactjs.org/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies)中, 已经详细说明了针对这个问题的正确做法. 我们回到现在例子:)

"但是我只希望在组件挂载的时候执行这个方法!" 你或许会这样说. 我们要记住这样一个原则: 如果你声明了依赖参数, **所有组件内部的值, 只要被副作用函数使用, 就一定要声明在依赖参数中.** 包括 props, state, 函数等任何在组件内部使用的值.

但是有时候我们可能会遇到一些问题. 比如, 你可能会遇到无限循环请求的情况, 或者是某个 socket 不断被重新创建. **针对这些问题的处理方式并不是删除这个依赖.** 后面我们会谈到如何解决这些问题.

在我们查看解决方案之前, 先更深入地了解一下我们的问题.

### 当你对 React 撒谎 传入了错误的依赖参数会发生什么

如果依赖数组中包含了所有副作用函数会用到的值, React 就能够知道何时需要重新执行副作用函数:

```jsx
useEffect(() => {
  document.title = "Hello, " + name
  // highlight-next-line
}, [name])
```

![](./deps-compare-correct.gif)

(依赖数组中的值在渲染前后有所差异, 因此我们需要重新执行副作用函数.)

如果我们在依赖数组中传入空数组`[]`, 新的副作用函数就不会被执行:

```jsx
useEffect(() => {
  document.title = "Hello, " + name
  // highlight-next-line
}, []) // 缺少了 name 参数
```

![](./deps-compare-wrong.gif)

(依赖数组前后一致, 因此跳过这次副作用函数的执行)

通过以上的场景对比, 问题就很明显地体现出来了. <mark>But the intuition can fool you in other cases where a class solution “jumps out” from your memory.</mark>

举个例子, 比如我们想要实现一个计数器, 每隔一秒数字加 1. 如果是类式组件的话, 我们会这样实现它: 在组件挂载的时候设置计时器函数, 组件卸载的时候清楚.

这是具体的[代码示例](https://codesandbox.io/s/n5mjzjy9kl). 当我们想要把类式组件转换成函数式组件的时候, 会习惯性地使用 `useEffect`, 设置依赖参数为 `[]`, 表示希望副作用函数只执行一次.

```jsx
function Counter() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1)
    }, 1000)
    return () => clearInterval(id)
    // highlight-next-line
  }, [])

  return <h1>{count}</h1>
}
```

但是真正执行了会发现, 结果并不是我们所预期的那样. [示例代码](https://codesandbox.io/s/91n5z8jo7r).

如果你的心智模型是这样的: "依赖的作用是让我声明什么时候需要重新触发副作用函数的执行", 那么你写出来的代码就会很危险, 就像上面的例子一样. 但是问题是, 你希望只触发一次副作用方法, 因为这是一个间隔执行的 API, 实际上并没有错, 但是为什么会带来问题呢?

之前已经提到过, 副作用函数中使用到的*所有*值, 我们都要在依赖参数数组中声明. 由于内部使用到了 `count`, 但是我们并没有在依赖参数数组中声明, 因此会引起 bug.

在首次渲染的时候, `count` 的值是 `0`. `setCount(count + 1)` 在首次渲染时实际执行的是 `setCount(0 + 1)`. **但是因为我们声明的依赖数组参数中数组值为 `[]`, 因此副作用函数不会变化, 每隔一秒执行的函数都是 `setCount(0 + 1)`**:

```jsx
// 首次渲染, state 是 0
function Counter() {
  // ...
  useEffect(
    // 首次渲染的副作用函数
    () => {
      const id = setInterval(() => {
        // highlight-next-line
        setCount(0 + 1) // 始终是 setCount(1)
      }, 1000)
      return () => clearInterval(id)
    },
    // highlight-next-line
    [] // 始终不会重新执行
  )
  // ...
}

// 下一次渲染, state 是 1
function Counter() {
  // ...
  useEffect(
    // highlight-next-line
    // 副作用函数被忽略了, 因为我们的依赖参数传递错误.
    () => {
      const id = setInterval(() => {
        setCount(1 + 1)
      }, 1000)
      return () => clearInterval(id)
    },
    []
  )
  // ...
}
```

由于在依赖参数中传递了一个空数组, 表明了我们的副作用函数不依赖任何值, 但是实际上副作用函数中存在依赖其他值的部分. 

我们的副作用函数使用到了 `count`, 这个值声明于组件之内, 副作用函数之外:

```jsx
// highlight-next-line
 const count = // ...

  useEffect(() => {
    const id = setInterval(() => {
  // highlight-next-line
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
```

因此, 将依赖参数设置为 `[]` 会引起 bug. React 会比较依赖项数组中的内容, 然后跳过副作用函数的更新:

![](./interval-wrong.gif)

_(依赖项中的内容始终没有区别, 因此跳过副作用函数的更新)_

这样的问题比较难定位. 因此, 我建议大家在传递依赖项参数的时候对 React 保持诚实, 声明所有依赖参数. (我们提供了一个 [lint 规则插件](https://github.com/facebook/react/issues/14920) 以供用户在开发阶段使用.)

### Two Ways to Be Honest About Dependencies

1. 在依赖项数组中声明副作用函数中的所有依赖项
2. 用函数的方式更新状态
### Functional Updates and Google Docs

