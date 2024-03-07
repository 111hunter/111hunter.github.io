# ES6 学习笔记


ECMAScript 6(以下简称 ES6)是 JavaScript 语言的下一代标准,已经在 2015 年 6 月正式发布了。Mozilla 公司将在这个标准的基础上,推出 JavaScript 2.0。ECMAScript 和 JavaScript 是什么关系？简单来说，ECMAScript 是 JavaScript 语言的国际标准，JavaScript 是 ECMAScript 的实现。

## let 和 const

var 函数作用域 function scope,不在函数内时作用域是全局的
用 let 和 const 声明变量, let, const 块级作用域 block scope,作用域是{}内

eg.执行以下语句判断区别：

```js
for (var i = 0; i < 10; i++) {
  console.log(i);
  setTimeout(function () {
    console.log(`i:${i}`);
  }, 1000);
}

for (let i = 0; i < 10; i++) {
  console.log(i);
  setTimeout(function () {
    console.log(`i:${i}`);
  }, 1000);
}
```

let, const 不能重复声明变量值

- let 声明的变量是可以重新赋值的, const 声明的变量只能修改引用类型的属性值
- 变量提升：let, count 有变量提升,未声明先使用存在临时性死区(Temporal dead zone),详见 mdn

## 箭头函数

特点：简明的语法,隐式返回(省去 return 关键字),匿名函数
this:普通函数 this 是动态绑定的

```js
const Jelly = {
  name: "Jelly",
  hobbies: ["Coding", "Sleeping", "Reading"],
  printHobbies: function () {
    // console.log(this);
    this.hobbies.map(function (hobby) {
      // console.log(this);
      console.log(`${this.name} loves ${hobby}`);
    });
  },
};
Jelly.printHobbies();
```

map 中的回调函数是一个独立的函数,不作为对象的方法,并且没有通过 call bind apply
来改变里面的 this，this 指向 window,严格模式下指向 undefined

```js
const Jelly = {
  name: "Jelly",
  hobbies: ["Coding", "Sleeping", "Reading"],
  printHobbies: function () {
    // console.log(this);
    var self = this;
    this.hobbies.map(function (hobby) {
      // console.log(this);
      console.log(`${self.name} loves ${hobby}`);
    });
  },
};
Jelly.printHobbies();
```

箭头函数的 this 值继承父作用域,是词法作用域,定义的时候就指向明确,且不会绑定 this：

```js
const Jelly = {
  name: "Jelly",
  hobbies: ["Coding", "Sleeping", "Reading"],
  printHobbies: function () {
    // console.log(this);
    this.hobbies.map((hobby) => {
      // console.log(this);
      console.log(`${this.name} loves ${hobby}`);
    });
  },
};
Jelly.printHobbies();
```

命名函数在递归,事件绑定时有用,在箭头函数中使用：

```js
const greet = (name) => {
  alert(`Hello ${name}`);
};
```

箭头函数不适用的情况：

- 需要使用 this 慎用
- 需要使用 arguments(箭头函数没有 arguments)

## 模板字符串

- 模板字符串中的换行和空格都是会被保留的。
- 模板字符串可嵌套,支持三元表达式。
- 标签模板字符串,是一个函数的调用。

```js
alert`Hello world!`;
// 等价于alert('Hello world!');
```

## 解构赋值

- 针对**数组**或者**对象**进行模式匹配,然后对其中的变量进行赋值。
- 是对赋值运算符的扩展,方便提取对象属性值,可嵌套可忽略。

```js
let [a, b, c, d, e] = "hello";

let obj = { p: ["hello", { y: "world" }] };
let {
  p: [x, { y }],
} = obj;
// x = 'hello'
// y = 'world'
let obj = { p: ["hello", { y: "world" }] };
let {
  p: [x, {}],
} = obj;
// x = 'hello'
```

## 计算属性

对象字面定义属性名位置的 [ ] 中可以放置任意合法表达式。

```js
const keys = ["name", "age", "birthday"];
const values = ["jelly", 18, "2016-01"];
const Laravist = {
  [keys.shift()]: values.shift(),
  [keys.shift()]: values.shift(),
  [keys.shift()]: values.shift(),
};
console.log(Laravist);
```

## Symbol

ES6 引入了一种新的原始数据类型表示独一无二的值,最大的用法是用来定义对象的唯一属性名。
用于生成唯一标识符避免命名冲突,可作为私有属性在对象内部使用,不能 for 循环遍历

```js
const classRoom = {
  [Symbol("lily")]: { grade: 60, gender: "female" },
  [Symbol("nina")]: { grade: 70, gender: "female" },
  [Symbol("nina")]: { grade: 90, gender: "female" },
};
const syms = Object.getOwnPropertySymbols(classRoom).map(
  (sym) => classRoom[sym]
);
console.log(syms);
```

## 剩余参数

```js
const player = ["jelly", 123, 2.4, 3.6, 1.7];
const [name, id, ...scores] = player;
console.log(name, id, scores);
```

扩展运算符可以将可遍历对象元素扩展成新的参数序列,而不用改变原来的对象

```js
const younger = ["aaa", "bbb", "ccc"];
const older = ["xxx", "yyy", "zzz"];
const members = [...younger, "ddd", ...older];
const newmembers = members;
```

## Promise

Promise 用于避免回调地狱

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject(Error("Laravist isn't awesome!"));
  }, 2000);
});
p.then((data) => {
  console.log(data);
}).catch((err) => {
  console.error(err);
});
```

await 操作符用于等待一个 Promise 对象,它只能在异步函数 async function 内部使用。

await 针对所跟不同表达式的处理方式：

- Promise 对象：await 会暂停执行,等待 Promise 对象 resolve,然后恢复 async 函数的执行并返回解析值。
- 非 Promise 对象：直接返回对应的值。

```js
function testAwait(x) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(x);
    }, 2000);
  });
}
async function helloAsync() {
  var x = await testAwait("hello world");
  console.log(x);
}
helloAsync();
```

## Class

- class 是语法糖,本质是 funciton,没有变量提升

一个继承的例子:

```js
function MyArray() {
  Array.apply(this, arguments);
}
const colors = new MyArray();
colors[0] = "red";
console.log(colors.length); //undefined
colors.length = 0;
console.log(colors[0]); //red
```

ES5 是先新建子类的实例对象 this,
再将父类的属性添加到子类上,原生构造函数会忽略 apply 方法传入的 this,
父类的内部属性无法获取,导致无法继承原生的构造函数。

```js
class MyArray extends Array {
  constructor() {
    super();
    console.log(this);
  }
}
const colors = new MyArray();
colors[0] = "red";
console.log(colors.length); // 1
colors.length = 0;
console.log(colors[0]); // undefined
```

ES6 允许继承原生构造函数定义的子类,因为 ES6 是先新建父类的实例对象 this,
然后再用子类的构造函数修饰 this,使得父类的素有行为都可以继承。
ES6 可以自定义原生数据结构的子类，这是 ES5 无法做到的。

## 一些补充

### 新增 for...of 循环：

<br>先回顾 js 中 for 循环的几种写法

```js
const fruits = ["apple", "banana", "orange", "mango"];
for (let i = 0; i < fruits.length; i++) {
  console.log(fruits[i]);
}
```

可读性不是很好

```js
fruits.forEach((fruit) => {
  console.log(fruit);
});
```

不能在循环中 break 或 continue

```js
for (let index in fruits) {
  console.log(fruits[index]);
}
```

会遍历对象上所有可枚举属性

```js
for (let fruit of fruits) {
  console.log(fruit);
}
```

不会遍历数组中非数字属性,能够 break 或 continue
<br>应用数组解构语法

```js
for (let [index, fruit] of fruits.entries()) {
  console.log(`${fruit} rank in ${index + 1} in my favorite fruits`);
}
```

for...of 可以应用于可迭代对象(部署了 iterator 接口或提供 Symbol.iterator 方法的数据结构)
<br>数组,字符串,arguments,NodeList,map.set 等,但不支持对象

### Array.from()和 Array.of()

es6 新增数组方法 Array.from()和 Array.of()：

- 注意是数组原型对象上的静态方法
- Array.from()用于把可迭代对象转化成数组,Array.of()传入参数生成数组

### Proxy 与 Reflect

Proxy 与 Reflect 是 ES6 为了操作对象引入的 API 。

- Proxy 可以对目标对象的读取、函数调用等操作进行拦截,然后进行操作处理。它不直接操作对象,而是像代理模式,通过对象的代理对象进行操作。
- ES6 中将 Object 的一些明显属于语言内部的方法移植到了 Reflect 对象上。

### 迭代器与生成器

Iterator 是 ES6 引入的一种新的遍历机制,通过一个键为 Symbol.iterator 的方法来实现。
<br>Generator 函数:在 function 后面,函数名之前有个\*,函数内部有 yield 表达式。

### Map 与 Set

Object 的键只能是字符串或者 Symbols, Map 的键可以是任意值,Map 中的键值是有序的(FIFO 原则),Map 的键值对个数可以从 size 属性获取。
<br>Set 对象允许你存储任何类型的唯一值,NaN 与 NaN 是不恒等的,但是在 Set 中只能存一个。

