# Tailwind CSS + cva 实现样式变体组件

## 前言

button 组件是我们在开发中经常会用到的组件，这是一个看起来十分简单的组件，但是在实际开发中，我们经常会遇到这样的需求：同一个 button 组件，需要实现不同的样式，比如：不同的颜色、不同的大小、不同的形状等等。那么，我们如何通过 Tailwind CSS 优雅的实现这样的需求呢？

## 需求

为了方便演示，我们先来简单定义一下我们的基本需求：

- button 组件有几种语义类型 type：primary、secondary、success、danger 等
- button 组件有三种大小 size：small、medium、large
- button 组件有三种填充 fill：solid、outline、text
- button 组件有两种形状 shape：方形（小圆角）、胶囊形（full rounded）
- button 组件支持 disabled
- button 组件支持 Icon
- button 组件支持 loading

简单分析一下需求，前四个需求都是对应按钮的不同样式变体（variants），我们只需要给每种变体指定自己特有的样式即可。而 disabled 和 loading 对应按钮的不同状态。我们使用组合的方式构建 Button 组件，因此 Icon 的逻辑不必写在 Button 组件内部。这样的话，loading 态也可以分解成一个 loading-icon + disabled 的组合。

## 初步实现

我们会用到一个 [clsx](https://github.com/lukeed/clsx) 库，它可以帮助我们更方便的通过条件去控制样式的变化。我们先来看一下我们的 Button 组件的代码：

```jsx
import clsx from "clsx";

interface Props
  extends Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, "type"> {
  type?: "primary" | "success" | "danger";
  size?: "small" | "large" | "middle";
  fill?: "solid" | "outline" | "text";
  shape?: "round" | "square";
  disabled?: boolean;
}
export const Button = ({
  type = "primary",
  size = "middle",
  fill = "solid",
  shape = "square",
  children,
  ...props
}: Props) => {
  return (
    <button
      className={clsx(
        "inline-flex items-center justify-center rounded-lg text-sm font-medium disabled:opacity-20",
        {
          "bg-primary text-primary border-primary": type === "primary",
          "bg-success text-success border-success": type === "success",
          "bg-danger text-danger border-danger": type === "danger",
          "h-8  px-2 py-2": size === "small",
          "h-10 px-4 py-2": size === "middle",
          "h-12 px-6 py-2": size === "large",
          "text-white": fill === "solid",
          "border bg-white": fill === "outline",
          "bg-white": fill === "text",
          "!rounded-full": shape === "round",
        }
      )}
      {...props}
    >
      {children}
    </button>
  );
};
```

以上代码基本实现了我们的需求，但是存在两个问题：

1. clsx 中大量的条件判断代码，使代码看起来很臃肿，没有组织性，不利于代码的阅读与维护。
2. 我们无法保证 tailwind 中 class 的优先级，我们无法保证写在后面的样式可以覆盖前面的样式，比如 shape 为 round 的情况下，我们希望`rounded-full`更够覆盖基本样式中`rounded-lg`,但是我们无法保证这一点。只能通过`!rounded-full`来覆盖`rounded-lg`，`!important`的使用会导致样式的不可预测性，不利于代码的维护。

## 使用 cva + tailwind-merge 改写

[cva 是 Class Variance Authority 的缩写](https://cva.style/docs)，cva 是一个用于管理样式变体的库，它可以帮助我们更好的组织样式变体，使得代码更加清晰，更加易于维护。我们可以使用 cva 来重写我们的 Button 组件，使得代码更加清晰。具体的用法，可以去看官方文档。这里就不做过多用法介绍了。

[tailwind-merge](https://github.com/dcastil/tailwind-merge) 用来处理 tailwind 样式冲突问题，它可以让写在后面的样式覆盖前面的样式，这样我们就不需要使用`!important`来覆盖样式了。

以下是重构后的代码：

```jsx
import { cva, type VariantProps } from "class-variance-authority";
import clsx from "clsx";
import { twMerge } from "tailwind-merge";

interface Props
  extends Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, "type">,
    VariantProps<typeof buttonVariants> {}

const buttonVariants = cva(
  "inline-flex rounded-lg items-center justify-center text-sm font-medium disabled:opacity-50",
  {
    variants: {
      type: {
        primary: "bg-primary text-primary border-primary",
        success: "bg-success text-success border-success",
        danger: "bg-danger text-danger border-danger",
      },
      size: {
        middle: "h-10 px-4 py-2",
        small: "h-8  px-2 py-2",
        large: "h-12 px-6 py-2",
      },
      fill: {
        solid: "text-white",
        outline: "border bg-white",
        text: "bg-white",
      },
      shape: {
        round: "rounded-full",
        square: "rounded-lg",
      },
    },
    defaultVariants: {
      type: "primary",
      size: "middle",
      fill: "solid",
      shape: "square",
    },
  }
);

export const Button = ({
  type,
  size,
  fill,
  shape,
  children,
  ...props
}: Props) => {
  return (
    <button
      className={twMerge(clsx(buttonVariants({ type, size, fill, shape })))}
      {...props}
    >
      {children}
    </button>
  );
};
```

可以看到，改写后的 Button 组件清晰简洁了很多，我们把样式变体的定义和组件的实现分离开来，使得代码更加清晰，更加易于维护。

使用 twMerge 来处理样式冲突问题，使得我们可以更加方便的控制样式的优先级。

## 一些优化点

1. 如果组件的使用场景是 PC 端，可以添加`focus-visible`伪类，美化键盘聚焦 Button 时的样式。还可以给不同的 type 加上对应的 hover 态样式。
2. 如果组件的使用场景是移动端，可以添加`active`伪类，美化点击 Button 时的样式。
3. 如果希望外部对 Button 组件的样式进行覆盖，可以使用`className`属性，这样可以让 Button 组件更加灵活。代码如下：

```jsx
export const Button = ({
  className,
  type,
  size,
  fill,
  shape,
  children,
  ...props
}: Props) => {
  return (
    <button
      className={twMerge(
        clsx(buttonVariants({ type, size, fill, shape, className }))
      )}
      {...props}
    >
      {children}
    </button>
  );
};
```

## 总结

本文通过一个简单的 Button 组件，介绍了如何使用 cva 和 tailwind-merge 来更好的组织样式变体，使得代码更加清晰，更加易于维护。
