---
title: 使用RxJS实现快捷键
date: 2021-02-09 15:51:23
---

项目中经常需要实现快捷键功能，通过 RxJS 可以很容易实现！

<!-- more -->

首先拦截所有键盘事件

```typescript
import { fromEvent, merge } from "rxjs";

const keyDown$ = fromEvent<KeyboardEvent>(document.body, "keydown");
const keyUp$ = fromEvent<KeyboardEvent>(document.body, "keyup");

const keyAction$ = merge(keyDown$, keyUp$);

keyAction$.subscribe((event) => console.log(event.key, event.type));
```

对于任意按键的 down 和 up 都会触发 `console.log`

```text
z keydown
z keyup
x keydown
x keyup
...
```

测试发现一个问题，按住按键不放会一直触发 down 事件，解决只需用 `distinctUntilChanged` 去重即可，当然为了共享流，还需要添加 `share` 操作符

```typescript
const isSameKey = (a: KeyboardEvent, b: KeyboardEvent): boolean =>
  a.code === b.code && a.type === b.type;
const keyAction$ = merge(keyDown$, keyUp$).pipe(
  distinctUntilChanged(isSameKey),
  share()
);
```

一般快捷键要求同时按下组合键来生效，例如 `C-x` 等于先按住 `Control`，再按住 `x`，相当于同时检测两个流，两个流的最新 `event.code` 都为预期值时，触发效果

于是可以定义一个数组 `keyCodes`，数组中的每个 `keyCode` 就是要同时按住的按键。通过 `map` 获取一个流数组，每个流仅当 `keyCode` 匹配时流动

```typescript
const keyCodes = ["ControlRight", "KeyX"];
const observables = keyCodes.map((keyCode) =>
  keyAction$.pipe(filter((event) => event.code === keyCode))
);
```

然后使用 `combineLatest` 合并每个流的最新值，保证它们都是按下的情况，触发合并流

```typescript
const isAllKeyDown = (events: KeyboardEvent[]): boolean =>
  events.every((event) => event.type === "keydown");
const C_x$ = combineLatest(observables).pipe(filter(isAllKeyDown));
```

得到的 `C_x$` 就是同时按下 `Control` 和 `x` 的流

测试又发现一个问题，同时按下不存在顺序，先按 `x` 再按 `Control` 也能触发流，希望避免这种情况

可以再定一个操作符来过滤顺序不正确的情况

```typescript
const keepOrder = () => (source: Observable<KeyboardEvent[]>) =>
  source.pipe(
    filter((events) => {
      const sortedSeq = events
        .slice()
        .sort((a, b) => a.timeStamp - b.timeStamp)
        .map((event) => event.code)
        .join();
      const originalSeq = events.map((event) => event.code).join();
      return sortedSeq === originalSeq;
    })
  );

const C_x$ = combineLatest(observables).pipe(filter(isAllKeyDown), keepOrder());
```

But 不知道怎么回事 `Control` 始终会触发两次，导致保顺失败，换成其他字母按键没有问题。

所有代码如下：

```typescript
import { combineLatest, fromEvent, merge, Observable } from "rxjs";
import { distinctUntilChanged, filter, share } from "rxjs/operators";

const keyDown$ = fromEvent<KeyboardEvent>(document.body, "keydown");
const keyUp$ = fromEvent<KeyboardEvent>(document.body, "keyup");

const isSameKey = (a: KeyboardEvent, b: KeyboardEvent): boolean =>
  a.code === b.code && a.type === b.type;
const keyAction$ = merge(keyDown$, keyUp$).pipe(
  distinctUntilChanged(isSameKey),
  share()
);

const keyCodes = ["KeyX", "KeyY", "KeyZ"];
const observables = keyCodes.map((keyCode) =>
  keyAction$.pipe(filter((event) => event.code === keyCode))
);

const isAllKeyDown = (events: KeyboardEvent[]): boolean =>
  events.every((event) => event.type === "keydown");

const keepOrder = () => (source: Observable<KeyboardEvent[]>) =>
  source.pipe(
    filter((events) => {
      const sortedSeq = events
        .slice()
        .sort((a, b) => a.timeStamp - b.timeStamp)
        .map((event) => event.code)
        .join();
      const originalSeq = events.map((event) => event.code).join();
      return sortedSeq === originalSeq;
    })
  );

const xyz$ = combineLatest(observables).pipe(filter(isAllKeyDown), keepOrder());
xyz$.subscribe(() => console.log("x y z"));
```
