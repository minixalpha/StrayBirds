StrayBirds
==========

基于 GitHub Pages 搭建的极简博客，所有操作都可以直接通过浏览器完成。

## 示例

可以通过访问 [StrayBirds](http://minixalpha.github.io/StrayBirds/) 看到最终
的效果，下面是截图:

![ui-demo](/images/ui_demo.png)

## 教程

### 使用方法

1. 注册 GitHub，得到用户名，例如 minixbeta
2. 到 [StrayBirds](https://github.com/minixalpha/StrayBirds) 页面，单击右上
角的 Fork
3. 到你 Fork 后的项目中，将 `_config.yml` 中的 username 修改为你的用户名 minixbeta
4. 访问你的博客 http://minixbeta.github.io/StrayBirds/

![create_project](/images/create_project.gif)

**注意如果你是第一次使用 GitHub Pages，可能不会马上生效，等一段时间即可**

**按照配置中说的方法修改项目名称可能会加快这一进程**

### 配置

* 修改主题

在 `_confg.yml` 下修改 theme 的值。

**注意修改主题后，并不会马上生效，GitHub 还要反应一段时间，所以请耐心等待**

**修改主题后, 按照配置中说的方法修改项目名称可能会加快这一进程**

可选主题包括：

- hack
	![hack-demo](/images/hack-demo.png)
- leap-day
	![leap-day-demo](/images/leap-day-demo.png)
- merlot
	![merlot-demo](/images/merlot-demo.png)
- midnight
	![midnight-demo](/images/midnight-demo.png)
- minimal
	![minimal-demo](/images/minimal-demo.png)
- modernist
	![modernist-demo](/images/modernist-demo.png)
- slate
	![slate-demo](/images/slate-demo.png)
- time-machine
	![time-machine-demo](/images/time-machine-demo.png) 
- kunka
	![kunka-demo](/images/kunka-demo.png)

* 修改项目名

例如将 StrayBirds 修改为 blog，那么你需要做的是

1. 在项目的 Setting 中将 Repository name 从 StrayBirds 修改为 blog
2. 将 `_config.yml` 中的 baseurl 修改为 /blog
3. 通过 http://minixbeta.github.io/blog/ 来访问你的新博客

![create_post](/images/change_project_name.gif)


* 修改评论系统用户名

我们的评论系统使用的是 [Disqus](https://disqus.com/)，如果你想在这份博客模板中使用，需要先去注册一下，然后得到一个用户名，例如 minixalpha。然后在 `_config.yml` 中将 disqusname 修改为 minixalpha。

**千万注意: 如果你开启评论系统一定要修改这个值，不然就评论到我的评论系统中去了**

### 添加文章

在 `_post` 目录下添加形如 `2014-10-26-title.md` 的文章，用 markdown 格式
撰写博客。

例如：

```
---
layout: post
title: Java 中的并发
comments: true
category: 技术
---


## 如何创建一个线程

按 Java 语言规范中的说法，创建线程只有一种方式，就是创建一个 Thread 对象。而从 HotSpot 虚拟机的角度看，创建一个虚拟机线程
有两种方式，一种是创建 Thread 对象，另一种是创建 一个本地线程，加入到虚拟机线程中。

...

```

其中 `layout` 表示布局，不用改变，`title` 表示文章题目，`comments` 表示是否要开户评论。

![create_post](/images/create_post.gif)

## 感谢

Thanks to authors of the themes:

* [hack](https://github.com/sundaykofax/baby-legs), Licence: None
* [leap-day](https://github.com/mattgraham/leapday), Licence: [Creative Commons Attribution](http://creativecommons.org/licenses/by/3.0/)
* [merlot](https://github.com/cameronmcefee/headsmart/tree/gh-pages), Licence: None
* [midnight](https://github.com/briandoll/change-inside-surroundings.vim/tree/gh-pages), Licence: None
* [minimal](https://github.com/orderedlist/minimal), Licence: [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/)
* [modernist](https://github.com/orderedlist/modernist), Licence: [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/)
* [slate](https://github.com/jasoncostello/slate), Licence: MIT
* [time-machine](https://github.com/jonrohan/time-machine-theme), Licence: None
* [kunka](https://github.com/pizn/kunka), Licence: MIT, author: [zhanxin.info](http://www.zhanxin.info/)

All the themes are intergrated in the blog template, with some modifies.
