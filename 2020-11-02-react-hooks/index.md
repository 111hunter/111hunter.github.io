# 以 useEffect 为圆心


以 useEffect 为圆心，其他 Hooks 为半径，构建 React Hooks 的知识圆环。为什么会想出这样一个标题呢？Hooks 的知识点过于分散，很多朋友在读过 React 官方文档后，还是不知道 Hooks 如何在实际项目中使用。本文希望从 useEffect 的具体用法中引出其他 Hooks，从而构建出完整的 React Hooks 知识体系。

## 心智模型

学习 Hooks 的使用，重点是心智模型的转变。useEffect 的心智模型是实现状态同步，而不是响应生命周期事件。每次触发时 useEffect，它都会捕获本次调用时组件中的数据，也就是所谓的 Capture Value 特性：**组件每次渲染都有自己的数据，组件内的函数(包括 effects，事件处理函数，定时器或者 API 调用等)会捕获该次渲染的组件数据。**

##  状态同步

首先看这段代码，请判断最终计时器中的 count 和 组件中的 count 分别是多少？

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  console.log("组件中的count", count);
  useEffect(() => {
    console.log("触发useEffect");
    const id = setInterval(() => {
      console.log("计时器中的count", count);
      setCount(count + 1);
    }, 1000);
    return () => {
      console.log("销毁了定时器");
      clearInterval(id);
    }
  }, []);

  return <h1>{count}</h1>;
}
```

答案分别是 0 和 1，多少有点基础的人都能想通或者猜对。现在更改需求，让组件中的 count 每秒加 1。我们有两种写法：

- 不对依赖数组撒谎

```jsx
// 每次因 count 变化触发的重渲染都会触发 useEffect
function Counter() {
  const [count, setCount] = useState(0);
  console.log("组件中的count", count);
  useEffect(() => {
    console.log("触发useEffect");
    const id = setInterval(() => {
      console.log("计时器中的count", count);
      setCount(count + 1);
    }, 1000);
    return () => {
      console.log("销毁了定时器");
      clearInterval(id);
    }
  }, [count]);

  return <h1>{count}</h1>;
}
```

每次重渲染创建 `Counter` 组件时都会触发 useEffect，销毁上一次的计数器，并创建新的计数器。因此组件中的 count 和计时器中的 count 是同步的。

- 让 Effect 自给自足

```jsx
// 移除 useEffect 的非必需的依赖，减少不必要的触发。
function Counter() {
  const [count, setCount] = useState(0);
  console.log("组件中的count", count);
  useEffect(() => {
    console.log("触发useEffect");
    const id = setInterval(() => {
      console.log("计时器中的count", count);
      // 这里接收的函数描述 count 如何变化(action)
      setCount(c => c + 1);
    }, 1000);
    return () => {
      console.log("销毁了定时器");
      clearInterval(id);
    }
  }, []);
  
  return <h1>{count}</h1>;
}
```

通过控制台我们发现，计时器中的 count 始终为 0，怎么解释？useEffect 只在组件初次渲染后触发一次，它创建了计时器，计时器记住了当时的 count。既然计时器中的 count 始终为 0，那么 setCount 是怎样让组件状态同步的呢？想搞清楚这一点就不得不提 useReducer 了。

## useReducer

事实上，useState 是预置了如下 reducer 的 useReducer，相关 [源码](https://github.com/facebook/react/blob/v16.13.1/packages/react-reconciler/src/ReactFiberHooks.js#L623) ：

```js
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

也就是说，setCount 接收的 action 函数是在下一次函数组件渲染时，在 useState 中调用的。描述动作和执行动作分开进行，感觉有 Redux 的内味了？没错，React 还为 useReducer 提供了配套的 dispatch 方法：

```js
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

再次更改需求，我们不让计时器每秒加 1，而是由输入的 step 控制。还是对比两种写法：

- 不对依赖数组撒谎

```js
function Counter() {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState(0);
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count => count + step);
    }, 1000);
    return () => clearInterval(id);
  }, [step]);
  ...
}
```

- 让 Effect 自给自足

```jsx
const initialState = {
  count: 0,
  step: 0,
};

function reducer(state, action) {
  const { count, step } = state;
  if (action.type === 'tick') {
    return { count: count + step, step };
  } else if (action.type === 'step') {
    return { count, step: action.step };
  } else {
    throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  const { count, step } = state;
  useEffect(() => {
    const id = setInterval(() => {
      dispatch({ type: 'tick' });
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return (
    <div>
      <h1>{count}</h1>
      <input value={step} onChange={e => {
        dispatch({
          type: 'step',
          step: Number(e.target.value)
        });
      }} />
    <div/>
  );
}
```

移除 useEffect 的非必需的依赖，就能减少不必要的触发，是一种性能优化思路。还有一种思路是保持 useEffect 的依赖不变，也能减少不必要的触发，这会用到 useCallback 和 useMemo。

## useCallback

React 判断组件中的数据是否发生改变时使用了 Object.is 进行比较。当 useEffect 的依赖数组的元素为引用数据类型时，每次的比较结果都是发生改变，这就失去了依赖数组本身的意义(条件式地触发 useEffect)。看这段代码，我们希望复用网络请求逻辑，这样可行吗？

```jsx
function SearchResults() {
  const getFetchUrl = (query) => {
    return 'https://hn.algolia.com/api/v1/search?query=' + query;
  }; 

  useEffect(() => {
    const url = getFetchUrl('react');
  // Fetch data and do something 
  ...
  }, [getFetchUrl]); 

  useEffect(() => {
    const url = getFetchUrl('redux');
  // Fetch data and do something 
  ...
  }, [getFetchUrl]);
  
  ...
}
```

当我们写这段代码时，我们发现网络请求将无限重复，因为函数调用会生成不同引用。一个可能的解决办法是把 `getFetchUrl` 从依赖中去掉，前提是你能确保它不受数据流变化的影响，否则就会出现意想不到的 bugs。

然而 useEffect 的设计意图就是要强迫你关注数据流的变化，然后去同步状态。当不能把函数从依赖中去掉时，我们可以使用 useCallback 来包装函数从而确保函数的**引用相等**：

```js
/** useCallback 在其依赖变化时，才生成新的函数
  * 现在依赖为空，getFetchUrl 永远调用同一个函数 */
const getFetchUrl = useCallback((query) => {
  return 'https://hn.algolia.com/api/v1/search?query=' + query;
}, []); 
```

更改 `getFetchUrl` 后就能避免网络请求重复的问题了。useCallback 本质上是对函数添加了一层依赖检查，让函数只在需要改变的时候才改变。

useCallback 的另一个使用场景：当父组件传递函数给子组件的时候，由于父组件的更新会导致该函数重新生成，从而传递给子组件的函数引用发生变化，这就会导致子组件也会更新，这时我们可以通过 useCallback 来缓存该函数，然后传递给子组件同一个函数避免子组件更新(子组件需用 memo 包装)，看这篇 [文章](https://juejin.im/post/6844904101445124110) 的例子。

useMemo 扩展了 useCallback 的功能，useCallback 只能缓存函数，而 useMemo 可以缓存任何类型的值，同样是为了确保引用相等。除此之外，useMemo 还可以用于避免重复计算。

## useRef

现在来看这个例子，我们连续点击 Add count 按钮时，3s 后会发生什么？

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setTimeout(() => {
      console.log(count);
    }, 3000);
  });

  return (
    <div>
      <h1>{count}</h1>
      <button onClick={()=>setCount(count+1)}>Add count</button>
    <div/>
  );
}
```

3s 后会在控制台依次打印 count 的值 1，2，3 ...，然而在一些场景中，我们只想得到最新的 count 值，该怎么做？这就会用到 useRef，useRef 返回一个可变的 ref 对象，它的属性 current 被初始化为传入的参数，并且 useRef 始终返回同一个 ref 对象。

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const latestCount = useRef(count);

  useEffect(() => {
    latestCount.current = count;
    setTimeout(() => {
      console.log(latestCount.current);
    }, 3000);
  });

  return (
    <>
      <h1>{count}</h1>
      <button onClick={()=>setCount(count+1)}>Add count</button>
    </>
  );
}
```
现在我们连续点击会发现 3s 后控制台将多次打印最新的 count 值，这证明我们修改的是同一个 ref 对象。除了用来缓存变量，useRef 还能获得 DOM 元素，需要在元素上绑定 ref 属性：

```jsx
function RefDemo() {
  const titleRef = useRef();

  function changeDOM() {
    titleRef.current.innerHTML = 'hello world';
    titleRef.current.style.color = 'red';
  }
  return (
    <div>
      <h2 ref={titleRef}>RefDemo</h2>
      <button onClick={changeDOM}>修改DOM</button>
    </div>
  );
}
```

## 思考

暂时就先这样吧，后面还会再补充。

**参考资料**

- [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
