# Next.js 概览


![https://www.reddit.com/r/nextjs/comments/17iy9sx/nextjs_server_action_is_crazy](/img/nextjs-slug.webp "Next.js Reddit Meme")

我们知道，如今流行的前端框架都是 SPA(单页应用)，在投入生产时会出现中首屏加载慢，不利于 SEO 等问题。于是，现代前端同构框架应运而生。Next.js 是 React 的同构框架，它的页面由 React 组件构成。

## 路由系统

Next.js 的路由系统基于文件路径自动映射，一般约定在根目录的 pages 文件夹内：

- `pages/index.js` --> `/`

- `pages/about.js` --> `/about`

- `pages/blog/[slug].js` --> `/blog/:slug`( slug 是动态生成的)

- `pages/post/[...all].js`--> `/post/*`(匹配 `/post`,`/post/a`,`/post/a/b` 等)

Next.js 创建的是多页应用，pages 内的每个文件都是单个页面。Next.js 中用形如 `[params]` 文件(文件夹)表示动态路由页面。

## 路由跳转

Next.js 中路由跳转方式有两种，使用的 api 分别是 `next/link` 和 `next/router`。

### next/link

从 `next/link` 导入的 `<Link>` 是 React 组件，可接收以下属性:

- `href` 是导航到的路径，是页面跳转的必需属性，href 可以是字符串或者对象

```jsx
<Link href="/about?name=jackylin">
//这里 href 有两层 {}, github page 无法识别语法，只能写为一层了
<Link href={ pathname: "/about", query: { name: "jackylin" },}>
```

- `as` 是浏览器 url 栏显示的路径，当 `href` 中包含动态页面 (`[param]`) 时使用

```jsx
const pids = ["id1", "id2", "id3"];
{
  pids.map((pid, index) => (
    <Link href="/post/[pid]" as={`/post/${pid}`} key="index">
      <a>Post {pid}</a>
    </Link>
  ));
}
```

- `passHref` 将 `<Link>` 的 `href` 传递给子项，当子项是包装 `<a>` 的组件时，此属性必需

- `prefetch` 预加载，将页面提前加载到本地缓存

官方文档还有一些其他 [属性](https://nextjs.org/docs/api-reference/next/link) 和用法示例，需要注意的是 `<Link>` 只能有一个子项。

### next/router

相较于 `next/link`，`next/router` 能自定义配置复杂的路由跳转。`next/router` 提供如下 api：

- `useRouter` 是 React hook，只能用于函数组件

- `withRouter` 是高阶组件，可用于类组件和函数组件

他们的实例对象 router 具有以下的属性，方法，事件等。

属性：`pathname` 是文件名，`query` 是查询参数，`asPath` 是浏览器中显示的路径。

方法：`router.push(url, as, options)` 是路由跳转方法，跳转的页面路径(url)必需。url 可以是字符串形式，也可以是对象形式。在需路由跳转的元素上绑定点击事件。

```jsx
export default function ReadPost({ post }) {
  const router = useRouter();

  return (
    <span
      onClick={() => {
        router.push({
          pathname: "/post/[pid]",
          query: { pid: post.id },
        });
      }}
    >
      查看文章
    </span>
  );
}
```

对比前面讲的 `<Link>` 组件，能看出 `<Link>` 组件其实是封装了 router，点击事件等。

事件：Next.js 在路由跳转的生命周期内置了一些的钩子事件，若我们有监听路由变化的需求，可订阅这些钩子事件来实现需求。具体用法请参阅 [官方文档](https://nextjs.org/docs/api-reference/next/router#routerevents)。

### 路由传参

Next.js 支持查询字符串格式的参数传递，参数以字符串或者对象的格式传递: `<Link>` 的 `href` 属性或者 `router.push` 中的 url。参数的接收可以用 `useRouter` 或 `withRouter`:

```jsx
// router 直接读取参数
const Post = () => {
  const router = useRouter();
  return <div>文章编号：{router.query.pid}</div>;
};

export default Post;
```

```jsx
//使用 withRouter 接收参数时，router 作为组件参数
const Post = ({ router }) => {
  return <div>文章编号：{router.query.pid}</div>;
};

export default withRouter(Post);
```

## 获取数据

Next.js 中获取数据的方法有 `getServerSideProps`，`getStaticProps` 和 `getStaticPaths`。还有一个 `getInitialProps`，官方文档已不推荐使用。这些方法都是服务端的异步方法，只能在 pages 文件夹内使用。

Next.js 有两种预渲染形式：

- 服务端渲染(SSR)：html 在每次访问路由时都会重新生成。对应的数据获取方法：`getServerSideProps`。由于“服务端渲染”比“静态生成”慢，因此常用于数据频繁更新的页面。

- 静态生成(SSG)：html 是在构建时生成的，并且会在每次请求时重用。对应的数据获取方法： `getStaticProps` 和 `getStaticPaths`。这对于可以在用户请求之前就预渲染的页面非常有用，可以将其与客户端渲染结合使用以引入其他数据。

Next.js 引入了自动静态优化的功能，就是说如果页面中没有使用 SSR 方法，Next.js 在 build 阶段就会生成 html，访问页面路由直接返回生成的 html，以此来提升性能。

选择 SSR 还是 SSG？

- 如果页面内容真动态(例如来源数据库且经常变化)，使用 SSR。

- 如果是静态页面或者伪动态(来源数据库但是不变化)，使用 SSG。

### getServerSideProps

在数据频繁更新的页面使用，每次访问路由时都会调用。`getServerSideProps` 方法是升级了 9.3 之前的 `getInitialProps` 方法。

```jsx
const Blog = ({ data }) => {
  return <div>title: {data.title}</div>;
};

// 在每次页面请求时才会运行，在构建时不运行。
export async function getServerSideProps() {
  const res = await fetch("https://jsonplaceholder.typicode.com/todos/1");
  const data = await res.json();

  return { props: { data } };
}

export default Blog;
```

### getStaticProps

页面内容取决于外部数据时使用。

```jsx
// posts 依赖外部数据
const Blog = ({ posts }) => {
  return <div>title: {posts.title}</div>;
};

// 此函数只在构建时被调用一次，后面不会再次调用
export async function getStaticProps() {
  // 调用外部 API 获取内容
  const res = await fetch("https://jsonplaceholder.typicode.com/todos/1");
  const posts = await res.json();

  // 在构建时将接收到 `posts` 参数
  return {
    props: {
      posts,
    },
  };
}

export default Blog;
```

### getStaticPaths

页面路径取决于外部数据时使用，结合 getStaticProps 使用。

```jsx
const Post = ({ post }) => {
  return (
    <div>
      文章id: {post.id}, 文章标题: {post.title}
    </div>
  );
};
// 此函数只在构建时被调用一次，后面不会再次调用
export async function getStaticPaths() {
  // 取全部文章数据
  const res = await fetch("https://jsonplaceholder.typicode.com/todos");
  const posts = await res.json();
  const paths = posts.map((post) => `/posts/${post.id}`);
  // fallback为 false，表示不在 getStaticPaths 的路径是 404 页面。
  return { paths, fallback: false };
}
// params 来自 paths: [{ params: { ... } }]
export async function getStaticProps({ params }) {
  // 取具体文章数据
  const res = await fetch(
    `https://jsonplaceholder.typicode.com/todos/${params.id}`
  );
  const post = await res.json();
  return { props: { post } };
}
export default Post;
```

api 的更多细节用法请阅读 [官方文档](https://nextjs.org/docs/basic-features/data-fetching)。

## 其他功能

### 自定义配置

Next.js 在 pages 文件夹内的默认配置文件有 `_app.js`,`_document.js`,`404.js` 等。

以 `_app.js` 为例，它的功能是初始化当前路由的页面组件，接口如下：

```jsx
/**
 * Component 是当前路由的页面组件，每次路由切换时，Component 都会更新
 * pageProps 是初始属性，该初始属性由某个数据获取方法预先加载到你的页面中，否则它将是一个空对象
 */
function MyApp({ Component, pageProps }) {
  return <Component {...pageProps} />;
}

export default MyApp;
```

比如我们要使用 recoil 进行状态管理，所有页面组件都应该为 `<RecoilRoot>` 的子组件。

```jsx
import { RecoilRoot } from "recoil";

export default function MyApp({ Component, pageProps }) {
  return (
    <RecoilRoot>
      <Component {...pageProps} />
    </RecoilRoot>
  );
}
```

其他配置文件的作用请阅读 [官方文档](https://nextjs.org/docs/advanced-features/custom-document)。

### 自定义构建

Next.js 根目录 next.config.js 可配置项目构建。例如扩展默认 webpack 配置，接口如下：

```jsx
module.exports = {
  webpack: (config, { buildId, dev, isServer, defaultLoaders, webpack }) => {
    // Note: we provide webpack above so you should not `require` it
    config.plugins.push(new webpack.IgnorePlugin(/\/__tests__\//));
    // Important: return the modified config
    return config;
  },
};
```

例如在 Next.js 默认的 babel 配置中添加一个 loader：

```jsx
// https://github.com/vercel/next.js/tree/canary/packages/next-mdx
module.exports = {
  webpack: (config, options) => {
    config.module.rules.push({
      test: /\.mdx/,
      use: [
        options.defaultLoaders.babel,
        {
          loader: "@mdx-js/loader",
          options: pluginOptions.options,
        },
      ],
    });
    return config;
  },
};
```

### api 路由

Next.js 提供简单的后端 api 能力，在 `pages/api` 内的文件都将映射为 `/api/*` 的后端接口。它们不会和页面一起打包。

```jsx
// pages/api/post.js

import { getPosts } from "lib/posts";

const Posts = async (req, res) => {
  const posts = await getPosts();
  res.statusCode = 200;
  res.setHeader("Content-Type", "application/json");
  res.end(JSON.stringify(posts));
};

export default Posts;
```

目前 Next.js 没有提供数据库和测试相关的功能，需自行配置或与其他框架配合使用。

**参阅资料**

- [Next.js 官方文档](https://nextjs.org/)
- [Next.js 简明教程](https://hicc.pro/p/UEvoioY/T622AgI)
- [手把手带你入门 NextJs](https://juejin.im/post/6863336367309455373)

**进阶阅读**

Next.js Severless 全栈开发:

- [使用 Next.js、Prisma 和 Vercel Postgres 构建全栈应用程序](https://vercel.com/guides/nextjs-prisma-postgres)
- [使用 Next、Prisma 和 MongoDB 进行身份验证和数据库访问](https://blog.openreplay.com/authentication-and-db-access-with-next-prisma-and-mongodb/)

