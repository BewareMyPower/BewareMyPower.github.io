---
title: hexo多终端同步
date: 2018-10-15 20:07:19
tags:
---
## 简介
hexo的安装还是很简单的，操作也很傻瓜式。常用的基本就下面几个命令
1. `hexo s`
s即server，默认在`localhost:4000`启动服务器，在浏览器中即可看到效果，可以通过`-p`选项指定端口。
2. `hexo d`
d即deploy(部署)，上传到服务器。一般会加上`-g`选项在本地生成静态文件。如果不上传服务器，可以直接`hexo g`生成静态文件。
3. `hexo new`
新建博客，后接博客名，比如`hexo new "test"`，此时hexo框架就会自动生成.md文件。自动新建的.md文件会生成一些模板信息，因此最好使用命令新建博客。生成之后，用vim等本地编辑器修改即可。
另外在`_config.yml`设置`post_asset_folder`为true，则`hexo new`会新建.md文件的同名目录用来存放图片，并且作为图片的默认路径，也就是说如果待插入图片放在该目录下，路径直接写文件名即可。

至于其他命令可以参考[hexo中文文档](https://hexo.io/zh-cn/docs/commands.html)，`_config.yml`的配置也有较为详细地注释。

## 新建分支保存源码
hexo是将Markdown文本和图片生成静态网页上传至github的。
![master分支目录](master分支目录.jpg)

上图所示目录即`hexo d -g`或`hexo g`在本地生成的静态文件，为`public`子目录。`hexo d`即上传该目录到仓库master分支。
而博客目录是保存在`source/_posts`目录下的，因此需要在git仓库中新建一个分支保存博客源文件。创建过程如下
```
git init
git add source themes _config.yml package.json
git commit -m "blog source"
git branch hexo
git checkout hexo
git remote add origin git@github.com:BewareMyPower/BewareMyPower.github.io.git
git push origin hexo
```
普通的git命令，初始化、添加、提交、创建分支、切换分支、添加远程仓库、推送分支到该仓库。完成后远程仓库hexo分支如下图所示。
![hexo分支目录](hexo分支目录.jpg)

## 新环境下部署博客并添加新博客
首先克隆源码分支。
```
git clone -b hexo git@github.com:BewareMyPower/BewareMyPower.github.io.git
```
由于hexo分支仅仅是源码，缺少了将源码转换成网页的nodejs模块(即hexo)，所以需要安装hexo在该目录。
```
cd BewareMyPower.github.io/
npm install hexo
npm install
```
然后会发现目录下多了个`node_modules`目录，即保存nodejs相关模块。
注意后面一个`npm install`不可少！否则会缺少某些模块，比如缺少hexo-asset-image模块导致图片无法显示。
此外无论是生成静态文件还是启动服务器，源码目录不受影响，因此可以放心大胆地操作。
```
hexo new "hexo多终端同步.md"
```
边修改边`hexo s`查看效果，直到修改完毕，便可以上传了，当然，也是分为上传源码到hexo分支和上传编译后静态文件到master分支这两步。
```
git add source/_post/多终端同步.md source/_posts/hexo多终端同步
git commit "new blog"
git push origin hexo
hexo d
```

## 其他问题
之前在阿里云上`hexo init`出错
```
> nunjucks@3.1.3 postinstall /root/hexo/node_modules/nunjucks
> node postinstall-build.js src

sh: 1: node: Permission denied
```
原因是我是root用户，而`npm`的默认用户ID是500，因此要设为0。可以直接修改`.npmrc`文件，也可以`npm config`命令修改：
```
~# npm config set user 0
~# npm config set unsafe-perm true
~# cat .npmrc
user=0
unsafe-perm=true
```