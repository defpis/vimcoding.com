---
title: promise在angular1.x中的实现
date: 2020-03-24 16:23:11
---

angular1.x 中实现 Promise 由两部分组成 Deferred 和 Promise。如果 Promise 承诺将来会提供某些值，那么 Deferred 就是使该值可用的计算过程。两者总是成对出现，但是通常可以被代码的不同部分访问。

![Xnip2020-03-21_20-03-10.jpg](https://i.loli.net/2020/03/21/ny8SQKuqtCGwfaP.jpg)

从数据流的角度来考虑它，则数据的生产者有一个 Deferred，而数据的使用者有一个 Promise。在未来的某个时刻，当生产者计算出了数据时，消费者将获得 Promise 的值。

angular1.x 使用 `$rootScope.$evalAsync` 来在整理周期内处理 Promise，这里仅仅通过 setTimeout 来延迟评估价值。

> 源码仓库：<https://github.com/defpis/promise-in-angular1.x>

- 实现：src/promise.ts
- 测试：test/promise.spec.ts

实现参考了《build-your-own-angularjs》，以 TypeScript 重新实现。
