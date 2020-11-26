# λ-calculus: 重新认识计算


由邱奇 (Alonzo Church) 创造的 [Lambda 演算(λ-calculus)](https://en.wikipedia.org/wiki/Lambda_calculus) 是世界上最小的程序设计语言。虽然没有数(number)，字符串(string)，布尔型(boolean) 或其他任何非函数(non-function) 的数据类型，但 λ-演算只用匿名单参函数就能模拟图灵机(图灵完备)。

## Lambda 演算

λ 演算仅由 3 种元素组成：**变量**、**函数** 和 **应用**

<div align=center><img src="/img/lambda.png" width="100%"></div>

最基本的函数是恒等函数：$ λx.x $，等同于 $ f(x) = x $，第一个 x 是函数参数，第二个是函数体。

### 变量

λ 演算中的变量分为自由变量和约束变量：

- 在函数 $ λx.x $ 中 x 被称为约束变量，x 被称为约束变量，因为它既在函数体中又是形参。
- 在 $ λx.y $ 中 y 被称为自由变量，因为它没有被预先声明。

### 求值

求值通过 [β规约(β-Reduction)](https://en.wikipedia.org/wiki/Lambda_calculus#Beta_reduction) 完成，它将替换作用于应用，简单理解就是函数调用。例如对表达式 $ (λx.x)a $ 求值时，我们把函数体中所有的 x 都替换为 a：

- $ (λx.x) \ a = a $
- $ (λx.y) \ a = y $

你可以这样表示高阶函数：

- $ (λx.λy.x) \ a = λy.a $

为了简化表示，λx.λy. 与 λxy. 等价：

- $ (λxy.x) \ a = λy.a $

图灵机的逻辑操作实际上是二进制的 0 1 比较，而 λ 演算中连数字都没有，怎么定义程序逻辑？既然没有数，那就用 λ 函数来表示数！

### Church 编码

**邱奇数**

[邱奇数](https://en.wikipedia.org/wiki/Church_encoding#Church_numerals) 是用 λ 函数表示的自然数。某个程序过程(函数) f 和它的执行次数 n，存在对应关系：

$ 0 = λf.λx.x $<br>$1 = λf.λx.f x$<br>$ 2 = λf.λx.f (f x) $<br>$ 3 = λf.λx.f ( f (f x)) $<br>...

于是，

$ n = λf.λx.f^nx $

由 β规约得出：$ n \ f \ x = (λf.λx.f^nx) \ f \ x = f^nx $

由恒等式 $ f^{m+n}x = f^m(f^nx) $ 推出加法定义：

- $ plus = λm.λn.λf.λx.m \ f \ (n \ f \ x) $

由恒等式 $ f^{m*n}x = (f^n)^mx $ 推出乘法定义：

- $ mult = λm.λn.λf.m (n \ f) $

其他的运算都能用类似的方法推出。

**布尔逻辑**

布尔逻辑可被看做一种选择，ture 和 false 可以被编码为有两个参数的函数：

ture — 选择第一个参数

- $ true = λa.λb.a $

false — 选择第二个参数

- $ false = λa.λb.b $

布尔函数本身是条件分支，那 if-else 语句就成语法糖了。因为判定条件最终是布尔函数，直接能将 if-else 的条件分支作为布尔函数的参数(if-else 语句有三个参数，第一个是判定条件)：

- $ ifelse = λp.λt.λf. p \ t \ f $ &#8194; (p = true or p = false)

接着定义与、或、非的逻辑运算符：

- $ and = λp.λq. p \ q \ p $

- $ or = λp.λq. p \ p \ q $

- $ not = λp. p \ false \ true $

若要图灵完备，必然能实现无限递归。而 λ 演算中只有匿名单参函数，也就是没有函数声明，这能实现无限递归？真的能！

### 不动点组合子

**不动点**

函数 $ f $ 的不动点指的是将函数应用在输入值 x 时，会传回与输入值相同的值，使得 $ f(x) = x $。例如，0 和 1 是函数 $ f(x) = x^2 $ 的不动点。现在，假设有某个函数 Y 和任意函数 g，满足：

Y g = g (Y g) 

就是说将 g 作为 Y 的参数时，得到的新函数 Y g 是 g 的不动点。那神奇的事情就发生了：

Y g = g (Y g) = g (g (Y g)) = g (...g (Y g)...)

一旦调用 Y g 就会产生新的 g，任意函数 g 的无限递归不就产生了吗？Amazing！

**Y combinator**

数学家 [Haskell Curry](https://en.wikipedia.org/wiki/Haskell_Curry) 发现了这个 Y 的存在：

$ Y := λf.(λx.f(x \ x))(λx.f(x \ x)) $

证明过程：

<div align=center><img src="/img/y-combinator.png" width="100%"></div>

例如我们用匿名函数表达求阶乘时，我们暂称它为 g，实际的 g 和 Y 没有名字， 

g = λf. λx. (iszero x) 1 (mult x (f (pred x)))

当调用 Y g 时，得到 g(Y g)，由 β规约得出：
 
g(Y g) = λx. (iszero x) 1 (mult x ((Y g) (pred x))) 

于是，Y g 就成为了递归函数：

Y g = λx. (iszero x) 1 (mult x ((Y g) (pred x))) 

Y 能够组合一个匿名函数成为递归函数，因此被称为 Y combinator(Y组合子)。Church 编码和不动点组合子表明了程序不用其他原始数据类型就能模拟图灵机。

## 实践

直接将 λ演算翻译为 JS 代码：

```js
let mark = x => console.log("λ");

let F = a => b => b;
let T = a => b => a;

let zero = f => x => x;
let one = f => x => f(x);
let two = f => x => f(f(x));

let add = m => n => f => x => m(f)(n(f)(x));
let succ = n => f => x => f(n(f)(x));
let mult = m => n => f => m(n(f));

// console.log(T(mark)("no mark"));
// console.log(two(mark)("no mark"));
// console.log(mult(zero)(two)(mark)("no mark"));

let Y = (f =>
    (x => f(v => x(x)(v)))
    (x => f(v => x(x)(v)))
)
let fact = (mul => n => n < 2 ? 1 : n * mul(n - 1));

// console.log(Y(fact)(5));
```

**参考资料**

- [康托尔、哥德尔、图灵——永恒的金色对角线](http://mindhacks.cn/2006/10/15/cantor-godel-turing-an-eternal-golden-diagonal/)
- [Church encoding](http://www.cse.unt.edu/~tarau/teaching/PL/docs/Church%20encoding.pdf)
- [Learn Lambda Calculus in Y minutes](https://learnxinyminutes.com/docs/lambda-calculus/)
- [Lambda演算系列](http://cgnail.github.io/academic/lambda-index/)

