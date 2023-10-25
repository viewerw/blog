## 前言

一般情况下，子组件通过 props 接受父组件的状态，父组件通过 props 中的回调函数被动接收子组件的状态。但是有些情况下，由于子组件声明的位置不同，父子组件无法传递 props，这时候该如何通信呢？此外，父组件如果想主动获取子组件的当前内部状态，又该如何做呢？本文将带你探索父子组件间传递数据的诸多方式。

## 区分父子组件在组件树上的位置和声明的位置

当我们谈论父子组件的父子关系时，其实指的是两个组件在组件树上的位置关系，而不是指的两个组件的声明的位置关系。比如下面这个例子：

```jsx
const App = () => {
  return (
    <div>
      <Parent>
        <Child />
      </Parent>
    </div>
  );
};

const Parent = ({ children }) => {
  const [count, setCount] = useState(0);
  return <div>{children}</div>;
};

const Child = ({ count }) => {
  return <div>count:{count}</div>;
};
```

在这个例子中，从组件树（结构类 Fiber 树、DOM 树）的角度，Parent 组件是 Child 组件的父组件，Child 组件是 Parent 组件的子组件。但是，Parent 组件和 Child 组件的声明位置是一样的，都是在 App 组件内部。

接下来我们来理解一下组件的声明，在 jsx 中声明一个组件其实就是对 React.createElement()的一次调用，`<Child count={count} />` 等同于 `React.createElement(Child, { count })`。

假设 Child 是一个计数器组件，它需要从 Parent 中获取计数器的初始值， 那么在上面这种代码结构下， Child 组件在声明的时候无法初始化自己的 props,因为它不能访问不属于自己作用域的值（不能访问 Parent 内的 state,因为不在作用域链上）。

```jsx

const App = () => {
  return (
    <div>
      <Parent>
        <Child count={/*?*/} />
      </Parent>
    </div>
  );
};
```

所以看起来，如果 Child 组件在 Parent 组件的外部声明，那么父子组件就无法传递数据了。但是，其实还是有一些办法的，下边我们分情况细致的讨论一下。

## 子组件声明在父组件内部

如果子组件 Child 在父组件 Parent 内部声明，那么父子组件间可以方便的通过 Props 来通信：

```jsx
const Parent = () => {
  const [count, setCount] = useState(0);
  return (
    <div>
      <Child value={count} onChange={(value) => setCount(value)} />
    </div>
  );
};

const Child = ({ value, onChange }) => {
  return (
    <div>
      <button onClick={() => onChange(value + 1)}>+</button>
      <span>{value}</span>
    </div>
  );
};
```

上面例子中的 Child 是个受控组件， Child 没有自己的内部状态，它的状态由父组件 Parent 控制，父组件通过 props 传递给子组件一个回调函数 onChange，子组件通过调用 onChange 来通知父组件自己的状态变化。

现在我们把 Child 改写成非受控组件：

```jsx
const Child = ({ initialValue, onChange }) => {
  const [value, setValue] = useState(initialValue);

  const handleChange = (value) => {
    setValue(value);
    onChange(value);
  };
  return (
    <div>
      <button onClick={() => handleChange(value + 1)}>+</button>
      <span>{value}</span>
    </div>
  );
};
```

这样， Child 组件有了自己的内部状态， Parent 组件还是可以通过 onChange 回调函数来接收 Child 组件的状态变化。但是如果 Parent 组件内部其他方法需要用到 Child 的状态，那么就需要把 Child 的状态提升到 Parent 组件内部，这样就又变成了受控组件。（关于组件设计成受控还是非受控，这里不做过多讨论，后面可能会写篇文章详细讨论）或者，在 Parent 组件内部，用 useState 新建一个 state,并在 onChange 回调里同步最新的 state，这样或造成状态的冗余。此外，还可以用 ref 来主动获取 Child 组件的状态（后面会简单介绍）。

再来看一个子组件声明在父组件内部的特殊例子：

```jsx
//app.jsx
const App = () => {
  return <Parent child={Child} />;
};
//parent.jsx
const Parent = ({ child }) => {
  const Child = child;
  const [count, setCount] = useState(0);
  return <Child count={count} />;
};
//child.jsx
const Child = ({ count }) => {
  return <div>count:{count}</div>;
};
```

通过 Props 传递组件 Functon,这种方式有什么好处呢？ 首先这是一种控制反转的体现，解除 Parent 和 Child 在代码层面的耦合（不必在 parent.jsx 文件内部 import child.jsx），方便 Parent 组件可以拆分自己逻辑，将一部分的逻辑/UI 渲染的控制权释放出去，这也是分解巨石组件的一种方式，同时保证自身的多态性。其次，这种方式可以让方便 Child 组件获取到 Parent 组件的内部状态，因为声明 Child 仍然发生在 Parent 组件内部。

## 子组件声明在父组件外部

接下来我们看看子组件声明在父组件外部的情况，通常，这种情况是在组件组合模式下的产生的。这种情况下，父组件无法通过 props 传递数据给子组件。

还是用之前的例子：

```jsx
const App = () => {
  return (
    <div>
      <Parent>
        <Child />
      </Parent>
    </div>
  );
};
```

在组合模式下，我们有三种方式可以建立父子组件间的通信：

### React.cloneElement

React.cloneElement 可以克隆一个 React Element，并且可以为克隆出来的 React Element 添加 props。这样一来，我们就可以在父组件中为子组件添加 props 了。

```jsx
const App = () => {
  return (
    <Parent>
      <Child />
    </Parent>
  );
};
const Parent = ({ children }) => {
  const [count, setCount] = useState(0);
  return React.cloneElement(children, { count });
};

const Child = ({ count }) => {
  return <div>count:{count}</div>;
};
```

通过 React.cloneElement,我们破坏了 React 自顶向下的数据流，在 React 运行时中动态添加了 props,这样会使得代码的可读性变差，同时也会使得代码的维护变得困难。此外，还会引发组件定义时的 Props 类型和声明时的 Props 类型不一致的问题，这个问题在 TypeScript 中会更加明显。因此，这种方式官方并不推荐使用。但是这种方式在一些组件库中有着大量使用场景。

### Context

我们可以使用 createContext 创建一个 CountContext :

```jsx
const CountContext = createContext(0);
```

然后在 Parent 组件中使用 CountContext.Provider 来包裹子组件：

```jsx
const Parent = ({ children }) => {
  const [count, setCount] = useState(0);
  return (
    <CountContext.Provider value={count}>{children}</CountContext.Provider>
  );
};
```

最后在 Child 组件中使用 useContext 来获取父组件传递的数据：

```jsx
const Child = () => {
  const count = useContext(CountContext);
  return <div>count:{count}</div>;
};
```

这样我们就可以在父组件中向子组件传递数据了：

```jsx
const App = () => {
  return (
    <Parent>
      <Child />
    </Parent>
  );
};
```

这种方式不会有类型问题，但还是增加了组件间的耦合，维护性变差，这种方式也在组件库中大量使用。

### render prop

render prop 是一种组件复用的方式，它的核心思想是将组件的渲染逻辑作为一个函数传递给组件，组件内部通过调用这个函数来渲染内容。这种方式可以让父组件向子组件传递数据，同时又不会增加组件间的耦合。

```jsx
const App = () => {
  return <Parent>{(count) => <Child count={count} />}</Parent>;
};

const Parent = ({ children }) => {
  const [count, setCount] = useState(0);
  return children(count);
};
```

这里我们把 children 属性当成了 " render prop "。`render prop`是一个函数，它接受父组件的状态作为参数，返回一个 React Element。这样一来，父组件就可以向子组件传递数据了。这种方式不会增加组件间的耦合，也不会破坏 React 自顶向下的数据流，是一种比较好的组件通信方式。

## forwardRef + useImperativeHandle

forwardRef 可以让组件有 ref prop,然后父组件通过 ref.current 来获取子组件的实例。useImperativeHandle 可以让我们在子组件中暴露一些方法给父组件调用。

通过这个组合，我们可以在父组件中主动（命令式）的获取子组件的状态。

比如，我们有个子组件是一个计数器，我们希望父组件可以主动获取到子组件的当前计数器的值，那么我们可以这样做：

```jsx
const Child = forwardRef((props, ref) => {
  const [count, setCount] = useState(0);
  useImperativeHandle(ref, () => ({
    getCount: () => count,
  }));
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>+</button>
      <span>{count}</span>
    </div>
  );
});
```

在父组件中获取子组件的状态：

```jsx
const Parent = () => {
  const childRef = useRef(null);
  const handleClick = () => {
    alert(childRef.current.getCount());
  };
  return (
    <div>
      <Child ref={childRef} />
      <button onClick={handleClick}>获取计数值</button>
    </div>
  );
};
```

这里只是提出这样一种思路，实际上，这种方式并不推荐使用，因为它会使得父组件和子组件耦合在一起，父组件需要知道子组件的内部实现，这样会使得父组件的可维护性变差。

这种方式，一般用于子组件内部的一些 DOM 相关操作，比如 input 的聚焦、视图的滚动等等。如果某些操作可以通过 props 实现，那么就不要使用这种方式。在子组件内部通过 useEffect 来监听 props 的变化，然后执行相应的操作，可以获得类似命令式操作的效果，而且，这样可以使得子组件更加灵活，更加易于维护。

## 总结

本文介绍了父子组件间传递数据的几种方式，每种方式都有自己的适用场景。在我们日常开发中，如果子组件声明在父组件的外部，我们应该优先考虑使用`render prop` 或者 `context` 去传递数据，这样可以避免父子组件间的耦合，同时也不会破坏 React 自顶向下的数据流。如果子组件声明在父组件的内部，那么我们可以优先考虑使用 props 来传递数据，这样可以使得代码更加清晰，更加易于维护。
