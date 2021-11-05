# Docker 极速入门


Docker 包括三个基本概念: Image(镜像)、Container(容器)、Repository(仓库)，理解了这三个概念，就理解了 Docker 的整个生命周期。

## 基本概念

Docker 里的基本概念:

<div align=center><img src="/img/docker.jpg" width="66.7%"></div>

- Image(镜像): 镜像类似于创建虚拟机时只读的系统镜像文件
- Container(容器): 容器可类比于可以运行的虚拟机，容器可以被创建、启动、停止、删除、暂停等。
- Repository(仓库): 仓库是集中存放镜像文件的地方
- tar 文件: 类似于 vmware 中的 vmdk 文件
- Dockerfile: 定义镜像如何构建的配置文件

镜像与容器类似对象与实例的关系，一个镜像可以创建多个容器

## 实战

在 Docker 官网注册好账号后，进入 [play-with-docker](https://labs.play-with-docker.com/) 添加一个新的实例，就会进入 ssh 页面。

### 创建容器

搜索远程仓库中的 nginx 镜像

```bash
$ docker search nginx
```

从远程仓库拉取 nginx 镜像, 可以在 [dockerhub](https://hub.docker.com/) 中查指定版本的拉取命令

```bash
$ docker pull nginx
```

查看本地已有的镜像:

```bash
$ docker images
```

用镜像创建容器, -d 指定后台运行, -p 指定外，内端口映射

```bash
$ docker run -d -p 80:80 nginx
```
 
play-with-docker 的网页中出现了可点击的 80 字段，可以再启动一个外部 81 端口的映射

```bash
$ docker run -d -p 81:80 nginx
```

### 修改容器

查看正在运行的容器信息

```bash
$ docker ps
```

通过容器 ID 进入外部 81 端口的容器

```bash
$ docker exec -it 80fca9b9217d bash
```

```bash
$ cd usr/share/nginx/html && ls
```

里面的 index.html 就是外部 81 端口网页中显示的 html, 修改这个文件

```bash
cat index.html
```

```bash
echo hello > index.html
```

强制刷新浏览器，网页内容为 “hello” 修改成功

```bash
$ exit 
```

退出容器后，删除外部 80 端口容器，我们不用这个未修改的容器

```bash
$ docker rm -f 42a0565b5ba8
```

#### commit 构建镜像

将修改后的 81 端口容器重新保存为镜像后，启动新的 82 端口容器

```bash
docker commit 80fca9b9217d m1
```

```bash
docker images
```

```bash
$ docker run -d -p 82:80 m1
```

这是对 commit 的实践，符合预期，另一种构建镜像的方式是使用 Dockerfile


#### Dockerfile 构建镜像

```bash
$ vim Dockerfile
```

按照 dockerfile 语法写配置，FROM 指定新镜像的基础镜像，ADD 将当前文件夹下的所有文件拷贝到指定目录

```
FROM nginx
ADD ./ /usr/share/nginx/html/
```

```bash
$ vim index.html
hello, 83
```

```bash
$ ls
Dockerfile  index.html
```

以当前目录的 Dockerfile 作为镜像构建配置文件

```bash
$ docker build -t m2 .
```

```bash
$ docker run -d -p 83:80 m2
```

容器启动后就可以网页查看了效果了, 网页内容为 “hello, 83”

### 导出镜像

```bash
$ docker save m2 > m2.tar
```

删除镜像 m2

```bash
$ docker rmi m2
```

提示 container 0618f427bb2b 正在用这个镜像, 需要先删除容器

```bash
$ docker rm -f 0618f427bb2b
```

```bash
$ docker rmi m2
```

```bash
$ docker images
```

m2 镜像删除了，用 m2.tar 重新生成 m2 镜像

```bash
$ docker load < m2.tar
```

```bash
$ docker images
```

### 补充

到这里，我们已入门 docker，其他内容待补充。。。


**参考资料**

- [10分钟，快速学会docker](https://www.bilibili.com/video/BV1R4411F7t9)
- [Docker 从入门到实践](https://yeasy.gitbook.io/docker_practice/)
