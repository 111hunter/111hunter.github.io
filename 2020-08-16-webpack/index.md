# Webpack 概念梳理


Webpack 是一个前端构建工具，前端构建工具的作用就是把开发环境的代码转化成运行环境代码。一般来说，开发环境的代码是为了更好的阅读，而运行环境的代码则是为了能够更快地执行。因此开发环境和运行环境的代码形式也不相同。比如，开发环境的代码，要通过混淆压缩后才能放在线上运行，这样代码体积更小，但对代码执行不会有任何影响。

<!-- <div align=center><img src="/img/webpack.png" width="90%"></div> -->

## 应用场景

一般的构建工具可以处理但不限于以下场景：

- 代码压缩

将 JS、CSS 代码混淆压缩，让代码体积更小，加载更快。

- 语法编译

编写CSS时使用 Less、Sass，编写 JS 时使用 ES6、TypeScript 等，这些标准目前都无法被浏览器兼容，因此需要构建工具编译，例如使用 Babel 编译 ES6 语法。

- 模块化处理

CSS 和 JS 的模块化语法，目前无法被浏览器兼容。因此开发环境可以使用既定的模块化语法，但是需要构建工具将模块化语法编译为浏览器可识别形式。例如使用 Webpack、Rollup 等处理 JS 模块化。

使用 webpack，构建的前端项目是高度可配置的(替换 react，vue 默认 cli 工具)。

## 核心概念

以下概念提取自 webpack 的官方文档，学习更多细节请参阅[官方文档](https://www.webpackjs.com/)。

### Entry 

入口起点(entry point)指示 webpack 应该使用哪个模块,来作为构建其内部依赖图的开始。

进入入口起点后, webpack 会找出有哪些模块和库是入口起点(直接和间接)依赖的。

每个依赖项随即被处理,最后输出到称之为 bundles 的内存文件中。

### Output 

output 属性指定 webpack 在哪里输出它所创建的 bundles,以及如何命名这些文件,默认值为 ./dist。

基本上,整个应用程序结构,都会被编译到你指定的输出路径的文件夹中。

### Module 

模块,在 Webpack 里一切皆模块,一个模块对应着一个文件。Webpack 会从配置的 Entry 开始递归找出所有依赖的模块。

### Chunk 

代码块,一个 Chunk 由多个模块组合而成,用于代码合并与分割。

### Loader 

loader 让 webpack 能够去处理那些非 JS 文件(webpack 自身只理解 JS)。

loader 可以将所有类型的文件转换为 webpack 能够处理的有效模块,然后你就可以利用 webpack 的打包能力,对它们进行处理。

本质上,webpack loader 将所有类型的文件,转换为应用程序的依赖图(和最终的 bundle)可以直接引用的模块。

### Plugin 

loader 被用于转换某些类型的模块,而插件则可以用于执行范围更广的任务。

插件的范围包括,从打包优化和压缩,一直到重新定义环境中的变量。插件接口功能极其强大,可以用来处理各种各样的任务。

## 构建流程

Webpack 在启动后，会从 Entry 开始，递归解析 Entry 依赖的所有 Module，每找到一个 Module，就会根据 Module.rules 里配置的 Loader 规则进行相应的转换处理，对 Module 进行转换后，再解析出当前 Module 依赖的Module，这些 Module 会以 Entry 为单位进行分组，即为一个 Chunk。因此一个 Chunk 就是一个 Entry 及其所有依赖的 Module 合并的结果。最后 Webpack 会将所有的 Chunk 转换成文件输出 Output。在整个构建流程中，Webpack 会在恰当的时机执行 Plugin 里定义的逻辑，从而完成 Plugin 插件的优化任务。

简单的解释就是这样，详细构建流程请看这篇[文章](https://juejin.im/post/6844904038543130637)。

## 配置入门

下面以一个配置一个 react 开发环境为例，学习 webpack 的基本配置方法。

新建文件夹 webpack-demo，终端进入文件夹执行 ` npm init ` 初始化项目。

安装 react: ` yarn add react react-dom `

安装 webpack: ` yarn add webpack webpack-cli webpack-dev-server -D `

- webpack-cli 提供了一组用于运行和设置 webpack 的命令
- Webpack-dev-server 提供 http 服务，实时重载(hot模式)，cors 配置，端口配置等

安装 babel: ` yarn add @babel/core @babel/preset-react @babel/preset-env babel-loader -D`

- @babel/core 是核心依赖项
- @babel/preset-react 添加对 JSX 支持
- @babel/preset-env 添加对 ES6 的支持
- babel-loader 使用 Babel 和 webpack 转换 react 代码为 JS

安装 CSS Loaders: `yarn add css-loader style-loader -D`

- css-loader 从收集 CSS 并将 CSS 转化为字符串
- style-loader 将从 css-loader 中获得的字符串嵌入在 html 中的 style 标签中

安装插件: `yarn add html-webpack-plugin -D`

- html-webpack-plugin 用于将生成的 output 文件嵌入到指定 html 文件

准备文件: 在根文件夹下创建 src 和 dist 文件夹，在 src 文件夹下创建 main.js，app.js，index.css，
在 dist 文件夹下创建 index.html。

创建 webpack.config.js，这是默认的 webpack 配置文件：

```js
//webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
module.exports = {
    entry: './src/main.js',
    output: {
        path: path.join(__dirname, '/dist'),
        filename: 'bundle.js'
    },
    devServer: {
        port: 8080
    },
    module: {
        rules: [
            {
                test: /\.jsx?$/,
                exclude: /node_modules/,
                loader: 'babel-loader',
            },
            {
                test: /\.css$/,
                use: ['style-loader', 'css-loader']
            }
        ]
    },
    plugins: [
        new HtmlWebpackPlugin({
            template: './dist/index.html'
        })
    ]
}
```

- "entry": 这是入口 js，webpack将从此处开始打包。
- "output": 打包的文件将位于 "/dist/bundle.js"。
- "devServer": 它定义了 weback-dev-server 的配置，开发服务器的默认端口是8080。
- 模块规则-这些是转译规则：
  - "test": 正则表达式，指定哪种文件需要通过 loader 转译。
  - "exclude": 指定 loader 应忽略的文件。
  - "use": 应用 loader 的数组，**注意是从右往左加载 loader**。

babel 转译的配置文件 .babelrc：

```
{
    "presets":["@babel/preset-env", "@babel/preset-react"]
}
```
在 package.json 中添加脚本：

```
"start": "webpack-dev-server --mode development --open --hot",
"build": "webpack --mode production"
```

将创建的空文件补充完整：

```html
<!-- dist/index.html -->

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>React Web</title>
</head>

<body>
    <div id="root"></div>
    <!-- html-webpack-plugin 插件生成如下标签
    <script src='bundle.js'></script> -->
</body>

</html>
```

```js
// src/main.js

import React from 'react';
import ReactDOM from 'react-dom';
import App from './App.js';
ReactDOM.render(<App />, document.getElementById('root'));
```

```js
// src/app.js 

import React, { Component } from 'react';
import './index.css';
class App extends Component {
    render() {
        return (
            <div>
                <h1>Hello!!</h1>
                <h2>Welcome to your React App..!</h2>
            </div>
        );
    }
}
export default App;
```

```css
/* src/index.css */

* {
    margin: 0;
    padding: 0;
}

```

运行代码: ` yarn start `，打包文件: ` yarn run build `，动手试试吧！

**参考资料**

- [webpack 官方文档](https://www.webpackjs.com/)
- [webpack打包原理? 看完这篇你就懂了!](https://juejin.im/post/6844904038543130637)
- [实现一个简单的Webpack](https://juejin.im/post/6844903858179670030)
