# Node.js 搭建博客后台


本次实践将不使用框架，尝试直接使用 Node.js 处理 http 请求，连接数据库，创建 api 接口。以及介绍一些 web 开发相关的常识。

## 搭建开发环境

- 使用 nodemon 监测文件变化，自动重启 node
- 使用 cross-env 设置环境变量，兼容Mac Linux 和 Windows
- 配置完后使用 ` $ npm run dev ` 命令启动项目

## Node.js 处理 get，post 请求

```javascript
const http = require('http')
const querystring = require('querystring')

const server = http.createServer((req, res) => {
    const method = req.method
    const url = req.url
    const path = url.split('?')[0] //重点：split('?'[0])语法弄清楚
    const query = querystring.parse(url.split('?')[1])

    //设置返回值格式为 JSON
    res.setHeader('Content-type', 'application/json')   
    //返回的数据
    const resData = {
        method,
        url,
        path,
        query
    }
    
    //返回
    if (method === 'GET') {
        res.end(
            JSON.stringify(req.query)
        )
    }   
    if (req.method === 'POST'){
        let postData = ''
        //res.on('data')指每次发送的数据
        //chunk 逐步接收数据 req绑定一个data方法 chunk是变量
        req.on('data', chunk => {
            postData += chunk.toString()
        })
        //req.on(end)数据发送完成；
        req.on('end', () => {
            console.log('postData:', postData)
            console.log('resData:', resData)
            resData.postData = postData
            //返回
            res.end(
                JSON.stringify(resData)
            )
        })
    }
})

server.listen(8000)
console.log('OK')
```
https://github.com/username/xxx 每个斜线后面的唯一标识就是路由

## 接口设计方案

| 描述 | 接口 | 方法 | url参数 | 备注 |
| :--: | :--: | :--: | :--: | :--: |
| 获取博客列表 | /api/blog/list | get | author，keyword | 参数为空则不进行查询过滤 |
| 获取一篇博客 | /api/blog/detail | get | id |
| 新增一篇博客 | /api/blog/new | post |  | postData 有新增信息 |
| 更新一篇博客 | /api/blog/update | post | id | postData 有更新信息 |
| 删除一篇博客 | /api/blog/del | post | id |
| 登录 | /api/user/login | post |  | postData 有用户名和密码 |

## 开发路由

业务分层 拆分业务

   - createServer 业务单独放在 ` ./bin/www.js `
   - 系统基本设置、基本信息 ` app.js ` 放在根目录
   - 路由功能 ` ./src/router/xxx.js `
   - 数据管理 ` ./src/contoller/xxx.js `
   - 使用 promise 读取文件，避免 callback-hell

## 使用MySQL数据库

### 根据需求设计表

users：

|  id  | username | password | realname |
| :--: | :------: | :------: | :------: |
|  1   | zhangsan |   admin  |   张三   |
|  2   |   lisi   |   123    |   李四   |

blogs：

|  id  | title | content |  createtime   |  author  |
| :--: | :---: | :-----: | :-----------: | :------: |
|  1   | 标题A |  内容A  | 1573989043149 | zhangsan |
|  2   | 标题B |  内容B  | 1573989111301 |   lisi   |

## 用 Node.js 操作 MySQL

1. 示例：文件 [mysql-test/index.js](https://github.com/111hunter/node-demo/blob/master/mysql-test/index.js)  演示 Node.js 操作 MySQL

2. 封装：将其封装为系统可用的工具, 封装 [exec](https://github.com/111hunter/node-demo/blob/master/blog-1/src/db/mysql.js) 函数

3. 使用：路由 api/user/xxx，api/blog/xxx 对接 MySQL

## 用户登录

### Cookie

- 什么是 Cookie

  - 存储在浏览器的字符串（最大5KB）
  - 跨域不共享
  - 格式如 K1=V1;K2=V3;K3=V3; 因此可以存储结构化数据
  - 每次发送 Http 请求，会将请求域的 Cookie 一起发送给 Server
  - Server 可以修改 Cookie 并返回给浏览器
  - 浏览器也可以通过 JavaScript 修改 Cookie （有限制）

- Server 端操作 Cookie，实现登录验证

### Session

- Cookie 存放用户信息非常危险
- 如何解决：cookie 中存储 userId， server 端对应 username
- 解决方案：session ，即 server 端储存用户信息
- 直接在代码中使用 session 的问题

  - session 是 JS 变量，放在 Node.js 进程内存中
  - 进程内存有限，访问量过大，内存暴增
  - 正式上线是多进程，进程之间内存无法共享

## 在 Redis 中存储 Session

- Web Server 最常用的缓存数据库，数据储存在内存中
- 相比于 MySQL，访问速度极快，成本更高，储存空间小
- 将 Web Server 和 Redis 拆分为两个单独服务，双方独立，可扩展

1. 示例：文件 [redis-test/index.js](https://github.com/111hunter/node-demo/blob/master/redis-test/index.js)  演示 Node.js 操作 Redis

2. 封装：将其封装为系统可用的工具, 封装 [set, get](https://github.com/111hunter/node-demo/blob/master/blog-1/src/db/redis.js) 函数

3. 使用：路由 api/user/xxx 对接 Redis

## Nginx 反向代理前后端联调

- 登录依赖 Cookie，必须用浏览器
- Cookie 跨域不共享，前端和 server 端必须同域
- 需要用到 Nginx 做代理，让前后端共域，实现前后端联调
1. 开发前端界面：文件夹 [html-test](https://github.com/111hunter/node-demo/tree/master/html-test) 包含用户登录，博客管理界面

2. 启动服务：

 ` $ cd html-test `
 
 ` $ npm i http-server -g `
 
 ` $ http-server -p 8001 `

3. 修改 nginx 配置:

 ` ..\nginx-1.12.2\conf\nginx.conf `

```
worker_processes  auto;
 listen       8080;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }

　　　　location / {
　　　　　　　　proxy_pass http://localhost:8001;
　　　　}

　　　　location /api/ {
　　　　　　　　proxy_pass http://localhost:8000;
　　　　　　　　proxy_set_header Host $host;
　　　　}
```

4. 启动 nginx:

` $ start nginx `

## Web 安全

### 常见安全问题

- SQL 注入：窃取数据库内容
- XSS攻击：窃取前端的 Cookie 内容
- 密码加密：保障用户信息安全（重要）
- Server 端攻击方式很多，预防手段也很多
- 有些攻击需要**硬件和服务**来支持（需要 OP 支持），如 DDOS

### SQL 注入

- 最原始、最简单的攻击，从有了 Web2.0 就有了 SQL 注入攻击
- 攻击方式：输入一个 SQL 片段，最终拼接成一段攻击代码

解决方案：使用 MySQL　的 escape 函数处理输入数据内容

### XSS 攻击

- 前端同学熟知的攻击方式，但 Server 端更应该掌握
- 攻击方式：在页面展示内容中参杂 JS 代码，以获取网页信息

解决方案：在文件夹 blog-1 中安装 xss 库： ` npm i xss `

### 密码加密

- 万一数据库被攻破，避免泄露用户信息
- 攻击方式：获取用户名和密码，再去尝试登录其它系统

解决方案：密码加密，密文储存，MD5加密

附：[源码地址](https://github.com/111hunter/node-demo)
