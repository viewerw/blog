## 什么是组合（composition) 模式

`composition` 是一种自下而上的组件设计模式，由一些细小的、单一职责的、原始组件通过自由搭配组合在一起构成一个功能完善的大组件，这样的一种组件设计模式就是组合模式。有很多组件库都是基于这种模式设计的，比如 `Radix`, `Chakra UI`, `Reakit` 等等。

## 组合模式的好处:组件复用性、可读性、有能力处理未来可能发生的变化

通过构建一些基础的原始组件，这些原始组件的使用场景可能覆盖整个应用，这样就不用拷贝、粘贴代码，提高组件的复用性。

通过组合模式，我们可以将组件的逻辑拆分成多个小组件，这样一来，每个小组件的逻辑就会变得简单，可读性也会提高。同时，这些小组件的逻辑也会变得更加稳定，因为它们的职责更加单一，更加专注。此外，组合模式可以解决`属性打洞（props drilling）`的问题，使得组件的逻辑更加清晰。

当我们的组件逻辑发生变化时，我们只需要改动部分涉及的小组件，改动范围更小，更容易维护，也共容易迭代。

## 在 React 中使用组合模式

通常，我们使用 `children` 属性来实现组合模式：

```jsx
const Parent = ({ children }) => {
  return <div>{children}</div>;
};
```

当然，我们命名一些更加语义化的属性名，并放在组件内部的特定位置，比如 `header`，`body`,`footer` 。

```jsx
const Parent = ({ header, body, footer }) => {
  return (
    <div>
      <div>{header}</div>
      <div>{body}</div>
      <div>{footer}</div>
    </div>
  );
};
```

但是这种方式，会使得组件声明不够美观：

```jsx
<Parent
  header={<Header />}
  body={<Body />}
  footer={<Footer />}
/>
//不如下面的直观
<Parent>
  <Header />
  <Body />
  <Footer />
</Parent>
```

因此组件库一般都是采用 `children` 属性来实现组合模式的。内部使用 React.Children.map 来遍历 `children`，然后对每个 `child` 进行处理，比如添加样式、添加事件等等。

```jsx
const Parent = ({ children }) => {
  return (
    <div>
      {React.Children.map(children, (child) => {
        return React.cloneElement(child, {
          style: { color: "red" },
        });
      })}
    </div>
  );
};
```

这里如果涉及父子组件的通信，对每个 `child` 的处理一般是通过 `React.cloneElement` 或者 包裹一层 `Context.Provider` 来实现的。具体参见[React 中父子组件通信的几种方式]()

## 组合模式与单体模式的对比

这里所说的 `单体模式` 是指按照自上而下的设计方式，将所有的组件逻辑放到一个大组件内部的这种组件设计模式。

任何一个复杂的组件都可以通过单体模式或者组合模式实现，我们常用的组件库 `antd` 就是用单体模式实现的大部分组件，而 `Radix` 组件库就是基于组合模式实现组件的，所以我们通过这两个库里一个常见的组件 `Tabs` 来看看不同的实现模式的对比。

## 声明的区别

antd 的 Tabs 组件的声明方式是这样的：

```jsx
const items = [
  {
    key: "1",
    label: "Tab 1",
    children: "Content of Tab Pane 1",
  },
  {
    key: "2",
    label: "Tab 2",
    children: "Content of Tab Pane 2",
  },
  {
    key: "3",
    label: "Tab 3",
    children: "Content of Tab Pane 3",
  },
];
const App = () => (
  <Tabs defaultActiveKey="1" items={items} onChange={onChange} />
);
```

我们可以看到，我们只需要声明一个 Tabs 组件，然后通过 `items` 属性来传递数据，这样的声明方式非常简洁，而且也很容易理解。

而 Radix 的 Tabs 组件的声明方式是这样的：

```jsx
import * as Tabs from "@radix-ui/react-tabs";

export default () => (
  <Tabs.Root defaultValue="tab1" orientation="vertical">
    <Tabs.List aria-label="tabs example">
      <Tabs.Trigger value="tab1">One</Tabs.Trigger>
      <Tabs.Trigger value="tab2">Two</Tabs.Trigger>
      <Tabs.Trigger value="tab3">Three</Tabs.Trigger>
    </Tabs.List>
    <Tabs.Content value="tab1">Tab one content</Tabs.Content>
    <Tabs.Content value="tab2">Tab two content</Tabs.Content>
    <Tabs.Content value="tab3">Tab three content</Tabs.Content>
  </Tabs.Root>
);
```

我们可以看到，Radix 的 Tabs 组件的声明方式更加复杂，需要声明多个组件，而且还需要为每个组件添加不同的属性，这样的声明方式不够直观，也不够简洁。

## 实现的区别

在 [这里](https://github.com/ant-design/ant-design/blob/master/components/tabs/index.tsx)可以查看 antd Tabs 组件的实现，其内部主要是调用了 `rc-tabs` 这个库来实现的。

可以看到 Tabs 透传了大量属性给 rc-tabs ,rc-tabs 本身接收了大量 props, 所以，其内部实现必然相当复杂。

接下来，我们看看 Radix Tabs 组件的实现。在 [这里](https://github.com/radix-ui/primitives/blob/main/packages/react/tabs/src/Tabs.tsx)查看源码

通过这个文件的 export,我们可以发现 Radix Tabs 组件被拆分成了 4 个小组件：Root、List、Trigger、Content。

Root 组件充当一个容器，用来整合各个子组件，通过提供一个 Provider 来共享组件的全局状态。

List 组件是 Tabs 标题的容器。负责标题的布局方向、是否循环等等

Trigger 组件是 Tabs 标题。负责标题的样式、选中态、禁用态、处理点击事件等等

Content 组件是 Tabs 内容。负责内容的样式、判断隐藏或展示、动画等等

我们可以看到，通过对组件的拆分，组件的交互逻辑也被分散到各个小组件中，每个小组件只负责自己相关的功能，彼此相对独立。

对于组件逻辑的拆分、分散，一定程度上，可以提高组件的可读性、可维护性。后续如果有需求变更，我们只需要改动相关的组件，而不需要改动整个组件。明确的改动范围，可以减少我们的心里负担。

## 样式的覆盖

单体组件模式下，一般可以通过 `className` 或 `style` 属性来修改最外层组件的默认样式，对于组件内部比较深层的组件，我们一般很难去改写它们的样式，除非作者将属性打洞暴露出来。

但是，组合模式下，样式的重写就相对容易了，只要每个子组件都支持 `className`属性，我们可以随便更改深层组件的样式。其实原理就是：`组合模式可以解决属性打洞问题`。

## 总结

单体模式，对于内部逻辑复杂的组件，其实是一种不太好的设计模式，它会导致组件的可读性、可维护性变差，后续的需求变更也会变得困难。但是他对于使用者来说，比较友好，使用者只需要关心一个组件，而不需要关心组件内部的实现细节，不需要想着如何去拼凑。

组合模式，代表着一种自下而上的组件设计模式，其实对于组件的设计者有着很高的编码能力要求。对于使用者来说，则需要多写一些代码来声明、搭配子组件来实现想要的功能。但是，这样的设计方式使得组件有着更灵活的使用场景，更好的可读性、可维护性，也更容易迭代。

对于一个喜欢 Tailwind CSS 的人来说，组合模式的组件无疑是更好的选择，因为它可以更好的支持 Tailwind CSS 的样式覆盖。所以很多 headless 组件库都是基于组合模式来实现的，比如：`Radix`, `Chakra UI`, `Reakit` 等等。
