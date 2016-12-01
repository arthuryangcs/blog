---
title: 基于 Git 的 hexo 备份与恢复
date: 2016-12-01 10:35:27
categories: hexo
tags:
  - hexo
  - 备份
  - 同步
---

当我们基于 github 架设 hexo 博客时，github 上存储的只包含静态网页信息。当我们想要在另一台机器上写博客，或者是备份整个系统的时候，这些是远远不够的。这时我们可以通过 Git 存储整个博客系统，用来不同机器之间的同步，还同时提供了版本控制功能。

<!-- more -->

## 备份

1. 首先在 Git 上建立一个新的项目，用来保存博客系统。
2. 提交 hexo 到 Git 上
    
    ``` bash
    $ cd <your hexo dir>
    $ git init
    $ git add .
    $ git commit -m "first commit"
    $ git remote add origin <your git server address>
    $ git push
    ```

这样整个系统就上传到 Git 服务器上了，我们现在在 git 服务器上查看备份的文件的话可能会发现少了一些文件。因为 hexo 系统在我们通过 `hexo init` 命令初始化时就会自动生成一个 `.gitignore` 文件，去除不必要上传的文件，`.gitignore` 文件信息如下:

```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

这样我们就可以只备份博客、自建页面、主题等必须的文件，降低备份空间大小。

## 版本控制

当我们对博客进行了修改操作后，可以通过 Git 原本的版本控制功能进行更新，或者是还原。这也是通过 Git 而不是简单地文件或者网盘备份的好处。

## 恢复

当我们想在另一个电脑上恢复整个系统，或者在另一台电脑上写文章时，通过以下命令便可重新生成一套与原先一样的 hexo 系统：

1. 首先通过 Git 克隆备份的文件

    ``` bash
    $ git clone <your git server address>
    ```

2. 安装必须的 node.js 包

    ``` bash
    $ cd <your hexo dir>
    $ npm install
    ```

    > 需要确保电脑已经安装了 node.js

3. 生成博客界面

    ``` bash
    $ hexo g
    ```

    > 需要确保电脑安装了 hexo

这样整个博客系统就算回复完成了，我们可以通过 `hexo s` 命令查看博客是否正常。
