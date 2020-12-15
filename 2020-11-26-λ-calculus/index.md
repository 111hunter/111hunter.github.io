# λ 演算: 程序从无到有


由邱奇 (Alonzo Church) 创造的 [λ 演算(λ-calculus)](https://en.wikipedia.org/wiki/Lambda_calculus) 是世界上最小的程序设计语言。虽然没有数(number)，字符串(string)，布尔型(boolean) 或其他任何非函数(non-function) 的数据类型，但 λ 演算只用匿名单参函数就能模拟图灵机(图灵完备)。

## λ 演算

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

若要图灵完备，必然能实现无限递归。而 λ 演算中只有匿名单参函数，也就是没有提前的函数声明，这能实现无限递归？真的能！

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

Y 能够组合一个匿名函数成为递归函数，因此被称为 Y combinator(Y组合子)。由于 lambda 表达式是[惰性求值](https://en.wikipedia.org/wiki/Lazy_evaluation)，而很多编程语言例如 JS 中使用[严格求值](https://en.wikipedia.org/wiki/Eager_evaluation)，因此 Y组合子在 JS 中这样表示：

```js
let Y = f => (x => f(y => x(x)(y)))(x => f(y => x(x)(y)));
```

Church 编码和不动点组合子表明了程序不用其他原始数据类型就能模拟图灵机。

## 实践

代码实践参考了[计算的本质](https://www.ituring.com.cn/book/1098)一书，原文是 Ruby 代码，我把它翻译为 JS 代码了：

```js
const Y = f => (x => f(y => x(x)(y)))(x => f(y => x(x)(y)));

const F = a => b => b;
const T = a => b => a;

const zero = f => x => x;
const one = f => x => f(x);
const two = f => x => f(f(x));
const three = f => x => f(f(f(x)));
const four = f => x => f(f(f(f(x))));
const five = f => x => f(f(f(f(f(x)))));

const pair = x => y => z => z(x)(y);
const left = p => p(x => y => x);
const right = p =>p(x => y => y);
const empty = pair(T)(T);
const ushift = l => x => pair(F)(pair(x)(l));
const is_empty = left;
const first = l => left(right(l));
const rest = l => right(right(l));

const if_else = b => b;
const is_zero = f => f(x => F)(T);
const is_less_or_equal = m => n => is_zero(minus(m)(n));

const succ = n => f => x => f(n(f)(x));
const slide = p => pair(right(p))(succ(right(p)));
const pred = n => left(n(slide)(pair(zero)(zero)));

const plus = m => n => n(succ)(m);
const minus = m => n => n(pred)(m);
const mult = m => n => n(plus(m))(zero);
const div = Y(f => m => n =>
    if_else(is_less_or_equal(n)(m))
        (x=>succ(f(minus(m)(n))(n))(x))
        (zero)
);

const a = two;
const b = succ(a);
const aa = ushift(ushift(empty)(a))(a);
const ab = ushift(ushift(empty)(b))(a);
const abaa = ushift(ushift(aa)(b))(a);

const to_boolean = p => if_else(p)('T')('F');
const to_char = c => 
    if_else(is_zero(c))('0')(
        if_else(is_zero(pred(c)))('1')(
            if_else(is_zero(two(pred)(c)))('a')('b')
        )
    );
const fold = Y(f => l => x => g => 
    if_else(is_empty(l))
        (x)
        (y=>g(f(rest(l))(x)(g))(first(l))(y))
);
const pushs = l => x => fold(l)(ushift(empty)(x))(ushift);
const to_digits = Y(f => n => pushs(
    if_else(is_less_or_equal(n)(pred(two)))
        (empty)
        (x => f(div(n)(two))(x))
    )(mod(n)(two))
)
const mod = Y(f => m => n => 
    if_else(is_less_or_equal(n)(m))
        (x => f(minus(m)(n))(n)(x))
        (m)
);
const range = Y(f => m => n => 
    if_else(is_less_or_equal(m)(n))
        (x => ushift(f(succ(m))(n))(m)(x))
        (empty)
);
const maps = k => f => fold(k)(empty)(l => x => ushift(l)(f(x)));
const twenty = mult(four)(five);

const my_list = maps(range(one)(twenty))(n => 
    if_else(is_zero(mod(n)(succ(five))))(abaa)(
        if_else(is_zero(mod(n)(three)))(aa)(
            if_else(is_zero(mod(n)(two)))(ab)(to_digits(n))
        )
    )        
);

// The above code only uses functions to compconste all calculations, 
// and the calculation result is a single-character linked list

// But the above code does not encode the characters related to the output format, 
// so use an array to store the result to change the output format

const to_array = proc => {
    const arr = [];
    while(to_boolean(is_empty(proc))!='T'){
        arr.push(first(proc));
        proc = rest(proc);
    }
    return arr;
}
const to_string = s => to_array(s).map(c => to_char(c)).join('');
console.log(to_array(my_list).map(v=>to_string(v)));

// If you don’t use arrays, you can also use functions to simulate

const fact = Y(f => n => is_zero(n)(one)(
    x => mult(n)(f(pred(n)))(x)
));

const arr = s => to_char(first(s));
const s1 = to_digits(fact(four));
const s2 = rest(s1);
const s3 = rest(s2);
const s4 = rest(s3);
const s5 = rest(s4);

// console.log(arr(s1) + arr(s2) + arr(s3) + arr(s4) + arr(s5));
```

附：[源码地址](https://github.com/111hunter/process-zero-to-hero)

**参考资料**

- [康托尔、哥德尔、图灵——永恒的金色对角线](http://mindhacks.cn/2006/10/15/cantor-godel-turing-an-eternal-golden-diagonal/)
- [Church encoding](http://www.cse.unt.edu/~tarau/teaching/PL/docs/Church%20encoding.pdf)
- [Learn Lambda Calculus in Y minutes](https://learnxinyminutes.com/docs/lambda-calculus/)
- [Lambda演算系列](http://cgnail.github.io/academic/lambda-index/)

