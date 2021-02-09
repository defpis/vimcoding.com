---
title: 了不起的RxJS
date: 2021-02-09 15:54:54
---

最近在处理项目中复杂的鼠标 + 键盘交互时使用到了 rxjs，之前使用 rxjs 还是在写 angular，angular 本身就是通过 rxjs 构建的，默认的 http 请求数据返回的就是包裹了一层 Observable，所以对我来说还算是有一定基础，常见的操作符 map、filter、reduce、merge 等也能熟练使用，配合 angular 的单例 service 可以很方便的管理和分发状态。但对于传统 react 项目已经使用 redux 单一 store 管理和分发状态，引入 rxjs 可能并不是一个明智选择，因为两者混用会带来维护的复杂性和困难性。虽然社区内也有 rxjs-hooks 这种将 rxjs 和 react hooks 结合的库，但并不主流。

项目之所以引入 rxjs，是想借助 rxjs 处理异步事件的强大能力，来抽象通用的操作逻辑。例如拖拽这种常见操作，没引入 rxjs 前，总是单独处理 mousedown、mousemove、mouseup 回调函数

<!-- more -->

```typescript
class Screen {
  enableMoving() {
    this.on("mousedown", this.onMousedown);
  }

  disableMoving() {
    this.off("mousedown", this.onMousedown);
  }

  onMousedown = () => {
    this.on("mousemove", this.onMousemove).on("mouseup", this.onMouseup);
  };

  onMousemove = () => {
    // 处理鼠标移动
  };

  onMouseup = () => {
    this.off("mousemove", this.onMousemove).off("mouseup", this.onMouseup);
  };
}
```

回调函数的处理隐含了时间上的联系，处理起来需要十分小心。这个例子只是最简单的情况，复杂情况会让我们代码变得支离破碎，难以维护。使用 rxjs 来写却非常清晰

```typescript
const dragMoveSub = mousedown$
  // mousedown -- switch --> mousemove -- util --> mouseup
  .pipe(switchMap(() => mousemove$.pipe(takeUtil(mouseup$))))
  .subscribe(() => {
    // 处理鼠标移动
  });

// 禁用拖拽
dragMoveSub.unsubscribe();
```

当然你可能会质疑这个跟不同人写代码水平有关，用第一种方式一样能写出可维护的代码，不可否认是这样，用 rxjs 也不能解决不合理架构或不清晰流程带来的灾难。而且对于大多数交互简单的项目，根本没必要使用 rxjs，回调依然是最简单可靠的解决方式。

知乎上有一讨论什么情况下才需要使用到 rxjs：[redux、mobx、rxjs 这三款数据流管理工具在你项目中是如何取舍的？](https://www.zhihu.com/question/277530559)，感兴趣可以研究研究。

## 什么是响应式编程

由于 js 单线程执行，所以需要引入异步来规避一些耗时操作，相较于一直同步等待结果返回，更好的方式是注册一个回调函数，等任务完成后触发，在此期间可以去干别的事情，而不至于阻塞浏览器，让用户以为卡死了。回调函数某种意义上实现了控制反转（IOC），可以通过发送请求来举例：

![](https://tech-proxy.bytedance.net/tos/images/1609258143809_61695b95b9c21e1f87f223e1f3cd64cd.jpg)

同步代码一步一步执行，每一步依赖上一步的状态和执行完成，因此在时空上是连续和因果的

![](https://tech-proxy.bytedance.net/tos/images/1609258143786_7e9d09961f09e348a7b1bf2cc662c0da.jpg)

而异步代码执行开始是按顺序的，但是执行完成却是不确定的

![](https://tech-proxy.bytedance.net/tos/images/1609258143818_d748618ed0a36627e008db74e5406e57.jpg)

过程中没法确定何时 step2 可以访问到 step1 产生的数据，当然做到这一点只需嵌套回调来保证时序的正确即可

![](https://tech-proxy.bytedance.net/tos/images/1609258143818_047f72dcdcdd54c2dc2486dae3f59531.jpg)

哈哈，这不就相当于把异步调用同步化了么，可以这么理解，但是代码写起来却变复杂了，举个例子

```typescript
ajax("<host1>/items", (items) => {
  for (let item of items) {
    ajax(`<host2>/items/${item.getId()}/info`, (dataInfo) => {
      ajax(`<host3>/files/${dataInfo.files}`, processFiles);
    });
  }
});
beginUiRendering();
```

如果嵌套的再深一点，就出现了大家所熟知的回调地狱，代码会疯狂的水平生长，难以维护。还有一个问题就是在异步中使用循环也很危险，循环中的异步代码可不保证按照遍历的时序执行，常常引起不可预料或难以理解的 bug。

由于回调难以处理复杂情况下的异步问题，提出了 Promise 的解决方案。作为对于未来值的“承诺”，Promise 允许你将未来的值和对未来值的操作链接起来形成“延续（continuation）”，延续只是一种编写回调的花哨术语，与之前所提到的控制反转原理有很大关系。为了更好的理解，可以将之前的 ajax 请求转换为 Promise 写法来举例。

首先分解抽象它的逻辑，then 关键字定义时序关系

```text
Fetch all items, then
  For-each item fetch all files, then
    Process each file
```

将每一个异步调用的返回值封装为 Promise，待值 Fulfilled 时触发 then 的回调，以此触发下一步操作

```typescript
let getItems = () => ajax("<host1>/items");
let getInfo = (item) => ajax(`<host2>/data/${item.getId()}/info`);
let getFiles = (dataInfo) => ajax(`<host3>/data/files/${dataInfo.files}`);

getItems()
  .then((items) => items.map(getInfo))
  .then((promises) => Promise.all(promises))
  .then((infos) => infos.map(getFiles))
  .then((promises) => Promise.all(promises))
  .then(processFiles);
```

相比于回调函数那一版，不再出现嵌套的水平生长，回调函数通过 then 定义的时序依次调用，可维护性大大增强。

![](https://tech-proxy.bytedance.net/tos/images/1609258143820_102bd6b0feef0013a6bfa4b35eb74d8a.jpg)

Promise 创建一个由 then 方法链接的调用流。如果 Promise 被完成，功能链将继续；否则，将错误委托给 Promise catch 块。

Promise 的值无论成功还是失败只会被回调使用一次，它无法处理产生多个值的数据源。例如鼠标移动或者文件流中的字节序列。而且由于状态的不可逆转，它缺乏从失败中重试的能力。此外 Promise 是不可变的，一旦产生无法被取消。例如用来包裹 http 请求的返回数据时，希望中途取消请求，但却没有相应的机制取消 Promise then 的订阅。

因此需要一种完全不同的方式来解决所有这些问题：

- 熟悉的循环不能很好的协同工作，因为它们不了解异步，忽略了迭代之间的等待时间。
- 无法捕获异步中的错误，你需要在回调中嵌套 try/catch 块，但这样错误处理策略会变得很复杂。
- 代码频繁嵌套，过度耦合，难以分解测试。
- 创建太多的闭包，关联太多副作用，导致行为难以预测。
- 很难检测到事件或长时间运行的操作何时失控并需要取消。
- 没有限制用户与 UI 的交互，避免系统不必要的过载。
- 随着 UI 的复杂和丰富，未释放的事件侦听器导致内存泄漏。

## 未完待续
