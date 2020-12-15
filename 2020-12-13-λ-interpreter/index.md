# 实现 λ 演算解释器


如果你没有听说过 λ 演算，可以阅读我的这篇[文章](https://111hunter.github.io/2020-11-26-%CE%BB-calculus/)。如果你没有编译原理相关知识，可以阅读我的这篇[文章](https://111hunter.github.io/2020-12-10-lex-parse)。首先介绍调度场算法，后面的实现中会用到。

## 调度场算法

调度场算法是一种将中缀表达式转换为后缀表达式的经典算法，由 Dijkstra 提出，因其操作类似于火车调车场而得名。为什么要转换，为了处理运算符的优先级：

<div align=center><img src="/img/shunting-yard.png" width="80%"></div>

之后，程序处理后缀表达式 output 就像玩消消乐。

### 处理括号

首先后缀表达式里没有括号，所以括号不应该被输出。由于括号里一定是一个完整的表达式，可以这样修改算法。把左括号的优先级当作最低，这样括号就不会立即出栈：

- 1.依次按顺序读入，
  - 读到数字：直接输出；
  - 读到一般运算符：如果栈顶的运算符优先级不低于该运算符，则输出栈顶运算符并使之出栈，直到栈空或不满足上述条件为止；然后入栈；
  - 读到左括号：直接入栈；
  - 读到右括号：输出栈顶运算符并使之出栈，直到栈顶为左括号为止；令左括号出栈。
- 2.当读入完毕时，依次输出并弹出栈顶运算符，直到栈被清空。

这样处理后，就能让括号内的运算符优先级最高。

除括号外，在 λ 演算有语义的运算符只有 `.` 和函数与参数之间的空格。`.` 用于区分函数变量和函数体，空格表示将参数作用于函数。它们都是中缀表达式，我们用调度场算法把它们转换为后缀表达式。

## 词法分析

对任意的 λ 演算表达式，只保留函数间的空格。

```js
/** remove any space that isn't in one of the following spots:
)_(, x_(, )_x, x_x, x_\, )_\ */
function remove_extra_spaces(str) {
    return str.trim()
    .replace(/\s+([^\(\wλ])/g, '$1')
    .replace(/([^\)\w])\s+/g, '$1');
}
// all spaces are lambda application => whitespace matters
function lex(str) {
    let tokens = remove_extra_spaces(str)
    .split(/(\)|\(|λ|\.|\w+|\s+)/)
    .filter(t => t != '');
    return tokens;
}
```

```js
lex(' ( λ a. λ b. a ) a b ')
// ["(", "λ", "a", ".", "λ", "b", ".", "a", ")", " ", "a", " ", "b"]
```

## 语法分析

按照调度场算法处理运算符，空格的优先级高于 `.`，因为函数的参数是函数。设计如下[递归下降解析器](https://en.wikipedia.org/wiki/Recursive_descent_parser)：

```js
// shunting yard algorithm
function parse(tokens) {
    let output = [];
    let stack = [];
    while(tokens.length > 0) {
        let current = tokens.shift();

        if(current.match(/\w+/)) {
            output.push(new Var(current));
        }
        else if(current.match(/\s+/)) { // swap if both o1 and o2 are application
            stack_to_output(stack, output, () => stack[stack.length-1].match(/\s+/));
            stack.push(current);
        }
        else if(current == "(" || current == '.') {
            stack.push(current);
        }
        else if(current == ")") {
            stack_to_output(stack, output, () => stack[stack.length-1] != '(');
            if(stack.length == 0) {
                console.log("mismatched parenthesis");
            }
            stack.pop(); // pop off left paren
        }
    }
    if(stack.indexOf('(') != -1 || stack.indexOf(')') != -1) {
        console.log("mismatched parenthesis");
    } else {
        stack_to_output(stack, output, () => true);
    }
    return output.pop();
}
```

加入运算符到后缀表达式 output

```js
function stack_to_output(stack, output, condition) {
    while(stack.length > 0 && condition()) {
        let top = stack.pop();
        let s = output.pop();
        let f = output.pop();
        output.push(top == '.' ? new Abs(f, s) : new App(f, s));
    }
}
```

这里用到了 AST 节点，给出定义：

```js
function Var(id) {
    this.type = 'var';
    this.id = id;
    this.free_vars = new Set([id]);
}

function App(func, arg) {
    this.type = 'app';
    this.func = func;
    this.arg = arg;
    this.free_vars = new Set([...func.free_vars, ...arg.free_vars]);
}

function Abs(v, expr) {
    this.type = 'abs';
    this.var = v;
    this.expr = expr;
    this.free_vars = new Set([...expr.free_vars]);
    this.free_vars.delete(this.var.id);
}
```

只有 `abs` 类型里的参数 `var` 是约束变量，例如 `(λa.b)` 生成的 AST 如下

```js
Abs {
  type: 'abs',
  var: Var { type: 'var', id: 'a', free_vars: Set(1) { 'a' } },
  expr: Var { type: 'var', id: 'b', free_vars: Set(1) { 'b' } },
  free_vars: Set(1) { 'b' }
}
```

`(λa.b) c` 生成的 AST 如下

```js
App {
  type: 'app',
  func: Abs {
    type: 'abs',
    var: Var { type: 'var', id: 'a', free_vars: [Set] },
    expr: Var { type: 'var', id: 'b', free_vars: [Set] },
    free_vars: Set(1) { 'b' }
  },
  arg: Var { type: 'var', id: 'c', free_vars: Set(1) { 'c' } },
  free_vars: Set(2) { 'b', 'c' }
}
```

## 实现解释器

在每个 AST 节点上封装一个 `stepped` 布尔值，表示对应的表达式是否能够 β 规约。只有在调用函数时，才能进行 β 规约，替换掉函数里的约束变量。代码的实现思路是观察 AST 的结构，例如 `(λa.b) c` 能够被替换的是 `(λa.b)`，对应的 AST 结构是 `app` 里的 `abs`

```js
var redexes = 0;
function stepper(node) {
    switch (node.type) {
        case 'var': return { stepped: false, node: node };
        case 'app':
            switch(node.func.type) {
                case 'var':
                case 'app':
                    let func_evaled = stepper(node.func);
                    if (func_evaled.stepped) {
                        return {stepped: true, node: new App(func_evaled.node, node.arg)};
                    }
                    let arg_evaled = stepper(node.arg);
                    return {stepped: arg_evaled.stepped, node: new App(node.func, arg_evaled.node)};
                case 'abs': // redex
                    redexes++;
                    return {stepped: true, node: substitute(node.arg, node.func.var, node.func.expr)};
            }
            break;
        case 'abs':
            let new_expr = stepper(node.expr);
            return { stepped : new_expr.stepped, node : new Abs(node.var, new_expr.node) };
    }
}
```

实现替换函数 `substitute`，代码的实现思路同样是观察 AST 的结构

```js
// substitute e for x (variable) in expr
function substitute(e, x, expr) {
    switch (expr.type) {
        case 'var': return expr.id == x.id ? e : expr;
        case 'app': return new App(substitute(e, x, expr.func), substitute(e, x, expr.arg));
        case 'abs':
            if(expr.var.id == x.id) {
                return expr;
            }
            else if(!e.free_vars.has(expr.var.id)) {
                return new Abs(expr.var, substitute(e, x, expr.expr));
            }
            else {
                do {
                    var z = rename(expr.var.id);
                } while(e.free_vars.has(z) || variables(expr.expr).has(z));
                return new Abs(new Var(z), substitute(e, x, substitute(new Var(z), expr.var, expr.expr)));
            }
    }
}
```

进行 β 规约时，约束变量的名字可能重复，这就需要使用 α 变换重命名约束变量

```js
function rename(variable) {
    let [match, prefix, num] = /^(.*?)([\d]*)$/.exec(variable);
    return prefix + (num == '' ? 1 : parseInt(num) + 1);
}

function variables(expr) {
    switch (expr.type) {
        case 'var': return new Set([expr.id]);
        case 'app': return new Set([...variables(expr.func), ...variables(expr.arg)]);
        case 'abs': return new Set([...variables(expr.expr), expr.var.id]);
    }
}
```

最后实现一个由 AST 节点重新生成 λ 演算表达式的函数

```js
// unnecessary parenthesises uses in some abstractions
function ast_to_expr(expr) {
    switch (expr.type) {
        case 'var': return expr.id;
        case 'abs': return `(λ${expr.var.id}.${ast_to_expr(expr.expr)})`;
        case 'app':
            return ast_to_expr(expr.func) + ' ' + (expr.arg.type == 'app' ? 
            '('+ast_to_expr(expr.arg)+')' : 
            ast_to_expr(expr.arg));
    }
}
```

现在可以自顶向下的解释 λ 演算表达式的计算规则了

```js
let expr = '(λn. (λf. (λx. (f ((n f) x))))) (λf. (λx. x))';
function run(expr) {
    let t = new Date().getTime();
    let ast = parse(lex(expr));
    let new_expr = stepper(ast);
    while(new_expr.stepped) {
        console.log(redexes + ': ' + ast_to_expr(new_expr.node));
        new_expr = stepper(new_expr.node);
    }
    let delay = new Date().getTime() - t;
    console.log('delay: ' + delay);
    console.log('redexes: ' + redexes);
    console.log('final: ' + ast_to_expr(new_expr.node));
}

run(expr);
```

expr 中第一个表达式是数字的后继函数，第二个表达式是数字 0 

```js
1: (λf.(λx.f ((λf.(λx.x)) f x)))
2: (λf.(λx.f ((λx.x) x)))
3: (λf.(λx.f x))
delay: 10
redexes: 3
final: (λf.(λx.f x))
```

可以看到经过 3 次计算后，最终的表达式是数字 1，我们实现了 λ 演算解释器！

附：[源码地址](https://github.com/111hunter/process-zero-to-hero)

**参考资料**

- [算法学习笔记: 调度场算法](https://zhuanlan.zhihu.com/p/147623236)
- [parkertimmins/lambda_interpreter](https://github.com/parkertimmins/lambda_interpreter)
