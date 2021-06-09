---
title: React 作为 UI 运行时
date: "2021-05-28"
template: "post"
draft: true
toc: true
category: "React"
tags:
  - "React"
description: "React as a UI Runtime"
---

原文: [React as a UI Runtime](https://overreacted.io/react-as-a-ui-runtime/)

---

几乎大部分文章都将 React 介绍为一个 UI 库. 这当然是合理的, 官网上就是这样介绍它的.

![](./react.png)

对于 UI 设计的挑战, 我之前写过一篇[文章](https://overreacted.io/the-elements-of-ui-engineering/) . 这篇文章和之前的文章有点区别, 从一个完全不同的角度来看待 React, 本文认为 React 是一个[编程运行时](https://en.wikipedia.org/wiki/Runtime_system).

**这篇文章不会教你任何创建 UI 有关的知识.** 不过阅读了本文之后, 能够帮助你更深刻地理解 React 的编程模型.

---

如果你目前正在学习 React, 那么你不是本文的目标读者, 可以先去查看[官方文档](https://reactjs.org/docs/getting-started.html#learn-react).

**本文是深入探讨 React 的文章, 因此初学者不适合阅读这篇文章.** 在这篇文章中, 我会从最基本原理(first principles)的角度介绍一些 React 的编程模型. 我不会教你如何使用, 只会说明具体的原理.

**许多有多年 React 开发经验的开发者, 可能也没有对这些话题有深入的思考.** 本文针对 React 的探讨角度, 更偏向于编程角度而非[设计师的角度](https://mrmrs.cc/writing/developing-ui/) . 当然两者都了解是再好不过的.

接下来开始我们的正文.

---

## 宿主树

有些程序输出数字, 有些程序输出诗篇. 不同的语言以及它们对应的运行时经常会针对某些特定场景做出一些适合该场景的优化. React 也不例外.

React 程序**输出"树"结构, 这棵树会实时变化.** 它可以是[DOM 树](https://www.npmjs.com/package/react-dom), [IOS 中的结构树](https://developer.apple.com/library/archive/documentation/General/Conceptual/Devpedia-CocoaApp/View%20Hierarchy.html), [PDF 原始类型的树](https://react-pdf.org/), 甚至是 [JSON 对象](https://reactjs.org/docs/test-renderer.html) . 通常我们会依赖这些树结构来呈现一些 UI. 后面我们会把他们叫做宿主树. 因为它们属于 React 之外的宿主环境的一部分, 像 DOM 和 iOS 都是宿主环境. 宿主树有属于[他们](https://developer.mozilla.org/en-US/docs/Web/API/Node/appendChild)[自己](https://developer.apple.com/documentation/uikit/uiview/1622616-addsubview)的关键 API. React 属于另一层的实现.

那么 React 是做什么的呢? 抽象地来说, 它会帮助你构造并控制你的程序所需要的宿主树, 针对外部的事件, 比如用户交互, 网络响应, 计时器等, 做出对应的处理.

在某些场景下, 一个具有特定功能的工具会比功能广泛的工具更好用, 因为可以从一些特定功能中收获一些益处. React 的其中两个实现理念, 就在这方面做出了一些有挑战性的尝试:

**稳定性.** 宿主树会始终保持相对稳定. 所有的更新都不会引起整体结构的根本性变化. 如果一个程序每隔一秒就重新组织它可交互的部分, 那么这个程序就会很难用. 你可能会遇到按钮突然消失, 屏幕抖动等各种影响体验的问题.

**规律性.** 宿主树能够被拆解成对应的 UI 模式, 并且他们是一一对应且行为一致的. The host tree can be broken down into UI patterns that look and behave consistently (such as buttons, lists, avatars) rather than random shapes.

**以上的这两个原则, 正好对于大部分 UI 来说, 也是适用的.** 但是当程序的输出并非一个稳定"模式"时, React 就不适合用于构建这样的程序. 比如, React 能够构建一个推特客户端, 但是构建 3D 管道的屏幕保护程序, 就不太合适了.

### 宿主实例

宿主树由各个节点组成. 我们把这些节点叫做"宿主实例".

在 DOM 作为宿主树的场景下, 宿主示例就是普通的 DOM 的节点 — j 节点就是当你调用 `document.createElement('div')` 所产生的那部分东西. 在 iOS 中, host instances could be values uniquely identifying a native view from JavaScript.

宿主实例有他们各自的属性(比如 `domNode.className` 或 `view.tintColor` ). 实例本身也可能包含其他实例(子节点).

(这些部分跟 React 完全没有关系, 都是宿主环境中的概念)

通常来说, 宿主环境都会提供一些 API 用以操作宿主示例, 这些 API 的包括: `appendChild`, `removeChild`, `setAttribute` 等. 在 React 应用中, 你通常不需要手动调用这些 API. 这些工作是由 React 完成的.

### 渲染器

渲染器会告诉 React 去和特定的宿主环境进行交互, 同时它还负责了操控宿主示例的任务. React DOM, React Native 甚至 [Ink](https://mobile.twitter.com/vadimdemedes/status/1089344289102942211), 都属于 React 渲染器. 你还可以[创建自己的 React 渲染器](https://github.com/facebook/react/tree/master/packages/react-reconciler).

React 渲染器能够在两种不同的模式下工作.

大部分渲染器使用的是可变(mutating)的模式. 这也是 DOM 采取的模式: 我们能够创建节点, 设置节点的属性, 从节点中添加或者删除子节点. 在这种模式下宿主实例是可变的.

React 同样能够在另一种[持久(persistent)](https://en.wikipedia.org/wiki/Persistent_data_structure)模式下工作. 在这个模式下, 宿主实例没有提供任何类似 `appendChild()` 之类的 API, 对于每次修改, 都会克隆整体的父节点树, 然后替换最顶层的子节点. 由于宿主树的不可变特性使得多线程的实现变得更容易. [React Fabric](https://reactnative.dev/blog/2018/06/14/state-of-react-native-2018)就利用了这一特性.

作为一个 React 用户, 正常来说不需要去思考这些模式. 我只是想要突出这一点: React 并不仅仅是一个简单的适配器而已. Its usefulness is orthogonal to the target low-level view API paradigm.

### React 元素

在宿主环境下, 一个宿主实例(比如 DOM 元素)是最小的构建单元. 在 React 中, 最小的构建单元则是 _React element_ (React 元素).

一个 React 元素仅仅是一个普通的 JavaScript 对象. 它能够 _描述_ 一个宿主实例.

```js
// JSX 是以下对象的语法糖
// <button className="blue" />
{
  type: 'button',
  props: { className: 'blue' }
}
```

React 元素十分轻量, 并且它本身和宿主实例没有关系. 再重申一遍, 它仅仅只是你想要在屏幕中所看到内容的一个表达形式.

和宿主实例一样, React 元素是构建树的最小单元.

```jsx
// JSX 是以下对象的语法糖
// <dialog>
//   <button className="blue" />
//   <button className="red" />
// </dialog>
{
  type: 'dialog',
  props: {
    children: [{
      type: 'button',
      props: { className: 'blue' }
    }, {
      type: 'button',
      props: { className: 'red' }
    }]
  }
}
```

(注意: 我省略了一些对于教学目的没有帮助的[属性](https://overreacted.io/why-do-react-elements-have-typeof-property/))

不过, 我们要记住一点, **React 元素没有它们自己能够持久化的签名.** 这意味着, 它们会经历不断地被移除和重新创建的过程.

React 元素是不可变的. 举个例子, 我们不能够修改元素的子节点或者是元素的属性. 如果在下一次渲染中你希望展示不同的内容. 就需要创建一个全新的 React 元素树来表达你想要的内容.

我倾向于将 React 元素看作是电影中的每一帧. 它们记录了在某个时间点, UI 应该以怎样的形式呈现. 每一帧的内容本身, 始终是不会变的.

### 入口 Entry Point

每个 React 渲染器都存在一个 "入口", 这个入口是一个 API, 我们利用这个 API 来告诉 React 应该在宿主实例的容器中渲染什么样的内容.

举个例子, React DOM 的入口就是 `ReactDOM.render`:

```jsx
ReactDOM.render(
  // { type: 'button', props: { className: 'blue' } }
  <button className="blue" />,
  document.getElementById("container")
)
```

`ReactDOM.render(reactElement, domContainer)` 这段代码表达的意思是: "React, 在宿主树 `domContainer` 中渲染我的 `reactElement.`

在此之后, React 就会去确认 `reactElement.type` (需要渲染的元素的类型) 的值是什么(在我们的例子中, 这个值是 `button`). 然后 React 再告诉 React DOM 渲染器去创建一个宿主实例并设置对应的属性.

```js
// ReactDOM renderer 渲染器做的事情(简化版本)
function createHostInstance(reactElement) {
  let domNode = document.createElement(reactElement.type)
  domNode.className = reactElement.props.className
  return domNode
}
```

在我们的例子中, React 做的是这些工作:

```js
let domNode = document.createElement("button")
domNode.className = "blue"

domContainer.appendChild(domNode)
```

如果 React 元素存在子元素(`reactElement.props.children`)的话, 在首次渲染时, 就会递归地创建宿主实例.

## 协调 Reconciliation

如果我们在同样的 container 中重复调用 `ReactDOM.render()`, 会发生什么呢?

```jsx
ReactDOM.render(
  <button className="blue" />,
  document.getElementById("container")
)

// 是会完全替换宿主实例, 还是仅仅修改一个属性呢?
ReactDOM.render(
  <button className="red" />,
  document.getElementById("container")
)
```

这里想要再强调一次, React 的工作是使得宿主树的内容匹配 React 元素树的内容. 在接受到新的内容之后, 处理宿主实例树的过程叫做[协调(reconciliation)](https://reactjs.org/docs/reconciliation.html).

针对我们以上的代码示例, React 可以有两种方式进行处理, 简单的方式就是移除当前存在的整棵树, 然后重新创建一棵用户需要的树:

```js
let domContainer = document.getElementById("container")
// 清除整棵树
domContainer.innerHTML = ""
// 重新创建一个宿主实例树
let domNode = document.createElement("button")
domNode.className = "red"
domContainer.appendChild(domNode)
```

但是在 DOM 中, 这样的处理是很耗费性能的, 与此同时还会丢失一些重要信息, 比如 focus, selection, 滚动的状态等. 因此, React 选择了以下这种做法:

```js
let domNode = domContainer.firstChild
// 更新已存在的宿主实例
domNode.className = "red"
```

也就是说, React 承担了这样的任务, 决定何时应该更新已有宿主实例, 何时应该创建一个新的宿主实例.

这就引发了一个问题: 如何区分函数实例. React 元素或许每一次都是不同的, 但是函数实例可能是会被复用的.

在我们的例子中, 很简单. 我们首先渲染了一个 `<button/>` 作为元素的子节点. 第二次依然是一个 `<button/>`, 唯一的区别是属性变了, 因此我们完全没有必要重新创建一个宿主实例. 直接复用就可以.

这样的想法, 很接近 React 的处理方式.

**如果在同样的位置下, 两次渲染时, 元素的类型是一致的, 那么就会复用已存在的宿主实例.**

以下的代码示例呈现了 React 具体的操作:

```jsx
// let domNode = document.createElement('button');
// domNode.className = 'blue';
// domContainer.appendChild(domNode);
ReactDOM.render(
  <button className="blue" />,
  document.getElementById("container")
)

// 确认是否可以复用宿主实例? 可以 (因为前后两次都是 button )
// 于是复用实例, 直接修改属性 domNode.className = 'red';
ReactDOM.render(
  <button className="red" />,
  document.getElementById("container")
)

// 确认是否可以复用宿主实例? 不可以 (因为前后两次不一致 button → p)
// 于是删除原有的实例, 重新创建一个新的实例
// domContainer.removeChild(domNode);
// domNode = document.createElement('p');
// domNode.textContent = 'Hello';
// domContainer.appendChild(domNode);
ReactDOM.render(<p>Hello</p>, document.getElementById("container"))

// 确认是否可以复用宿主实例? 可以 (因为前后两次都是 p )
// domNode.textContent = 'Goodbye';
ReactDOM.render(<p>Goodbye</p>, document.getElementById("container"))
```

子节点对应的树, 也经历了以上类似的过程. 举个例子, 当我们更新一个 `<dialog>` 组件时, 会在其中添加两个 `<button>`. React 先决定是否要复用 `<dialog>`, 再决定是否要复用子节点所对应的实例.

## 不同情况 Conditions

如果 React 只复用类型一致的实例, 那么针对动态渲染的内容, 是如何处理的呢?

举个例子, 我们现在希望先展示一个 input 框, 之后再在 input 框之前展示一些信息:

```jsx
// 首次渲染
ReactDOM.render(
  <dialog>
    <input />
  </dialog>,
  domContainer
)

// 后续的渲染
ReactDOM.render(
  <dialog>
    // highlight-next-line
    <p>I was just added here!</p>
    <input />
  </dialog>,
  domContainer
)
```

在上面的例子中, `<input>` 宿主实例可能会被重新创建, 因为等 React 遍历完整棵树之后, 与前一个版本进行对比会发现:

- `dialog → dialog`: 是否可以复用原有的宿主实例? **可以 -- 两者的类型是一致的.**

  - `input → p`: 是否可以复用原有的宿主实例? **不可以, 因为类型已经改变了!** 需要把当前的 `input` 元素删除然后创建一个新的 `p` 宿主实例.
  - `(nothing) → input`: 需要创建一个新的类型为: input 的宿主实例.

以上更新的过程, 由代码呈现就是这样的:

```jsx
// highlight-start
let oldInputNode = dialogNode.firstChild
dialogNode.removeChild(oldInputNode)
// highlight-end

let pNode = document.createElement("p")
pNode.textContent = "I was just added here!"
dialogNode.appendChild(pNode)

// highlight-start
let newInputNode = document.createElement("input")
dialogNode.appendChild(newInputNode)
// highlight-end
```

这并不是最佳的更新策略, 因为 `<input>` 并不是被 `<p>` 替换, 而仅仅是移了个位置. 我们不能够因此就将它整个删除.

想要修复这个问题其实很简单(我们很快会讲到), 这种情况在 React 应用中其实出现得并不多. 探究一下其中的原因, 其实会觉得很有意思.

在实际应用中, 我们几乎很少会直接调用 `ReactDOM.render` 来渲染页面, 我们会将程序进行抽象, 拆分成多个组件:

```jsx
function Form({ showMessage }) {
  let message = null
  if (showMessage) {
    message = <p>I was just added here!</p>
  }
  return (
    <dialog>
      {message}
      <input />
    </dialog>
  )
}
```

以上的例子并不会遇到我们先前所描述的那个问题. 我们用对象的形式表示组件, 而不是 JSX 的形式. 我们来看一看以上形式的 `dialog` 子组件树:

```jsx
function Form({ showMessage }) {
  let message = null
  if (showMessage) {
    message = {
      type: "p",
      props: { children: "I was just added here!" },
    }
  }
  return {
    type: "dialog",
    props: {
      // highlight-start
      children: [message, { type: "input", props: {} }],
      // highlight-end
    },
  }
}
```

**不管 `showMessage` 的值是 `true` 还是 `false`, `<input>` 组件始终是第二个子节点. 这样一来, `input` 在树中的位置始终不会变化.**

当 `showMessage` 从 `false` 变为 `true` 的时候, React 会遍历整棵元素树, 对比前一个版本, 这次 React 是这样做的:

- `dialog → dialog`: 是否可以复用原有的宿主实例? **可以 -- 两者的类型是一致的.**

  - `null → p`: 需要插入一个新的宿主实例, 类型为 `p`.
  - `input → input`: 是否可以复用原有的宿主实例? **可以 -- 两者的类型是一致的.**

以上由代码呈现就是这样:

```jsx
let inputNode = dialogNode.firstChild
let pNode = document.createElement("p")
pNode.textContent = "I was just added here!"
dialogNode.insertBefore(pNode, inputNode)
```

现在, input 的相关状态就不会丢失了~

## 列表

对比同一位置的元素类型, 在大部分情况下已经能够区分是否要复用或者重新创建对应的宿主实例了.

可是只有当子节点位置是固定不变且不会被重新排序的时候, 我们上面的规则才会奏效. 在上面的例子中, 即使 `message` 的值可能不存在, 我们依然知道 input 是在 message 之后的, 并且 `Dialog` 下也不存在任何其他子节点.

而对于动态的列表, 情况就不一样了, 我们无法确保顺序是始终一致的.

```jsx
function ShoppingList({ list }) {
  return (
    <form>
      {list.map(item => (
        <p>
          You bought {item.name}
          <br />
          Enter how many do you want: <input />
        </p>
      ))}
    </form>
  )
}
```

如果我们需要购买的东西的 `list` 被重新排序过, React 会认为所有的 `p` 和 `input` 元素都属于同样的类型, 因此不会移动它们. (在 React 看来, 是列表中的所有内容变化了, 而不是它们的顺序变化了.)

执行重新排序的操作由代码呈现, 将会是下面这样:

```js
for (let i = 0; i < 10; i++) {
  let pNode = formNode.childNodes[i]
  let textNode = pNode.firstChild
  textNode.textContent = "You bought " + items[i].name
}
```

React 并没有将它们进行重新排序, 而是更新了每一个节点的内容.
这无疑会引起很大的性能问题和 bug . 举例来说, 排序之后第一次输入的内容的状态始终会被保留 -- 可是实际上, 它所关联的产品应该已经和上一次完全不一样了!

**这也是为什么 React 强制用户针对每一个列表声明一个特殊的属性 `key` 的原因:**

```jsx
function ShoppingList({ list }) {
  return (
    <form>
      {list.map(item => (
        // highlight-next-line
        <p key={item.productId}>
          You bought {item.name}
          <br />
          Enter how many do you want: <input />
        </p>
      ))}
    </form>
  )
}
```

`key` 属性是一个节点的唯一标识, 使得 React 能够在每一次渲染中辨认出这个节点.

当 React 看到 `<form>` 中的 `<p key="42">` 的时候, 会检查前一次渲染的同一个 `<form>` 中是否同样包含 `<p key="42">`. 即使 `<form>` 的子节点被重新排序, React 依然能够通过 `key` 找到先前的宿主实例.

有一点要注意的是, 只有当 React 元素存在子节点的时候, `key` 这个属性才是有用的

并且, key 这个属性的只需要在同一个父元素中是唯一的就可以了, React 不会混乱不同父节点中的相同 key 属性节点. (对于不同父元素中的不同子节点交换顺序的情况, React 所采取的机制是删除并重新创建.)

那么对于 `key`, 它最佳的取值是什么呢? 一个比较简单的答案是: **当子元素的顺序变化的时候, 它的 key 始终保持一致.** 举个例子, 在我们的购物列表中, 产品 ID 就适合作为 key.

## 组件

我们之前已经了解了函数式组件: 它是返回 React 元素的函数.

```jsx
function Form({ showMessage }) {
  let message = null
  if (showMessage) {
    message = <p>I was just added here!</p>
  }
  return (
    <dialog>
      {message}
      <input />
    </dialog>
  )
}
```

我们把这类函数叫做组件. 我们使用组件来创建 UI 的各个组成部分, 组件对于 React 来说, 就是面包和黄油.

组件接受一个参数 -- 对象哈希值. 这个对象包含了组件所需要的各个属性(props). 在我们上面的例子中, `showMessage` 就是一个属性. 属性和具名参数很相似.

## Purity

React 组件是一个纯函数, 我们不能够在组件内部修改所传入的属性.

```jsx
function Button(props) {
  // 🔴 不会生效
  props.isActive = true
}
```

通常情况下, React 不支持可变. (后面我们会提到, 对于不同的事件, 应该做出怎样的合适的响应.)

不过, 在组件内部实现可变, 是可以接受的:

```jsx
function FriendList({ friends }) {
  // highlight-next-line
  let items = []
  for (let i = 0; i < friends.length; i++) {
    let friend = friends[i]
    // highlight-next-line
    items.push(<Friend key={friend.id} friend={friend} />)
  }
  return <section>{items}</section>
}
```

在 `FriendList` 中, 我们通过循环遍历在 `items` 中插入新的 `Friend` 组件, 这种操作完全不影响其他组件. 在组件渲染之前, 可变操作是合理的, 没有必要刻意避免这种操作.

同样的, 延迟初始化的操作也是能够被接受的, 尽管这种操作本质上并不"纯".

```jsx
function ExpenseForm() {
  // 只要不影响其他组件, 这种操作就能够被接受
  SuperCalculator.initializeIfNotReady()

  // 继续渲染
}
```

有时候, 我们可能会多次调用同一个组件, 但是只要不影响其他组件的渲染, 这种操作就是安全的. React 不会强制要求组件 100% 纯粹(pure, 函数式变成中的术语). 对于 React 来说, [幂等性](https://stackoverflow.com/questions/1077412/what-is-an-idempotent-operation)比纯粹性更加重要.

也就是说, React 组件中不允许出现用户可见的副作用, 更实际一点的表述就是: 仅仅调用某个组件本身, 不能够引发屏幕上可感知的渲染结果改变.

## 递归

如果我们想要复用来自其他组件的组件, 应该怎么做呢? 组件的本质是函数, 因此我们可以直接调用它们:

```jsx
let reactElement = Form({ showMessage: true })
ReactDOM.render(reactElement, domContainer)
```

然而这种方式并不符合 React 的设计初衷, 我们不该在运行时创建组件.

更合理的方式是, 用 React 元素的形式使用组件. **这意味着, 我们不能直接调用组件所对应的函数, 而是让 React 来负责调用的工作, 我们只需要声明即可:**

```jsx
// { type: Form, props: { showMessage: true } }
let reactElement = <Form showMessage={true} />
ReactDOM.render(reactElement, domContainer)
```

然后 React 就会在内部调用这个函数组件:

```jsx
// React 内部
let type = reactElement.type // Form
let props = reactElement.props // { showMessage: true }
let result = type(props) //  Form 函数 return 出的结果
```

按照惯例, 我们应该把函数式组件声明为首字母大写. 这样的话, JSX 就不会把自定义组件 `Form` 认错为原生的 HTML 标签 `<form>`. 这样一来, 函数所对应的对象的 `type` 属性就可以是一个签名而非单纯的字符串:

```jsx
console.log((<form />).type) // 'form' string
console.log((<Form />).type) // Form function
```

我们没有全局注册检查的机制, 当看到 `<Form/>` 组件的时候, 只是单纯地寻找对应的 `Form`. 如果没法找到 `Form` 的话, 就会抛出一个 JS 错误, 这类错误, 和生命了一个错误的变量名是同类的错误.

**那么当 React 发现元素的类型是函数的时候, React 会做些什么呢? 它会调用你的函数组件, 从而得出需要渲染哪些组件的信息.**

这个操作在 React 内部递归地进行, 文档中有对相关概念详细的[解释](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html). 用几句话概括, 就是这样的:

- **开发者**: `ReactDOM.render(<App />, domContainer)`
- **React**: Hi, `<App>`, 你要渲染的是什么内容?
  - `App`: 我想渲染一个 `<Layout>` 组件, 它的子组件是 `<Content>`
- **React**: Hi, `<Layout>`, 你要渲染什么内容?
  - `Layout`: 我要把我的子组件都渲染到 `<div>` 中. 我的子组件是 `<Content>`.
- **React**: Hi, `<Content>`, 你要渲染什么内容?
  - `Content`: 我要渲染一些文本到 `<article>` 标签中, 然后再加一个 `<Footer>` 组件.
- **React**: Hi, `<Footer>`, 你要渲染什么内容?
  - `Footer`: 我要渲染一些文本到 `<footer>` 标签中.
- **React**: 好的.

```jsx
// React 生成的 DOM 结构
<div>
  <article>
    Some text
    <footer>some more text</footer>
  </article>
</div>
```

看到上面的例子, 就能很容易理解为什么协调 (reconciliation) 是递归的操作了. 当 React 遍历元素树的时候, 遇到 `type` 为组件的时候, 就调用它, 然后不断深入组件树直到最后一层为止.

同样的协调规则在这种情况下同样适用. 如果同一个位置的 `type` 改变了(这里的变化是通过索引值和一个可选的 `key` 共同判断出来的), React 就会抛弃原有的宿主实例然后重新创建一个.

## 控制反转 Inversion of Control

你可能会疑惑: 为什么不允许我们直接调用组件而要让 React 来做这个事情呢? 为什么要声明式得编写 `<Form/>` 而不是直接调用它呢 `Form()` ?

**这是因为 React 如果知道你的组件树是怎样的, 就能够将这件事情完成地比我们更好.**

```jsx
// 🔴 因为是你负责调用这些组件, React 完全不知道 Layout 和 Article 的存在.
ReactDOM.render(Layout({ children: Article() }), domContainer)

// ✅ React 知道 Layout 和 Article 的存在. 它负责调用它们
ReactDOM.render(
  <Layout>
    <Article />
  </Layout>,
  domContainer
)
```

这是一个典型的[控制反转](https://en.wikipedia.org/wiki/Inversion_of_control)的例子. 让 React 来执行调用组件的操作, 我们还能发现一些有意思的东西:

- **Component became more than functions.** React 能够扩展组件的功能, 例如给予组件存储内部状态的能力, <mark>React can augment component functions with features like local state that are tied to the component identity in the tree. Marked text</mark>一个良好的运行时能够针对所可能遇到的问题提供更多底层的抽象. 我们之前提到过, React 更适合用于实现 UI 渲染和用户交互的程序. 如果我们自己负责调用组件, 就比如自己来实现这些能力.

- **协调的过程利用组件的类型做出了一些判断.** 让 React 来调用我们的组件, 我们能够更加直观地看出组件树本身的结构. 举个例子, 当你从 `<Feed>` 页面转到 `<Profile>` 页面的时候. React 不会尝试对其中的宿主实例进行复用 -- 其实和普通的标签渲染的过程没有区别. 组件内部所有的状态都会消失 -- 当两个组件的内容差异很大时, 这样的处理方式是十分合适的. 当我们我们从 `<PasswordForm>` 组件转到 `<MessengerChat>` 组件时, 一定不会希望其中状态依然保留, 即使 input 框在页面中的位置刚好一致.

- **React 会延迟协调的过程.** 如果 React 控制了调用组件的逻辑, 它就能够做出血多有意思的事情. 比如说, 它能够在两个组件被调用的间隙让浏览器做一些事情, 这样的话, 一个比较大的组件的重新渲染, 也[不会阻塞主线程中的任务](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html). 用户如果希望手动实现这种优化, 会耗费很大的力气, 同时 React 本身也需要修改一大部分的内容.

- **更好的调试体验.** React 能够直接与作为一等公民的组件进行交互, 因此我们可以实现功能更加丰富的[调试工具](https://github.com/facebook/react-devtools).

React 负责调用组件带来的最后一个好处就是 _延迟计算_. 我们来看看这意味着什么.

## 延迟计算

当我们在 JavaScript 中调用函数的时候, 参数会在调用之前先被计算出来:

```jsx
// (2) 后计算
eat(
  // (1) 先计算
  prepareMeal()
)
```

JavaScript 的开发者当然能够预期到这一点, 因为在 JavaScript 中, 函数会带来一些隐式的副作用. 想象这种情况, 如果我们调用了一个函数, 但是只有当它被使用的时候, 这部分代码才被真正执行, 大部分 JavaScript 开发者肯定都会觉得这个现象很奇怪.

React 组件[相对](https://overreacted.io/react-as-a-ui-runtime/#purity)来说是比较纯的函数. 并且很显而易见的一点是, 只有当需要在屏幕上渲染出内容的时候, 我们才需要调用这个函数.

来看下面这一个示例, 这个组件将会渲染 `<Page>` 组件, `Page` 的子组件是 `<Comments>`:

```jsx
function Story({ currentUser }) {
  // return {
  //   type: Page,
  //   props: {
  //     user: currentUser,
  //     children: { type: Comments, props: {} }
  //   }
  // }
  return (
    <Page user={currentUser}>
      // highlight-next-line
      <Comments />
    </Page>
  )
}
```

`Page` 组件中的内容就是将传入其中的 `children` 渲染到 `Layout` 中:

```jsx
function Page({ user, children }) {
  return (
    <Layout>
      // highlight-next-line
      {children}
    </Layout>
  )
}
```

(在 JSX 中, `<A><B /></A>` 与 `<A children={<B />}/>` 实际上是一样的.)

但是如果有一些动态渲染其他内容的逻辑, React 会怎么处理呢?

```jsx
function Page({ user, children }) {
  // highlight-start
  if (!user.isLoggedIn) {
    return <h1>Please log in</h1>
  }
  // highlight-end
  return <Layout>{children}</Layout>
}
```

如果我们将 `Comments()` 作为函数来调用的话, 不管是否被渲染, 只要一被调用, 就会立即执行:

```jsx
// {
//   type: Page,
// highlight-next-line
//   props: {
//     children: Comments() // Always runs!
//   }
// }
<Page>
  // highlight-next-line
  {Comments()}
</Page>
```

如果我们传入的是 React 组件, 就不会直接先执行 `Comment`:

```jsx
// {
//   type: Page,
//   props: {
// highlight-next-line
//     children: { type: Comments }
//   }
// }
<Page>
  // highlight-next-line
  <Comments />
</Page>
```

声明组件的方式, 使得 React 能够决定何时去调用组件. 如果我们的 `Page` 组件忽略了它的 `children` 属性, 然后渲染了 `<h1>Please log in</h1>`, React 就不会去调用 `Comment` 函数, 因为已经完全没有必要了.

这样的方式带来了比较大的性能提升, 因为不仅避免了多余的渲染, 还使得代码更加的健壮. (当用户没有登录的情况下, 我们完全不需要理会 `Comments` 组件发生了什么, 因为它不可能被调用.)

## 状态

[先前](https://overreacted.io/react-as-a-ui-runtime/#reconciliation)我们提到了函数签名的问题, 还提到元素在树中的位置是如何影响 React 决定复用还是重新创建新的宿主实例的. 宿主实例可能会有多种状态: 聚焦(focus), 选择(selection), 输入(input)等. 我们希望, 更新前后在渲染同样的 UI 的情况下, 这个状态依然能够保留. 我们还希望, 在更新前后 UI 产生巨大变化的情况的下, 能够提前预测并销毁目标实例. (比如从 `<SignupForm>` 更新成 `<MessengerChat>` 的时候).

**内部的状态十分重要, 因此 React 给了组件存储内部状态的能力.** 组件依然是函数, 不过 React 扩展了它们的功能, 这些功能对于 UI 的渲染来说, 是很有必要的. 存储状态就是这些有用的功能之一.

我们把这些扩充的能力, 叫做 _Hooks_ . `useState` 就是一个 Hook.

```jsx
function Example() {
  // highlight-next-line
  const [count, setCount] = useState(0)

  return (
    <div>
      // highlight-start
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        // highlight-end Click me
      </button>
    </div>
  )
}
```

`useState` 返回了一对值: 当前的 state 和更新这个 state 的函数.

[数组解构](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#array_destructuring)的语法使得我们能够给 state 变量任意命名. 在上面的例子中, 我就将它们命名为 `count` 和 `setCount`, 当然也可以是其他名字, 比如`banana` 和 `setBanana`. 接下来, 无论第二个参数(更新函数)是什么名字, 我都会使用 `setState` 来指代它.

(你可以在[这里](https://reactjs.org/docs/hooks-intro.html)查看 `useState` 和其他 Hooks 的文档.)

## 一致性

即使我们希望将协调的过程拆分成小块, 以免[阻塞](https://www.youtube.com/watch?v=mDdgfyRB5kg)浏览器的正常工作, <mark>we should still perform the actual host tree operations in a single synchronous swoop.</mark> 这样的话, 我们才能够确保用户不看到更新了一半的 UI, 与此同时, 浏览器也不会执行不必要布局和样式计算, 因为用户是看不到中间过程的状态变化的.

这也是为什么, React 要把这部分相关的工作分为两个阶段("渲染阶段"和"提交阶段")的原因.在 _渲染阶段_ React 调用你的组件并执行协调的操作. 这一过程即使被打断, 也能够确保正常的渲染流程, [未来](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html), 这一过程将会是异步的.

在 _提交阶段_, React 才会与宿主树进行交互. 这一过程始终是同步的.

## 缓存 Memoization

当父节点开始通过调用 `setState` 来规划一次 UI 更新的时候, 默认情况下 React 会对这个父节点下的所有子节点进行协调的操作. 这是因为 React 不知道父节点的更新是否会影响它的子节点. 同时, React 的默认行为是保持一致性. 这样的更新方式听起来成本很大, 但其实在实际生产过程中, 对于规模较小或者是中等的子树, 都不是什么大问题.

如果树的层级很深的话, 你可以告诉 React 把这些子树[缓存](https://en.wikipedia.org/wiki/Memoization)下来, 并且复用前一次渲染的结果, 前提是浅比较下 props 没有变化.

```jsx
function Row({ item }) {
  // ...
}
// highlight-next-line
export default React.memo(Row)
```

现在父组件中的 `<Table>` 中的 `setState` 会跳过对 `Row` 的协调, 因为它内部的 `item` 与上一次渲染时的 `item` 是同一个引用.

我们还能够使用 [`useMemo` Hook](https://reactjs.org/docs/hooks-reference.html#usememo) 来实现更加细粒度的缓存操作. 缓存位于组件内部, 如果组件内部的状态消失, 缓存也会一同消失. <mark>It only holds one last item.</mark>

React 内部默认不会对组件进行缓存. 一般情况下, 组件会接受很多 props, 因此缓存它们会是一个净损耗.

## Raw Models

很讽刺的是, React 对于十分细粒度的更新并没有采取实时响应的方式(并不那么 reactive). <mark>In other words, any update at the top triggers reconciliation instead of updating just the components affected by changes.</mark>

我们是刻意这样设计的. 在面向用户的 web 应用中, [可交互时间](https://calibreapp.com/blog/time-to-interactive)是一个关键性能指标, 遍历模型并且设置细粒度时间监听器的时间, 就相当于上述的可交互时间. 除此之外, 在许多应用中, <mark>交互所带来的更新: 比较小的是按钮 hover 这类的, 较大的则是页面的跳转.</mark>. 在这些场景下, 采取细粒度的响应方式其实会造成资源非必要的浪费.

React 中最核心的设计原则之一是: 它与 <mark>row data</mark> 进行交互. 如果你现在从网络请求中获取到一系列的 JavaScript 对象数据, 甚至可以不做任何预处理直接将它们扔到你的组件中. <mark>There are no gotchas about which properties you can access, or unexpected performance cliffs when a structure slightly changes. React rendering is O(view size) rather than O(model size), and you can significantly cut the view size with windowing.</mark>

当然, 还存在一些适合细粒度订阅更新的应用 -- 比如证券报价机. 这类需要实时更新的应用比较罕见, 它是其中之一. 对于这类应用, <mark>While imperative escape hatches can help optimize such code, React might not be the best fit for this use case. </mark> 尽管如此, 你还是可以基于 React 实现适合自己的细粒度订阅更新系统.

**不过需要注意的是, 这类系统会存在一个共性的性能问题.** 比如说, 当我们要渲染一棵很深的树(这种情况常在页面变化(transition)的时候发生)却不希望阻塞浏览器处理正常的任务. 实时更新的特性会使得这样的场景变得十分影响性能. 另一个问题是, 在开始渲染视图之前, 我们必须要等待数据准备好. 在 React 中, 我们使用了 [Concurrent Rendering](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html) 的方式来解决上面的问题.

## 合并更新

Several components may want to update state in response to the same event. This example is contrived but it illustrates a common pattern:

```jsx
function Parent() {
  let [count, setCount] = useState(0)
  return (
    // highlight-next-line
    <div onClick={() => setCount(count + 1)}>
      Parent clicked {count} times
      <Child />
    </div>
  )
}

function Child() {
  let [count, setCount] = useState(0)
  return (
    // highlight-next-line
    <button onClick={() => setCount(count + 1)}>
      Child clicked {count} times
    </button>
  )
}
```

当分发了一个事件之后, 子元素的 `onClick` 首先被触发(然后触发 `setState` 方法). 之后父元素在它的 `onClick` 处理器中调用 `setState` 方法.

如果 React 对于所有 `setState` 调用都立即重新渲染组件, 最终会造成子组件被渲染两次:

```jsx
*** Entering React's browser click event handler ***
Child (onClick)
  - setState
  // highlight-next-line
  - re-render Child // 😞 不必要的渲染
Parent (onClick)
  - setState
  - re-render Parent
  // highlight-next-line
  - re-render Child
*** Exiting React's browser click event handler ***
```

第一次`Child`的渲染是完全没有必要的. 我们不能让 React 跳过第二次 `Child` 的渲染, 因为 `Parent` 状态更新引起的重新渲染或许会传递给 `Child` 一些不同的数据, 如果跳过的话会引起一些问题.

**这也是为什么 React 会在事件处理器内部合并更新的原因:**

```jsx
*** Entering React's browser click event handler ***
Child (onClick)
  - setState
Parent (onClick)
  - setState
*** Processing state updates                     ***
  - re-render Parent
  - re-render Child
*** Exiting React's browser click event handler  ***
```

组件内部的 `setState` 调用不会立即引起重新渲染. 取而代之的是, React 会先执行所有的处理函数, 然后针对所有的更新执行一次页面的重新渲染.

合并更新对于性能优化有比较大的好处, 但是如果你的代码是下面这样的, 可能就会遇到一些问题:

```jsx
const [count, setCount] = useState(0)

function increment() {
  setCount(count + 1)
}

function handleClick() {
  increment()
  increment()
  increment()
}
```

如果我们最开始将 `count` 设置为 `0`, 以上的代码就是调用了三次 `setCount(1)`. 为了解决这个问题, `setState` 提供了一个重载的方式, 接受一个 "updater" 方法:

```jsx
const [count, setCount] = useState(0)

function increment() {
  setCount(c => c + 1)
}

function handleClick() {
  increment()
  increment()
  increment()
}
```

React 内部将这个 updater 方法放在一个队列中, 然后按顺序调用这些方法, 最终 `count` 被更新成了 `3`, 并且只引发了一次重新渲染.

当 state 中所存储的状态结构变得更复杂了之后, 我们可以使用 [`useReducer` Hook](https://reactjs.org/docs/hooks-reference.html#usereducer)来处理内部状态的更新. 这个更新方式其实就是 "updater" 模式的进阶版本, 唯一的区别是每一次状态更新都有一个名字:

```jsx
const [counter, dispatch] = useReducer((state, action) => {
  if (action === "increment") {
    return state + 1
  } else {
    return state
  }
}, 0)

function handleClick() {
  dispatch("increment")
  dispatch("increment")
  dispatch("increment")
}
```

`action` 参数可以是任何东西, 但是常见的方式是使用对象作为 `action` 参数的内容.

## 调用树

编程语言的运行时通常存在一个[调用栈](https://www.freecodecamp.org/news/understanding-the-javascript-call-stack-861e41ae61d4/). 当函数 `a()` 调用函数 `b()`, 函数 `b()` 调用函数 `c()` 的时候, 在 JavaScript 引擎中针对这种情况会存储类似 `[a, b, c]` 这样的数据结构, 在这里追踪你目前运行到了哪个位置, 接下来需要运行哪部分代码. <mark>Once you exit out of c, its call stack frame is gone — poof! It’s not needed anymore. We jump back into b. By the time we exit a, the call stack is empty.</mark> 一旦 `a` 执行结束之后, 它的调用栈就空了.

React 是基于 JavaScript 实现的, 因此遵循 JavaScript 的语法规则. 我们可以认为 React 内部维护了自己的调用栈, 以记录目前正在渲染的组件, 比如说 `[App, Page, Layout, Article /* 我们目前在这里 */].`

React 与普通的编程语言进行时有一些区别, 它的主要目的是渲染 UI 树. 这些树需要时刻保持激活状态, 这样我们才能够与它们进行交互. DOM 节点直到首次 `ReactDOM.render()` 被调用之后, 才会真正被移除.

这样的类比可能有点夸大了, 但是我倾向于认为 React 组件处于 "调用树" 中而不是 "调用栈" 中, 当 `Article` 元素调用完成之后, React 调用树桢不会被破坏. 我们需要在[某些地方](https://medium.com/react-in-depth/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)存储这些内部状态和宿主实例的引用.

当[协调](#协调-reconciliation)的规则认为必要的情况下, 这些"调用树"桢会与内部的状态以及宿主实例一同消失. 如果你曾经阅读过 React 的源码, 肯定知道 [Fiber](<https://en.wikipedia.org/wiki/Fiber_(computer_science)>) 的概念, 在 React 中, 桢指的就是 Fibers.

组件的内部状态就存储在 Fiber 中. 当状态更新的时候, React 会将 Fiber 设置为需要协调的状态, 然后调用那些组件.

## 上下文

一般情况下, 组件之间的数据会通过 props 传递. 某些情况下, 会存在一个全局的数据, 而大部分组件都需要这类数据, 举例来说就是页面当前的主题数据. 如果这类数据通过 props 一层一层传递的话, 就会十分繁琐.

在 React 中, 我们可以通过 [Context(上下文)](https://reactjs.org/docs/context.html) 来简化流程. 它就像是组件的[动态作用域](http://wiki.c2.com/?DynamicScoping)一样. It’s like a wormhole that lets you put something on the top, and have every child at the bottom be able to read it and re-render when it changes.

```jsx
const ThemeContext = React.createContext(
  "light" // Default value as a fallback
)

function DarkApp() {
  return (
    <ThemeContext.Provider value="dark">
      <MyComponents />
    </ThemeContext.Provider>
  )
}

function SomeDeeplyNestedChild() {
  // Depends on where the child is rendered
  const theme = useContext(ThemeContext)
  // ...
}
```

当 `SomeDeeplyNestedChild` 组件渲染的时候, `useContext(ThemeContext)` 会寻找调用树中最近的 `<ThemeContext.Provider>`, 然后使用它的值.

(在实际生产过程中, React 会在其渲染时维持一个上下文的栈.)

如果在调用树中没有发现 `ThemeContext.Provider`, `useContext(ThemeContext)` 的调用结果就会是 `createContext()` 中声明的默认值. 在我们的示例中就是 `'light'`.

## 副作用 Effects

前面我们说到过, React 组件在渲染的过程中不应该出现明显的影响用户体验的副作用. 但是有时候有些副作用是不可避免的, 比如说管理聚焦的动作, 在 canvas 中描绘一些内容, 订阅某个数据源等等.

在 React 中, 可以通过声明一个 effect 来实现:

```jsx
function Example() {
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

React 会在浏览器开始重绘之后, 才开始执行副作用. 这样的方式不会影响页面的[可交互时间(TTI)](https://calibreapp.com/blog/time-to-interactive/)和[首次有效绘制时间(First Meaningful Paint)](https://web.dev/first-meaningful-paint/).(React 还提供了一个[使用频率比较低](https://reactjs.org/docs/hooks-reference.html#uselayouteffect)的 Hook `useLayoutEffect`, 可以跳过 `useEffect` 的限制, 同步地进行渲染. 但是由于对性能的影响, 所以不建议使用这个 hook).

副作用并不只执行一次, 组件首次渲染和更新的时候的, 副作用都会执行. 在执行的期间, 读取到的是当时的 props 和 state, 比如以上示例中的 `count`.

声明副作用方法的同时还需要声明一个清除副作用的方法, 比如声明订阅操作为副作用方法的时候, 就需要同时声明一个取消订阅的方法:

```jsx
useEffect(() => {
  DataSource.addSubscription(handleChange)
  return () => DataSource.removeSubscription(handleChange)
})
```

React 会在下一次执行这个副作用方法之前执行一次清除副作用的方法, 组件卸载的时候也会执行一次这个方法.

在某些情况下, 我们不会希望每一次渲染都执行副作用方法. `useEffect` 提供了一个入口, 可以跳过某些非必要的渲染, 用户能够在这个入口中某些依赖, 只有在依赖项变化的时候, 其中的方法才会重新执行:

```jsx
useEffect(() => {
  document.title = `You clicked ${count} times`
  // highlight-next-line
}, [count])
```

但是如果你不了解 JavaScript 闭包的原理, 滥用 `useEffect`, 就可能会做出过度的优化, 并且引起许多不必要的问题.

举个例子, 下面的代码就是有问题的:

```jsx
useEffect(() => {
  DataSource.addSubscription(handleChange)
  return () => DataSource.removeSubscription(handleChange)
}, [])
```

因为我们没有在依赖项中传入任何参数, 因此副作用方法只会执行一次. 但是在副作用内部的 `handleChange` 是在外部定义的, 并且 `handleChange` 方法中可能引用了 props 或者 state:

```jsx
function handleChange() {
  console.log(count)
}
```

如果我们不允许副作用方法再次执行的话, `handleChange` 中的所引用的 `count`, 就会始终是第一次渲染时的值: `0`.

为了解决这个问题, 我们要确保依赖项数组定义的准确性, 其中需要包括副作用方法内部会有所变化的内容, 包括函数:

```jsx
useEffect(() => {
  DataSource.addSubscription(handleChange)
  return () => DataSource.removeSubscription(handleChange)
  // highlight-next-line
}, [handleChange])
```

修改了依赖之后, 我们就能够观察到订阅更新了, 但是还存在一个问题, 订阅被注册了多次. 这是因为每次重新渲染, `handleChange` 都发生了变化. 我们可以使用 [`useCallback`](https://reactjs.org/docs/hooks-reference.html#usecallback) hook 来帮助我们缓存这个方法, 避免不必要的重新注册. 当然你也可以不做优化, 让它进行重新注册. 浏览器的 `addEventListener` API 是很快的, 使用 `useCallback` 甚至可能带来更多的问题.

(你可以在[这里](https://reactjs.org/docs/hooks-effect.html)了解更多有关于 `useEffect` 以及其他 Hooks 的内容.)

## 自定义 Hooks

官方提供的内置 hooks, 都是一些函数调用方法, 我们可以将这些方法进行组合, 实现我们自己的 Hooks:

```jsx
function MyResponsiveComponent() {
  // highlight-next-line
  const width = useWindowWidth() // Our custom Hook
  return <p>Window width is {width}</p>
}
// highlight-next-line
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth)
  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth)
    window.addEventListener("resize", handleResize)
    return () => {
      window.removeEventListener("resize", handleResize)
    }
  })
  return width
}
```

自定义 hooks 使得不同的组件之间能共享一些状态相关的逻辑. 当然 `state` 本身是没有共享的, 各个组件之间都维护自己各自的状态.

(你可以在[这里](https://reactjs.org/docs/hooks-custom.html)学习如何实现自己的 Hooks)

## Static Use Order

你可以将 `useState` 看作是定义 React 状态变量的一种语法. 当然它并不是一种语法. 我们依然在写 JavaScript 代码. 但是既然我们把 React 看做是运行时环境, 同时 React 基于 JavaScript 实现的结果, 更好地描述了 UI 组件树, 在某种程度上看来, 它的特性其实十分接近一门编程语言.

如果 `use` 是一种语法的话, 那么一切看起来就十分合理了:

```jsx
// 😉 并不是真实的语法
component Example(props) {
  // highlight-next-line
  const [count, setCount] = use State(0);

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

What would putting it into a condition or a callback or outside a component even mean?

```jsx
// 😉 并不是真实的语法

// 这个是什么东西内部的变量? 
const [count, setCount] = use State(0);

component Example() {
  if (condition) {
    // 当条件的结果是 false 的时候, 会发生什么?
    const [count, setCount] = use State(0);
  }

  function handleClick() {
    // 函数又代表了什么?
    // 它们和变量有什么区别?
    const [count, setCount] = use State(0);
  }
```

React 的状态属于*组件*内部 and its identity in the tree. 如果 `use` 的语法确实存在, 强制将它置于组件内部的第一行其实很合理:

```jsx
// 😉 不是真正的语法
component Example(props) {
  // 在这里声明才是合理的
  const [count, setCount] = use State(0);

  if (condition) {
    // 是一个语法错误
    const [count, setCount] = use State(0);
  }
```

这和 `import` 的位置必须置于模块的顶部是类似的.

**当然, `use` 并非真正的语法.** (即使是的话, 它带来的好处其实也不大, 甚至会带来一些不便之处.)

不过, 对于 hooks 的位置, React 依然要求用户将其置于组件的顶层, 并且不要将它放在条件语句中. 在开发过程中, 我们可以用一个 [linter 插件](https://www.npmjs.com/package/eslint-plugin-react-hooks)来强制自己遵循 [hooks 的编写规则](https://reactjs.org/docs/hooks-rules.html). 这样的设计曾经带来过很大的争议, 但是在实际生产过程中, 这样的规则并没有对用户带来过多的困扰. 我还写过一篇文章, 提及了[为什么必须按照这样的规则来编写 hooks](https://overreacted.io/why-do-hooks-rely-on-call-order/).

hooks 的实现, 利用了[链表](https://dev.to/aspittel/thank-u-next-an-introduction-to-linked-lists-4pph)的数据结构. 当你调用 `useState` 的时候, 我们会将指针移到下一个 hooks. 当我们退出组件的["调用树"桢](#调用树)的时候, 会一直存储这些结果列表直到下一次渲染.

[这篇文章](https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e)对 hooks 的实现原理做出了一些简单的介绍. 用数组去塑造理解理解 hooks 的心智模型, 可能比链表更加容易一些:

```jsx
// 伪代码
let hooks, i;
function useState() {
  i++;
  if (hooks[i]) {
    // 后续的渲染
    return hooks[i];
  }
    // 首次渲染
  hooks.push(...);
}

// 准备渲染
i = -1;
hooks = fiber.hooks || [];
// 调用组件
YourComponent();
// 记住 hooks 的状态
fiber.hooks = hooks;
```

(生产版本的代码在[这里](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.new.js).)

以上就是 `useState()` 调用获取到 state 的代码实现. 我们[之前](#协调-reconciliation)已经了解到, 找到所匹配的部分在 React 中是已经存在的模式, 协调的过程就依赖元素之间的匹配. 

## 其他

我们已经谈到了几乎所有 React 运行时环境的重要方面. 如果你已经看到了这里, 那么你对 React 的了解已经超过了 90% 的 React 用户.

尽管如此, 本文还是缺少了一部分内容 -- 大多数是因为这些内容对于 React 开发团队来说, 也存在一些疑惑. <mark>React doesn’t currently have a good story for multipass rendering, i.e. when the parent render needs information about the children. </mark> 同时, 针对组[件渲染错误处理的 API](https://reactjs.org/docs/error-boundaries.html), 目前还没有对应的 hooks 版本. 这两个问题有可能会一起解决. Concurrent 模式目前依然处于不稳定的状态, and there are interesting questions about how Suspense fits into this picture. Maybe I’ll do a follow-up when they’re fleshed out and Suspense is ready for more than [lazy loading](https://reactjs.org/blog/2018/10/23/react-v-16-6.html#reactlazy-code-splitting-with-suspense).

I think it speaks to the success of React’s API that you can get very far without ever thinking about most of these topics. Good defaults like the reconciliation heuristics do the right thing in most cases. Warnings, like the key warning, nudge you when you risk shooting yourself in the foot.

If you’re a UI library nerd, I hope this post was somewhat entertaining and clarified how React works in more depth. Or maybe you decided React is too complicated and you’ll never look at it again. In either case, I’d love to hear from you on Twitter! Thank you for reading.