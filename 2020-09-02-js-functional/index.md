# 浅析函数式编程


>在计算机科学中，[函数式编程](https://en.wikipedia.org/wiki/Functional_programming)是一种编程范式，其中通过应用和组合函数来构造程序。它是一种声明式编程范式，其中函数定义是每个返回一个值的表达式树，而不是一系列更改程序状态的命令性语句。

## 声明式与命令式

假设我们有个需求：把下面字符串变成每个单词首字母大写。

```js
var string = 'functional programming is great';
```

### 命令式

如果你没有听说过函数式编程，用传统的编程思路，很自然的写出如下 [命令式编程](https://en.wikipedia.org/wiki/Imperative_programming) 代码：

```js
var string = 'functional programming is great';
var arrays = string.split(' ');
var newArray = [];
for (var i = 0; i < arrays.length; i++){
	var str = arrays[i].slice(0, 1).toUpperCase() + arrays[i].slice(1);
	newArray.push(str);
}
var newString = newArray.join(' ');
```

这样当然能完成任务，结果是产生了一堆临时变量。光是变量名就不好想，同时过程中掺杂了大量逻辑，一个函数需要从头读到尾才知道它具体做了什么，并且一旦出问题很难定位。

### 声明式

[声明式编程](https://en.wikipedia.org/wiki/Declarative_programming) 被看做是形式逻辑的理论，把计算看做推导。常见的声明式编程有数据库查询(SQL语句)，正则表达式，函数式编程等。函数式编程倡导利用若干简单的执行单元让计算结果不断渐进，逐层推导复杂的运算，而不是设计一个复杂的执行过程。

```js
var string = 'functional programming is great';
var newString = string
  .split(' ')
  .map(str => str.slice(0, 1).toUpperCase() + str.slice(1))
  .join(' ');
```

函数式编程的核心思想：通过函数转换数据，组合多个函数来求结果。

对比两种编程思想：命令式编程考虑我该如何做，而声明式编程考虑我要做什么。

## 核心概念-纯函数

函数式编程中的“函数”指满足以下特性的函数，也被称为 [纯函数](https://en.wikipedia.org/wiki/Pure_function)：

- 输出仅取决于输入(无状态，每次的执行结果都是可预测和易测试的)
- 不产生副作用(只计算输出值，不修改输入值，不做其他任何操作)

因此纯函数更像数学中的函数，只是描述输入与输出之间映射关系的表达式。

一个典型的纯函数设计是 redux 中的 reducer。好的我懂了，但是为什么要强调纯函数呢？因为纯函数的特性决定了它的众多优点：

### 易读易推理

纯函数容易阅读和推理，因为所有依赖关系都由参数提供。这意味着我们只需阅读函数的声明即可快速了解函数的作用及其依赖关系，而不用担心函数内有其他行为(副作用)。

### 移植性

对于常见的普通函数，同一函数不能直接在移植到别的上下文中使用，通常会为了实现同一类功能而编写不同的函数。

```js
// 普通函数
var signUp = function(attrs) {
  var user = saveUser(attrs);
  welcomeUser(user);
};
// 依赖 Db
var saveUser = function(attrs) {
    var user = Db.save(attrs);
    ...
};
// 依赖 Email
var welcomeUser = function(user) {
    Email(user, ...);
    ...
};
```

编写纯函数的好处是它需要的东西都在输入参数中已经声明，所以它方便移植到别的地方，因为它的依赖关系是很清晰的。

```js
// 纯函数
var signUp = function(Db, Email, attrs) {
  return function() {
    var user = saveUser(Db, attrs);
    welcomeUser(Email, user);
  };
};

var saveUser = function(Db, attrs) {
    ...
};

var welcomeUser = function(Email, user) {
    ...
};
```

### 引用透明性

因为纯函数内部没有全局引用，所以在任何使用纯函数的地方中把纯函数替换成它的执行结果，都不会对程序的整体运行产生影响，不会产生隐性问题。

```js
const greet = (name) => {
  return `hello, ${name}`;
};

console.log(greet('beijing'));

// 可做如下等价替换
console.log('hello, beijing');
```

### 可缓存

纯函数对相同输入总有相同输出，可以根据输入来做缓存，相同的输入可以不做重新计算。

```js
// 下面的代码我们可以发现相同的输入，再第二次调用的时候都是直接取的缓存
let squareNumber  = memoize((x) => { return x*x; });
squareNumber(4);
//=> 16
squareNumber(4); // 从缓存中读取输入值为 4 的结果
//=> 16
squareNumber(5);
//=> 25
squareNumber(5); // 从缓存中读取输入值为 5 的结果
//=> 25
```
这是怎么实现的呢? 请看下面的代码:
```js
const memoize = (f) => {
  // 由于使用了闭包，所以函数执行完后 cache 不会立刻被回收
  const cache = {};
  return () => {
    var arg_str = JSON.stringify(arguments);
    // 利用 cache 做一个简单的缓存，当这个参数之前使用过时，我们立即返回结果就行
    cache[arg_str] = cache[arg_str] || f.apply(f, arguments);
    return cache[arg_str];
  };
};
```

### 并行处理

纯函数不会访问共享的内存，因此不用担心线程的执行顺序，对任何纯表达式的求值都是线程安全的。

```js
var x = f(a);
var y = g(b);
var z = h(c);
// 线程安全
var result = x + y + z;
```

前三个表达式之间没有数据依赖关系，它们的执行顺序可以颠倒，或者并行执行也互不干扰。只要它们能在分配给 result 之前执行。

说了这么多优点，其实纯函数的优秀的原因是因为它不使用全局引用：

{{< admonition note "大神语录" >}}
Shared mutable state is the root of all evil(共享的可变状态是万恶之源)  -- Pete Hunt
{{< /admonition >}}

## 应用和组合函数

### 高阶函数

在数学和计算机科学中，[高阶函数](https://en.wikipedia.org/wiki/Higher-order_function) 是至少执行以下一项的函数：

- 将一个或多个函数作为参数(即过程参数)
- 返回一个函数作为其结果

ES6 中常用的高阶函数包括：map，filter，reduce，find，some，every 等。

```js
// 数组求和
const arr = [5, 7, 1, 8, 4];

// 不使用高阶函数
let sum = 0;
for (let i = 0; i < arr.length; i++) {
  sum = sum + arr[i];
}
console.log(sum);  //25

// 使用高阶函数
const sum = arr.reduce((accumulator, currentValue) => accumulator + currentValue,0);
console.log(sum);  //25

```

### 闭包

通常情况下我们说的 [闭包](https://en.wikipedia.org/wiki/Closure_(computer_programming)) 指的是函数内部的函数。闭包的形成条件：

- 存在内、外两层函数
- 内层函数对外层函数的局部变量进行了引用

闭包的用途：定义一些作用域局限的持久化变量，这些变量可用来做缓存或者计算的中间量等。

闭包的弊端：持久化变量不会被正常释放，持续占用内存造成内存浪费，所以需要额外的手动清理机制。

```js
// 匿名函数创造了一个闭包，实现简单的缓存工具
const cache = (function() {
  const store = {};
  
  return {
    get(key) {
      return store[key];
    },
    set(key, val) {
      store[key] = val;
    }
  }
}());
console.log(cache) //{get: ƒ, set: ƒ}
cache.set('a', 1);
cache.get('a');  // 1
```

### 柯里化

[柯里化](https://en.wikipedia.org/wiki/Currying) 是一种将多参函数拆解为多个单参函数序列的技术。

```js
function curryIt(fn) {
  // 参数fn函数的参数个数
  var n = fn.length;
  var args = [];
  return function(arg) {
    args.push(arg);
    if (args.length < n) {
      return arguments.callee; // 返回这个函数的引用
    } else {
      return fn.apply(this, args);
    }
  };
}

function add(a, b, c) {
  return [a, b, c];
}
// c 是内部匿名函数
var c = curryIt(add); 

// 可以分步传参
var c1 = c(1);        // 将 1 加入 args 中，返回 c 的引用
var c2 = c1(2);       
var c3 = c2(3);       // [1, 2, 3]

// 也可以直接调用
var c3 = c(1)(2)(3);  // [1, 2, 3]
```

可以看出，柯里化是一种函数的“预加载”技术，可以通过闭包实现对参数的缓存。

类似的概念有将多参函数拆解为任意参数个数的[部分函数应用](https://en.wikipedia.org/wiki/Partial_application)：

```js
// Currying  f(a)(b)(c)
var f = a => b => c => a + b + c;

// Partial application  f(a)(b,c) 
var f = a => (b, c) => a + b + c;
```

### 函数组合

柯里化是函数的拆解，函数组合就是多个函数组合为一个函数。compose 简单实现：

```js
var compose = (f, g) => x => f(g(x));

var g = x => x + 1;
var f = x => x * 5;
var fg = compose(f, g); 
fg(2); // 15
```

我们要合成的函数可能不止两个，更通用的 compose 实现：

```js
function compose(...args) {
    return function(x) {
        var composeFun = args.reduceRight(function(first, second) {
          //从右边开始迭代，这里实际是把右边放入左边
            return second(first); 
        }, x);
        return composeFun;
    }
};
// 简化为箭头函数
var compose = (...args)=>(x)=> args.reduceRight((f,s)=>s(f),x);
```

现在我们可以自由组合函数：

```js
function addHello(str){
    return 'hello ' + str;
}
function toUpperCase(str) {
    return str.toUpperCase();
}
function reverse(str){
    return str.split('').reverse().join('');
}

var composeFn=compose(reverse,toUpperCase,addHello);
composeFn('ttsy');  // YSTT OLLEH
```

最后，软件工程没有银弹。每种编程范式各有利弊，我们要根据实际需求选择合适的编程范式。

**参考资料**

- [维基百科](https://www.wikipedia.org/)
- [JavaScript函数式编程入门经典](https://juejin.im/post/6844903837715660814)
