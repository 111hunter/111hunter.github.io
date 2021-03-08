# [译]元编程


本文翻译自 stereobooster 的博客文章：[metaprogramming](https://stereobooster.com/posts/metaprogramming/)，少量删改。完整内容参阅原文。

## 什么是元编程

不好的是，没有一个达成共识的单一定义。让我们参阅一下：

>元编程是一种编程技术，指计算机程序具有将其他程序视为其数据的能力。 – [wikipedia](https://en.wikipedia.org/wiki/Metaprogramming)

>元编程是指程序具有了解自身或操纵自身的多种方式。 – [stackoverflow 上的流行答案](https://stackoverflow.com/questions/514644/what-exactly-is-metaprogramming)

>“支持元编程”意味着用户可以有效修改该语言内置语法(例如 Lisp 的宏)或扩展该语言常规语法(例如 C 的预处理程序)。 – [rosettacode](https://rosettacode.org/wiki/Metaprogramming)

没有一个很好的定义，让我们看一些例子。当人们谈论元编程时，他们可能指的是：

- macros in Lisp (1960)
- preprpcessor in C (1973)
- hygenic macros in Scheme (1986)
- C++ templates (1986)
- "Dynamic" metaprogramming in Smalltalk (1980) and Ruby (1995-2005?)
- Reflections in Java (1997)

## 两类元编程

元编程大致分为两类：

- 一种是(编译时)作为源代码(例如宏，预处理器，模板)，通常称为“宏”
- 另一种是(运行时)基于“OOP 技巧”(例如动态调度和反射)以支持其他行为，这没有名字，我把它称为“动态”

|   | compile time | runtime |
| :----: | :----: | :----: |
| macros in Lisp | ? | + |
| Preprocessor, templates | + |   |	
| Dynamic metaprogramming	|	  | + |

### 动态元编程

>元编程是编写在运行时操纵(自身)语言结构的代码。 – [Ruby 元编程](https://book.douban.com/subject/7056800/)

元编程在 Ruby 中比在其他的动态类型语言中更常用，尤其是在 Rails 中，例如：[Path and URL Helpers](https://guides.rubyonrails.org/routing.html#path-and-url-helpers)。动态元编程的缺点是“事物”没有源代码：你看到了一个函数，但是你不知道它的定义位置，这破坏了“[grep test](http://jamie-wong.com/2013/07/12/grep-test/)”。另一个缺点是它趋向于变慢，例如，参见 [Rails / DynamicFindBy](https://docs.rubocop.org/rubocop-rails/cops_rails.html#railsdynamicfindby)。

编程语言：
- [Ruby](http://media.pragprog.com/titles/ppmetr/spells.pdf)
- [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Meta_programming)
- [Python 2](https://python-3-patterns-idioms-test.readthedocs.io/en/latest/Metaprogramming.html), [Python 3](https://www.youtube.com/watch?v=sPiWg5jSoZI)

主要用法：
- 消除重复代码(DRY)，例如，David Beazley 在[演讲中](https://www.youtube.com/watch?v=sPiWg5jSoZI)展示了很多例子。
- 创建嵌入式领域特定语言(EDSL)，Martin Fowler 称它们为[内部 DSL](https://www.martinfowler.com/bliki/DomainSpecificLanguage.html)。例如，[Sass](https://github.com/sass/ruby-sass)(能转换为 CSS 的 Ruby EDSL)，[Haml](https://haml.info/)(能转换为 HTML 的 Ruby EDSL)，Active Record 查询接口(能转换为 SQL 的 Ruby EDSL)，最重要的是 Rake(替代 Make 的 Ruby EDSL)。
- “扩展语言”

#### 关于扩展语言

如何扩展语言？可以添加更多关键字(扩展词法)，也可以添加更多关键字的组合规则(扩展语法)。

我们很容易添加更多关键字，例如，定义新的函数、模块、变量，但并不是所有种类 -- 仅限于语言语法允许使用的标识符(例如，我不能定义 `:?:`)。在 Ruby 和 Python 中，可以重载运算符(`+`,`-`,`>`,`<` 等)，但不能定义新的运算符。据我所知，这些语言均不允许我定义新的语法规则，例如，我无法定义自己的 `if/else` 版本。

但程序员们总能找到一种方法来解决这个问题 -- 可以重用现有的语法，让它看起来像是另一种语法。例如，在函数式语言中，有一个漂亮的“模式匹配”的概念。[OCaml:](https://ocaml.org/learn/tutorials/data_types_and_matching.html)

```OCaml
match value with
| pattern -> result
| pattern -> result
```

或是 [Scheme:](https://www.gnu.org/software/guile/manual/html_node/Pattern-Matching.html)

```lisp
(let ((l '(hello (world))))
 (match l
 ((x y)
 (values x y))))
```

这是在 JavaScript 中的实现：

```js
const { matches } = require("z");
const result = matches(1)(
 (x = 2) => "number 2 is the best!!!",
 (x = Number) => `number ${x} is not that good`,
 (x = Date) => "blaa.. dates are awful!"
);
```

这是一个旧语法，但如果你细心，它看起来就像 OCaml 中的模式匹配。在幕后，它使用 `toString` 检查实际的代码，因为以前没有一等公民的反射对象。另一个值得注意的技术是“链式”(例如 jQuery 和 Active Record 查询接口)。

### Macros(宏)

宏是一个宽泛的范畴，让我们看一下使用示例来了解这一点。

#### 语法扩展

在 Lisp 中 `if/else` 表达式像这样：

```lisp
(if condition
 (print 1)
 (print 2))
```

定义具有相同结构的函数很容易：

```lisp
(my-if condition
 (print 1)
 (print 2))
```

关键在于，Lisp 中的函数是立即执行的。这意味着在将参数传递给函数之前，它就会执行 `then` 和 `else` 两个分支，这就是宏的作用。有了宏，就可以定义自己的 `if` 版本，像你期望的那样。

另请参阅：

- [Idris Syntax Extensions](http://docs.idris-lang.org/en/latest/tutorial/syntax.html)
- [Racket Module Syntax](https://docs.racket-lang.org/guide/Module_Syntax.html), [Racket Macros](https://docs.racket-lang.org/reference/Macros.html)

#### DSL

>JSX 是 ECMAScript 中类似 XML 的语法扩展，没有任何定义的语义  – [Draft: JSX Specification](https://facebook.github.io/jsx/)

它本质上是一个 DSL。而负责编译它的 Babel 插件是一个预处理器。你可以使用其他的元编程技术来实现同样的结果 -- 参见 [JSX 的替代方案](https://blog.bloomca.me/2019/02/23/alternatives-to-jsx.html)。

#### 多态性

>...多态语言，其中一些值和变量可能有一个以上的类型。多态函数是指其操作数(实际参数)可以有一个以上类型的函数。多态类型是指其操作可以适用于一种以上类型的值的类型。 – [On Understanding Types, Data Abstraction, and Polymorphism](http://lucacardelli.name/Papers/OnUnderstanding.A4.pdf)

令我惊讶的是:

- 动态类型语言，是非常灵活的(但也很容易给自己找麻烦)。
- 静态类型的语言，具有完全的多态性支持，如 OCaml，Haskell 等。
- 没有多态性或在多态性上有一定限制的静态类型语言(Pascal，Go)。

最后一类编程语言可以使用元编程来实现类似多态性的东西("提高灵活性")。在 GO 中，没有参数多态(或类型参数，或泛型)，于是人们创造了解决方法，例如，[gengen](https://github.com/joeshaw/gengen)(类似的解决方案 [genny](https://github.com/cheekybits/genny)，[generic](https://github.com/taylorchu/generic)，[gen](https://github.com/clipperhouse/gen))。

```go
package list

import "github.com/joeshaw/gengen/generic"

type List struct {
    data generic.T
    next *List
}
```

然后，您需要运行预处理器：

```bash
$ gengen github.com/joeshaw/gengen/examples/list string
```

你会得到类型准确的代码:

```go
package list

type List struct {
    data string
    next *List
}
```

另请参阅：[Who needs generics? Use … instead!](https://appliedgo.net/generics/), [The Next Step for Generics.](https://blog.golang.org/generics-next-step)

#### DRY

>模板元程序员利用这种机制来提高：源代码的灵活性和运行时性能。 – [Walter E. Brown “Modern Template Metaprogramming: A Compendium, Part I”](https://www.youtube.com/watch?t=607&v=Am2is2QCvxY&feature=youtu.be)

在C++中，有函数重载(即一种多态)，但它会产生很多重复：

```cpp
double abs(double x) {
 return (x >= 0) ? x : -x;
}
int abs(int x) {
 return (x >= 0) ? x : -x;
}
```

你可以编写函数模板：

```cpp
template<typename T>
T abs(T x) {
  return (x >= 0) ? x : -x;
}
```

### 性能

通常认为，在编译时进行宏扩展可以提高性能。对我来说这很合理，但我没有很好的例子。

相关：[Compile-time reflection and compile-time code execution in Zig.](https://ziglang.org/#Compile-time-reflection-and-compile-time-code-execution)

### 宏和类型

Lisp(和Scheme)宏非常强大，但它们不能与静态类型检查器一起很好地工作。假设我们有确保能够终止的宏，并且能在编译时扩展(语法糖)并进行类型检查生成的代码，下一个问题是在生成的代码中报告类型错误，这也会很混乱。

有多种尝试使宏与静态类型更好地配合使用，例如：

- [Hackett](https://lexi-lambda.github.io/hackett/index.html)
- [Inferring Type Rules for Syntactic Sugar](https://cs.brown.edu/~sk/Publications/Papers/Published/pk-resuarging-types/paper.pdf)

**参考资料**

- [Metaprogramming](https://stereobooster.com/posts/metaprogramming/)
- [谈元编程与表达能力](https://draveness.me/metaprogramming/)
- [Emacs之魂：宏与元编程](https://zhuanlan.zhihu.com/p/34106430)
