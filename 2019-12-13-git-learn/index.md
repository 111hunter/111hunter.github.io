# Git 快速上手指南


Git 是一个全世界开发者都在用的版本控制系统。它会帮助你与其他开发者合作，以及跟踪你不同版本的代码。如果你长时间在一个项目工作，你也许会想对那些有改动的地方保持跟踪：是谁，以及什么时候改动的。要是最终你发现你的代码里有 bug，这跟踪就变得更重要了。

![git版本控制](/img/git.png "git版本控制")

本文实践较多，建议跟着文章的步骤敲一遍代码以加深理解。

## 1. 使用帮助

`$ git help` 可以查看 git 常用命令

`$ git help -a` 可以查看 git 所有命令，F 或者 空格 向下查看命令，B 向上查看命令，Q 退出 git-cli

`$ git help add`  help后接一个指令可以查看该指令的详细用法

## 2. git 配置范围

git 配置分为三个范围 system, global 和 项目范围
<br>一般选择global进行配置 `$ git config --global user.name '111hunter' `

`$ git config --list` 查看当前配置信息

`$ git config --unset --global user.name` 取消 user.name 配置

配置文件是当前用户主目录 `$ cat ~/.gitconfig`

## 3. git 项目文件

`$ mkdir movietalk && cd movietalk` 新建文件夹

`$ git init` 初始化项目

`$ cd .git && ls -a` config 目录就是项目级别的配置

`$ cd .. && touch index.html` 新建文件

`$ vim index.html` 编辑文件后保存

`$ git add .` 提交所有文件到暂存区

`$ git commit -m "first commit"` 暂存区文件提交仓库区

`$ git log` 查看提交日志信息

`$ vim index.html` 将index.html中 charset="UTF-8" 改为 charset="GBK"

`$ git status` 查看文件状态

`$ git diff index.html` 查看暂存区文件与本地工作区的对比

`$ git diff --staged` 查看仓库区与暂存区对比，此时一致

`$ git add .` 然后 `$ git diff index.html` 此时没有区别了，因为已将文件提交暂存区

`$ vim index.html` 再次修改，在文件中新增适应移动端的 meta 标签

`$ git diff --staged` 查看仓库区与暂存区对比

`$ git commit -m "修改了charset属性"` 暂存区提交仓库

`$ git diff --staged` 此时暂存区与仓库一致

`$ git log` 查看提交日志信息, 此时有两条提交信息

`$ git status` 查看文件状态

`$ git add . && git commit -m "新增meta标签"` 工作区文件提交到仓库

### 重命名 git 已跟踪文件

`$ touch style.css && vim style.css` 新建css文件

`$ mv style.css theme.css` 修改文件名

`$ git status` 查看文件状态

`$ git rm style.css && git add theme.css` 就能修改文件名字

`$ git commit -m "mv style.css theme.css"` 上传仓库区

### git 移动文件

`$ git mv theme.css aha.css` 移动文件，重命名

`$ mkdir css && git status` git 不会跟踪空文件

`$ git mv aha.css /css` 移动文件

`$ git commit -m "move aha.css"` 上传仓库区

### git 删除文件与恢复

`$ git rm index.html` 从工作区与暂存区中删除index.html

`$ git checkout HEAD -- index.html` HEAD 指针指向最近一次提交， -- 表示当前分支，将index.html恢复到当前 commit 的状态

`$ git rm index.html && git commit -m "删除了index.html"` 从工作区与暂存区中删除index.html, 并提交仓库

`$ git checkout "HEAD^" -- index.html` 恢复到上一次提交，windows cmd 中 ^ 会被当做换行处理，需要加上引号

### git 恢复某个操作状态

在 css 文件夹引入 bootstrap

`vim index.html` 在 index.html 中引入 bootstrap

`git commit -am "增加了bootstrap"` 提交仓库

新建 js 文件夹并引入 jquery

`vim index.html` 在 index.html 中引入 jquery

`git commit -am "增加了jquery"` 提交仓库

`git log --oneline` 查看提交日志 id, 添加 bootstrap 的 id 为 981eb52

`git revert 981eb52` 撤销对 bootstrap 的提交，查看工作区文件夹发现 index.html 对 bootstrap 的引入消失了

每次 git commit 后 HEAD 都会指向最后一次提交，用 git reset 可以帮助回到某次提交时的状态，有 3 个可选配置参数: --mixed, --soft, --hard

 - --soft 软重置，不会修改任何文件状态。该参数用于git commit后，又要恢复还没commit的场景，重新审查代码，然后再推上去覆盖之后的提交。

 `$ git log --oneline` 查看添加 jquery 的 id 为 e9ae8b5，Revert "添加了bootstrap"的 id 为 3e3da01

 `$ git reset --soft e9ae8b5` 回到提交 jquey 的 commit，但是不会对文件做任何操作

 `$ git status` 会提示 撤销 bootstrap 时的消息

 - 默认是 --mixed, 只影响暂存区文件状态

 `$ git reset --mixed e9ae8b5 && git status` 发现 bootstrap 已经不在暂存区了

 - --hard 会直接重置暂存区和工作区文件到指定 id 状态，用 git reset --hard 可直接在不同提交状态切换。

 `$ git reset --hard e9ae8b5 && git log --oneline` 查看工作区文件发现 bootstrap又回来了

 `$ git reset --hard 3e3da01 && git log --oneline` 文件又到了最后一次提交时的状态

## 4. git 项目分支

`$ git branch testing && git checkout testing` 新建并切换分支，此时对文件的修改只影响 branch 分支

`$ vim index.html` 在 index.html 中引入 link 标签

`$ git commit -am "添加link标签"` 提交仓库区

`$ git checkout master` 切换回 master 分支，查看工作区文件发现对文件的修改没有了

`$ git diff master..testing index.html` 查看分支间的文件对比

`$ git merge testing` 分支合并

`$ git diff master..testing` 没有不同，已经合并了分支

### 解决分支合并冲突

`$ vim index.html` 修改 document 为 Movietalk

`$ git commit -am "修改标题为Movietalk"` 提交仓库区

`$ git checkout testing && vim index.html` 切换分支，修改 document 为 Movie-talk

`$ git commit -am "修改标题为Movie-talk"` 提交仓库区

`$ git merge master` 产生冲突。git 发现冲突，查看文件会有提示，编辑保留其中一个 

 document 修改

 `$ git commit -am "解决冲突"` 提交

 `$ git log --oneline --all -10 --graph` 查看所有分支提交信息

### 保存当前工作状态

git stash 指令能够保存当前工作状态到 git 栈

`$ touch human.txt && git commit -am "add human.txt"` 新建空文件

`$ vim human.txt` 加入任意内容

`$ git status` 查看修改，不提交

`$ git stash save "修改了human.txt"` 保存工作进度，查看文件 human.txt 又变成了空文件

`$ git stash list` 查看工作进度

`$ git stash show -p stash@{0}` 以补丁的方式查看工作进度与工作目录的区别

`$ git stash apply stash@{0}` 切换到之前的工作进度，发现对 human.txt 的修改又生效了

`$ git stash pop` 切换栈顶工作状态

`$ git stash drop stash@{0}` 删除工作状态

## 5. git 远程仓库

 新建远程仓库后请清空仓库，不要保留任何文件

`$ git remote add origin https://gitee.com/hunter_111/movietalk.git` 关联远程仓库

`$ git remote -v` 查看是否关联, fetch 远程用来提取，push 远程用来推送

`$ git push -u origin master` 推送到远程分支，并跟踪远程分支变化

`$ git push origin testing` 推送远程分支，不跟踪变化

**参考资料**

- [廖雪峰的 git 教程](https://www.liaoxuefeng.com/wiki/896043488029600)

