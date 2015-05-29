---
layout: post
title: "用jekyll bootstrap在github上快速搭建个人博客"
category: tutorial
tags: jekyll, bootstrap, github, 个人博客
---

### 1. 创建Repository

访问https://github.com, 创建名为USERNAME.github.com的repository.

<font color="red">**USERNAME是你github的用户名**</font>

### 2. 下载jekyll-bootstrap

	git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com

### 3. 将代码上传到代码库

	cd USERNAME.github.com

	git remote set-url origin git@github.com:USERNAME/USERNAME.github.com.git

	git push origin master

等待几分钟后，访问<font color="red">**http://USERNAME.github.com**</font>, 就能看到你的博客已经神奇地搭建好了。

### 4. 在本地运行jekyll

为了能够在本地预览博客，你需要安装jekyll ruby gem。 当然，你首先要安装ruby, 这里假设你已经安装好了ruby, 接着输入下面的命令。

	gem install jekyll

安装好jekyll后，进入你博客所在的目录启动jekyll server

	cd USERNAME.github.com
	jekyll serve

打开浏览器，访问[http://localhost:4000](http://localhost:4000)

### 5. 写博客

jekyll-bootstrap提供了rake命令方便的创建博客。

	rake post title="XXX"

### 6. 发布

当你写完一篇博客或者改变了主题后，需要把你的修改提交到github上。

	git add .
	git commit -m "add some content"
	git push origin master
