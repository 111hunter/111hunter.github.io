# 入门 AST 抽象语法树


如果你想了解 Javascript 的编译原理，那么你就得了解 AST(Abstract Syntax Tree)，目前前端常用的一些插件或者工具，比如 JS 转译、代码压缩、CSS 预处理器、ESLint、Prettier 等功能的实现，都是建立在 AST 的基础之上的。

## JS 编译流程

首先是 JS 引擎读取 JS 文件中的字符流，然后通过 **词法分析** 生成 tokens，之后再通过 **语法分析** 生成 AST，最终 JS 引擎将 AST 编译成字节码或机器码，然后再运行。

### 词法分析

词法分析，也称为扫描(scanner)，简单来说就是调用 next() 方法，一个一个字母的来读取字符，然后与定义好的 JavaScript 关键字符做比较，生成对应的 Token。Token 是 JS 代码在语法含义上不可分割的最小单元。除此之外，还会过滤掉源程序中的注释和空白字符(换行符、空格、制表符等)。

最终，整个代码被分割进一个 tokens 的数组中。如下代码：

```js
const href = 'https://github.com/'
```
经过词法分析生成类似这样的 tokens：
```json
[
    {
        "type": "Keyword",
        "value": "const"
    },
    {
        "type": "Identifier",
        "value": "href"
    },
    {
        "type": "Punctuator",
        "value": "="
    },
    {
        "type": "String",
        "value": "'https://github.com/'"
    }
]
```

### 语法分析

语法分析会将词法分析出来的 tokens 转化成有语法含义的 AST 结构。同时，验证语法，如果语法有错，抛出语法错误。

```json
{
  "type": "Program",
  "body": [
    {
      "type": "VariableDeclaration",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "href"
          },
          "init": {
            "type": "Literal",
            "value": "https://github.com/",
            "raw": "'https://github.com/'"
          }
        }
      ],
      "kind": "const"
    }
  ],
  "sourceType": "script"
}
```
[这里](https://esprima.org/demo/parse.html?code=const%20href%20%3D%20'https%3A%2F%2Fgithub.com%2F') 可以看到代码的转换。[这里](https://mp.weixin.qq.com/s/qTPhYD8PXpeAxwmdlUmdEA) 有 tokens 和 AST 的简单 JS 实现。

## AST 节点规范

业界已经有很多成熟的解析库，常用的库都集成在 [AST Explorer](https://astexplorer.net/) 中，可以实现代码与符合 [The ESTree Spec](https://github.com/estree/estree) 的 AST 之间的相互转换。下面对规范里的 ES5 的 API 做简要说明。

ESTree AST 中每个节点都要实现以下的 Node 接口，loc 字段表示相关代码的位置信息：

```ts
interface Node {
    type: string;
    loc?: SourceLocation;
}

interface SourceLocation {
    source: string | null;
    start: Position;
    end: Position;
}

interface Position {
    line: number; // >= 1
    column: number; // >= 0
}
```
### Programs 根节点

```ts
interface Program <: Node {
    type: "Program";
    body: [ Statement ];
}
```

AST 的顶部， body 包含了多个 Statement(语句)节点。

### Patterns 模式

```ts
interface Pattern <: Node { }
```

在 ES6 的解构赋值中有意义，如 `let {name} = user`，其中`{name}`部分为 ObjectPattern, 对于 ES5，唯一的子类是 Identifier

### Expression 表达式

```ts
interface Expression <: Node { }
```

表达式，子类很多，有二元表达式(n*n)、函数表达式(var fun = function(){})、数组表达式(var arr = [])、对象表达式(var obj = {})、赋值表达式( a=1)等。

### Identifier 标识符

```ts
interface Identifier <: Expression, Pattern {
    type: "Identifier";
    name: string;
}
```

写代码时自定义的名称，如变量名，函数名，属性名等。

### Literal 字面量

```ts
interface Literal <: Expression {
    type: "Literal";
    value: string | boolean | null | number | RegExp;
}
```

从 value 的类型可以看出，字面量就是值，他的类型有字符串，布尔，数值，null 和正则。

### Statement 语句

```ts
interface Statement <: Node { }
```

语句，子类有很多， 块语句、 if/switch语句、 return语句、 for/while语句、 with语句等。

### Declaration 声明

```ts
interface Declaration <: Statement { }
```

声明，子类主要有变量申明、函数声明。

ES6，7，8，... 的更多类型补充可以看这一篇 [文章](https://vincentstudio.info/2020/05/27/056_Javascript_ast/)。

## AST 的运用

将原代码转化为 AST，修改 AST，再重新转化为新代码就能完成代码转译。Babel 将最新语法的 JS 代码转化为 ES5 的原理就是这样的。

<div align=center><img src="/img/ast.png" width="90%"></div>

Babel 操作 AST 会用到以下工具包：

- @babel/parser 用于将代码转换为 AST
- @babel/traverse 用于对 AST 的遍历，包括节点增删改查、作用域等处理
- @babel/generator 用于将 AST 转换成代码
- @babel/types 用于 AST 节点操作的 Lodash 式工具库,各节点构造、验证等

更多api详见 [Babel手册](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/README.md)。

下面是用一个例子讲述具体操作步骤：

```js
var obj = {
  fn(){
    console.log("hello")
  }
}
```

我们需要把以上代码转换成下面这样：

```js
const obj = {
  fn(){
    console.log("hello","world")
  }
}
```

将两份代码在 [AST Explorer](https://astexplorer.net/) 中打开。选择 @babel/parser 为解析器，右边有选项隐藏不需要的属性。对比两颗 AST 发现差异是 kind 和 arguments，因此代码如下：

```js
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generate = require("@babel/generator").default;
const t = require("@babel/types");

let sourceCode = `
var obj = {
    fn(){
      console.log("hello")
    }
  }
`
let ast = parser.parse(sourceCode);
traverse(ast, {
    VariableDeclaration(path) {
        let { kind } = path.node
        if (kind === "var") {
            kind = "const"
        }
    },
    CallExpression(path) {
        let { callee, arguments } = path.node
        if (t.isMemberExpression(callee) && callee.object.name === "console" && callee.property.name === "log") {
            arguments.push(t.stringLiteral("world"))
        }
    }
})

console.log(generate(ast).code);
```

[这里](https://mp.weixin.qq.com/s/Fz9H5dscj5Oy__daecAYvg) 还有更多例子。

**参考资料**

- [JS之 执行过程](https://mp.weixin.qq.com/s/qTPhYD8PXpeAxwmdlUmdEA)
- [JS 语法树学习](https://vincentstudio.info/2020/05/27/056_Javascript_ast/)
- [Javascript抽象语法树](https://mp.weixin.qq.com/s/Fz9H5dscj5Oy__daecAYvg)

