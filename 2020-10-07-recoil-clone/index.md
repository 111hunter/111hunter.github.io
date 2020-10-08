# 实现仿 Recoil 的状态共享


本文是我最近阅读一篇英文技术文章后写的小结。阅读前请注意，本文不涉及任何 Recoil 源码。仿写的代码并不是  Recoil 真正的实现方式，本文只仿造实现了 Recoil 中两个重要的 API 接口：Atom 和 Selector。

如果你不熟悉 Recoil，请先阅读我的这篇 [文章](https://111hunter.github.io/2020-07-27-recoil/) 或者阅读它的 [官方文档](https://recoiljs.org/)。然后新建 React 项目：

```bash
npx create-react-app recoil-clone --typescript
```

在根目录下新建 coiled.tsx 文件，下面的代码都在这个文件中实现。

## 状态基类

定义一个 Stateful 表示共享状态的基类，Atom 和 Selector 需继承这个基类。为了监听状态的变化，我们使用观察者模式。这种设计模式在 RxJS 之类的库中很常见，我将从头开始编写一个简单的同步版本。

```ts
interface Disconnect {
  disconnect: () => void;
}

export class Stateful<T> {
  // Set 是 callback 的集合
  private listeners = new Set<() => void>();

  constructor(protected value: T) {}
  // 取值函数
  snapshot(): T {
    return this.value;
  }
  // 此处才会调用所有的监听者
  private emit() {
    for (const listener of Array.from(this.listeners)) {
      console.log('调用监听者: ' + listener);
      listener();
    }
  }
  // update 方法可以被 Stateful 的子类 Atom 和 Selector 继承
  protected update(value: T) {
    if (this.value !== value) {
      this.value = value;
      console.log('新值: ' + this.value);
      this.emit();
    }
  }
  // 订阅就加入监听者的 Set 集合，此方法接收 callback，返回也是 callback 
  subscribe(callback: () => void): Disconnect {
    console.log('注册监听者：' + callback);
    this.listeners.add(callback);
    return {
      disconnect: () => {
        console.log('注销监听者：' + callback);
        this.listeners.delete(callback);
      },
    };
  }
}
```

## 自定义 hook

下面是只读 hook 的实现方式。atom 和 selector 均可读，因此参数只需满足 Stateful 类型。这里注册的监听者 **updateState 巧妙地利用了函数组件的重渲染机制。**
 
```ts
export function useCoiledValue<T>(value: Stateful<T>): T {
  // 只要调用 updateState 就会触发重渲染
  const [, updateState] = useState({});
    useEffect(() => {
      console.log('渲染结束调用 useEffect, 添加监听者')
      // 注册 updateState 为监听者, 监听者是 callback 的 Set 集合
      const { disconnect } = value.subscribe(() => updateState({}));
      return () => disconnect();
    }, [value]);
  console.log('此时 useCoiledValue 的值: ' + value.snapshot());
  return value.snapshot();
}
```

下面是读写 hook 的实现方式，这里的读写 hook 只适用于 atom，默认 selector 不可写。

```ts
export function useCoiledState<T>(atom: Atom<T>): [T, (value: T) => void] {
  const value = useCoiledValue(atom);
  return [value, useCallback(value => atom.setState(value), [atom])];
}
```

## Atom

Atom 继承 Stateful，需要一个默认的写值方法。

```ts
export class Atom<T> extends Stateful<T> {
  public setState(value: T) {
    super.update(value);
  }
}
```

暴露的接口函数是仿写 Recoil 中的 atom 函数。

```ts
export function atom<V>(value: { key: string; default: V }): Atom<V> {
  return new Atom(value.default);
}
```

## Selector

Selector 继承 Stateful，Selector 是 Atom 或其他 Selector 的派生值，因此需要添加依赖。

```ts
export class Selector<T> extends Stateful<T> {
  // 将 dep 加入 Set 集合
  private registeredDeps = new Set<Stateful<any>>();

  private addDep<V>(dep: Stateful<V>): V {
    if (!this.registeredDeps.has(dep)) {
       // 注册 updateSelector 为监听者，并将 dep 加入 Set 集合
      dep.subscribe(() => this.updateSelector());
      this.registeredDeps.add(dep);
    }
    return dep.snapshot();
  }
  // 调用 generate 直接返回当前的 dep 值
  private updateSelector() {
    this.update(this.generate({ get: dep => this.addDep(dep) }));
  }

  constructor(private readonly generate: SelectorGenerator<T>) {
    super(undefined as any);
    this.value = generate({ get: dep => this.addDep(dep) });
  }
}
// selector 接收 atom 或者其他 selector 作为依赖
type SelectorGenerator<T> = (context: { get: <V>(dep: Stateful<V>) => V }) => T;
```

暴露的接口函数是仿写 Recoil 中的 selector 函数。

```ts
export function selector<V>(value: {
  key: string;
  get: SelectorGenerator<V>;
}): Selector<V> {
  return new Selector(value.get);
}
```

## 使用

将 index.tsx 作如下修改后，启动项目 `yarn start`，查看浏览器的 cosole 面板，项目成功运行。

```jsx
import React from 'react';
import ReactDOM from 'react-dom';
import { atom, useCoiledState, useCoiledValue, selector } from './coiled';
import './App.css';

const textState = atom<string>({
  key: 'textState', 
  default: '', 
});

const charCountState = selector<number>({
  key: 'charCountState',
  get: ({get}) => {
    const text = get(textState);
    return text.length;
  },
});

function TextInput() {
  const [text, setText] = useCoiledState(textState);

  const onChange = (event: any) => {
    setText(event.target.value);
  };

  return (
    <div>
      <input type="text" value={text} onChange={onChange} />
      <br />
      Echo: {text}
    </div>
  );
}

function CharacterCount() {
  const count = useCoiledValue(charCountState);
  return <>Character Count: {count}</>;
}

function App() {
  return (
    <div className="App">
      <TextInput />
      <CharacterCount />
    </div>
  );
}

ReactDOM.render(<App />, document.getElementById('root'));
```

## 思考

我们仿造 Recoil 实现了自己的状态共享。但请思考以下内容：

- Selectors 不会取消对 atoms 的监听。这意味着当你不再使用他们时，会造成内存泄漏。

- Selectors 和 Atoms 在重渲染前仅做一个浅比较。在某些场景下，使用深比较更加合理。

- Recoil 使用唯一 key 值标识每一个 atom 或 selector，并且它被用作支持 “App-wide observation” 的元数据。这里的实现仅仅是为了保持 API 相似。

- Recoil 在 selectors 里支持异步，这里没有实现这个特性。

我在 github 上发现了 [jotai](https://github.com/pmndrs/jotai) 项目。它与我的仿写非常相似，并且支持异步。

**参考资料**

- [Rewriting Facebook's "Recoil" React library ...](https://bennetthardwick.com/blog/recoil-js-clone-from-scratch-in-100-lines/)

