---
title: '我的vim开发环境搭建(3): Go开发配置'
date: 2019-06-21 00:00:49
tags:
  - 搭环境
  - vim
  - golang
top: 1
---

## 1. Go的安装

嗯因为Go的官网被墙了所以需要自行准备梯子。Linux安装Go很简单，即使是CentOS 6，直接去[golang下载页](https://golang.org/dl/)下载二进制文件解压即可，比如我写这篇博客时的最新版本是1.12.6，下载解压即可：

```
wget https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz
tar zxvf go1.12.6.linux-amd64.tar.gz -C ~/local/
cd ~/local/go
mkdir gopath
```

我这里依旧是解压到`~/local`目录，另外新建了`gopath`目录。然后在`~/.bashrc`中添加环境变量：

```
export PATH=$LOCAL/go/bin:$PATH
export GOROOT=$LOCAL/go
export GOPATH=$GOROOT/gopath
export GOBIN=$GOPATH/bin
```

`GOROOT`指的是Go的安装目录，而`GOPATH`则是自定义路径，不一定要在`GOROOT`下，而且可以有多个路径，每个路径代表Go的一个工作区，一般的工作区由`src`、`pkg`、`bin`三个目录组成，比如`go get`远程下载的项目，而`GOBIN`则是安装的二进制文件的路径。

## 2. 重新编译YCM

之前安装的YCM能够对C/C++进行补全，要对Go进行补全需要加上`--go-completer`选项重新编译，不过由于已经存放过编译的中间文件了，所以这次编译会很快。

```
CC="$LOCAL/gcc-5.4.0/bin/gcc" CXX="$LOCAL/gcc-5.4.0/bin/g++" ./install.py  \
  --clang-completer --system-libclang --go-completer
```

编译完毕后基本的补全功能已经有了，如下图所示：

![YCM对Go的补全](YCM对Go的补全.jpg)

Go的编码规范建议使用TAB而非空格表示缩进，因此之前的vim配置中`set expandtab`最好注释掉，防止把TAB扩展成空格。

## 3. vim-go

```
Plug 'fatih/vim-go', { 'do': ':GoUpdateBinaries' }
```

安装、使用可参考[vim-go](https://github.com/fatih/vim-go)的README，某些二进制文件地址被墙了，所以需要去镜像站下载。

然而打开.go文件时会提示

```
vim-go: could not find 'gopls'. Run :GoInstallBinaries to fix it
```

然而用`:GoInstallBinaries`会失败，因为某些二进制文件被墙了，打开vim-go项目目录下的`plugin/go.vim`可以找到：

```
" these packages are used by vim-go and can be automatically installed if
" needed by the user with GoInstallBinaries.
let s:packages = {
      \ 'asmfmt':        ['github.com/klauspost/asmfmt/cmd/asmfmt'],
      \ 'dlv':           ['github.com/go-delve/delve/cmd/dlv'],
      \ 'errcheck':      ['github.com/kisielk/errcheck'],
      \ 'fillstruct':    ['github.com/davidrjenni/reftools/cmd/fillstruct'],
      \ 'gocode':        ['github.com/mdempsky/gocode', {'windows': ['-ldflags', '-H=windowsgui']}],
      \ 'gocode-gomod':  ['github.com/stamblerre/gocode'],
      \ 'godef':         ['github.com/rogpeppe/godef'],
      \ 'gogetdoc':      ['github.com/zmb3/gogetdoc'],
      \ 'goimports':     ['golang.org/x/tools/cmd/goimports'],
      \ 'golint':        ['golang.org/x/lint/golint'],
      \ 'gopls':         ['golang.org/x/tools/cmd/gopls'],
      \ 'gometalinter':  ['github.com/alecthomas/gometalinter'],
      \ 'golangci-lint': ['github.com/golangci/golangci-lint/cmd/golangci-lint'],
      \ 'gomodifytags':  ['github.com/fatih/gomodifytags'],
      \ 'gorename':      ['golang.org/x/tools/cmd/gorename'],
      \ 'gotags':        ['github.com/jstemmer/gotags'],
      \ 'guru':          ['golang.org/x/tools/cmd/guru'],
      \ 'impl':          ['github.com/josharian/impl'],
      \ 'keyify':        ['honnef.co/go/tools/cmd/keyify'],
      \ 'motion':        ['github.com/fatih/motion'],
      \ 'iferr':         ['github.com/koron/iferr'],
\ }
```

如注释所言，`:GoInstallBinaries`实际上是把这些二进制文件安装到本地，由于`golang.org`被墙了，所以相关工具无法下载。即使是github上的项目，克隆速度也慢得感人，所以还是手动下载安装。

### 3.1 手动安装二进制文件

以`fillstruct`为例：

```
mkdir -p $GOPATH/src/github.com/davidrjenni
cd $GOPATH/src/github.com/davidrjenni
git clone https://github.com/davidrjenni/reftools.git 
cd reftools/cmd/fillstruct
go install
```

注意`go.vim`中的`cmd/fillstruct`后缀指的是克隆的项目的子目录，因此流程是先克隆项目本身，再进入相应子目录安装。`go install`安装完后会发现`$GOPATH/bin`下多了二进制文件`fillstruct`。

实际上上述流程是手动进行了`go get`的流程，但是能够判断出错到底是`git clone`还是`go install`，前者出错可能就是访问太慢甚至无法访问，后者则是可能项目里引用了未安装的包。

### 3.2 安装依赖包

并非所有项目都像`fillstruct`那么顺利，比如在安装`errcheck`时就提示：

```
$ go install
internal/errcheck/errcheck.go:19:2: cannot find package "golang.org/x/tools/go/packages" in any of:
    /home/xyz/local/go/src/golang.org/x/tools/go/packages (from $GOROOT)
    /home/xyz/local/go/gopath/src/golang.org/x/tools/go/packages (from $GOPATH)
```

其中`/home/xyz`是我的HOME目录，出现这个错误就是因为`"golang.org/x/tools/go/packages"`包没有安装，而由于`golang.org`被墙了，`go get`安装会失败，不过好在可以从[golang的github镜像站](https://github.com/golang)去找到对应的项目，然后下载安装，以此为例：

```
cd $GOPATH
mkdir -p src/golang.org/x
cd src/golang.org/x
git clone https://github.com/golang/tools.git
```

然后再重新在`errcheck`目录下`go install`即可。

### 3.3 特别操作

1. `stamblerre/gocode`，由于生成的是`gocode-gomod`而非默认的`gocode`，因此需要指定二进制名称（似乎`go install`无法做到）：

```
go build -o gocode-gomod
mv gocode-gomod $GOPATH/bin
```

2. `honnef.co/go/tools/cmd/keyify`，在github上有[镜像](https://github.com/dominikh/go-tools)：

```
cd $GOPATH/src
mkdir -p honnef.co/go
git clone https://github.com/dominikh/go-tools.git honnef.co/go/tools
cd honnef.co/go/tools/cmd/keyify
go install
```

### 3.4 最终安装结果

安装的二进制文件如下：

```
$ ls $GOBIN
asmfmt    fillstruct    godef      golangci-lint  gomodifytags  gotags  impl
dlv       gocode        gogetdoc   golint         gopls         guru    keyify
errcheck  gocode-gomod  goimports  gometalinter   gorename      iferr   motion
```

从golang的github镜像上分别克隆了3个项目：

```
$ ls $GOPATH/src/golang.org/x
lint  sync  tools
```

至于具体使用方式，参考[vim-go Wiki](https://github.com/fatih/vim-go/wiki)。

### 3.5 vimrc配置

```
" 打开go文件时会出现下列错误，解决方法参考[#2301](https://github.com/fatih/vim-go/issues/2301)
" vim-go: Features that rely on gopls will not work correctly in a null module.
let g:go_null_module_warning = 0
```

## 4. 总结

由于是在之前的基础上进行，所以Go开发环境的搭建较为简单，安装Go、重新编译YCM，Ultisnip已经有了，剩下的就是vim-go的安装了。其实最重要的还是YCM的补全功能。

虽然我vim-go用得也不多，主要就是用了下将gofmt集成进vim的`:GoFmt`命令，毕竟上手Go的时间也不长。话说才发现现在保存文件时就会自动格式化了，无需手动输入`:GoFmt`命令。

安装vim-go的主要难点在于各种远程包，在我的电脑上几乎是全程开代理下载，有时候git代理不太好使就直接在VPS上克隆完毕后，打包、FTP下载、FTP上传、解压。

之前实习时安装vim-go去解决各种`go get`的问题了，`go get`虽然方便，但是出错的话难以及时发现错误在哪，感觉在国内这个网络环境下，还不如手动克隆、安装。
