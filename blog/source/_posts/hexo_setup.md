# Hexo + Github 搭建博客与多终端同步
``


### 需求

* 搭建一个便捷的博客。
* 家里的 Mac 和公司的 Windows 能同时修改和提交。



### 方案概要

* 在 Mac 上配置 Hexo 、 NPM 、 Git 环境，并将 Hexo 静态网页关联到 Github Page 。
* 在 Windows 上配置 Hexo 、 NPM 、 Git 环境。
* 在 Github Page 的仓库中创建一个分支用于保存 Hexo 源码，今后在任意终端提交完博客的修改后同时再执行Hexo -d发布即可。



### 参考
以下是在搭建过程中参考的网站。参考下列网站步骤搭建，**半个小时**基本能搭建好Hexo了,另外的时间用于学习Git基本知识和如何多个客户端修改博客。总计**半天**可完成博客的搭建和部署。

* [Mac上搭建基于Github的Hexo博客](http://www.jianshu.com/p/13e64c9e2295)
* [Windows上搭建基于Github的Hexo博客](https://stormzha.github.io/2017/03/26/blogFinish/)
* [在不同的电脑维护Hexo和写作](http://www.rvclient.com/2016/05/21/hexo-everywhere/)
* [Git教程](http://www.runoob.com/git/git-tutorial.html)
* [极好用的MD编辑器](https://www.typora.io/)

>注意：用于存放Hexo源码的分支，在提交源码时记得把文件夹中的.gitignore去掉，否则上传文件不全。




































