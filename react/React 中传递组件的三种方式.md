# React 中传递组件的三种方式

之前的那篇组件设计的文章中提到过，采用自上而下的方式去设计一些较为复杂的组件，随着组件内部复杂度的提升，巨石组件也就产生了。为了解决巨石组件的打包体积大、复用性差、逻辑复杂问题，我们需要将巨石组件拆分成多个小组件，然后通过某种方式将这些小组件组合起来。那么如何组合起来呢？当然是通过传递组件的方式啦。

通过传递组件，我们可以将复杂组件内部的一部分 UI 交由外部组件来控制渲染，这也是控制反转（Inversion of Control）的一种体现。在 React 中，我们可以通过三种方式来传递组件，分别是：

## 1. React Element as Prop (即我们通常所说的组合)

这种方式是最常见的一种方式，我们可以将一个 React Element 作为另一个组件的 prop，然后在内部通过`props.children`来渲染这个 React Element。这种方式的好处是简单易用，但是缺点也很明显，那就是我们不方便对传递进来的 React Element 进行控制，比如我们很难对其 props 进行控制。

假如我们设计一个 Button 组件，Button 内部的 Icon 区域交由外部来控制，那么我们可以这样设计：

```jsx
// App.js
const App = () => <Button icon={<Icon />}>按钮</Button>;

// Button.js
function Button(props) {
  return (
    <button>
      {props.icon}
      {props.children}
    </button>
  );
}
```

如果按钮 Icon 的颜色要受到 APP 内的 state 控制，我们很容易实现这样的功能：

```jsx
// App.js
const App = () => {
  const [color, setColor] = useState("red");
  return (
    <Button
      icon={<Icon color={color} />}
      onClick={() => setColor(color === "red" ? "blue" : "red")}
    >
      按钮
    </Button>
  );
};
```

可见，这种方式下，传递的组件 Icon 很容易获取外部环境 APP 组件的 state。但是如果 Icon 想获取 Button 组件的内部 state ,那么就不太容易了，因为 Button 组件无法控制 Icon 组件的 props。那么有没有一种方式可以让 Icon 组件获取到 Button 组件的内部 state 呢？答案是肯定的，那就是下面要介绍的第二种方式。

## 2. Component Function as Prop(传递组件函数)

这种方式，我们传递的是组件的函数，而不是组件本身。这样一来，我们就可以在组件内部控制传递进来的组件的 props 了。我们可以通过这种方式来实现上面的需求：

```jsx
// App.js
import Icon from "./Icon";
const App = () => {
  return <Button icon={Icon}>按钮</Button>;
};

// Button.js
function Button(props) {
  const Icon = props.icon; // 这里的 Icon 就是一个组件函数,注意Icon的首字母要大写
  const [color, setColor] = useState("red");
  return (
    <button>
      <Icon color={color} />
      {props.children}
    </button>
  );
}
```

可见，这种组件传递方式，我们可以在 Button 组件内部控制 Icon 组件的 props，这样 Icon 组件就可以获取到 Button 组件的内部 state 了。但是，这种方式也有缺点，那就是 Icon 不能获取外部环境（APP）的 state 了。那么有没有一种方式可以让 Icon 组件既能获取到 Button 组件的内部 state，又能获取到外部环境（APP）的 state 呢？答案是肯定的，那就是下面要介绍的第三种方式。

## 3. Render Function as Prop(即 render props 渲染属性)

这种方式，我们传递的是一个函数，通过函数的参数，我们可以获取 Button 组件的内部状态。因为函数是在外部环境（APP) 内声明的，所以也很容易获得外部状态。我们可以通过这种方式来实现上面的需求：

```jsx
// App.js
import Icon from "./Icon";
const App = () => {
  const [size, setSize] = useState(16);
  return (
    <Button renderIcon={(color) => <Icon color={color} size={size} />}>
      按钮
    </Button>
  );
};

// Button.js
function Button(props) {
  const [color, setColor] = useState("red");
  return (
    <button>
      {props.renderIcon(color)}
      {props.children}
    </button>
  );
}
```

## 总结

1. 看完上面的分析，你可能会认为 render props 是最好的传递组件的方式，但是其实不然，render props 也有缺点：a.组件层级不清晰，可读性差；b.re-render 问题（参考之前的 re-render 文章）。所以，我们在实际开发中，应该根据实际情况来选择传递组件的方式：

> 如果传递的组件只需要获取外部环境的 state，那么我们可以使用 React Element as Prop 的方式；

> 如果传递的组件需要获取宿主组件的 state，那么我们可以使用 Component Function as Prop 的方式；

> 如果传递的组件需要获取宿主组件的 state，同时也需要获取外部环境的 state，那么我们可以使用 Render Function as Prop 的方式。

2. 其实 React Element as Prop 的方式可以通过 React.cloneElement() 来获取宿主组件的内部状态。Componet Function as Prop 的方式可以通过高阶组件（HOC）的方式来注入外部状态。所以三种组件传递方式都可以做到内外部状态的获取，只不过那样的代码不够优雅。

3. 对于 Button 组件来说，第一种方式 props.icon 是一个 object, 后面两种方式 props.icon 都是 function ，可能有些人会搞不懂他们的区别，其实区别在于 Button 内部消费 props.icon 的方式不同，一种是当成函数来调用执行，一种是当做函数组件来声明。
