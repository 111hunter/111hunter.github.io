# 构建自己的 mini React


现在，我们遵循 React 16.8 的代码体系结构，一步一步地构建我们自己的 mini React。

## 基础回顾

开始之前，我们先回顾 React 是怎么将 JSX 转换成 DOM 节点的：

```jsx
const element = <h1 title="foo">Hello</h1>
const container = document.getElementById("root")
ReactDOM.render(element, container)
```

第一行使用 JSX 来创建元素，但 JSX 不是有效的 JS 代码。React 用 Babel 将 JSX 代码转换为原生 JS 代码。转换过程就是调用 createElement 函数，并将 JSX 的元素类型、props 属性和 childen 元素作为参数依次传入：

```jsx
const element = <h1 title="foo">Hello</h1>
// Babel 调用 createElement 函数完成转换
const element = React.createElement(
  "h1",
  { title: "foo" },
  "Hello"
)
// createElement 根据参数生成 object
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
}
```

这就是 React 元素的本质，包含 type 和 props 属性的对象(还有其他属性，我们只关注这两个)。现在我们就能用 element 生成 DOM 节点了。

```js
​const container = document.getElementById("root")
​
const node = document.createElement(element.type)
node["title"] = element.props.title
​
const text = document.createTextNode("")
text["nodeValue"] = element.props.children
​
node.appendChild(text)
container.appendChild(node)
```

## 渲染阶段

### createElement

现在我们来实现自己的 createElement 函数。注意一个细节，JSX 叶子节点可能是基本数据类型。我们把它包装为对象，统一数据类型(React 不会包装基本类型值或创建空数组，我们这样做是为了数据判断和修改的方便)。

```js
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object" ? child : createTextElement(child)
      )
    }
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: []
    }
  };
}
```

### render

创建 DOM 节点并添加元素的 props 属性。

```js
function render(element, container) {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type);
  const isProperty = key => key !== "children";
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name];
    });
  element.props.children.forEach(child => render(child, dom));
  container.appendChild(dom);
}
```

现在将我们的库取名 Didact，让 Babel 调用我们的库转换 JSX。

```jsx
const Didact = {
  createElement,
  render
};

/** @jsx Didact.createElement */
const element = (
  <div style="background: salmon">
    <h1>Hello World</h1>
    <h2 style="text-align:right">from Didact</h2>
  </div>
);
const container = document.getElementById("root");
Didact.render(element, container);
```
这样就实现了 JSX 的转换和渲染，在 [codesandbox](https://codesandbox.io/s/didact-2-k6rbj) 中试试看。

### Concurrent

在我们加入更多特性之前我们先对代码进行一次重构。因为递归调用存在一个问题：渲染开始就会一直阻塞主进程。如果浏览器需要处理一些高优先级的任务(像用户输入或者保持动画流畅)，需要等到所有元素渲染完成之后再进行处理，这是不好的用户体验。

```js
let nextUnitOfWork = null
​
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    // 迭代子任务
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}
​// 主进程空闲时才会调用回调函数
requestIdleCallback(workLoop)
​// 执行当前子任务并返回下一个子任务
function performUnitOfWork(nextUnitOfWork) {
  // TODO
}
```

现在我们拆分整个任务为一个个小的子任务，浏览器可以在执行完每个小任务后中断渲染流程去处理其他事情。因为我们使用 `reqeustIdleCallback` 来创建一个循环任务(React 现在使用 [scheduler](https://github.com/facebook/react/tree/master/packages/scheduler))，它在主进程空闲时才会执行回调函数，它为回调函数提供一个 deadline 参数，据此我们可以知晓还剩多少时间浏览器会拿回控制权。

### Fibers

我们使用 fiber 树连接所有子任务，为每个元素创建一个 fiber，每个 fiber 对应一个子任务。假如我们渲染如下一颗元素树：

```jsx
Didact.render(
  <div>
    <h1>
      <p />
      <a />
    </h1>
    <h2 />
  </div>,
  container
)
```

生成对应的 fiber 树：

<div align=center><img src="/img/fiber-tree.png" width="35%"></div>

在渲染中，我们将 container 创建为 root 并将其设置为 `nextUnitOfWork`。而元素的 fiber 由 `performUnitOfWork` 生成，我们将为每个 fiber 做三件事：

- 1.将元素添加到 DOM 中。
- 2.为元素的子元素创建 fiber。
- 3.寻找下一个子任务。

现在我们从 render 中提取出创建 DOM 节点的逻辑，稍后我们会使用它。

```js
function createDom(fiber) {
  const dom = fiber.type == "TEXT_ELEMENT"? 
    document.createTextNode(""): 
    document.createElement(fiber.type)
  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })
​
  return dom
}
```

在 render 函数中，我们将 `nextUnitOfWork` 的 DOM 属性设置为 container。

```js
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  }
}
​
let nextUnitOfWork = null
```

接下来在 `performUnitOfWork` 中完成每个 fiber 的三件事。

```js
function performUnitOfWork(fiber) {

  // 需要与父节点的 DOM 连接时才创建 DOM 节点
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
​   
  // 为子元素创建 newFiber，dom 属性为空
  const elements = fiber.props.children
  let index = 0
  let prevSibling = null
​
  while (index < elements.length) {
    const element = elements[index]
​
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }
   
    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }
​
    prevSibling = newFiber
    index++
  }
  
  // 寻找下一个子任务，优先级依次是子节点、兄弟节点、叔叔节点。
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
```

### Commit

处理元素时我们每次向 DOM 添加一个新节点。但浏览器是会中断渲染过程的，这样用户会看到不完整的 UI。我们怎么避免这种情况呢？答案是重构操作 DOM 的代码。

首先在删除 `performUnitOfWork` 中添加 DOM 节点的代码：

```js
// if (fiber.parent) {
//   fiber.parent.dom.appendChild(fiber.dom)
// }
```

然后在 render 中用 wipRoot 保存 fiber root 节点。

```jsx
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}
​
let nextUnitOfWork = null
let wipRoot = null
```

直到本次全部元素渲染结束时，我们才将整个 fiber 树提交到 DOM 中。

```js
function commitRoot() {
  // 将所有元素的 fiber 递归附加到 DOM 
  commitWork(wipRoot.child)
  wipRoot = null
}
​
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}

function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  } 
​ // 直到没有下一个子任务，将整个 fiber 树提交到 DOM 节点中
  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
  }
  requestIdleCallback(workLoop)
}
​// 主进程空闲时才会调用回调函数
requestIdleCallback(workLoop)
```

## 更新阶段

### Reconciliation

到目前为止，我们只是将元素添加到了 DOM 中，但是我们怎么去更新或者删除节点呢？(diff)

在每个 fiber 中添加 alternate 属性保存上一次提交到 DOM 中的 fiber。先在 wipRoot 中添加：

```js
function commitRoot() {
  commitWork(wipRoot.child)
  // 渲染结束时存储当前的 fiber root
  currentRoot = wipRoot
  wipRoot = null
}
​
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  }
  nextUnitOfWork = wipRoot
}
​
let nextUnitOfWork = null
// 增加 currentRoot 用于保存当前提交到 DOM 中的 fiber 树
let currentRoot = null
let wipRoot = null
```

将 `performUnitOfWork` 中创建新 fiber 的逻辑移到 reconcileChildren 函数中，给每个子 fiber 添加 alternate 和 effectTag 属性(effectTag 属性用于 Commit 阶段)：

```js
// 协调(比较和复用)当前 fiber 的所有子 fiber
function reconcileChildren(wipFiber, elements) {
  let index = 0
  let oldFiber = wipFiber.alternate && wipFiber.alternate.child
  let prevSibling = null
  
  while ( index < elements.length || oldFiber != null ) {
    const element = elements[index]
    let newFiber = null
    const sameType = oldFiber && element && element.type == oldFiber.type
    // 需要添加新的 fiber
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null, 
        effectTag: "PLACEMENT",
      }
    }
    // 需要更新原来的 fiber
    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        // oldFiber 被替换时才需要用 alternate 保存
        alternate: oldFiber, 
        effectTag: "UPDATE",
      }
    }
    // 需要删除原来的 fiber
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION"
    }
    if (oldFiber) {
      oldFiber = oldFiber.sibling
    }
    if (index === 0) {
      wipFiber.child = newFiber
    } else if (element) {
      prevSibling.sibling = newFiber
    }
    prevSibling = newFiber
    index++
  }
}
```

### Commit

现在我们需要修改 commitWork 函数完成 DOM 的修改。

```js
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  if (
    fiber.effectTag === "PLACEMENT" &&
    fiber.dom != null
  ) {
    domParent.appendChild(fiber.dom)
  } else if (
    fiber.effectTag === "UPDATE" &&
    fiber.dom != null
  ) {
    updateDom(
      fiber.dom,
      fiber.alternate.props,
      fiber.props
    )
  } else if (fiber.effectTag === "DELETION") {
    domParent.removeChild(fiber.dom)
  }
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```

这里的 updateDom 就是用来更新 DOM 节点的属性。

```js
const isProperty = key => key !== "children" && !isEvent(key)
const isEvent = key => key.startsWith("on")
const isNew = (prev, next) => key => prev[key] !== next[key]
const isGone = (prev, next) => key => !(key in next)

function updateDom(dom, prevProps, nextProps) {
  // 删除旧的属性
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach(name => {
      dom[name] = ""
    })
  // 删除旧的事件监听
  Object.keys(prevProps)
    .filter(isEvent)
    .filter(
      key =>
        !(key in nextProps) ||
        isNew(prevProps, nextProps)(key)
    )
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.removeEventListener(
        eventType,
        prevProps[name]
      )
    })
  // 设置新的属性
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      dom[name] = nextProps[name]
    })
  // 添加新的事件监听
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.addEventListener(
        eventType,
        nextProps[name]
      )
    })
}
```

在 [codesandbox](https://codesandbox.io/s/didact-6-96533) 中查看完整代码。

## 函数组件

### Commit

现在，我们考虑在已有代码的基础上，增加对函数组件和 Hooks 的支持。看这个函数组件：

```jsx
/** @jsx Didact.createElement */
function App(props) {
  return <h1>Hi {props.name}</h1>
}
const element = <App name="foo" />
const container = document.getElementById("root")
Didact.render(element, container)
```

如果我们将 jsx 转换为 js，Babel 的解析方式会是这样：

```js
function App(props) {
  return Didact.createElement(
    "h1",
    null,
    "Hi ",
    props.name
  )
}
// 这里虽然会调用 createElement，但并不会调用 App 获取子元素
const element = Didact.createElement(App, {name: "foo"}, )
```

观察 Babel 的解析后会发现：

- 我们不能为函数 App 创建 DOM 节点，因为没有 html 标签，只能渲染它的子元素。
- 子元素不会通过 createElement 的第三个参数传递，子元素需手动调用函数获取。

为函数创建子 fiber 我们可以这样做：

```js
function updateFunctionComponent(fiber) {
  // 由于 Babel 调用 createElement，得到的 fiber.type 就是函数名
  const children = [fiber.type(fiber.props)]  
  reconcileChildren(fiber, children)
}
```

我们需要修改 commitWork，因为函数的 fiber 没有 DOM 节点。我们要考虑如果在 fiber 树中存在无 DOM 节点的 fiber 时，如何连接 DOM：

- 要找到一个 DOM 节点的父节点，我们需要找到一个包含 DOM 节点的 fiber。
- 当需要移除一个 fiber 节点时，我们需要找到一个包含 DOM 节点的子节点。

```js
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  // 寻找到包含 DOM 的父节点
  let domParentFiber = fiber.parent
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent
  }
  const domParent = domParentFiber.dom
​
  if (
    fiber.effectTag === "PLACEMENT" &&
    fiber.dom != null
  ) {
    domParent.appendChild(fiber.dom)
  } else if (
    fiber.effectTag === "UPDATE" &&
    fiber.dom != null
  ) {
    updateDom(
      fiber.dom,
      fiber.alternate.props,
      fiber.props
    )
  } else if (fiber.effectTag === "DELETION") {
    // 寻找到包含 DOM 的子节点
    commitDeletion(fiber,domParent);
  }
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}

function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom)
  } else {
    commitDeletion(fiber.child, domParent)
  }
}
```

### Hooks

最后一步，既然我们使用了函数组件，那么就要给它加上状态。

首先更新为函数创建子 fiber 的函数：

```js
let wipFiber = null
let hookIndex = null
​
function updateFunctionComponent(fiber) {
  wipFiber = fiber
  // 记录当前 hook 的索引
  hookIndex = 0
  // 支持在同一个组件中多次调用 useState 函数
  wipFiber.hooks = []
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```

然后我们写自己的 useState 函数：

```js
function useState(initial) {
  const oldHook = 
    wipFiber.alternate && 
    wipFiber.alternate.hooks && 
    wipFiber.alternate.hooks[hookIndex]
  // 如果存在旧的 hook，我们从旧的 hook 中拷贝状态到新的 hook
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue:[]
  }
  // 拿到 action(更新状态的回调函数) 处理 state
  const actions = oldHook ? oldHook.queue : []
  actions.forEach(action => {
    hook.state = action(hook.state)
  })
  // setState 将 action 添加到 hook 的 queue 中
​ const setState = action => {
    hook.queue.push(action)
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    }
    // 重新渲染这颗 fiber tree
    nextUnitOfWork = wipRoot
  }
  wipFiber.hooks.push(hook)
  hookIndex++
 // 返回 state 和 setState
  return [hook.state, setState]
}
```

就这样，我们构建出了我们自己的 mini React，在 [codesandbox](https://codesandbox.io/s/didact-8-21ost) 中查看完整代码。

## 经验总结

本文的目的之一是让你更轻松地深入学习 React，这就是我们在几乎所有地方都使用与 React 相同的变量和函数名称的原因。但是我们构建的代码库并没有包含很多的 React 特性以及优化，以下是 React 与我们的实现做得不同的地方：

- 我们的渲染阶段会遍历整棵树，而 React 会跳过那些没有发生改变的子树。
- 我们会在提交阶段遍历整个树，而 React 只会保留产生影响的 fiber 节点。
- 我们为每个 fiber 创建一个新的对象，而 React 会复用之前树上的 fiber 节点。
- 我们在渲染阶段收到一个新的更新时，会丢弃之前的工作树，从根节点重新开始。而 React 给每一个更新标记一个过期时间戳，通过这个时间戳来决定各个更新之间的优先级。
- 除此之外还有很多...

**参考资料**

- [Build your own React](https://pomb.us/build-your-own-react/)
