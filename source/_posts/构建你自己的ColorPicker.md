---
title: 构建你自己的ColorPicker
date: 2020-11-24 20:13:17
---

好久没有更新博客了，真的印证了那句话：“字节一年，人间三年”，从入职到现在一直忙于项目上的事情。

最近在项目中使用到了颜色选择器，公司的组件库无法满足需要，所以我去 Github 找了相关实现，最后通过 react-color 凑合完成了任务。但是完成任务的我一直很好奇它是如何实现的，一不做二不休，干脆自己搞一个，万一 react-color 也没法满足项目需要的时候，我还可以手撸顶上去。

<!-- more -->

作为一个前端程序猿，还是比较熟悉颜色编码的，RGB 和 HEX 都能熟练使用，但是对于更多编码 HSL、HSB/HSV、CMYK 和 YUV 等就知之甚少了，并不清楚为啥同一个东西要整出这么多表现形式，为此我查阅了相关资料，收获颇丰。

RGB 和 HEX 本质上是一样的，都是通过 10 进制 0-255 或 16 进制 00-ff 来表示 R-Red、G-Green 和 B-Blue 三种光的权重，通过混合三种光来获的各种颜色。这种表示方法对于机器很好理解，但是对于人类就不那么直观了，一眼瞧过去，很难知道这是啥颜色，鲜艳不鲜艳，明亮还是灰暗，所以人们就基于这种痛点重新对颜色进行了编码，主要分为了 HSL、HSB/HSV 两种编码方式。（CMYK 和 YUV 暂时没有了解，如果有兴趣可以一起讨论）

> 查看[色彩空间中的 HSL、HSV、HSB 有什么区别](https://www.zhihu.com/question/22077462)

首先，HSB 和 HSV 是同一个东西，只是名称不同。

HSB 和 HSL 在字面意思上是一样的：

- H 指的是色相（Hue），就是颜色名称，例如“红色”、“蓝色”；
- S 指的是饱和度（Saturation），即颜色的纯度；
- L（Lightness） 和 B（Brightness）是明度，颜色的明亮程度

在表现和原理上，HSB 和 HSL 中的 H（色相）完全一致，但是二者的 S（饱和度）和不一样，L 和 B（明度）也不一样。

- HSB 中的 S 控制纯色中混入白色的量，值越大，白色越少，颜色越纯；
- HSB 中的 B 控制纯色中混入黑色的量，值越大，黑色越少，明度越高；
- HSL 中的 S 和黑白没有关系，饱和度不控制颜色中混入黑白的多寡；
- HSL 中的 L 控制纯色中的混入的黑白两种颜色，作用等同于 HSB 中的 S 和 B 之和；

想必看到这里的你一定还是很迷糊为啥可以通过参杂黑白控制饱和度或明度，而饱和度具体定义有代表什么？别着急，可以看一下这篇文章[公式剖析色彩三要素：色相、饱和度、明度](https://zhuanlan.zhihu.com/p/30409532)，里面比较详细阐述了色彩三要素具体的含义，并且非常好的为每一种要素的提供了数学概念，通过数学概念能深入理解颜色的表示方法。

至此理论知识补充的差不多了，可以开始实践操作了。我也去网上找了已有的实现，按照自己的理解重新撸了一个版本。

实际效果如图

![](https://i.loli.net/2020/11/24/Lqf9GJEj4W8HyrN.png)

UI 主要分为三部分

- 饱和度面板
- 色相拖动条
- 透明度拖动条

目录结构如下

```
ColorPicker
├── ColorPicker.scss
├── ColorPicker.tsx
├── components
│   ├── Alpha                 // 透明度
│   │   ├── Alpha.scss
│   │   └── Alpha.tsx
│   ├── Hue                   // 色相
│   │   ├── Hue.scss
│   │   └── Hue.tsx
│   └── Saturation            // 饱和度
│       ├── Saturation.scss
│       └── Saturation.tsx
└── hooks                     // 通用逻辑
```

首先来构建饱和度面板

```tsx
<div
  style={{ width: `${width}px`, height: `${height}px` }}
  className="color-picker-wrapper color-picker-saturation"
>
  <div
    ref={ref}
    style={{ background: `hsl(${hsva.h}, 100%, 50%)` }}
    className="saturation"
  >
    <div
      className="saturation-thumb"
      style={{
        left: `${pos.offsetX}px`,
        top: `${pos.offsetY}px`,
        borderColor: pos.offsetY < height / 2 ? "black" : "white",
      }}
    />
  </div>
</div>
```

通过伪元素来叠加两层透明渐变遮罩层，一层从全白到透明，另一层从全黑到透明，分表向 X 和 Y 轴渐变，样式如下

```scss
.color-picker-saturation {
  @mixin saturation-pseudo-element {
    @include full();
    content: "";
    position: absolute;
  }

  .saturation {
    @include full();
    background: transparent;
    cursor: crosshair;

    &::before {
      @include saturation-pseudo-element();
      background: linear-gradient(to right, white, transparent);
    }
    &::after {
      @include saturation-pseudo-element();
      background: linear-gradient(to top, black, transparent);
    }

    .saturation-thumb {
      position: absolute;
      transform: translate(-10px, -8px);
      z-index: 1;
      width: 15px;
      height: 15px;
      border-radius: 50%;
      border: 1px solid;
      background-color: transparent;
    }
  }
}
```

预留了一个 div 作为滑块，通过设置为绝对定位，然后根据传入的坐标来渲染。

至于拖动的逻辑，封装到成了 useMouseDrag，方便之后 Slider 组件复用

可以像 useRef 一样使用获得一个 ref 引用

```typescript
const [ref, pos, setPos] = useMouseDrag<HTMLDivElement>(null);
```

通过 ref 引用需要检测位置变化的父容器

```tsx
<div
  ref={ref}
  style={{ background: `hsl(${hsva.h}, 100%, 50%)` }}
  className="saturation"
>
  ...
</div>
```

这样 pos 就可以获取鼠标拖动在父容器的坐标

useMouseDrag 具体实现如下：

```typescript
interface IPosition {
  offsetX: number;
  offsetY: number;
}

export function useMouseDrag<T extends HTMLElement>(
  element: T | null
): [RefObject<T>, IPosition, Dispatch<SetStateAction<IPosition>>] {
  const ref = useRef<T>(element);
  const [start, toggle] = useToggle(false);
  const [pos, setPos] = useState<IPosition>({ offsetX: 0, offsetY: 0 });

  const setPosition = useMemo(
    () =>
      throttle(({ clientX, clientY }: MouseEvent) => {
        const element = ref.current;
        if (!element) return;

        const { left, top } = element.getBoundingClientRect();
        const offsetX = clamp(clientX - left, 0, element.clientWidth);
        const offsetY = clamp(clientY - top, 0, element.clientHeight);
        setPos({ offsetX, offsetY });
      }, 40),
    []
  );
  const startDrag = useCallback(
    (event: MouseEvent) => {
      setPosition(event);
      toggle(true);
    },
    [toggle, setPosition]
  );
  const dragging = useCallback((event: MouseEvent) => setPosition(event), [
    setPosition,
  ]);
  const stopDrag = useCallback(() => toggle(false), [toggle]);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    element.addEventListener("mousedown", startDrag);
    return () => {
      element.removeEventListener("mousedown", startDrag);
    };
  }, [startDrag]);

  useEffect(() => {
    const element = ref.current;
    if (!element || !start) return;

    element.addEventListener("mousemove", dragging);
    document.addEventListener("mouseup", stopDrag);
    return () => {
      element.removeEventListener("mousemove", dragging);
      document.removeEventListener("mouseup", stopDrag);
    };
  }, [start, dragging, stopDrag]);

  return [ref, pos, setPos];
}
```

之后来构建色相拖动条，还是同样的套路，熟悉的配方

```tsx
<div
  style={{ width: `${width}px`, height: `${height}px` }}
  className="color-picker-wrapper color-picker-hue"
>
  <div ref={ref} className="hue">
    <div
      className="hue-thumb"
      style={{
        left: `${pos.offsetX}px`,
      }}
    />
  </div>
</div>
```

通过指定关键颜色渐变形成色相环

```scss
.color-picker-hue {
  box-sizing: border-box;
  border-left: 4px solid #f00;
  border-right: 4px solid #f00;

  .hue {
    @include full();
    background: linear-gradient(
      to right,
      #f00 0%,
      #ff0 16.66%,
      #0f0 33.33%,
      #0ff 50%,
      #00f 66.66%,
      #f0f 83.33%,
      #f00 100%
    );

    .hue-thumb {
      position: absolute;
      width: 8px;
      height: 100%;
      background-color: #fff;
      box-shadow: 0 0 2px rgba(0, 0, 0, 0.6);
      transform: translateX(-4px);
    }
  }
}
```

拖动的逻辑复用 useMouseDrag

同理透明度拖动条也是一样

```tsx
<div
  style={{
    width: `${width}px`,
    height: `${height}px`,
    borderRightColor: color,
  }}
  className="color-picker-wrapper color-picker-alpha"
>
  <div
    ref={ref}
    className="alpha"
    style={{
      background: `linear-gradient(to right, transparent, ${color})`,
    }}
  >
    <div
      className="alpha-thumb"
      style={{
        left: `${pos.offsetX}px`,
      }}
    />
  </div>
</div>
```

```scss
.color-picker-alpha {
  box-sizing: border-box;
  border-left: 4px solid;
  border-left-color: transparent;
  border-right: 4px solid;
  background-color: white;
  background-image: linear-gradient(
      45deg,
      #c5c5c5 25%,
      transparent 0,
      transparent 75%,
      #c5c5c5 0,
      #c5c5c5
    ), linear-gradient(45deg, #c5c5c5 25%, transparent 0, transparent 75%, #c5c5c5
        0, #c5c5c5);
  background-size: 10px 10px;
  background-position: 0 0, 5px 5px;

  .alpha {
    @include full();

    .alpha-thumb {
      position: absolute;
      width: 8px;
      height: 100%;
      background-color: #fff;
      box-shadow: 0 0 2px rgba(0, 0, 0, 0.6);
      transform: translateX(-4px);
    }
  }
}
```

这样通过计算拖动位置和宽度或高度的比值，就可以计算相应的 H、S、V 和 A 值，计算方式如下：

```typescript
import { round } from "lodash";

const s = round(pos.offsetX / element.clientWidth, 2);
const v = round((element.clientHeight - pos.offsetY) / element.clientHeight, 2);
const h = round((pos.offsetX / element.clientWidth) * 360);
const a = round(pos.offsetX / element.clientWidth, 2);
```

为了显示获得 HSV 值，将其转换为了 HSL（CSS 支持 HSL 而不是 HSV）作为 div 的 CSS 样式

```tsx
const hsl = hsv2hsl(hsva);
const color = `hsl(${hsva.h}, ${hsl.s * 100}%, ${hsl.l * 100}%)`;

<div
  style={{ backgroundColor: color, opacity: `${hsva.a * 100}%` }}
  className="color-picker-block"
/>;
```

HSV 转 HSL 的公式网上也能随便找到

```typescript
const hsv2hsl = ({ h, s, v }: IHSVA): { h: number; s: number; l: number } => {
  return {
    h,
    s: round((s * v) / ((h = (2 - s) * v) < 1 ? h : 2 - h) || 0, 2),
    l: round(h / 2, 2),
  };
};
```

这样任何颜色更改都会在这个 div 上生效

至此这个版本颜色选择器就搞定了，非常简单，但是对于我来说简单的东西想要做好做完美也很不容易。日常工作中不重复造轮子用于生产，但是通过构建简单版本的轮子来学习我认为很有必要。
