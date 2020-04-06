---
title: 实现rxjs中的Observable
date: 2020-04-06 19:53:08
---

> 内容来源于 [`Ben Lesh`](https://github.com/benlesh) 的视频 [`Creating Observable From Scratch`](https://egghead.io/lessons/rxjs-creating-observable-from-scratch)，可以观看视频以加深理解。

```typescript
class SafeObserver {
  destination: any;
  isUnsubscribed = false;
  _unsubscribe: any = null;

  constructor(destination: any) {
    this.destination = destination;
  }

  next(value: any): void {
    const destination = this.destination;
    if (destination.next && !this.isUnsubscribed) {
      destination.next(value);
    }
  }

  error(error: any): void {
    const destination = this.destination;
    if (!this.isUnsubscribed) {
      this.unsubscribe();
      if (destination.error) {
        destination.error(error);
      }
    }
  }

  complete(): void {
    const destination = this.destination;
    if (!this.isUnsubscribed) {
      this.unsubscribe();
      if (destination.complete) {
        destination.complete();
      }
    }
  }

  unsubscribe(): void {
    this.isUnsubscribed = true;
    if (this._unsubscribe) {
      this._unsubscribe();
    }
  }
}

class Observable {
  _subscribe: any;

  constructor(subscribe: any) {
    this._subscribe = subscribe;
  }

  subscribe(observer: any): any {
    const safeObserver = new SafeObserver(observer);
    safeObserver._unsubscribe = this._subscribe(safeObserver);
    return {
      unsubscribe(): void {
        safeObserver.unsubscribe();
      },
    };
  }
}
```
