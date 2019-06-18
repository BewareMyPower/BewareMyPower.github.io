---
title: '我的vim开发环境搭建(1): 准备工作'
date: 2019-06-06 03:43:39
tags:
  - 搭环境
  - vim
top: 1
---

## 前言

即将毕业入职，之前实习时的开发机是CentOS 6.10，因此这里vim开发环境的搭建是基于CentOS 6.10的。

相比更流行的Ubuntu系统，CentOS的毛病实在太多了，折腾起来累死人。另一方面，由于公司开发机并不能像个人电脑一样随心所欲，所以像高版本的vim、gcc等，都是安装到自己的HOME目录，因此绝大多数软件需要手动下载源码、编译、安装到HOME目录。

问题来了，为什么要升级高版本？
1. CentOS 6.10的源默认软件太老了，比如gcc版本才4.4.7，连C++11都不支持；
2. 像YouCompleteMe(YCM)这种优秀的插件，需要各种高版本软件。

为了节约篇幅，后文的下载、解压的命令都省略了，比如下载基本就是找到官网下载链接，然后`wget`命令下载下来，对于github上的项目，直接`git clone`。有时候比较慢，我会在租的VPS上下载好，然后打包传回来。

使用`tar`解压，对`.tar.gz`后缀的用`zxvf`选项来解压，对`.tar.bz`后缀的用`jxvf`选项解压，对`.tar`或`.tar.xz`后缀的用`xvf`选项解压。其中`v`选项不是必要的，只是查看解压了什么内容。

本人在折腾的过程中踩了很多坑，遇到很多错误，借助Google解决了大量问题，后文中不会对这些问题进行描述，而是直接讲述最终尝试后可行的方案。所有尝试均在VMware中新安装的CentOS 6.10系统上进行，系统来源于[CentOS镜像站](http://mirrors.zju.edu.cn/centos/6.10/isos/x86_64/)的bin-DVD1版本。

所有的软件均安装在`$HOME/local`下，我将其设为了环境变量`LOCAL`，通过在`~/.bashrc`中加上如下内容：

```
export LOCAL=$HOME/local
```

## 1. 升级gcc

这里选用了我一直用的gcc版本，也是Ubuntu 16.04默认安装的最新版本：5.4。其实只需要4.8就够了，即完整支持C++11的功能。

在CentOS下需要安装交叉编译IDE依赖，参考[gcc installation error](https://stackoverflow.com/questions/24325754/gcc-installation-error)，我新安装的系统没有g++，因此还需要安装`gcc-c++`。

```
sudo yum install -y glibc-devel.i686 libgcc.i686 gcc-c++
```

去官网给出的[镜像列表](https://www.gnu.org/prep/ftp.html)中找到中国的地址，比如[ustc镜像](https://mirrors.ustc.edu.cn/gnu/)，下载gcc源码并解压，然后进入gcc源码目录，执行以下命令编译：

```
./contrib/download_prerequisites
mkdir build
cd build
../configure --prefix=$LOCAL/gcc-5.4.0 --enable-languages=c,c++
make -j4
make install
```

这里说明下，`make`选项是`-j4`是采用4个进程编译，因为我的个人电脑是4核。上述命令的逻辑就是新建`build`目录，然后`configure`、`make`、`make install`三步走的套路，通过`--prefix`选项指定`install`的目录。

修改`~/.bashrc`文件，设置环境变量来使本地的文件优先于全局的同名文件被选择：

```
export GCC_DIR=$LOCAL/gcc-5.4.0
export PATH=$GCC_DIR/bin:$PATH
export LD_LIBRARY_PATH=$GCC_DIR/lib:$GCC_DIR/lib64:$LD_LIBRARY_PATH
```

比如在我的电脑上这么配置后，`gcc`命令默认就是`~/local/gcc-5.4.0/bin/gcc`而非`/usr/bin/gcc`了。

## 2. 升级某些软件

有了高版本的gcc后便可从源码中直接编译安装高版本的软件了。本节升级的软件版本均已够用，如果需要更高版本的，可以选择下载更高版本的源码进行编译。

### 2.1 安装Python 2.7.16

去官网下载源码解压，进入目录。首先安装依赖项，然后编译、安装（其中`test_weakref`耗时比较久）：

```
sudo yum install -y zlib zlib-devel openssl* bzip2*
./configure --prefix=$LOCAL/python2.7.16/ --enable-shared --with-zlib --enable-optimizations
make -j4
make install
```

在`~/.bashrc`文件中添加环境变量：

```
export PY2_DIR=$LOCAL/python2.7.16
export PATH=$PY2_DIR/bin:$PATH
export LD_LIBRARY_PATH=$PY2_DIR/lib:$LD_LIBRARY_PATH
```

### 2.2 安装cmake 3.14.5

去官网下载源码解压，进入目录。直接编译、安装：

```
./configure --prefix=$LOCAL/cmake-3.14.5
gmake -j4
make install
```

这里的是`gmake`而非`make`，但在Linux上`gmake`只是指向`make`的符号链接，代表GNU Make。

在`~/.bashrc`文件中添加环境变量：

```
export CMAKE_DIR=$LOCAL/cmake-3.14.5
export PATH=$CMAKE_DIR/bin:$PATH
```

### 2.3 安装vim 8.1

去github上克隆vim源码，进入目录，首先安装依赖项，然后编译安装：

```
$ sudo yum -y groupinstall 'Development Tools'
$ sudo yum -y install ruby perl-devel python-devel ruby-devel perl-ExtUtils-Embed ncurses-devel
$ ./configure --prefix=$LOCAL/vim \
 --with-features=huge \
 --enable-multibyte \
 --enable-pythoninterp=yes \
 --with-python-config-dir=$PY2_DIR/lib \
 --enable-luainterp=yes \
 --enable-cscope
$ make VIMRUNTIMEDIR=$LOCAL/vim/share/vim/vim81
$ cd src
$ make
$ make install
```

在`~/.bashrc`文件中添加环境变量：

```
export VIMPATH=$HOME/local/vim
export PATH=$VIMPATH/bin:$PATH
```

### 2.4 安装clang 3.9

去[LLVM下载页](http://releases.llvm.org/download.html)分别下载llvm、clang、clang-tools-extra、compiler-rt源码按照以下命令解压、重命名：

```
wget http://releases.llvm.org/8.0.0/llvm-8.0.0.src.tar.xz
tar xvf llvm-8.0.0.src.tar.xz
mv llvm-8.0.0.src llvm
cd llvm/tools
wget http://releases.llvm.org/8.0.0/cfe-8.0.0.src.tar.xz
tar xvf cfe-8.0.0.src.tar.xz
mv cfe-8.0.0.src clang
cd clang/tools
wget http://releases.llvm.org/8.0.0/clang-tools-extra-8.0.0.src.tar.xz
tar xvf clang-tools-extra-8.0.0.src.tar.xz
mv clang-tools-extra-8.0.0.src extra
cd ../../../projects/
wget http://releases.llvm.org/8.0.0/compiler-rt-8.0.0.src.tar.xz
tar xvf compiler-rt-8.0.0.src.tar.xz
mv compiler-rt-8.0.0.src compiler-rt
```

最终层次结构为：

```
llvm
  \-- tools
     \-- clang
        \-- tools
           \-- extra
  \-- projects
     \-- compiler-rt
```

其中clang目录为重命名后的cfe目录，extra目录为重命名后的clang-tools-extra目录，并且都去掉了后缀版本号。

然后进入llvm目录，执行下列编译操作，注意cmake识别的gcc路径仍然是系统路径，因此需要手动指定CC和CXX路径

```
$ mkdir build
$ cd build
$ CC=$LOCAL/gcc-5.4.0/bin/gcc CXX=$LOCAL/gcc-5.4.0/bin/c++ \
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=$HOME/local/clang8.0 \
-DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD="X86" ..
$ make -j4
$ make install
```

这里多线程编译可能会有问题，我用`-j4`编译到68%的`ASTImporter`时`cc1plus`直接崩了，报错如下：

```
[ 68%] Building CXX object tools/clang/lib/AST/CMakeFiles/clangAST.dir/ASTImporter.cpp.o
c++: internal compiler error: Killed (program cc1plus)
```

在.bashrc文件中添加环境变量

```
export CLANG_DIR=$LOCAL/clang8.0
export PATH=$CLANG_DIR/bin:$PATH
export LD_LIBRARY_PATH=$CLANG_DIR/lib:$LD_LIBRARY_PATH
```

## 3. 安装vim-plug管理插件

```
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

之后编辑`~/.vimrc`便可添加管理的插件，其格式如下：

```
" vim插件下载目录
call plug#begin('~/.vim/plugged')

" vim插件列表，对应github项目地址
Plug 'tpope/vim-sensible'
Plug 'junegunn/seoul256.vim'

" 结束列表
call plug#end()
```

vim命令模式下`:PlugInstall`从github网址中克隆安装所有插件，`:PlugClean`卸载注释掉的插件列表。安装的插件会位于目录`~/.vim/plugged`下，如果要使用代理，可以手动克隆到该路径。

## 4. 插件无关的vimrc配置

将下列代码添加在`~/.vimrc`中：

```
" 定义<leader>为分号，此行代码必须在插件配置代码之前
let mapleader=";"  

" 在处理未保存或只读文件的时候，弹出确认
set confirm
" 自适应不同语言的智能缩进
filetype indent on
" 设置编辑时制表符占用空格数
set tabstop=4
" 设置格式化时制表符占用空格数
set shiftwidth=4
" 将制表符扩展为空格
set expandtab
" 让 vim 把连续数量的空格视为一个制表符
"set softtabstop=4
" 显示行号
set number
"搜索忽略大小写
"set ignorecase
"搜索逐字符高亮
set hlsearch
set incsearch
" 使回格键（backspace）正常处理indent, eol, start等
set backspace=2
" 允许backspace和光标键跨越行边界
set whichwrap+=<,>,h,l
" 在被分割的窗口间显示空白，便于阅读
set fillchars=vert:\ ,stl:\ ,stlnc:\
" 高亮显示匹配的括号
set showmatch
" 匹配括号高亮的时间（单位是十分之一秒）
set matchtime=1
" 光标移动到buffer的顶部和底部时保持3行距离
set scrolloff=3

" 括号补全
inoremap ( ()<ESC>i
inoremap [ []<LEFT>
inoremap " ""<ESC>i
inoremap ' ''<ESC>i
inoremap { {}<ESC>i
inoremap {<CR> {<CR>}<ESC>O

function! ClosePair(char)
    if getline('.')[col('.') - 1] == a:char
        return "\<Right>"
    else
        return a:char
    endif
endfunction
                            
inoremap ) <c-r>=ClosePair(')')<CR>
inoremap } <c-r>=ClosePair('}')<CR>
inoremap ] <c-r>=ClosePair(']')<CR>
```

## 5. 总结

本篇博客主要完成了gcc、python、cmake、vim、clang的升级/安装，均采取了直接编译源码的方式，而像gcc和clang的源码编译非常耗时，python的源码编译虽然较快，但是test_weakref脚本的执行非常慢。

编译源码的方式较为简单，但有些依赖项必须安装，否则找解决方法非常麻烦。

最后安装了vim-plug作为插件管理器，并给出了插件无关的部分vimrc脚本。下一章节将着重介绍我个人常用的插件。
