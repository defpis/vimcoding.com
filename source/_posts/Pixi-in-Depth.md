---
title: Pixi-in-Depth
date: 2021-03-28 12:37:46
---

pixi.js 部分源码解析

<!-- more -->

## 初始化应用

Application 主要完成以下任务：

1. 构建舞台 stage 作为渲染数据
2. 构建渲染器 renderer 渲染 stage
3. 依次调用 plugins 的 init 方法
4. 暴露 render 方法，渲染一帧

> packages/app/src/Application.ts

```ts
class Application{
  public stage: Container = new Container();

  constructor {
    this.renderer = autoDetectRenderer(options);
    Application._plugins.forEach((plugin) => {
      plugin.init.call(this, options);
    });
  }

  render() {
    this.renderer.render(this.stage);
  }
}
```

## 构建渲染循环

Application 通过调用 registerPlugin 往静态属性 \_plugins 数组中存入插件

其中一个名为 TickerPlugin 的插件，用来专门处理 GameLoop

> bundles/pixi.js/src/index.ts

```ts
Application.registerPlugin(TickerPlugin);
```

进入 TickerPlugin 的 init 方法可以发现 Ticker 绑定了 render 方法

> packages/ticker/src/TickerPlugin.ts

```ts
// this 指向 Application 实例
if (ticker) {
  ticker.add(this.render, this, UPDATE_PRIORITY.LOW);
}
```

在探究 ticker.add 之前，先理解一下什么是 GameLoop，如果你使用过 canvas 绘制动画，对下面的代码一定不陌生：

```ts
(function tick() {
  requestAnimationFrame(tick);
  // animate
})();
```

> The window.requestAnimationFrame() method tells the browser that you wish to perform an animation and requests that the browser calls a specified function to update an animation before the next repaint. The method takes a callback as an argument to be invoked before the repaint.

requestAnimationFrame 类似 setTimeout，tick 不断异步递归自己，相当于带间隔的死循环，每一帧调用，触发浏览器重绘

Ticker 做了类似的事情，只是额外增加了一些功能：

- 维护了任务队列，可以按优先级排序
- 过滤或取消多余的绘制请求，防止过载
- 绑定回调自动启动渲染循环（autoStart===true）
- 提供 add 和 addOnce 来实现每次和单次执行
- 帧率控制，优化渲染最小帧率和最大帧率范围

Ticker 在构造函数中创建了 \_tick 作为每一帧的回调函数

> packages/ticker/src/Ticker.ts

```ts
this._tick = (time: number): void => {
  this._requestId = null;

  if (this.started) {
    this.update(time);
    if (this.started && this._requestId === null && this._head.next) {
      this._requestId = requestAnimationFrame(this._tick);
    }
  }
};
```

- started 用于控制渲染循环的运行和退出
- \_head 以链表的方式组织所有回调任务
- \_requestId 来标记正在进行的渲染请求

回调任务通过 add 和 addOnce 插入链表，它们都调用 \_addListener 方法，只是构建的 TickerListener 传入参数有所区分

> packages/ticker/src/Ticker.ts

```ts
add<T = any>(fn: TickerCallback<T>, context?: T, priority = UPDATE_PRIORITY.NORMAL): this {
  return this._addListener(new TickerListener(fn, context, priority));
}

addOnce<T = any>(fn: TickerCallback<T>, context?: T, priority = UPDATE_PRIORITY.NORMAL): this {
  return this._addListener(new TickerListener(fn, context, priority, true));
}
```

\_addListener 通过遍历 \_head 之后每个节点的 priority 属性来确定插入位置

> packages/ticker/src/Ticker.ts

```ts
let current = this._head.next;
let previous = this._head;

while (current) {
  if (listener.priority > current.priority) {
    listener.connect(previous);
    break;
  }
  previous = current;
  current = current.next;
}
if (!listener.previous) {
  listener.connect(previous);
}
```

TickerListener.connect 实现了链表插入

> packages/ticker/src/TickerListener.ts

```ts
connect(previous: TickerListener): void {
  this.previous = previous;
  if (previous.next) {
    previous.next.previous = this;
  }
  this.next = previous.next;
  previous.next = this;
}
```

在 Ticker.update 方法中会遍历回调任务链表，依次调用 TickerListener.emit

> packages/ticker/src/Ticker.ts

```ts
const head = this._head;

let listener = head.next;

while (listener) {
  listener = listener.emit(this.deltaTime);
}
```

在 TickerListener.emit 执行回调函数，并返回下一个节点

addOnce 会将 TickerListener.once 标记为 true，执行一次后会从链表中删除

> packages/ticker/src/TickerListener.ts

```ts
emit(deltaTime: number): TickerListener {
  if (this.fn) {
    if (this.context) {
      this.fn.call(this.context, deltaTime);
    } else {
      (this as TickerListener<any>).fn(deltaTime);
    }
  }

  const redirect = this.next;

  if (this.once) {
    this.destroy(true);
  }

  return redirect;
}
```

Ticker.update 还存在一段逻辑用于控制帧率逻辑，不过有点混乱，暂时略过

## 构建舞台树

初始化 Application 默认会创建 stage 作为舞台的根容器，之后所有需要渲染的对象都需要存储到它或它下属节点的 children 属性中，形成一颗舞台树

常见的方式是通过 addChild 把一个对象 push 到 Container.children 中

```ts
const g = new PIXI.Graphics();
stage.addChild(g);
```

Graphics 本身也是 Container，所以可以在它的下面继续挂载节点

舞台上所有的对象以树形结构组织，渲染很自然变成了遍历整颗树，然后依次绘制每个物体的过程

每一个物体的绘制可以简单分为两部分（类似 DOM 绘制过程中的重排和重绘），一找到绘制的位置，二调用 API 绘制图像

对于第一个需求，每个对象可以从当前位置向上遍历至根节点，依次计算它们相对位置影响的总和，比如

```text
              A
    向x偏移10 /  \ 向x偏移-10
            B   C
  向y偏移-10 |   | 向y偏移10
            D   F
```

如果将 A 坐标系设为全局坐标系，原点为(0, 0)，D 遍历 D->B->A，得到 D 坐标系原点在 A 中为(10, -10)，A 原点在 D 中表示则为(-10, 10)

如果每个物体都这样计算，效率是极低的，况且很多时候 D 单独变化时，B->A 根本没变化，优化的思路就是为每一层级添加缓存

```text
              A(0, 0)
    向x偏移10 /  \ 向x偏移-10
     (10, 0)B   C(-10, 0)
  向y偏移-10 |   | 向y偏移10
   (10, -10)D   F(-10, 10)
```

如果 D 此时相对与父节点不是向 y 偏移 -10 而是向 y 偏移 10，那么可以很容易根据 B(10, 0) 重新计算得到 D(10, 10)

这样做还有一个优势就是非常容易的实现坐标在不同坐标系中的转换：

- 假设 D 中有个点 P1(1, 1)，可以计算得到 P1 在 A 中的坐标为(11, -9)
- 假设 A 中有个点 P2(2, 2)，可以计算得到 P2 在 D 中的坐标为(-8, 12)

Pixi 也是类似的思路，DisplayObject 上就有 Transform 对象负责管理和缓存位置变换信息，可以找到两个重要属性

> packages/math/src/Transform.ts

```ts
public worldTransform: Matrix;
public localTransform: Matrix;
```

对于 worldTransform 就类似 D 相对 A 的位置偏移(10, -10)，而 localTransform 就类似 `向y偏移-10` 可以表示为(0, -10)

节点的 worldTransform 总是通过它的父节点的 worldTransform 和它自己 localTransform 一起计算得到

缓存有了，何时更新缓存又是一个大问题！别着急，看看代码发现了几个计数器

> packages/math/src/Transform.ts

```ts
public _parentID: number;
_worldID: number; // => _currentParentID
protected _localID: number;
protected _currentLocalID: number;
```

Pixi 通过计数器来确定上层或本层是否发生改变，本层或下层是否需要更新，四个计数器成队出现，两两比较值是否相等

> packages/math/src/Transform.ts

```ts
updateTransform(parentTransform: Transform): void {
  const lt = this.localTransform;

  if (this._localID !== this._currentLocalID) {
      lt.a = this._cx * this.scale.x;
      lt.b = this._sx * this.scale.x;
      lt.c = this._cy * this.scale.y;
      lt.d = this._sy * this.scale.y;

      lt.tx = this.position.x - ((this.pivot.x * lt.a) + (this.pivot.y * lt.c));
      lt.ty = this.position.y - ((this.pivot.x * lt.b) + (this.pivot.y * lt.d));
      this._currentLocalID = this._localID;

      this._parentID = -1;
  }

  if (this._parentID !== parentTransform._worldID) {
      const pt = parentTransform.worldTransform;
      const wt = this.worldTransform;

      wt.a = (lt.a * pt.a) + (lt.b * pt.c);
      wt.b = (lt.a * pt.b) + (lt.b * pt.d);
      wt.c = (lt.c * pt.a) + (lt.d * pt.c);
      wt.d = (lt.c * pt.b) + (lt.d * pt.d);
      wt.tx = (lt.tx * pt.a) + (lt.ty * pt.c) + pt.tx;
      wt.ty = (lt.tx * pt.b) + (lt.ty * pt.d) + pt.ty;

      this._parentID = parentTransform._worldID;

      this._worldID++;
  }
}
```

- 当前节点的 \_localID 和当前节点的 \_currentLocalID 配对，每当修改了位置状态，使 \_localID 自增，以更新 localTransform，更新完成同步 \_localID 和 \_currentLocalID，并使当前节点的 \_parentID 为 -1，以便继续更新当前节点的 worldTransform
- 当前节点的 \_parentID 和上层节点的 \_worldID 配对，如果不相等，更新当前节点的 worldTransform，同步当前节点的 \_parentID 和上层节点的 \_worldID，自增当前节点的 \_worldID，以便继续更新下层节点的 worldTransform

回到添加物体到舞台的过程，查看 addChild 源码

> packages/display/src/Container.ts

```ts
if (child.parent) {
  child.parent.removeChild(child);
}

child.parent = this;
this.sortDirty = true;

child.transform._parentID = -1;

this.children.push(child);
```

除了 push 到 children 数组，额外的工作就是设置 transform.\_parentID 为 -1，以便在渲染循环中更新变换矩阵

更新发生在每一帧都会调用的 Renderer.render 方法中，skipUpdateTransform 默认为 undefined

> packages/core/src/Renderer.ts

```ts
if (!skipUpdateTransform) {
  const cacheParent = displayObject.enableTempParent();

  displayObject.updateTransform();
  displayObject.disableTempParent(cacheParent);
}
```

对于普通 DisplayObject 对象，updateTransform 仅仅更新它自己的状态

> packages/display/src/DisplayObject.ts

```ts
updateTransform(): void {
  // ...

  this.transform.updateTransform(this.parent.transform);
}
```

但是对于 Container 对象，继承 DisplayObject 重写了 updateTransform 方法

> packages/display/src/Container.ts

```ts
updateTransform(): void {
  // ...

  this.transform.updateTransform(this.parent.transform);

  for (let i = 0, j = this.children.length; i < j; ++i) {
    const child = this.children[i];

    if (child.visible) {
      child.updateTransform();
    }
  }
}
```

Container.updateTransform 方法以深度优先顺序调用，更新所有受到影响的下属节点

舞台树所有节点的位置信息准备就绪，开始绘制！

## 渲染到画布

待舞台树位置更新完成后，会开始绘制

> packages/core/src/Renderer.ts

```ts
if (!skipUpdateTransform) {
  const cacheParent = displayObject.enableTempParent();

  displayObject.updateTransform();
  displayObject.disableTempParent(cacheParent);
}

displayObject.render(this);
```

DisplayObject.render 是一个抽象方法，并没有实现，查看 Container.render 的实现

> packages/display/src/Container.ts

```ts
render(renderer: Renderer): void {
  // ...

  if (/* */) {
    // ...
  } else {
    this._render(renderer);

    for (let i = 0, j = this.children.length; i < j; ++i) {
      this.children[i].render(renderer);
    }
  }
}
```

Container.render 也是以深度优先顺序调用，具体过程由子类的 \_render 方法实现

Graphics 继承于 Container，以它举例，定位到 Graphics.\_render 方法

> packages/graphics/src/Graphics.ts

```ts
 protected _render(renderer: Renderer): void {
  this.finishPoly();

  const geometry = this._geometry;
  const hasuint32 = renderer.context.supports.uint32Indices;

  geometry.updateBatches(hasuint32);

  if (geometry.batchable) {
    if (this.batchDirty !== geometry.batchDirty)
    {
        this._populateBatches();
    }

    this._renderBatched(renderer);
  }
  else {
    renderer.batch.flush();

    this._renderDirect(renderer);
  }
}
```

geometry.batchable 等于 true 为批处理逻辑，是为了提升性能做的优化，先不管这个分支，之后研究批处理系统时再深入

进入 Graphics.\_renderDirect 方法

> packages/graphics/src/Graphics.ts

```ts
protected _renderDirect(renderer: Renderer): void {
  // ...

  renderer.shader.bind(shader);
  renderer.geometry.bind(geometry, shader);

  renderer.state.set(this.state);

  for (let i = 0, l = drawCalls.length; i < l; i++) {
    this._renderDrawCallDirect(renderer, geometry.drawCalls[i]);
  }
}
```

可以看到 renderer 绑定了 shader 和 geometry，还设置了状态，内部已经为开始绘制准备好了 WebGL 上下文 gl，现在只需根据生成的 drawCall 依次绘制即可，暂时忽略调用 WebGL API 绘制的过程，之后分析 Renderer 封装时详细研究

由于 geometry.drawCalls 和渲染的图形息息相关，为了查看它的信息，起了一个 demo，仅仅绘制一条线

> demo.ts

```ts
import "./index.scss";
import * as PIXI from "pixi.js";

const container = document.querySelector("#root")! as HTMLDivElement;

const { renderer, stage, ticker } = new PIXI.Application({
  backgroundColor: 0xf3f3f3,
  autoDensity: true,
  resolution: window.devicePixelRatio,
  antialias: true,
  autoStart: true,
  resizeTo: container,
});

container.append(renderer.view);

const g = new PIXI.Graphics();
stage.addChild(g);
g.lineStyle(1);
g.moveTo(0, 0);
g.lineTo(100, 100);
```

<!--
finishPoly -> drawShape -> graphicsData
updateBatches + graphicsData -> buildDrawCalls
 -->
