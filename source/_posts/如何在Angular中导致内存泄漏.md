---
title: 如何在Angular中导致内存泄漏
date: 2020-04-01 16:46:54
---

> 参考原文链接 <https://medium.com/angular-in-depth/how-to-create-a-memory-leak-in-angular-4c583ad78b8b>

## 什么是内存管理？

- 分配所需的内存
- 读写分配的内存
- 释放不用的内存

在 JavaScript 的世界中，通过 GC 完成内存的分配和释放，但是 GC 是如何工作的呢？

<!-- more -->

## 垃圾收集器的工作原理

基于引用计数实现：如果一块内存存在一个标签可以访问它，那么它的引用计数加一；丢失访问的标签时，引用计数减一；当没有任何标签可以访问它也就是引用计数为零时，GC 就会回收这块内存，清除数据。

> `WeakMap`使用对象作为 key 不会增加对象的引用计数

在 JavaScript 中，从全局对象（window）开始遍历所有对象，每个对象初始都包含一个 mark=false 标志

![1_IktekG0U4His_Uy8DKUZEA.png](https://i.loli.net/2020/04/01/R3lM4pN8QVhXWiK.png)

遍历过程中，可访问的对象设置 mark=true

![1_WUJP9RPgydkh6fHkt_-oxQ.png](https://i.loli.net/2020/04/01/Af9x4GqkzRlh5C8.png)

之后 GC 将所有 mark=false 的对象回收

![1_DMa0T-hrQflKbudQbofHXw.png](https://i.loli.net/2020/04/01/qHXuxFBASZUb8d9.png)

GC 会定期执行此算法，管理可用内存。

## 如何在 Angular 中制造内存泄漏

概括为一句话：在订阅了比组件生命周期长的服务暴露出来的可观察对象（Observable 或 Subject 等），但却没有在组件销毁时取消订阅的情况下，可能会出现内存泄漏。

如果可观察对象的声明周期与组件一致，不会出现内存泄漏。

![1_mX5Go7_ZKYQW3hnLjtXqFQ.png](https://i.loli.net/2020/04/01/zgtVecd5U93Qjfp.png)

反之则会内存泄漏

![1_b_YXWoQ_YAuvThekswP4Ug.png](https://i.loli.net/2020/04/01/lcNH1p7igmx4XoM.png)

## 解决订阅可观察对象导致的内存泄漏

**使用组件销毁钩子`ngOnDestroy`和`destroy$`取消订阅**

```typescript
import { Component, OnDestroy } from "@angular/core";
import { Subject } from "rxjs";
import { takeUntil } from "rxjs/operators";
import { DummyService } from "./dummy.service";

@Component({
  selector: "app-hello",
  template: "..."
})
export class HelloComponent implements OnDestroy {
  destroy$ = new Subject<void>();
  value: number;
  constructor(private dummyService: DummyService) {
    // 当destroy$触发时取消订阅
    this.dummyService.some$
      .pipe(takeUntil(this.destroy$))
      .subscribe((v: number) => {
        this.value = v;
      });
  }
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```
