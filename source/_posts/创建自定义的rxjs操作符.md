---
title: 创建自定义的rxjs操作符
date: 2020-04-02 15:41:29
---

首先，操作符一般是在`pipe`中使用的，所以了解它的原理至关重要

```typescript
class Observable {
  pipe(...operators): Observable<any> {
    return operators.reduce((source, next) => next(source), this);
  }
}
```

`pipe`会将所有的操作符归并（操作符嵌套调用，返回函数作为下一个操作符的参数）为一个函数

<!-- more -->

> Angular 中的 HTTP 拦截器也是同样的原理

为了更好地说明，举个例子

```typescript
type callable = (...args: any) => any;
const source = (): any => {
  console.log("call source");
  return "ok";
};
function A(callback: callable): callable {
  console.log("call A 1");
  return (): any => {
    console.log("call A 2");
    const result = callback();
    console.log("call A 3");
    return result;
  };
}
function B(callback: callable): callable {
  console.log("call B 1");
  return (): any => {
    console.log("call B 2");
    const result = callback();
    console.log("call B 3");
    return result;
  };
}
function C(callback: callable): callable {
  console.log("call C 1");
  return (): any => {
    console.log("call C 2");
    const result = callback();
    console.log("call C 3");
    return result;
  };
}
const func = [A, B, C].reduce((source, next: callable) => next(source), source);
// call A 1
// call B 1
// call C 1
const result = func();
// call C 2
// call B 2
// call A 2
// call source
// call A 3
// call B 3
// call C 3
console.log(result);
// ok
```

理解了`pipe`如何处理操作符，就可以很轻松的自定义操作符，只要最终返回为一个函数即可：

```typescript
import { interval, Observable } from "rxjs";
import { map } from "rxjs/operators";

function MyOperator<T>(callback: (v: T) => T = (v: T): T => v) {
  // 必须返回函数以嵌套调用
  // 可以在已有操作符上拓展，例如
  // function filterNil() {
  //   return filter((value) => value !== undefined && value !== null);
  // }
  return (source: Observable<T>): Observable<T> => {
    // 归并过程将source替换为新建Observable对象
    // 可以在已有操作符上拓展，例如
    // function filterNil() {
    //   return function <T>(source: Observable<T>) {
    //     return source.pipe(
    //       filter((value) => value !== undefined && value !== null),
    //     );
    //   };
    // }
    return new Observable(subscriber => {
      return source.subscribe({
        next(value: T) {
          subscriber.next(callback(value));
        },
        error(error) {
          subscriber.error(error);
        },
        complete() {
          subscriber.complete();
        }
      });
    });
  };
}

interval(1000)
  .pipe(
    // MyOperator is same as map
    MyOperator((v: number) => v + 1),
    map((v: number) => v + 1)
  )
  .subscribe((v: number) => console.log(v));
```
