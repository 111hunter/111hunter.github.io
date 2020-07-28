# Recoil 基础


最近，Facebook 官方开源了一个状态管理库 Recoil，我们来学习一下。Recoil 是基于 Immutable 的数据流管理方案，这是它值得学习的重要原因。Recoil非常易于学习，它的 API 简单强大，对于已经习惯使用 hooks 的人来说很自然。

## 核心概念

Recoil 中的核心概念只有 Atom(原子状态) 和 Selector(派生状态)。

<div align=center><img src="/img/recoil.png" width="80%"></div>

### Atom 

 Atom 是状态的单位。它们可更新也可订阅。当 atom 被更新，每个被订阅的组件都将使用新值进行重渲染。如果多个组件使用相同的 atom，则这些组件共享 atom 的状态。可以使用 atom 替代组件内部的 state。atom 也可以在运行时创建。

Atom 是使用 atom 函数创建的：

```ts
function atom<T>({
  key: string,
  default: T | Promise<T> | RecoilValue<T>,
  
  dangerouslyAllowMutability?: boolean,
}): RecoilState<T>
```

- key：标识 atom 的字符串，必须相对于其他 atom/selector 是唯一值
- default：atom 的初始值，可以是静态值，Promise，或返回值类型相同的另一个 atom/seletor
- 最后一个参数是允许 Mutable，由于 Recoil 默认的 Immutable 特性带来的可预测性更利于调试和维护，一般不设置这个值

定义一个 atom，用来获取输入字符:

```js
const textState = atom({
  key: 'textState', // unique ID (with respect to other atoms/selectors)
  default: '', // default value (aka initial value)
});
```

### Selector

selector 是一个纯函数，入参为 atom 或其他 selector。selector 被用于计算基于 atom 的派生数据，这使得我们避免了冗余 state，将最小粒度的状态存储在 atom 中，而其它所有内容根据最小粒度的状态进行有效计算。当上游 atom/selector 更新时，将重新执行 selector 函数。组件可以像 atom 一样订阅 selector，当 selector 发生变化时，重新渲染相关组件。

Selector 是使用 selector 函数创建的：

```ts
function selector<T>({
  key: string,

  get: ({
    get: GetRecoilValue
  }) => T | Promise<T> | RecoilValue<T>,

  set?: (
    {
      get: GetRecoilValue,
      set: SetRecoilState,
      reset: ResetRecoilState,
    },
    newValue: T | DefaultValue,
  ) => void,

  dangerouslyAllowMutability?: boolean,
}): RecoilValueReadOnly<T> | RecoilState<T>
```
```ts
type ValueOrUpdater<T> = T | DefaultValue | ((prevValue: T) => T | DefaultValue);
type GetRecoilValue = <T>(RecoilValue<T>) => T;
type SetRecoilState = <T>(RecoilState<T>, ValueOrUpdater<T>) => void;
type ResetRecoilState = <T>(RecoilState<T>) => void;
```

- key：标识 selector 的字符串，必须相对于其他 atom/selector 是唯一值
- get：get 参数中 get，可以从其他 atom/selector 取值，从而利用依赖关系计算 seletor，传递给此函数的 atom/selector 隐式添加到这个 seletor 的依赖项列表中
- set?：设置了该属性，selector 才会返回可写的 state

定义一个 selector，依赖的 atom 是我们上面定义的 textState，用来获取输入字符长度 :

```jsx
const charCountState = selector({
  key: 'charCountState', // unique ID (with respect to other atoms/selectors)
  get: ({get}) => {
    const text = get(textState);
    return text.length;
  },
});
```
测试 atom 和 selector [示例 demo](https://recoiljs.org/docs/introduction/getting-started#demo)

从组件的角度来看，selector 和 atom 具有相同的功能，因此可以交替使用。

## 订阅或更新状态

前面讲述如何用 atom 和 selector 定义 state，下面是 state 的取值和更新函数：

- useRecoilState：返回 atom/selector 的值和 set 函数，类似 useState。
- useRecoilValue：仅返回 atom/selector 的值。
- useSetRecoilState：仅返回 atom/seletor 的 set 函数。
- useResetRecoilState：重置 atom/selector 到默认值并读取。

在组件中使用这些 hooks 与使用其他 hooks 的方式基本相同：

```jsx
import React from 'react';
import { atom, useRecoilState, selector, useRecoilValue } from 'recoil';

const textState = atom({
  key: 'textState', // unique ID (with respect to other atoms/selectors)
  default: '', // default value (aka initial value)
});

const charCountState = selector({
  key: 'charCountState', // unique ID (with respect to other atoms/selectors)
  get: ({ get }) => {
    const text = get(textState);
    return text.length;
  },
});

export const CharacterCounter = () => {

  const [char, setChar] = useRecoilState(textState);
  // selector 没有定义 set，用 useRecoilValue 取值 
  const charCount = useRecoilValue(charCountState);

  return (
    <div>
      <input
        type="text"
        value={char}
        onChange={(e) => setChar(e.target.value)}
      />
      <div>Echo: {char}</div>
      <div>Character Count: {charCount} </div>
    </div>
  );
};

export default CharacterCounter;
```
atom，selector 的 state 的取值和更新函数是相同的，selector 未定义 set 只能用 useRecoilValue 取值，定义 set 之后也能用 useRecoilState，因此 atom 应该是基于 selector 的一个特定封装，帮我们封装好了 set，get，而无须自定义。


## 异步支持

在 selector 的数据流图中, Recoil 可以让你随意的混合使用同步和异步函数。只需从 selector get 回调中返回一个 Promise，接口完全一样。因为这些只是 selector，其他的 selector 也可以依赖它们来进一步变更数据。selector 是纯函数，是对只读数据库查询进行建模的好方法，其中重复查询可提供一致的数据。

```jsx
import React from 'react';
import {selector, useRecoilValue} from 'recoil';

const myQuery = selector({
  key: 'MyDBQuery',
  get: async () => {
    const response = await fetch(getMyRequestUrl());
    return response.json();
  },
});

function QueryResults() {
  const queryResults = useRecoilValue(myQuery);

  return (
    <div>
      {queryResults.foo}
    </div>
  );
}

function ResultsSection() {
  return (
    <React.Suspense fallback={<div>Loading...</div>}>
      <QueryResults />
    </React.Suspense>
  );
}
```
atom 是基于 selector 封装，也支持 Promise 做默认 state。不过官方的建议是当其从其他状态或异步请求时派生的 state，应该使用 selector。

## 参数查询

有时我们希望通过传递参数动态定义 state，你可以使用 atomFamily 或 selectorFamily 实现这类需求，
atom 与 atomFamily，selector 与 selectorFamily 的区别仅仅是定义 state 的时候是否需要参数：

```jsx
const myDataQuery = selectorFamily({
  key: 'MyDataQuery',
  get: (queryParameters) => async ({get}) => {
    const response = await asyncDataRequest(queryParameters);
    if (response.error) {
      throw response.error;
    }
    return response.data;
  },
});

function MyComponent() {
  const data = useRecoilValue(myDataQuery({userID: 132}));
  return <div>...</div>;
}
```

目前 Recoil 还属于实验阶段，能确定的是 Recoil 将兼容 React 并发模式。
我们可以在 Recoil 中学到 React Hook 时代的状态管理的基本模式：

- state 的读与写分离，做到最优按需渲染。
- 原子存储的数据相互无关联，关联的数据使用派生值的方式推导。
- 派生的值必须严格缓存，并在命中缓存时引用保证严格相等。

**参考资料**

- [Recoil 官方文档](https://recoiljs.org/)
- [精读《recoil》](https://zhuanlan.zhihu.com/p/143335599)



