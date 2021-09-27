---
title: Ref objects inside useEffect Hooks
date: "2021-09-22"
template: "post"
draft: false
toc: true
category: "React"
tags:
  - "React"
description: "Ref objects inside useEffect Hooks"
---

原文: [Ref objects inside useEffect Hooks](https://medium.com/@teh_builder/ref-objects-inside-useeffect-hooks-eb7c15198780)

---

当我们要实现一个自定义 hook, 索引到某个 DOM 节点的时候, 你或许首先会想到使用 `useRef` hook 来实现. 这也是我的首选项.

因此在我初次实现 [react-intersection-observer](https://github.com/thebuilder/react-intersection-observer) 的时候, 就使用了这种实现方式. 但是在实现过程中, 发现了一个很有趣的问题: [如何处理 hooks 的条件渲染?](https://github.com/thebuilder/react-intersection-observer/issues/162).

最主要的问题之一是, 我以为 `ref` 始终会有值, 但是实际情况是:

- `ref` 的初始值是 `undefined`, 之后才会被赋值.
- `ref` 有可能变为另一个完全不同的元素.
- `ref` 有可能从一个元素变为 `undefined`.

我的自定义 hook 并没有处理以上任何一种情况.

### ref 和 useEffect 的问题

`useRef` hook 对我们的自定义 hook 来说是一个陷阱, 如果搭配 `useEffect` 一起使用的话. 你的直觉应该是把 `ref.current` 加到 `useEffect` 的依赖参数中, 这样一来, 一旦 `ref` 变化, `useEffect` 中的函数也会重新执行.

但问题来了, 只有当组件重新渲染的时候, `ref`	值才会有所变化, 因此在 `useEffect` 中, 在下一次渲染完成之前, 都不会捕获到 `ref` 值的变化.

[CodeSandBox示例](https://codesandbox.io/s/01m0ok09rv?from-embed)

观察社区中的这些自定义 hooks, 会发现这是一个普遍存在的问题. 一些自定义 hooks 的网站, 比如[useHooks](https://usehooks.com/), [the-platform](https://github.com/jaredpalmer/the-platform) 都跌入了这个陷阱中, 因为大部分开发者都认为这个 `ref`, 在第一次渲染之后, 就不会变化了.

于是我[在 Twitter 上](https://twitter.com/teh_builder/status/1093497029693763584?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1093497348913803265%7Ctwgr%5E%7Ctwcon%5Es2_&ref_url=https%3A%2F%2Fcdn.embedly.com%2Fwidgets%2Fmedia.html%3Ftype%3Dtext2Fhtmlkey%3Da19fcc184b9711e1b4764040d3dc5c07schema%3Dtwitterurl%3Dhttps3A%2F%2Ftwitter.com%2Fdan_abramov%2Fstatus%2F1093497348913803265image%3Dhttps3A%2F%2Fi.embed.ly%2F1%2Fimage3Furl3Dhttps253A252F252Fpbs.twimg.com252Fprofile_images252F906557353549598720252FoapgW_Fp_400x400.jpg26key3Da19fcc184b9711e1b4764040d3dc5c07)直接去问了 Dan Abramov 如何解决这个问题. 

React 团队确实会给我们以[警示](https://reactjs.org/docs/hooks-effect.html#explanation-why-effects-run-on-each-update), 当我们跳过某些副作用的执行的时候.

### 如何解决

在这种情况下, 我们不应该使用 `useRef`, 而是用 `useCallback` 来创建一个回调 ref([callback ref](https://reactjs.org/docs/refs-and-the-dom.html#callback-refs)).



```js
function useHookWithRefCallback() {
  const ref = useRef(null);
  const setRef = useCallback(node => {
    if (ref.current) {
      // Make sure to cleanup any events/references added to the last instance
    }

    if (node) {
      // Check if a node is actually passed. Otherwise node would be null.
      // You can now do what you need to, addEventListeners, measure, etc.
    }

    // Save a reference to the node
    ref.current = node;
  }, []);

  return [setRef];
}
```

一旦某个 DOM 节点触发了这个回调函数, 你就能够根据需要消费对应的 DOM 节点.

> 这里有个比较完整的示例, 可以观察一下 [react-intersection-observer](https://github.com/thebuilder/react-intersection-observer) 内部所依赖的 [useInView](https://github.com/thebuilder/react-intersection-observer/blob/master/src/useInView.tsx) 的源码

### 官方的解决方案

React 团队在文档的 FAQ 中也有一个[相关的例子](https://reactjs.org/docs/hooks-faq.html#how-can-i-measure-a-dom-node), 使用 ref callback 来计算 DOM 元素的大小尺寸等. 对于我们所提出的问题, 这也是一个解决方案.

