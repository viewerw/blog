# 前言

相信大家在日常开发中，都会用到弹框组件，我们一般叫它`Modal`,`Alert`,`Dialog`等,一般如果我们使用 `antd` 这样的组件库，它们的`Modal` 组件除了可以声明式的使用外，也支持指令式的调用，类似`Modal.show()`这样的使用方式，如果我们用了自定义的`Modal`组件，该如何实现其指令式调用呢？下面我来带大家分析一下如何自己实现弹框的指令式调用，以及如何使用 `nice-modal-react` 这个库，优雅的管理你的弹框组件。

# 自己实现指令式调用

想象我们现在有一个自定义的 `Modal` 组件，它的使用方式如下：

```jsx
<Modal visible={visible} onClose={onClose} />
```

现在我们需要给它添加一个指令式调用的方法：show，可以通过 `Modal.show()` 来弹出这个弹框，那么我们该如何实现呢？

首先的想法是：我们写一个 `renderToBody` 的方法，调用 show 的时候，其实就是把 Modal 渲染到 body 上：

> 以下代码基于 React 18+

```jsx
function renderToBody(element) {
  const container = document.createElement("div");
  document.body.appendChild(container);
  const root = createRoot(container);
  root.render(element);

  function unmount() {
    root.unmount();
    if (container.parentNode) {
      container.parentNode.removeChild(container);
    }
  }
  return unmount;
}
```

然后实现 `Modal.show` 方法：

```jsx
Modal.show = (props) => {
  const unmount = renderToBody(
    <Modal
      {...props}
      visible
      onClose={() => {
        props.onClose();
        unmount();
      }}
    />
  );
  return unmount;
};
```

至此，我们实现了一个简单版本的 `Modal.show`,用起来，感觉也没啥没问题，直到我们想给 Modal 加上入场和退出动画，并且 Modal 内部的动画是基于 js（比如基于`react-spring` 库）实现的：

- Modal 的 visible prop 需要初始化为 false，然后变为 true 的时候，执行入场动画
- Modal 的 visible prop 从 true 变为 false 的时候，执行退出动画，动画结束后，执行 onAfterClose 回调, 在 onAfterClose 回调中，执行 unmount

这时候，我们就会发现，我们的 `Modal.show` 方法就不好用了，因为我们的 `Modal.show` 方法是直接把 Modal(visible=true) 渲染到 body 上，导致没有入场动画。而且在 onClose 中直接调用 unmount 会导致没有退出动画。这时，我们不得不改造一下 `Modal.show` 方法：

```jsx
Modal.show = (props) => {
  const Wrapper = forwardRef((_, ref) => {
    const [visible, setVisible] = useState(false);
    useEffect(() => {
      setVisible(true);
    }, []);
    function handleClose() {
      props.onClose?.();
      setVisible(false);
    }
    useImperativeHandle(ref, () => ({
      close: handleClose,
    }));

    function handleAfterClose() {
      props.afterClose?.();
      unmount();
    }

    return (
      <Modal
        {...props}
        visible={visible}
        onClose={handleClose}
        afterClose={handleAfterClose}
      />
    );
  });
  const ref = createRef();
  const unmount = renderToBody(<Wrapper ref={ref} />);
  const close = () => {
    ref.current?.close();
  };
  return {
    close,
  };
};
```

上面的代码主要做了三件事：

1. 创建 Wrapper 高阶组件，添加 visible 状态，visible 初始化为 false，然后在 useEffect 中，把 visible 设置为 true，这样就可以触发 Modal 的入场动画了
2. 在 onClose 中，把 visible 设置为 false，这样就可以触发 Modal 的退出动画了,在 handleAfterClose 中，执行 unmount，这样就可以在退出动画结束后，执行 unmount 了
3. 通过 forwardRef + useImperativeHandle，把 Wrapper 组件的 close 方法暴露出去，这样我们就可以在外部调用 Modal.show() 返回的对象的 close 方法，来关闭 Modal 了

到此，我们基本实现了带动画版本的 Modal.show ，其实这也是 `antd-mobile` 的 Modal.show 的实现方式，只不过，它的源码里还会考虑更多的细节问题，代码会比这个复杂一些。

所以，对于我们普通开发者来说，自己实现这样一套东西，还是有一定的难度的，而且还要考虑各种细节问题，那么有没有现成的工具库可以实现指令式调用弹框呢？答案是肯定的，下面我来介绍一下 `nice-modal-react` 这个库。

# nice-modal-react

这是 eBay 出品的一个基于 React Context 的全局弹框管理库，而且它还支持指令式的调用，下面我来带大家看一下它的使用方式：

## 安装

```bash
# with yarn
yarn add @ebay/nice-modal-react

# or with npm
npm install @ebay/nice-modal-react
```

## 创建自己的组件

使用 NiceModal，您可以轻松创建单独的 Modal 组件。这与创建普通组件相同，但通过 NiceModal.create 将其包装为高阶组件。例如，下面的代码显示了如何使用 Ant.Design 创建对话框：

```jsx
import { Modal } from "antd";
import NiceModal, { useModal } from "@ebay/nice-modal-react";

export default NiceModal.create(({ name }) => {
  // Use a hook to manage the modal state
  const modal = useModal();
  return (
    <Modal
      title="Hello Antd"
      onOk={() => modal.hide()}
      visible={modal.visible}
      onCancel={() => modal.hide()}
      afterClose={() => modal.remove()}
    >
      Hello {name}!
    </Modal>
  );
});
```

从代码中我们可以看到：

- useModal hook 返回的 modal 对象，包含了 visible 属性，show, hide, remove 方法，我们可以通过这些方法来控制弹框的显示和隐藏
- NiceModal.create 创建了一个高阶组件，这样保证了组件在它 visible 之前，不会被执行。

接下来，我们来看一下如何使用这个组件：

## 使用你的 Modal 组件

### 在适合的地方放置 NiceModal.Provider

NiceModal.Provider 是一个 React Context Provider，它需要在应用的根组件中放置，以便在整个应用中使用 NiceModal 组件。例如，下面的代码显示了如何在应用的根组件中放置 NiceModal.Provider：

```jsx
import NiceModal from "@ebay/nice-modal-react";
ReactDOM.render(
  <React.StrictMode>
    <NiceModal.Provider>
      <App />
    </NiceModal.Provider>
  </React.StrictMode>,
  document.getElementById("root")
);
```

如果使用 Next.js 的 app route,可以在 layout 中放置 NiceModal.Provider

```tsx
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={poppins.className}>
        <main>
          <NiceModalProvider>{children}</NiceModalProvider>
        </main>
      </body>
    </html>
  );
}
```

### 在任何组件内使用 NiceModal.create 创建的组件

通过 import 你的 modal 组件，然后使用它：

```jsx
import NiceModal from "@ebay/nice-modal-react";
import MyAntdModal from "./my-antd-modal"; // created by above code

function App() {
  const showAntdModal = () => {
    // Show a modal with arguments passed to the component as props
    NiceModal.show(MyAntdModal, { name: "Nate" });
  };
  return (
    <div className="app">
      <h1>Nice Modal Examples</h1>
      <div className="demo-buttons">
        <button onClick={showAntdModal}>Antd Modal</button>
      </div>
    </div>
  );
}
```

或者给你的 Modal 组件注册一个 id,然后使用这个 id 来调用：

```jsx
import NiceModal from "@ebay/nice-modal-react";
import MyAntdModal from "./my-antd-modal"; // created by above code

// If use by id, need to register the modal component.
// Normally you create a modals.js file in your project
// and register all modals there.
NiceModal.register("my-antd-modal", MyAntdModal);

function App() {
  const showAntdModal = () => {
    // Show a modal with arguments passed to the component as props
    NiceModal.show("my-antd-modal", { name: "Nate" });
  };
  return (
    <div className="app">
      <h1>Nice Modal Examples</h1>
      <div className="demo-buttons">
        <button onClick={showAntdModal}>Antd Modal</button>
      </div>
    </div>
  );
}
```

我们通常会把项目内所有 自定义 Modal 的注册放到一个单独的文件中，比如 `modals.js`，然后在项目的根组件中，引入这个文件，这样就可以在项目的任何地方，通过 id 来调用 Modal 了。

这种使用方式，可以解耦 Modal 组件和调用方。

### 通过 useModal hook 来使用 Modal

useModal 除了可以用在自己的 Modal 组件中，用来初始化弹框以外，还可以放在任意其他组件中，通过传入一个已注册的弹框 id,来调用弹框：

```jsx
import NiceModal, { useModal } from "@ebay/nice-modal-react";
import MyAntdModal from "./my-antd-modal"; // created by above code

NiceModal.register("my-antd-modal", MyAntdModal);
//...
// if use with id, need to register it first
const modal = useModal("my-antd-modal");
// or if with component, no need to register
const modal = useModal(MyAntdModal);

//...
modal.show({ name: "Nate" }); // show the modal
modal.hide(); // hide the modal
//...
```

### 声明式的使用 Modal，可取代 register

如果你不想使用 register 来注册你的 Modal 组件，你也可以通过在 Modal 组件上添加一个 id 属性，来声明式的注册你的 Modal 组件：

```jsx
import NiceModal, { useModal } from "@ebay/nice-modal-react";
import MyAntdModal from "./my-antd-modal"; // created by above code

function App() {
  const showAntdModal = () => {
    // Show a modal with arguments passed to the component as props
    NiceModal.show("my-antd-modal");
  };
  return (
    <div className="app">
      <h1>Nice Modal Examples</h1>
      <div className="demo-buttons">
        <button onClick={showAntdModal}>Antd Modal</button>
      </div>
      <MyAntdModal id="my-antd-modal" name="Nate" />
    </div>
  );
}
```

这种使用方式，你可以享受以下便利：

- 可以访问某个组件节点下的 React Context。
- 可以通过 props 传递参数

### 内置 promise 的使用方式

除了使用 props 与父组件中的模态交互之外，您还可以通过 Promise 来更轻松地做到这一点。这种使用场景一般常见于 Modal 组件内包含表单，需要在表单提交后，`resolve(表单结果)`。如果，我们需要弹出一个包含 Input 的弹框，来询问用户的年龄，然后根据用户输入的年龄，来执行不同的逻辑：

```jsx
NiceModal.show(UserAgeModal)
  .then((age) => {
    // 根据用户的年龄，执行不同的逻辑
  })
  .catch((err) => {
    // 用户取消了
  });
```

UserAgeModal 组件的实现如下：

```jsx
const PromiseModal = NiceModal.create(() => {
  const modal = useModal();
  const [age, setAge] = useState(0);
  const [error, setError] = useState(null);
  const handleOk = () => {
    modal.resolve(age);
    mode.hide();
  };
  const handleCancel = () => {
    modal.reject();
    mode.hide();
  };
  return (
    <Modal
      title="请输入您的年龄"
      visible={modal.visible}
      onOk={handleOk}
      onCancel={handleCancel}
    >
      <InputNumber
        value={age}
        onChange={(value) => {
          setAge(value);
        }}
      />
    </Modal>
  );
});
```

更多使用方式，可以参考 [nice-modal-react](https://github.com/eBay/nice-modal-react)的文档。

# 最后

`nice-modal-react` 解决了项目中弹框管理的问题，还支持指令式调用，支持 promise 化，还可以和大多数的弹框 UI 库配合使用，比如 `antd`，`antd-mobile`，`material-ui`,`shadcn` 等，非常方便，推荐大家使用。
