---
title: 采用Jekyll + github + pygments构建个人博客的最终说明
comments: true
category: 技术
---

# 安装步骤
1. 
gem sources --remove https://rubygems.org/   #移除原来的镜像
gem sources -a https://ruby.taobao.org/    #添加淘宝镜像
gem sources -l         #查看是否只有taobao镜像
gem update --system    #更新RubyGems软件

2 安装jekyll及依赖的东西 
gem install jekyll // 安装jekyll
gem install kramdown // markdown语言解析包
gem install pygments.rb // 代码高亮包
gem install liquid // 这个包我不知道，但是好像有用
// 还有些别的包，看需要来安装啦

其中安装jekyll会报错
Error installing jekyll:
	ERROR: Failed to build gem native extension.

    /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby extconf.rb
    
 解决办法：Looks like you need to install the OSX development tools. The simplest way to do this in Mavericks is by running xcode-select to trigger the install as described in this guide:


jekyll build 
jekyll serve
