# 程序解释与编译


程序的解释和编译通常需要经过词法分析，语法分析和生成抽象语法树等阶段。

## 前置知识

算术表达式根据运算符所在的位置可以分为三种表示方法：

- 前缀表达式(波兰式)，如 `(- (+ 3 (* 2 4)) 1)`，Lisp 语言就是使用这种表示方法
- 中缀表达式，如 `3 + 2 * 4 - 1`，最适合人阅读的表示方法
- 后缀表达式(逆波兰式)，如 `3 2 4 * + 1 -`，计算机处理起来比较方便

## 编译原理

为了讲清楚程序解释与编译，我们自定义一种类似 Lisp 的前缀表达式：

```js
mul 3 sub 2 sum 1 3 4
```

### 语法和语义

为了完整地定义编程语言，我们需要：

- [语法(Syntax)](https://en.wikipedia.org/wiki/Syntax_(programming_languages))，就是程序看起来的样子(我们已经定义了)。
- [语义(semantics](https://en.wikipedia.org/wiki/Semantics_(computer_science))，描述程序的含义。一些编程语言有官方的书面规范。而另一些只有一个可用的解释器或者编译器，它们的语义是 “靠实现规范” 的。

我们用 JS 代码来规范前缀表达式的语义：

```js
const OpMapper = {
  sum: args => args.reduce((a, b) => a + b, 0),
  sub: args => args.reduce((a, b) => a - b),
  div: args => args.reduce((a, b) => a / b),
  mul: args => args.reduce((a, b) => a * b, 1)
};
```

按照我们规范的语义，前缀表达式等价为如下 JS 代码：

```js
mul(3, sub(2, sum(1, 3, 4)))
// or 
3 * (2 - (1 + 3 + 4))
```

### 词法分析

词法分析将源代码中每一个有语义的字符(token)提取出来，用数组表示。

```js
const lex = str => str.split(' ').map(s => s.trim()).filter(s => s.length);
const tokens = lex('mul 3 sub 2 sum 1 3 4')
// tokens = ["mul", "3", "sub", "2", "sum", "1", "3", "4"]
```

### 语法分析

#### 语法描述

用 [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) 来描述我们的程序语法：

```js
digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
num = digit+
op = sum | sub | mul | div
expr = num | op expr+
```

#### 抽象语法树

确定了语法后，一开始定义的前缀表达式可以表示为如下程序树(AST)：

![程序树(AST)](/img/expr-ast.jpg "程序树(AST)")

尝试用 JS 代码解析这种逻辑：

```js
const Op = Symbol('op');
const Num = Symbol('num');

const parse = tokens => {
  let c = 0;
  const peek = () => tokens[c];
  const consume = () => tokens[c++];

  const parseNum = () => ({ val: parseInt(consume()), type: Num });
  const parseOp = () => {
    const node = { val: consume(), type: Op, expr: [] };
    while (peek()) node.expr.push(parseExpr());
    return node;
  };
  const parseExpr = () => /\d/.test(peek()) ? parseNum() : parseOp();

  return parseExpr();
};
```

程序树中每个节点都被表示为了 JS 对象，我们完成了程序的解析！

实际上，我们开发了一个简单的[递归下降解析器](https://en.wikipedia.org/wiki/Recursive_descent_parser)。每个对象的值其实是它的 val 属性，可以通过[分治法](https://en.wikipedia.org/wiki/Divide-and-conquer_algorithm)自顶向下地对整个前缀表达式进行求值。

```js
const evaluate = ast => {
  if (ast.type === Num) {
    return ast.val;
  } else {
    // Op needs parameters to be evaluated
    return OpMapper[ast.val](ast.expr.map(evaluate));
  }
};

const value = evaluate(parse(tokens));
console.log(value);
// -18
```

### 解释和编译

将程序转换为 AST，然后直接对 AST 求值就是程序的解释(Interpreted)，还有一种求值方式是由 AST 生成中间代码，再由别的解释器或编译器对中间代码求值，就是程序的编译(Compiled)。通过 AST 完成代码转换非常方便，只需设计转换前后的映射表，代码转换就是查表替换。

```js
const compile = ast => {
  const opMap = { sum: '+', mul: '*', sub: '-', div: '/' };
  const compileNum = ast => ast.val;
  const compileOp = ast => `(${ast.expr.map(compile).join(' ' + opMap[ast.val] + ' ')})`;
  const compile = ast => ast.type === Num ? compileNum(ast) : compileOp(ast);
  return compile(ast);
};

const newCode = compile(parse(tokens)); 
console.log(newCode);
// (3 * (2 - (1 + 3 + 4)))
```

这里生成的中间代码就可以被直接被低级一些的语言解释或编译。

**参阅阅读**

- [Implementing a Simple Compiler on 25 Lines of JavaScript](https://blog.mgechev.com/2017/09/16/developing-simple-interpreter-transpiler-compiler-tutorial/)
- [解谜计算机科学](http://www.yinwang.org/blog-cn/2018/04/13/computer-science)
