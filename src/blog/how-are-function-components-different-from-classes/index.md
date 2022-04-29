---
title: 函数式组件和类式组件的区别
date: "2021-07-06"
template: "post"
draft: true
toc: true
description: "How Are Function Components Different from Classes?"
---

原文(2019-03-03): [How Are Function Components Different from Classes?](https://overreacted.io/how-are-function-components-different-from-classes/) 


React 函数式组件和类式组件有什么区别的呢?

很长一段时间, 对于这个问题的权威回答是这样的: 类式组件提供了更多的能力(比如存储状态的能力). 不过当 [Hooks](https://reactjs.org/docs/hooks-intro.html) 特性出现之后, 这个答案也就过时了.

或许你听说过这样的答案, 其中一类组件的性能更佳. 那么是哪一类呢? 其实许多评估性能的指标都多多少少存在一些[缺陷](https://medium.com/@dan_abramov/this-benchmark-is-indeed-flawed-c3d6b5b6f97f), 因此不能太过信任性能评估的结果. 应用的性能在很大程度上跟应用的功能(是否复杂)有关, 跟你所选择的组件类型其实关系不大. 根据我们的观察, 选择不同类的组件实现相同应用时, 两者的性能差异可以忽略不计, 不过针对这两类组件的优化策略, 还是有所[差异](https://reactjs.org/docs/hooks-faq.html#are-hooks-slow-because-of-creating-functions-in-render)的. 

正常来说, 我们[不建议](https://reactjs.org/docs/hooks-faq.html#should-i-use-hooks-classes-or-a-mix-of-both)重写目前的类式组件, 除非你有其它的原因或者是希望拥抱新特性. Hooks 在当下还是一个比较新的特性(就像2014年的 React 一样), 同时现在的教程中还没有最佳事件可供参考, 因此可以等观望一段时间之后再投入生产使用.

那么 React 函数式组件和类组件之间究竟有什么差别呢? 当然有, 在心智模型上, 它们有本质的差别. **在本文中, 我将会阐释它们之间最大的差别.** 这个差别在 2015 年函数式组件[出现](https://reactjs.org/blog/2015/09/10/react-v0.14-rc1.html#stateless-function-components)之后就存在了, 但是很多时候都被忽略了: 

> 函数式组件会捕捉当下渲染的值.

这是什么意思呢?

--- 


**注意: 本文并不对函数式组件的类组件的价值作高下评判. 只是阐释两者在 React 编程模型中的区别. 关于函数式组件和 Hooks 的使用问题, 可以看[这里](https://reactjs.org/docs/hooks-faq.html#adoption-strategy).**

---

查看下面的代码:

```jsx
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

以上代码做了这样的事情: 对于页面上的 button, 当我们点击它的时候, 会使用 `setTimeout` API 模拟一个网络请求, 3秒之后展示确认框. 举个例子, 如果 `props.user` 的值是 `'Dan'`, 那么三秒之后, 确认框就会弹出 `'Follow Dan'` 的文案. 

*(注意: 不管在上面的示例代码中, 我用的是普通的函数还是箭头函数, `function handleClick` 做的事情都是一样的.)*

如果想要以类组件的形式实现上述代码, 应该怎么做呢? 一个比较简单的实现方式是这样的:

```jsx
class ProfilePage extends React.Component {
  showMessage = () => {
    alert('Followed ' + this.props.user);
  };

  handleClick = () => {
    setTimeout(this.showMessage, 3000);
  };

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

乍看我们会认为以上两部分代码做的是同样的事情, 于是就用这样的方式重写原有组件代码:

![](./wtf.gif)

**然而以上代码并不完全一致.** 仔细看就会发现它们的差别.

**如果你想自己寻找两者的差异, 可以查看[这里](https://codesandbox.io/s/pjqnl16lm7)的代码示例.** 在文章的最后我们会详细说明两者的差别, 以及区分他们的差异有多么重要. 

---

在进入下一部分之前, 我想重点说明的是, 我们所阐释的内容和 React Hooks 没有直接的关系. 示例中并没有使用到 Hooks!

我想要说明的, 仅仅是 React 函数式组件和类式组件之间的差别. 如果你想要在应用中使用更多的函数式组件, 可以进一步阅读下面的内容.

---

**我们从 React 应用中一个常见的 bug 开始介绍两者的区别.**

打开[示例代码](https://codesandbox.io/s/pjqnl16lm7), 其中包含两种 `ProfilePage` 的实现 -- 均渲染了一个"关注"按钮.

现在我们开始执行下面的操作:

1. **点击**其中一个关注按钮
2. 在 3s 内**修改**所选择的人员
3. **读取**弹出的文案

你会发现一个奇怪的差别:

- 对于 `ProfilePage` **函数组件**, 点击关注 Dan, 之后切换成 Sophie, 仍然会弹出 `'Followed Dan'`.
- 对于 `ProfilePage` **类组件**, 重复以上的行为之后, 会弹出 `'Followed Sophie'`:

![](./bug.gif)

---

在这个例子中, 第一种行为是正确的. **如果我关注了某个人之后, 切换为另一个人, 组件不应该混淆我所关注的人.** 类组件的行为是有问题的.

_(不过, 你确实应该关注 [Sophie](https://mobile.twitter.com/sophiebits))_

---

为什么我们的类组件的行为是这样的呢?

现在仔细观察一下我们类组件中的 `showMessage` 方法