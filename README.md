[新建一篇hugo博客流程](https://hanchuntao.github.io/articles/how-to-use-hugo/)

参数说明：

  --theme 选项可以指定主题。也可用-t
  
  --watch 选项可以在修改文件后自动刷新浏览器。也可用-w
  
  --buildDrafts 包括标记为草稿（draft）的内容。也可以用-D

1 . 切到 hugo 文件夹下的 myblog:

` $ cd myblog `

2 . 启动本地服务器时需要指定主题：

` $ hugo server --theme=hugo-lamp --buildDrafts `

出现 Web Server is available at http://localhost:1313/ (bind address 127.0.0.1) 表示启动成功，之后修改博客文件会打印日志。

3 . 新建博客：

` $ hugo new post/first.md `

4 . 生成HTML静态页面：

` $ hugo --theme=hugo-lamp --baseUrl="https://111hunter.github.io/"`

该指令执行成功会在 public/post 文件夹下生成 first 文件夹，其中包含html文件

5 . 上传github
