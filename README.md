StrayBirds
==========

基于 GitHub Pages 搭建的极简博客，所有操作都可以直接通过浏览器完成。

## 示例

可以通过访问 [StrayBirds](http://minixalpha.github.io/StrayBirds/) 看到最终
的效果，下面是截图:

![ui-demo](/images/ui-demo.png)

## 教程

### 使用方法

1. 注册 GitHub，得到用户名，例如 minixbeta
2. 到 [StrayBirds](https://github.com/minixalpha/StrayBirds) 页面，单击右上
角的 Fork
3. 到你 Fork 后的项目中，将 `_config.yml` 中的 username 修改为你的用户名 minixbeta
4. 访问你的博客 http://minixbeta.github.io/StrayBirds/

如果你想修改项目的名字，例如将 StrayBirds 修改为 blog，那么你需要做的是

1. 在项目的 Setting 中将 Repository name 从 StrayBirds 修改为 blog
2. 将 `_config.yml` 中的 baseurl 修改为 /blog
3. 通过 http://minixbeta.github.io/blog/ 来访问你的新博客

### 添加文章

在 `_post` 目录下添加形如 `2014-10-26-title.md` 的文章，用 markdown 格式
撰写博客。

## 感谢

博客主题来自 [modernist](https://github.com/orderedlist/modernist), 
开源协议为 [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/)
