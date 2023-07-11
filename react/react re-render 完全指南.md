# react re-render 完全指南

很多人刚开始接触到 React 的时候，都会有一个误区，就是 React 会在每次 state 或者 props 发生变化的时候，重新渲染整个组件树。这是不对的 ，首先，组件的重新渲染不只是因为 state 或者 props 的变化导致，其次，也不会重新渲染整个组件树。React 会从发生变化的组件开始，递归地重新渲染整个组件子树。如果子树中的组件依赖变化的 state,那么这次渲染就可以称之为必要渲染，否则就是不必要的渲染。本文中会针对函数式组件讨论什么时候会触发重新渲染，以及如何避免不必要的渲染。

## 什么时候会触发重新渲染

在 React 中，有以下几种情况会触发组件的重新渲染：

1.  props 发生变化
2.  state 发生变化
3.  父组件重新渲染
4.  组件依赖的 context 发生变化

其实，这些情况寻根溯源都可以归结到 state 发生了变化。

首先，我们需要先知道 React 的渲染机制：如果一个组件内的 state 发生了变化，那么 React 会重新渲染这个组件，以及这个组件所依赖的子组件。这个过程是递归的，所以，如果父组件重新渲染，那么子组件也会重新渲染。

这时候有两种情况：

1. 子组件依赖父组件中的那个变化的 state, 他们之间必然是通过 props 传递这个数据的，那么 Props 内的值必然也会发生变化。所以如果仅仅观察子组件，就会得到一个错误的结论：子组件 props 中的某个值发生了变化，所以子组件重新渲染了。其实，子组件重新渲染是因为父组件重新渲染了，而不是因为 props 的某个值发生了变化。
2. 子组件不依赖父组件中的那个变化的 state，那么子组件的 props 中值不会发生变化，但是子组件还是会重新渲染。这是因为父组件重新渲染，导致子组件也会重新渲染。（除非子组件是一个纯组件，纯组件的特点是，如果它的 props 没有发生变化，那么它就不会重新渲染。可以使用`React.memo()`将组件变为纯组件）。

> Tips: 很多人可能会有疑问，为什么 React 中的父组件发生渲染一定会导致普通子组件的重新渲染呢？这是 React 的机制问题。在当前轮次的渲染中，React 会比较当前组件的 Props 对象和上一轮渲染中缓存下来的 Props 对象是否引用相等（oldProps === newProps），如果引用相等，那么 React 认为这个 Props 没有发生变化，就不会触发重新渲染。但是，如果引用不相等，那么 React 不会继续比较这两个 Props 对象的每一个属性是否相等，React 会直接触发组件的重新渲染。而对于 `memo`包裹的组件，则会浅比较 Props 对象的每一个属性是否相等， 如果发现某个属性不相等，那么 React 认为 Props 发生了变化，就会触发重新渲染。如果都相等，就不会触发重新渲染。那么，为什么 React 不使用类似 `memo` 组件的比较策略呢，因为大部分的组件渲染都是非常快速的，即使会触发非必要渲染，也不会对性能造成很大的影响。如果使用 `memo` 的比较策略，那么每次渲染都会多出一次 Props 对象的浅比较，这样反而会影响性能。

由此可见，前三种情况都是因为 state 发生了变化，加上 React 的渲染机制导致的重新渲染。而最后一种情况， context 的 value 变化也是由于 Provider 中 state 变化导致的，所以，这四种情况都是因为 state 发生了变化。

在进入下一节之前，我们先来回顾一点 JSX 的基础知识。

JSX 本质上是一种语法糖，它会被编译成 `React.createElement()` 方法的调用。`createElement()` 的返回值是一个对象，这个对象就是 React 的虚拟 DOM 对象，也就是 React Element。React 会根据这个对象来创建真实的 DOM 对象。React Element 有一个属性叫做 `type`，这个属性的值就是组件的类型，可以是一个字符串，也可以是一个函数。如果是字符串，那么这个组件就是一个原生组件，如果是函数，那么这个组件就是一个自定义组件。React 会根据这个属性来判断如何创建真实的 DOM 对象。另外还有一个属性叫做 `props`，这个属性的值就是组件的属性对象。React 会对比这个属性对象是否发生了变化，如果发生了变化，那么就会触发重新渲染。下面看两个例子：

```jsx
   // 原生组件
   <div className="foo" />
   // 自定义组件
   <Foo className="foo" />
```

babel 会将上面的代码编译成如下形式：

```js
// 原生组件
React.createElement("div", { className: "foo" });
// 自定义组件
React.createElement(Foo, { className: "foo" });
```

对应的 React Element 如下：

```js
    // 原生组件
    {
        type: "div",
        props: {
            className: "foo"
        },
        key: null,
        ...
    // 自定义组件
    {
        type: Foo,
        props: {
            className: "foo"
        },
        key: null
        ...
    }
```

React 会比较新旧两次渲染产生的 React Elements,如果 props 引用相等，则不会重新渲染(re-render),否则会 re-render。如果 type 不相等，则会 unmount 原来的组件，mount 新的组件。这样既会触发组件的 re-render，也会触发组件的卸载和挂载（re-mount)。此外 key 的变化也会导致组件的卸载和挂载。这就是 React 的渲染机制。

> re-mount 一定会导致 re-render，但是 re-render 不一定会导致 re-mount。他们是 React 执行中的不同阶段。re-render 代表 函数组件的执行，组件及其子树的 DOM 还是复用的。re-mount 代表组件及其子树 DOM 的重新创建和插入。它会触发 useEffect 回调的执行。

## 如何避免不必要的重新渲染

先来看一个会触发 re-render 的例子：

```jsx
const Child = () => {
  console.log("Child render");
  return (
    <div>
      <span>Child</span>
    </div>
  );
};

const Parent = () => {
  const [count, setCount] = useState(0);
  console.log("Parent render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      <Child />
    </div>
  );
};

export const App = () => {
  const [count, setCount] = useState(0);
  console.log("App render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      <Parent />
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render1-lwqy7r?file=/src/App.tsx)

当我们点击 App 组件中 button, 会看到三条日志，分别是 App render, Parent render, Child render。这是因为 App 组件的 state 发生了变化，导致 App 组件重新渲染，App 组件重新渲染，导致 Parent 组件重新渲染，Parent 组件重新渲染，导致 Child 组件重新渲染。这验证了之前讨论的 React 的渲染机制。

现在假设 Child 组件内有着复杂的计算逻辑，渲染组件会花费较长的时间。所以我们要优化 Child 的非必要渲染。

### 状态下放

从上面的例子，我们发现 Child 并不依赖 Parent 的任何状态，但是，点击 Parent 中的 button, count 发生了变化，导致 Parent 组件重新渲染，Child 组件也会重新渲染。我们观察 Parent 组件，发现 count 这个 state 只有

```jsx
    <span>count: {count}</span>
    <button onClick={() => setCount(count + 1)}>+</button>
```

这部分元素用到了，所以我们可以抽离一个新的 Count 组件，然后将 count state 下放到 Count 组件中，这样就可以避免不必要的渲染了。

```jsx
const Count = () => {
  const [count, setCount] = useState(0);
  return (
    <>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
    </>
  );
};

const Parent = () => {
  console.log("Parent render");
  return (
    <div>
      <Count />
      <Child />
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render1-fix1-8p84mt?file=/src/App.tsx)

React 有一条规则，就是组件的 state 应该尽可能的少，尽可能的下放。这样可以避免不必要的渲染。

但是如果我们的组件是一个复杂的组件，它的 state 有很多，我们不可能将所有的 state 都下放到子组件中，这时候我们就可以考虑别的优化方式了。既然无关的 state 无法下放，那么是否可以将 Child 组件提升到组件外面，即采用`组合`的方式。

### 组合与 memo

我们可以使用 `组合` 的方式重构 Parent 组件，将 Child 组件提取出来，这样就可以避免不必要的渲染了。

```jsx
const Parent = (props) => {
  const [count, setCount] = useState(0);
  console.log("Parent render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      {props.children}
    </div>
  );
};

export const App = () => {
  const [count, setCount] = useState(0);
  console.log("App render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      <Parent>
        <Child />
      </Parent>
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render1-fix2-2rycc9?file=/src/App.tsx)

或者使用 `React.memo` 来包裹 Child 组件，这样也可以避免不必要的渲染。

```jsx
const Child = React.memo(() => {
  console.log("Child render");
  return (
    <div>
      <span>Child</span>
    </div>
  );
});
```

[code sandbox](https://codesandbox.io/s/re-render1-fix3-ftj62v?file=/src/App.tsx)

这样我们点击 Parent 组件中的按钮，只会看到一条日志， 'Parent render'。Child 组件不会重新渲染了。
从第一节的 tips 中，我们知道只有两种方法去优化 re-render,一种是保证 props 引用相等，一种是使用 `React.memo`。所以组合的方式一定是保证了新旧 Props 引用相等。那么，组合的方式是如何保证新旧 Props 的引用相等的呢？

当我们点击按钮触发 Parent 组件重新渲染的时候，Parent 组件会重新执行，对于非组合的方式，这时候 Parent 组件内部的 Child 组件会被重新创建，得到一个新的 React Element 对象，它的 props 属性也是新创建的，所以 Child 组件的 Props 引用会发生变化。但是，我们使用组合的方式，将 Child 组件提取出来，Child 组件的创建只会发生在 App 组件发生渲染的时候，Parent 组件从 props.children 中得到的 Child 的 Element 对象是复用之前创建好的对象，这样就保证了 Child 新旧 Props 的引用相等。说白了，组合方式下， Child 组件的 Element 创建(React.createElement(Child)的执行)只会发生在 App 组件发生渲染的时候，而不会发生在 Parent 组件发生渲染的时候。

到这里我们知道了两种`解决re-render的方法，一种是组合，一种是使用 React.memo。`那么，这两种方法该如何选择呢？我们先来看看上面例子的一些变种情况。

#### Child 组件的 props 依赖于 Parent 组件的 state

```jsx
const Child = React.memo((props) => {
  console.log("Child render");
  return (
    <div>
      <span>Child: {props.count}</span>
    </div>
  );
});

const Parent = () => {
  const [count, setCount] = useState(0);
  const [count1, setCount1] = useState(0);

  console.log("Parent render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      <Child count={count1} />
    </div>
  );
};
```

我们会发现当 Child 组件依赖 Parent 中 state 的时候，使用组合的方式很难在父子组件间传递数据。我们可以得到一个结论：`当子组件的 props 依赖于父组件的 state 的时候，使用 React.memo 会更加方便。否则，使用 组合 性能更好。`上面例子中，Child 组件的属性只有一个 count，而且是基本数据类型,`如果 Props 中有引用类型和函数，那么还要配合使用 useCallback 和 useMemo 来优化,以保证浅比较Props的每个字段引用相等`

```jsx
const Child = React.memo((props) => {
  console.log("Child render");
  return (
    <div style={props.style}>
      <span>Child: {props.count}</span>
      <button onClick={props.onClick}>+</button>
    </div>
  );
});

const Parent = () => {
  const [count, setCount] = useState(0);
  const [count1, setCount1] = useState(0);

  const style = useMemo(
    () => ({ color: count % 2 === 0 ? "red" : "blue" }),
    [count]
  );

  const onClick = useCallback(() => {
    console.log("click");
  }, []);

  console.log("Parent render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      <Child count={count1} style={style} onClick={onClick} />
    </div>
  );
};
```

### useMemo

组合是一种 React 中的组件组织架构，避免 re-render 只是它的一种使用方式，它还有许多其他的用途：用来拆分巨石组件、提高组件的可复用性、提高组件的可测试性，解决属性透传（Props drilling）等等。这里不做过多的讨论，我们这里只讨论组合的一种用法：避免不必要的渲染。它的原理是复用 React Element 对象，避免创建新的 React Element 对象，从而避免不必要的渲染。那么，我们是否可以使用 useMemo 来达到同样的效果呢？

```jsx
const Child = () => {
  console.log("Child render");
  return (
    <div>
      <span>Child</span>
    </div>
  );
};

const Parent = () => {
  const [count, setCount] = useState(0);
  console.log("Parent render");

  const child = useMemo(() => <Child />, []);
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      {child}
    </div>
  );
};

export const App = () => {
  const [count, setCount] = useState(0);
  console.log("App render");
  return (
    <div>
      <span>count: {count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
      <Parent />
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render-usememo-nf4cn5?file=/src/App.tsx)

当我们点击 Parent 组件中的按钮，发现 Child 组件不会重新渲染，点击 App 组件里的按钮，Child 组件也不会重新渲染。虽然原理都是复用 React Element 对象，但是它的行为却和 `React.memo()` 很相似。如果 Child 组件有 Props，我们需要吧依赖的 state 放到 useMemo 的依赖数组里面，这样才能保证 Child 组件在必要的时候可以更新：

```jsx
const child = useMemo(() => <Child count={count} />, [count]);
```

但是这样的写法并非主流，因为它的可读性太差了，而且还需要考虑到依赖数组的问题，如果依赖数组写的不对，就会导致 Child 组件不会更新。所以，我们还是推荐使用 React.memo() 来避免不必要的渲染。

### 列表渲染优化相关

我们在渲染列表的时候，React 为了性能优化，会让我们给每个子项添加 key 值，如果 key 没变，那么 React 就会复用这个子项。如下代码：

```jsx
const Child = () => {
  console.log("Child render");

  useEffect(() => {
    console.log("Child mount");
  }, []);

  return (
    <div>
      <span>Child</span>
    </div>
  );
};
const list = [1, 2, 3, 4, 5];
const Parent = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>+</button>
      {list.map((item) => (
        <Child key={item} />
      ))}
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render-list1-kd4qzl?file=/src/App.tsx)

上面例子中的 list 是固定的数据，所以 Child 组件的 key 在每次渲染中都不会变化，但是当我们点击 button 触发 Parent 组件重新渲染的时候，发现 `Child render` 会被打印 5 次。那我们可能会产生疑惑，key 没变，为什么还会重新渲染呢？React 中的 key 到底用来优化什么的？

`key 是用来优化 DOM 的，不是用来优化 re-render 的，如果 key 不变，而且组件的 type 也不变， 那么 React 会复用 DOM 节点，从而提高性能。`所以，`Child mount` 这条日志并没有被打印。

> 如果查看 React 源码，就会发现在 reconcile 阶段，React 会根据 key 和 type 来直接复用 Fiber Node,Fiber Node 中的 stateNode 就是 DOM 节点，所以 key 的作用是优化 DOM 节点的复用。

所以，如果要优化 re-render，我们需要使用 React.memo() 来包裹子组件，这样才能避免不必要的渲染：

```jsx
const ChildMemo = React.memo(Child);

const Parent = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>+</button>
      {list.map((item) => (
        <ChildMemo key={item} />
      ))}
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render-list1-fix-p9tt3w?file=/src/App.tsx)

假如我们是 React 新手，可能会写出这样的代码：

```jsx
const Parent = () => {
  const [count, setCount] = useState(0);

  const Child = () => {
    console.log("Child render");

    useEffect(() => {
      console.log("Child mount");
    }, []);

    return (
      <div>
        <span>Child</span>
      </div>
    );
  };

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>+</button>
      {list.map((item) => (
        <Child key={item} />
      ))}
    </div>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render-list2-jyqmqp?file=/src/App.tsx)

我把 Child 组件的定义放到了 Parent 组件的内部，这时候，点击 button 触发 Parent render,发现 `Child render` 和 `Child mount` 都被打印了，但是 key 并没有变化，为啥 Child 会 re-mount 呢？

这是因为， Child 函数在 Parent render 的时候重新创建了，所以`<Child key={item}/>`对应的 React Element 中的 type 属性发生了变化，所以 React 会创建新的 Fiber Node，所以 Child 组件会 re-mount。

我们应该尽量避免组件 re-mount ,因为 re-mount 会导致组件内部的 state 丢失，由于 DOM 的重新创建和插入，也会导致一些表单元素的光标位置丢失，页面闪烁，useEffect 重新执行等问题。

总结：` type 和 key 变化会触发 re-mount,而 props 变化会触发 re-render。`

### Context 优化相关

#### 避免 value 变化

我们在使用 Context 的时候，一般会这样写：

```jsx
const ThemeContext = React.createContext({
  theme: "light",
  setTheme: () => {},
});

const Toolbar = React.memo(() => {
  const { theme, setTheme } = useContext(ThemeContext);
  console.log("Toolbar render");
  return (
    <div>
      <button onClick={() => setTheme("dark")}>dark</button>
      <button onClick={() => setTheme("light")}>light</button>
    </div>
  );
});

const Btn = (props) => {
  console.log("Btn render");
  return <button onClick={props.onClick}>{props.count}</button>;
};
const App = () => {
  // App 内部的 state
  const [count, setCount] = useState(0);
  // context 相关state
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Btn count={count} onClick={() => setCount(count + 1)} />
      <Toolbar />
    </ThemeContext.Provider>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render-context1-kwm63z)

上面的例子中，存在两种 re-render 问题：

1. 当我们点击 Btn 触发 App render 的时候，发现 `Toolbar render` 会被打印，但是 Toolbar 依赖的 theme state 并没有变化，Toolbar 也被 memo 优化了，为什么还会重新渲染呢？
2. 当我们点击 dark 或者 light 按钮的时候，发现 `Btn render` 会被打印，但是 Btn 并不依赖 ThemeContext,它没必要重新渲染。

修改代码解决这两个问题：

```jsx
const ThemeContext = React.createContext({
  theme: "light",
  setTheme: () => {},
});

const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState("light");

  const value = useMemo(() => {
    return { theme, setTheme };
  }, [theme]);
  return (
    <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
  );
};

const App = () => {
  const [count, setCount] = useState(0);

  return (
    <ThemeProvider>
      <Btn count={count} onClick={() => setCount(count + 1)} />
      <Toolbar />
    </ThemeProvider>
  );
};
```

[code sandbox](https://codesandbox.io/s/re-render-context1-fix-vqjj2l?file=/src/App.tsx)

现在，当我们点击 Btn 触发 App render 的时候，发现 `Toolbar render` 不会被打印了，当我们点击 dark 或者 light 按钮的时候，发现 `Btn render` 也不会被打印了。想一想，我们是怎么通过使用 `组合`和 `useMemo` 来解决这两个问题的。

#### 拆分 Context

1. 如果 Context value 中包含了多个 state，那么我们可以把它们按照相关性拆分成多个 Context，这样可以避免不必要的 re-render。

```jsx
const Component = () => {
  const [data, setData] = useState();
  const [data1, setData1] = useState();
  return (
    <DataContext.Provider value={data}>
      <Data1Context.Provider value={data1}>
        <Child />
      </Data1Context.Provider>
    </DataContext.Provider>
  );
};
```

2. 将 Context 中变与不变的部分拆分成两个 Context，比如将 data 和 api 拆分成两个 Context，这样可以避免不必要的 re-render。

```jsx
const Component = () => {
  const [data, setData] = useState();

  return (
    <DataContext.Provider value={data}>
      <ApiContext.Provider value={setData}>
        <Child />
      </ApiContext.Provider>
    </DataContext.Provider>
  );
};
```

### 高阶组件实现 Context Selector

如果我们想实现一个类似于 redux 的 useSelector 的功能，只有在 Context 中的某个值发生变化的时候，才会触发 re-render，我们可以通过高阶组件来实现：

```jsx
const withSelector = (selector) => (Component) => {
  const MemoComponent = React.memo(Component);
  return (props) => {
    const value = useContext(selector.context);
    const selectedValue = selector.selector(value);
    return <MemoComponent {...props} {...selectedValue} />;
  };
};
```

withSelector(selector) 返回了一个 高阶组件，这个高阶组件在 selector.context 发生变化的时候，会触发 re-render，然后通过 selector.selector 来选择需要的值，然后传递给 Component 的 memo 版本，所以 Component 只在需要的值变化后，才会 re-render。

## 总结

1. React Element 对象的属性中，type 和 key 变化会触发 re-mount,而 props 变化会触发 re-render。
2. 可以通过使用 `组合`、`useMemo`、`memo` 来避免不必要的 re-render,`memo`可能要搭配 `useCallback`和`useMemo` 来使用。
3. 合理的设计和使用 Context，可以避免不必要的 re-render。
4. React 很快，不要盲目的使用 `useMemo`、`memo`、`useCallback` 来优化，只有在性能问题出现的时候，才需要考虑优化。因为这些性能优化的手段往往也带来了一些性能负担。
5. 如果你发现页面渲染卡顿，或许在解决 re-render 问题之前，你应该先花点时间，找出那个渲染缓慢的组件，看看能不能让它渲染的快一点。
