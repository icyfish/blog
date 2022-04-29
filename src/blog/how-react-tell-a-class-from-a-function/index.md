---
title: React 如何区分类组件和函数式组件?
date: "2020-01-15"
template: "post"
draft: false
category: "React"
description: "How Does React Tell a Class from a Function?"
---

原文: [How Does React Tell a Class from a Function?](https://overreacted.io/how-does-react-tell-a-class-from-a-function/)


下面这个 `Greeting` 是一个函数式 React 组件:

```jsx
function Greeting() {
  return <p>Hello</p>;
}
```

React 同样支持以类的形式定义一个组件:

```jsx
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}
```

(函数式组件可以通过 [hooks](https://reactjs.org/docs/hooks-intro.html) 保存状态)

不过当我们使用一个组件的时候, 我们并不在意它是如何定义的:

```jsx
// Class or function — whatever.
<Greeting />
```

不过 React 本身对组件是函数式还是类式, 十分在意.

如果 `Greeting` 是函数式组件, React 的使用方式就是调用它:

```jsx
// Your code
function Greeting() {
  return <p>Hello</p>;
}

// Inside React
const result = Greeting(props); // <p>Hello</p>
```

如果 `Greeting` 是类式组件, React 的使用方式就是使用 `new` 操作符来实例化这个组件, 并调用所创建实例的 `render` 方法.

```jsx
// Your code
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

// Inside React
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

以上两种情况下, React 的目标都是获取需要被渲染的 DOM 节点(在我们的例子中是 `<p>Hello</p>` ). 具体的获取步骤, 还是根据 `Greeting` 如何被定义来决定.

**那么 React 是如何知道组件是类还是函数的呢?**

就像我在[这篇文章](https://overreacted.io/why-do-we-write-super-props/)中说的, 即使不知道这点, **你也能够高效地用 React 构建出 web 应用.** 我也并不是一开始就了解这个知识的, 因此请不要将这个知识点作为面试题询问求职者. 实际上, 这篇文章所涉及的知识更偏向于 JavaScript 而不是 React.

这篇文章的目标读者是对 React 的底层原理有好奇心的开发者, 如果你是的话, 那么就让我们一起探索吧.

**我们的探索旅程会很长. 请系好安全带. 本文涉及到的 React 知识点其实并不多, 更多的是关于 JavaScript 的知识点, 比如说: `new` , `this` , `class` , 箭头函数, `prototype` , `__proto__`, `instanceof` 等. 幸运的是, 当你在使用 React 的时候, 并不需要特别考虑这些. 不过站在 React 的开发者角度, 情况就不一样了......**

(如果你不希望了解实现细节, 只想要知道答案, 请直接查看文末的内容.)

---

首先我们需要理解, 为什么区别对待函数式组件和类组件是重要的. 当我们调用类组件时候, 需要依赖 `new` 操作符:

```jsx
// 如果 Greeting 是函数式组件, 在react内部会被这样调用
const result = Greeting(props); // <p>Hello</p>
// 如果 Greeting 是 class 组件, 在react内部会被这样调用
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

现在我们重点关注一下, `new` 操作符在 JavaScript 中具体做了些什么.

---

过去 JavaScript 中没有类的概念. 但是我们可以构造一个与类有类似功能的函数. 具体见下面的例子, 针对这种特殊的函数, 我们使用 `new`关键字进行调用.

```jsx
// 只是一个函数
function Person(name) {
  this.name = name;
}

var fred = new Person('Fred'); // ✅ Person {name: 'Fred'}
var george = Person('George'); // 🔴 不会起作用, 
// 如果将代码在控制台执行, 会发现 this 指向全局 window.name 是 George
```

如果我们调用 `Person('Fred')` 时没有加上 `new` 关键字, 函数中的 `this` 会指向全局, 那么这个 `this` 就一点意义也没有了. 我们的代码会崩溃, 同时还做了件蠢事: 创建了一个无用的全局变量.

**那么使用 `new` 调用函数时具体发生了些什么呢?**

在调用函数时添加`new`, 相当于告诉 JavaScript, `Person`虽然只是个函数, 但是我们可以把它看做类构造器. **此时 JavaScript 就会创建一个`{}`对象, 然后`Person`中的`this`值指向该对象, 同时将 `name` 设置为该对象的属性, 并且 `return` 出该对象.**

```jsx
var fred = new Person('Fred'); // Same object as `this` inside `Person`
```

通过以上的 `new` 调用, 对象 `fred` 还可以读取构造器函数 `Person.prototype` 上的属性.

```jsx
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  console.log('Hi, I am ' + this.name);
}

var fred = new Person('Fred');
fred.sayHi();
```

在 JavaScript 不原生支持`class`之前, 开发者就是通过这种方式模拟类的.

---

`new` 关键字存在已久, 而 `class` 的支持却是近段时间的事. 现在我们使用`class`改写先前的代码.

```jsx
class Person {
  constructor(name) {
    this.name = name;
  }
  sayHi() {
    alert('Hi, I am ' + this.name);
  }
}

let fred = new Person('Fred');
fred.sayHi();
```

在设计编程语言和 API 时, 用户的痛点至关重要.

开发者在编写函数时, JavaScript 无法猜测到函数是需要被直接调用还是在之前加关键词 `new` 调用.  因此开发者在调用类似 `Person` 这样的构造函数时, 务必记得在前面添加 `new` 关键字, 否则就会给程序引入一些令人疑惑的问题.

**有了 Class 语法之后, 我们的语境中就有了类的概念, 对于这类特殊的函数, 我们就能说, 它不仅仅是一个函数, 而是一个类, 同时有自己的构造函数.** 如果你在调用它的时候忘了加上 `new` 关键字, JavaScript 就会抛出错误.

```jsx
let fred = new Person('Fred');
// ✅  If Person is a function: works fine
// ✅  If Person is a class: works fine too

let george = Person('George'); // We forgot `new`
// 😳 If Person is a constructor-like function: confusing behavior
// 🔴 If Person is a class: fails immediately
```

这样一来, 我们就避免写出造成许多隐藏bug的代码, 比如 `this.name` 被当作是 `window.name` 而不是我们所预期的 `george.name` .

不过如果 React 也必须遵循这个原则的话, 那么在调用任何类组件的时候, 就必须在之前加上 `new` 关键字, 否则的话 JavaScript 就会抛出异常.

```jsx
class Counter extends React.Component {
	render() {
		return <p>Hello</p>;
	}
}

// 🔴 React 无法这样做
const instance = Counter(props);
```

---

那么 React 是如何解决这个问题的呢? 大多数开发者在使用React时会同时使用`Babel` 来编译最新语法的代码以兼容旧版本的浏览器. 因此我们把解决方案的头绪放在了编译器上.

在前几个版本的 `Babel` 中, 类可以不需要声明 `new` 关键字直接调用, 但是这个问题很快被 `Babel` 团队修复了, 我们来看一下他们的修复方式:

```jsx
function Person(name) {
  // A bit simplified from Babel output:
  if (!(this instanceof Person)) {
    throw new TypeError("Cannot call a class as a function");
  }
  // Our code:
  this.name = name;
}

new Person('Fred'); // ✅ Okay
Person('George');   // 🔴 Cannot call a class as a function
```

如果你看过项目打包之后的代码的话, 可能会觉得这个函数似曾相识, 这个就是`_classCallCheck` 函数所做的事情. (你可以通过开启"宽松检查模式"来减少打包之后代码的量, 但是这当然也带来了一个问题: 当处理真正的原生类语法时, 情况会变得更加复杂. You can reduce the bundle size by opting into the “loose mode” with no checks but this might complicate your eventual transition to real native classes.)

---

至此, 我们明白了调用函数时加 `new` 与不加 `new` 的区别:

![compare](./compare.png)

这就是为什么, 在React中, 正确调用组件是多么重要. **如果组件被定义为类组件, React必须在前添加 `new` 关键字进行调用.**

那么 React 是否可以不依赖任何其他信息区分出组件是函数式组件还是class组件呢?

并不简单. 虽然我们[在 JavaScript 中可以区分出 class 函数和普通函数](https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function), 但是被 `Babel` 处理过的代码, 就很难区分了. 因为经过处理之后, 对于浏览器来说, 它们都是普通的函数. 不过在 React 中, 我们还是有办法处理的.

---

或许 React 可以针对每一次调用都加上 `new` 关键字? 遗憾的事, 这样的方式并不靠谱. 

调用普通函数时添加 `new` 关键字会为该函数创建一个对象实例. 针对构造器函数, 使用 `new` 调用是预期行为, 但是针对函数式组件, 这种调用方式就会让人觉得疑惑.

```jsx
function Greeting() {
  // We wouldn’t expect `this` to be any kind of instance here
  return <p>Hello</p>;
}
```

以上这个原因并没有到不能接受的程度, 但是还有其他两个重要原因决定了不可以始终添加 `new` 关键词进行函数调用.

---

第一个原因是使用 `new` 调用箭头函数(未被 Babel 编译过) 会抛出异常:

```jsx
const Greeting = () => <p>Hello</p>;
new Greeting(); // 🔴 Greeting is not a constructor
```

由于不符合箭头函数的设计语法, JavaScript 有意限制了这种行为. 箭头函数最主要的特点之一是没有自己的 `this` , 如果在箭头函数内部使用 `this` 的话, `this` 的值会由离它最近的普通函数决定.

```jsx
class Friends extends React.Component {
  render() {
    const friends = this.props.friends;
    return friends.map(friend =>
      <Friend
        // `this` is resolved from the `render` method
        size={this.props.size}
        name={friend.name}
        key={friend.id}
      />
    );
  }
}
```

箭头函数没有自己的 `this`. 因此我们没有办法将箭头函数改造为构造器函数.

```jsx
const Person = (name) => {
  // 🔴 This wouldn’t make sense!
  this.name = name;
}
```

**JavaScript** 禁止了使用 `new` 调用箭头函数的行为. 也是由于同样的原因, 我们必须使用 `new` 关键词实例化一个类. 

因此, 在React中用 `new` 调用函数是不合理的. 那么针对这种情况, 我们是否可以检测出函数是否为箭头函数(如果没有 `prototype` , 则非箭头函数), 然后针对它做些特殊处理 (不要用 `new` 去调用它).

```jsx
(() => {}).prototype; // undefined
(function() {}).prototype // {constructor: f}
```

但是函数经过Babel编译之后, 就[不能够用以上的方式判断](https://github.com/facebook/react/issues/4599#issuecomment-136562930)了. 编译之后的结果:

```jsx
"use strict";

(function () {}).prototype;
(function () {}).prototype; 
```

不过这个问题比较好解决, 影响比较严重的是另一个原因:

---

当函数的返回值是字符串或其他原始类型时, 使用 `new` 对其进行调用不会得到我们预期的结果.

```jsx
function Greeting() {
  return 'Hello';
}

Greeting(); // ✅ 'Hello'
new Greeting(); // 😳 Greeting {}
```

这个行为和 `new` [操作符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)本身的设计有关. 在前文中我们知道, `new` 会告诉 JavaScript 引擎创建一个对象,  `this` 指向所创建的对象实例, 然后 `return` 出该实例.

JavaScript  同样允许函数被 `new` 调用时, 通过控制 `return` 的值来覆盖默认返回的对象.  当在对象池模式下, 想要复用函数实例时, 这种模式就可以发挥作用了. 

```jsx
// Created lazily
var zeroVector = null;

function Vector(x, y) {
  if (x === 0 && y === 0) {
    if (zeroVector !== null) {
      // Reuse the same instance
      return zeroVector;
    }
    zeroVector = this;
  }
  this.x = x;
  this.y = y;
}

var a = new Vector(1, 1);
var b = new Vector(0, 0);
var c = new Vector(0, 0); // 😲 b === c
```

不过有一点需要注意的是, 当手动 `return` 出的不是对象, 而是字符串或者数字类型, 这些值就无法覆盖默认的对象.

```jsx
function Answer() {
  return 42;
}

Answer(); // ✅ 42
new Answer(); // 😳 Answer {}
```

因此, 如果始终用 `new` 调用函数, `React` 就永远无法支持返回值为数字或者字符串的组件了.

由于这个原因, 我们必须做出一些妥协.

---

至此我们学到了: React 针对类组件调用时(包括经过 Babel 处理的结果), 需要添加 `new` 关键字, 但是普通函数和箭头函数调用却不需要. 可是到目前为止并没有可靠的方法对这两种方式进行区分.

**不过我们是否可以转换看问题的角度, 针对通用的问题我们没有办法解决, 那么是不是可以找到一个更具体的问题去解决. If we can’t solve a general problem, can we solve a more specific one?** 

当我们定义一个 React 类组件时, 该组件一定会继承 `React.Component` 以便使用 React 的内部方法 `this.setState()` .  **这样的话, 我们是否就不需要检测所有的类, 只需要检测继承 `React.Component` 的组件即可?**

剧透: 这就是React的处理方式.

---

现在我们要检测 `Greeting` 是否是一个React 类组件, 最符合直觉的检测方式是 `Greeting.prototype instanceof React.Component` :

```jsx
class A {}
class B extends A {}

console.log(B.prototype instanceof A); // true
```

为了理解以上代码的含义, 我们首先要理解 JavaScript 原型.  

大家对"原型链"这个概念应该已经很熟悉了. 在JavaScript中, 所有对象可能存在一个"原型". 当我们执行这段代码 `fred.sayHi()` 时, `fred` 这个对象本身可能并不存在 `sayHi` 属性. 我们会从 `fred` 的原型上寻找这个属性. 如果没有找到, 则会再深入原型链寻找 — `fred` 的原型的原型. 直到找到这个属性为止.

但比较令人疑惑的一点是, 类或者函数的 `prototype` 属性与它的原型并不对等.

```jsx
function Person() {}

console.log(Person.prototype); // 🤪 Not Person's prototype
console.log(Person.__proto__); // 😳 Person's prototype
```

原型链的结构是 `__proto__.__proto__.__proto__` 而不是 `prototype.prototype.prototype` .

那这个 `prototype` 属性是什么呢?  当我们用 `new` 实例化一个类或者函数时, 实例的 `__proto__` 属性就是就是类或函数的 `prototype` 属性.

```jsx
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name);
}

var fred = new Person('Fred'); // Sets `fred.__proto__` to `Person.prototype`
```

JavaScript使用**`__proto__`** 链查询属性:

```jsx
fred.sayHi();
// 1. Does fred have a sayHi property? No.
// 2. Does fred.__proto__ have a sayHi property? Yes. Call it!

fred.toString();
// 1. Does fred have a toString property? No.
// 2. Does fred.__proto__ have a toString property? No.
// 3. Does fred.__proto__.__proto__ have a toString property? Yes. Call it!
```

不过在实际应用中, 你不应该直接使用 `__proto__` , 除非调试时需要查询与原型链相关的信息. 如果你想使某些属性可以利用 `fred.__proto__` 访问, 就应该将该信息放在 `Person.prototype` . 这也是 `prototype` 设计的初衷.

`__proto__` 属性其实并不应该暴露给外部, 因为它本质上只给 JavaScript 内部使用. 但是某些浏览器依然支持了该属性甚至将其标准化(不过它会逐渐被 `Object.getPrototypeOf()` 所替代的.)

**现在我依然疑惑的是: 对象的 `prototype` 属性对应的值竟然不是对象的原型**(举个例子: `fred.prototype` 的值为 `undefined` , 因为 `fred` 不是一个函数). 个人认为, 这是开发者难以理解 JavaScript 原型概念的主要原因.

---

现在我们知道了, 当我们输入 `obj.foo` 时, JavaScript 实际上会在 `obj` `obj.__proto__`, `obj.__proto__.__proto__` ...上寻找 `foo` 属性.

在 JavaScript 类中, 并没有直接暴露这个机制, 不过 `extends` 的实现基于这种原型链机制. 这也是我们的 React 类实例读取 `setState` 方法所利用的机制:

```jsx
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

let c = new Greeting();
console.log(c.__proto__); // Greeting.prototype
console.log(c.__proto__.__proto__); // React.Component.prototype
console.log(c.__proto__.__proto__.__proto__); // Object.prototype

c.render();      // Found on c.__proto__ (Greeting.prototype)
c.setState();    // Found on c.__proto__.__proto__ (React.Component.prototype)
c.toString();    // Found on c.__proto__.__proto__.__proto__ (Object.prototype)
```

也就是说, **当我们使用类的时候, 类实例的 `__proto__` 链反映了这种原型链关系:**

```jsx
// `extends` chain
Greeting
  → React.Component
    → Object (implicitly)

// `__proto__` chain
new Greeting()
  → Greeting.prototype
    → React.Component.prototype
      → Object.prototype
```

那么既然了解了 `__proto__` 链的概念, 我们就能通过这种方式检查 `Greeting` 是否继承自 `React.Component` , 从 `Greeting.prototype` 开始通过 `__proto__` 不断层层深入.

```jsx
// `__proto__` chain
new Greeting()
  → Greeting.prototype // 🕵️ We start here
    → React.Component.prototype // ✅ Found it!
      → Object.prototype
```

还有一个更加方便的方式, `x instanceof Y` 也可以实现与以上代码几乎差不多的功能. 跟着 `x.__proro__` 的原型链寻找 `Y.prototype` .

正常情况下, 我们通过这种方式来判断类的实例:

```jsx
let greeting = new Greeting();

console.log(greeting instanceof Greeting); // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype (✅ Found it!)
//     .__proto__ → React.Component.prototype 
//       .__proto__ → Object.prototype

console.log(greeting instanceof React.Component); // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype (✅ Found it!)
//       .__proto__ → Object.prototype

console.log(greeting instanceof Object); // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype
//       .__proto__ → Object.prototype (✅ Found it!)

console.log(greeting instanceof Banana); // false
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype 
//       .__proto__ → Object.prototype (🙅‍ Did not find it!)
```

同时也能用来判断某个类是否继承自另一个类:

```jsx
console.log(Greeting.prototype instanceof React.Component);
// greeting
//   .__proto__ → Greeting.prototype (🕵️‍ We start here)
//     .__proto__ → React.Component.prototype (✅ Found it!)
//       .__proto__ → Object.prototype
```

以上这种方式可以用以区分React 类组件和函数组件.

---

不过React在生产中并不是使用这种方式去检查的. 😳

使用 `instanceof` 的判断方式有个弊端, 当我们的页面中存在多个 React 的拷贝时,  我们检查的某个组件可能继承自另一个 React 拷贝的 `React.Component` . 在一个项目中使用多个 React 拷贝当然是不推荐的做法. 但是由于历史原因, 我们可能会用多个 React 拷贝来解决某些问题. (不过如果我们在项目中使用 Hooks 的话, [我们可能必须要禁止多个拷贝的存在](https://github.com/facebook/react/issues/13991).)

另一个判断方式是, 检查原型链中的 `render` 方法是否存在. 然而, 当时对于[组件相关的 API 规范并没有完全确定下来.](https://github.com/facebook/react/issues/4599#issuecomment-129714112) 随时可能会变化. 如果使用这种方式的话, Every check has a cost so we wouldn’t want to add more than one. 同时, 如果 `render` 被定义为实例方法的话, 这种方式也会失效.

因此 React 采用了一种给基础组件[添加](https://github.com/facebook/react/pull/4663)特殊标志的方式. React 会检查这个标志是否存在, 通过这个标志来判断这是个类组件还是函数组件.

在最初的实现版本中, 这个标志存在于 `React.Compoenent` 本身:

```jsx
// Inside React
class Component {}
Component.isReactClass = {};

// We can check it like this
class Greeting extends Component {}
console.log(Greeting.isReactClass); // ✅ Yes
```

然而, 一些类在[实现](https://github.com/scala-js/scala-js/issues/1900)上并没有拷贝静态属性或者设置非标准的 `__proto__` . 在这种情况下 React 添加的特殊标志就丢失了.

于是React团队[将这个标志转移](https://github.com/facebook/react/pull/5021)到了 `React.Component.prototype` 上了:

```jsx
// Inside React
class Component {}
Component.prototype.isReactComponent = {};

// We can check it like this
class Greeting extends Component {}
console.log(Greeting.prototype.isReactComponent); // ✅ Yes
```

也许很多人会疑惑, 为什么我们的标识不是布尔值而是对象. 这是因为我们在使用 Jest 进行自动化测试时, 前几个版本的 Jest 默认打开了[automocking](https://dev.to/slashgear_/discover-jest-hidden-feature-automock-2c9f), 生成的模拟测试数据会[忽略原始值](https://github.com/facebook/react/pull/4663#issuecomment-136533373), 因此如果用布尔值的话, 会影响自动化测试的结果.

`isReactComponent` 的检测方式至今依然在[使用](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L297-L300). 

如果你在编写React组件的时候没有继承 `React.Component` . React就不会在原型链中找到 `isReactComponent` 这个属性, 也不会将其看做 class 组件. 读到这里, 大家可能就都明白[这个问题](https://stackoverflow.com/questions/38481857/getting-cannot-call-a-class-as-a-function-in-my-react-project/42680526#42680526)的答案了: 如果在React的开发过程中遇到 `Cannot call a class as a function` 的报错, 记得在使用React 类组件时加上 `extends React.Component` . 同时如果你没有这样使用类组件, 却组件内却定义了 `render` 方法, React也会报出一个[⚠️](https://github.com/facebook/react/pull/11168). 

---

你可能会觉得这篇文章有点挂羊头卖狗肉 ([bait-and-switch](https://en.wikipedia.org/wiki/Bait-and-switch)) 的感觉. **实际的解决方案那么简单, 可是我们却不直接回答这个问题, 而是饶了那么一大圈去解释为什么替代方案是不可行的.** 

在我的经验中, 在实现一个库的 API 时, 这是经常会遇到的情况. 对于一个使用起来很方便的 API, 你必须要深入考虑语言本身的语义, 运行时的性能, 存在/不存在编译步骤时的效率, 相关生态的发展以及打包方案, 早期的告警以及很多其他问题. 最终设计出成果或许不会是最优雅的, 但是肯定是最实用的.

如果最终设计出来的 API 很成功的话, 那么 API 的使用者就完全不需要考虑它的设计过程, 只需要专注于创建应用即可.

如果使用者对 API 的设计感到好奇, 了解它的底层原理当然是很好的.