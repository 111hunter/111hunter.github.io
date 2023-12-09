# 初探 RxJS - Observable 的简单创建


> ReactiveX 结合了 **观察者模式**、**迭代器模式** 和 **使用集合的函数式编程**，以满足以一种理想方式来管理事件序列所需要的一切。

在 RxJS 中用来解决异步事件管理的的基本概念是：

- Observable (可观察对象): 表示一个概念，这个概念是一个可调用的未来值或事件的集合。

- Observer (观察者): 一个回调函数的集合，它知道如何去监听由 Observable 提供的值。

- Subscription (订阅): 表示 Observable 的执行，主要用于取消 Observable 的执行。

- Operators (操作符): 采用函数式编程风格的纯函数 (pure function)，使用像 map、filter、concat、flatMap 等这样的操作符来处理集合。

- Subject (主体): 相当于 EventEmitter，并且是将值或事件多路推送给多个 Observer 的唯一方式。

- Schedulers (调度器): 用来控制并发并且是中央集权的调度员，允许我们在发生计算时进行协调，例如 setTimeout 或 requestAnimationFrame 或其他。

以上文字来自 RxJS 中文文档，是 RxJS 的核心概念。

下面来学习创建 Observable 对象以加深对一些概念的理解。
本文将生成 observable 的操作符分为以下四类：

- 转换操作符：from，fromEvent，fromPromise，of

- 创建操作符：create, range

- 时间操作符：interval, timer

- 特殊操作符：empty, never, throw

## 项目初始化

我们使用 webpack 作为项目构建工具。使用 Babel 编译我们的代码。这是我们的项目依赖：

```js
"devDependencies": {
    "@babel/core": "^7.8.7",
    "@babel/preset-env": "^7.8.7",
    "babel-loader": "^8.0.6",
    "webpack": "^4.42.0"
  },
  "dependencies": {
    "jquery": "^3.1.0",
    "rxjs": "^5.0.0-beta.12"
  }
```

webpack 配置文件 webpack.config.js:

```js
module.exports = {
  entry: "./src/app.js",
  output: {
    path: __dirname + "/dist",
    filename: "app.bundle.js",
  },
  module: {
    rules: [
      {
        test: /\.m?js$/,
        exclude: /(node_modules|bower_components)/,
        use: {
          loader: "babel-loader",
          options: {
            presets: ["@babel/preset-env"],
          },
        },
      },
    ],
  },
};
```

新建文件夹 src, 在里面新建文件 app.js, 我们在 app.js 中引入 jquery 和 RxJS。

```js
import $ from "jquery";
import Rx from "rxjs/Rx";

console.log("Code Running...");
```

在 html 文件中引入编译后的 js 文件。

```html
<body>
  <input type="text" id="input" />
  <div id="output"></div>
  <ul>
    <li id="name"></li>
    <li id="artist"></li>
  </ul>
  <script src="./dist/app.bundle.js"></script>
</body>
```

执行 ` webpack --watch --mode development`，实时监视文件变化，并重新编译代码。
打开浏览器控制台没有任何报错，并输出 "Code Running..." 说明我们的项目构建成功。
高版本的 chrome 可能出现 DevTools failed to parse SourceMap，在控制台的 setting 中取消 Enable JavaScript source maps 这一项即可。

## 转换操作符

### Observable.from()

Observable.from() 将 **可迭代对象** 转化为 observables 序列, 传入数据集合。

```js
// from array
const numbers = [1, 2, 3, 4, 5];
const numbers$ = Rx.Observable.from(numbers);
numbers$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  () => console.log("complete")
);

// from string
const str = "hello world";
const str$ = Rx.Observable.from(str);
str$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  () => console.log("complete")
);

// from array of objects
const posts = [
  { title: "post 1", body: "body 1" },
  { title: "post 2", body: "body 2" },
  { title: "post 3", body: "body 3" },
];
const posts$ = Rx.Observable.from(posts);
posts$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  () => console.log("complete")
);

// from set
const set = new Set(["hello", 123, { title: "my title" }]);
const set$ = Rx.Observable.from(set);
set$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  () => console.log("complete")
);

// from map
const map = new Map([
  [1, 2],
  [3, 4],
  [5, 6],
]);
const map$ = Rx.Observable.from(map);
map$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  () => console.log("complete")
);
```

### Observable.fromEvent()

Observable.fromEvent() 将 **事件** 转化为 observables 序列, 传入两个参数：页面元素 和 事件名称。从事件中创建的 observable 对象是持续不断产生的，不会输出 "completed"。

转化键盘事件:

```js
const input = $("#input");
const output = $("#output");
const inputStream$ = Rx.Observable.fromEvent(input, "keyup");

inputStream$.subscribe(
  (e) => {
    console.log(e.target.value);
    output.text(e.target.value);
  },
  (err) => console.log(err),
  () => console.log("completed")
);
```

转化鼠标事件:

```js
const moveStream$ = Rx.Observable.fromEvent(document, "mousemove");

moveStream$.subscribe(
  (e) => {
    console.log(e.type);
    output.html("<h1>X: " + e.clientX + " Y: " + e.clientY + "</h1>");
  },
  (err) => console.log("err"),
  () => console.log("completed")
);
```

### Observable.fromPromise()

Observable.fromPromise() 将 promise 转化为 observables 序列, 传入 promise。

```js
const myPromise = new Promise((resolve, reject) => {
  console.log("creating promise");
  setTimeout(() => {
    resolve("hello from promise");
  }, 500);
});

const myPromiseSource$ = Rx.Observable.fromPromise(myPromise);
myPromiseSource$.subscribe((x) => console.log(x));
```

结合之前定义的 inputStream$ 嵌套使用：

```js
function getSong(username) {
  return $.ajax({
    type: "GET",
    url: `https://autumnfish.cn/search?keywords=` + username,
  }).promise();
}

const song = $("#input");
const inputStream$ = Rx.Observable.fromEvent(song, "keyup");
inputStream$.subscribe((e) => {
  Rx.Observable.fromPromise(getSong(e.target.value)).subscribe((x) => {
    $("#name").text("Song: " + x.result.songs[0].name);
    console.log(x.result.songs[0].name);
    $("#artist").text("Artist: " + x.result.songs[0].artists[0].name);
  });
});
```

### Observable.of()

of 操作符接收 1 个或多个参数。转换为 Observable 对象。

```js
const stream$ = Rx.Observable.of(1, 2, 3, "hello");
stream$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  (complete) => console.log("complete")
);
```

## 创建操作符

### Observable.create()

Rx.Observable.create 是 Observable **构造函数** 的别名，它接收一个以 observer 作为参数的回调函数。这个回调函数会定义 observable 将会如何发送值给 observer。observer 是什么？observer 就是我们之前传入 subscribe() 的参数，是一个有三个回调函数的对象。

```js
const source$ = new Rx.Observable.create((observer) => {
  observer.next("hello");
  observer.next("another hello");
  setTimeout(() => {
    observer.next("next hello");
    observer.complete();
  }, 2000);
});

const observer1 = {
  next: (v) => console.log(v + "1"),
  error: (err) => console.log(err),
  complete: () => console.log("complete"),
};

source$.subscribe(observer1);
```

observable 是数据流的生产者，决定数据怎么给。observer 是数据流的消费者，决定数据怎么用。observable 是老板，observer 是顾客。
observable.subscribe()会实例化一个 Subscription 对象。Subscription 表示 Observable 的执行，可以被清理。这个对象最常用的方法是 unsubscribe 方法。

### Observable.range()

接收两个数字参数产生有序序列，一个是开始序列数字。一个是序列个数。

```js
const rangeSource$ = Rx.Observable.range(6, 5);
rangeSource$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  (complete) => console.log("complete")
);
```

## 时间操作符

### Observable.interval 和 Observable.timer

从零开始产生数字，interval 的参数是数字产生的间隔时间，timer 多了个开始延迟时间作为第一个参数。

```js
const intervalSource$ = Rx.Observable.interval(1000).take(5);
intervalSource$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  (complete) => console.log("complete")
);

const timerSource$ = Rx.Observable.timer(2000, 1000).take(5);
timerSource$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  (complete) => console.log("complete")
);
```

## 特殊操作符

### Observable.empty(), Observable.never() 和 Observable.throw()

Observable.empty 创建的 Observable 开始就结束，Observable.never 创建的 Observable 不会结束，Observable.throw 抛出异常不会结束。

```js
const emptySource$ = Rx.Observable.empty();
emptySource$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  (complete) => console.log("complete")
);

const neverSource$ = Rx.Observable.never();
neverSource$.subscribe(
  (v) => console.log(v),
  (err) => console.log(err),
  (complete) => console.log("complete")
);

const errorSource$ = Rx.Observable.throw("err");
errorSource$.subscribe(
  (v) => console.log(v),
  (err) => console.log("Throw Error: " + err),
  (complete) => console.log("complete")
);
```

附：[源码地址](https://gitee.com/hildakik/rxjs01)

**参阅资料**

- [RxJS Ultimate 中文版](https://rxjs-cn.github.io/RxJS-Ultimate-CN/)
- [30 天精通 RxJS](https://ithelp.ithome.com.tw/users/20103367/ironman/1199)
- [RxJS 中文文档](https://cn.rx.js.org/manual/index.html)

