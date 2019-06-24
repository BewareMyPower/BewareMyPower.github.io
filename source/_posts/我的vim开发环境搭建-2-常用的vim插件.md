---
title: '我的vim开发环境搭建(2): 常用的vim插件'
date: 2019-06-19 16:19:34
tags:
  - 搭环境
  - vim
top: 1
---

## 1. NERDTree 

在[前文](https://bewaremypower.github.io/2019/06/06/%E6%88%91%E7%9A%84vim%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-1-%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C/)中，我主要讲了一堆软件的升级，其主要原因是为了支持YouCompleteMe这个安装起来超级复杂的插件，其余的插件安装很简单，有了vim-plug后，只需要2步，以NERDTree（目录树）为例

第1步，在`.vimrc`中添加插件项目地址，然后在vim中执行`:PlugInstall`命令（也可以直接克隆到`plug#begin`指定的目录，这点前文已经讲过）：

```
call plug#begin('~/.vim/plugged')

" NERDTree插件的github网址（不包含前缀https://github.com/）
Plug 'scrooloose/nerdtree'
" 其他插件
" ...

call plug#end()
```

第2步，添加插件相关的配置脚本，可以参考插件项目的README：

```
nmap <Leader><Leader> :NERDTreeToggle<CR>
let NERDTreeWinSize=32          " 设置NERDTree子窗口宽度
let NERDTreeWinPos="right"      " 设置NERDTree子窗口位置
let NERDTreeShowHidden=1        " 显示隐藏文件
let NERDTreeMinimalUI=1         " NERDTree 子窗口中不显示冗余帮助信息
```

前文配置过`<Leader>`即分号（;）的代码，`nmap`这句就是配置快捷键，也就是连按2个;就等价于执行`:NERDTreeToggle`命令，打开树形目录，如下图所示：

![树形目录](树形目录.jpg)

后面的则是插件的配置，在注释中也给出了。使用方式可以参考项目的README，vim可以通过`Ctrl+W+上/下/左/右`在不同切分窗口中切换，当光标在树形目录中时可以用上下键移动：

| 按键 | 功能 |
| :--- | :--- |
| Enter | 如果光标所在行是目录，展开当前目录，否则直接打开该文件 |
| s | 纵向切分窗口中打开光标所在行对应文件，等价于`:split <filename>` |
| i | 横向切分窗口中打开光标所在行对应文件，等价于`:vsplit <filename>` |
| r | 更新树形目录 |
| u | 将根目录退回到上一层 |
| C | 将根目录变回光标所在行对应目录 |

## 2. YouCompleteMe

```
Plug 'Valloric/YouCompleteMe'
```

[YouCompleteMe](https://github.com/Valloric/YouCompleteMe)（简称YCM）可谓是vim环境下C/C++补全的标配了，不同于其他插件，YCM非常重量级，200多M，而且它对vim版本要求比较高，还要求vim支持python，最后还需要用clang来编译，因此前文做的大量工作都是为这个插件准备的。对于较新的Linux系统，手动编译完高版本的vim后，直接调用`./install.sh --clang-completer`就行，它会自动下载新版本的clang，但是对CentOS 6.10行不通，因此之前我手动编译了高版本的clang。

虽然我也找过替代品，但亲自尝试的结果是，现阶段补全C/C++确实没有比YCM体验更好的。

### 2.1 YCM的安装

之前说过YCM比较大，考虑到国内访问github速度之慢，使用vim-plug安装不如手动克隆，而且这个项目有不少子模块，因此需要`git submodule`，命令如下所示（注意要先进入`~/.vim/plugged`目录）：

```
git clone https://github.com/Valloric/YouCompleteMe.git
git submodule update --init --recursive
```

可以借助代理来加速，不过我是直接下载完毕后将整个目录压缩打包保存起来，下次安装直接上传解压即可。

YCM的安装是利用python脚本完成的，不过安装时可能会提示python缺少某些模块。那是因为之前编译python时没有安装依赖项，参考[前文](https://bewaremypower.github.io/2019/06/06/%E6%88%91%E7%9A%84vim%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-1-%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C/)的2.1节。安装依赖项后重新编译python即可。

进入YouCompleteMe目录，使用本地的gcc、g++、cmake、clang来安装：

```
CC="$LOCAL/gcc-5.4.0/bin/gcc" CXX="$LOCAL/gcc-5.4.0/bin/g++" ./install.py \
  --clang-completer --system-libclang
```

最终会在`third_party/ycmd`目录下生成`ycm_core.so`。

### 2.2 YCM的配置

在`.vimrc`文件中添加如下内容：

```
"" YCM配置
" 全局YCM配置文件路径
let g:ycm_global_ycm_extra_conf = '~/.ycm_extra_conf.py'
let g:ycm_confirm_extra_conf = 0  " 不提示是否载入本地ycm_extra_conf文件
let g:ycm_min_num_of_chars_for_completion = 2  " 输入第2个字符就罗列匹配项

" Ctrl+J跳转至定义、声明或文件
nnoremap <c-j> :YcmCompleter GoToDefinitionElseDeclaration<CR>|

" 语法关键字、注释、字符串补全
let g:ycm_seed_identifiers_with_syntax = 1
let g:ycm_complete_in_comments = 1
let g:ycm_complete_in_strings = 1
" 从注释、字符串、tag文件中收集用于补全信息
let g:ycm_collect_identifiers_from_comments_and_strings = 1
let g:ycm_collect_identifiers_from_tags_files = 1

" 禁止快捷键触发补全
let g:ycm_key_invoke_completion = '<c-z>'  " 主动补全(默认<c-space>)
noremap <c-z> <NOP>

" 输入2个字符就触发补全
let g:ycm_semantic_triggers = {
            \ 'c,cpp,python,java,go,erlang,perl': ['re!\w{2}'],
            \ 'cs,lua,javascript': ['re!\w{2}'],
            \ }
                                    
let g:ycm_show_diagnostics_ui = 0  " 禁用YCM自带语法检查(使用ale)

" 防止YCM和Ultisnips的TAB键冲突，禁止YCM的TAB
let g:ycm_key_list_select_completion = ['<C-n>', '<Down>']
let g:ycm_key_list_previous_completion = ['<C-p>', '<Up>']
```

输入2个字符即可触发补全操作，对C++而言，输入`.`、`::`、`->`也会触发补全操作。

除了补全，YCM本身也含有跳转功能，这里设置快捷键为Ctrl+J。但是目前似乎还不太完善，比如有时候只能跳转到声明无法跳转到定义。这些功能可能还需要借助ctags、cscope来辅助。

YCM对C/C++的补全、跳转是基于`.ycm_extra_conf.py`文件的，比如指定语言种类、标准、头文件目录、编译选项，其中编译选项是用来实现语法检查的，这里禁止了YCM自身的语法检查，使用后文将介绍的ALE插件来完成。其配置参考我的博客：[为YCM配置ycm_extra_conf脚本](https://www.jianshu.com/p/5aaae8f036c1)

CentOS配置比较简单，C头文件都在`/usr/include`下，C++头文件则在自己安装的gcc目录下：

```
'-isystem',
'/usr/include',
'-isystem',
os.environ['HOME'] + '/local/gcc-5.4.0/include/c++/5.4.0',
'-isystem',
os.environ['HOME'] + '/local/gcc-5.4.0/include/c++/5.4.0/x86_64-unknown-linux-gnu'
```

需要注意的是`x86_64-unknown-linux-gnu`目录必须添加进去，否则会导致C++标准库无法补全，因为C++标准库头文件包含了该目录下的各种头文件，比如`thread`包含的3个头文件：

```
#include <bits/functexcept.h>
#include <bits/functional_hash.h>
#include <bits/gthr.h>
```

其中前2个都是出自`$GCC_DIR/include/c++/5.4.0/bits`目录，而第3个则是出自`$GCC_DIR/include/c++/5.4.0/x86_64-unknown-linux-gnu/bits`目录，因此必须把这个目录添加进去，否则头文件会解析失败。

## 3 ale静态语法检查

### 3.1 ale的安装和配置

```
Plug 'w0rp/ale'
```

其配置如下：

```
"" ale
let g:ale_linters = {
            \ 'c': ['gcc', 'cppcheck'],
            \ 'cpp': ['g++', 'cppcheck'],
            \ }
let g:ale_c_gcc_options = '-Wall -O2 -std=c99
            \ -I .
            \ -I /usr/include'
let g:ale_cpp_gcc_options = '-Wall -O2 -std=c++11
            \ -I .
            \ -I /usr/include
            \ -I $HOME/local/gcc-5.4.0/include/c++/5.4.0'
let g:ale_c_cppcheck_options = ''
let g:ale_cpp_cppcheck_options = ''

let g:ale_linters_explicit = 1  " 只显示运行ale_linters的文件
let g:ale_completion_delay = 500
let g:ale_echo_delay = 20
let g:ale_lint_delay = 500
let g:ale_echo_msg_format = '[%linter%] %code: %%s'
let g:ale_lint_on_text_changed = 'normal'  " 防止YCM不停补全
let g:ale_lint_on_insert_leave = 1
```

主要有用的是前面5个，后面照搬即可。`ale_c_gcc_options`和`ale_cpp_gcc_options`指定使用gcc/g++进行静态语法检查时的选项，和编译时的选项相同，类似YCM的`.ycm_extra_conf.py`，但是必须在`.vimrc`中配置而不能作为项目独立的配置文件。

另外，除了gcc/g++外，ale还使用了cppcheck进行语法检查，它能够检查出一些未定义的行为，比如没有`fclose`关闭打开的`FILE`指针。从源安装的cppcheck版本较老，不支持C++11的语法，因此去下载[cppcheck源码](http://cppcheck.sourceforge.net/)手动编译。

### 3.2 安装cppcheck

要支持`HAVE_RULES`，需要安装[pcre](www.pcre.org)，下载源码解压后，手动`./configure --prefix=<dirname>`、`make`、`make install`安装，然后在`~/.bashrc`中添加如下代码以指定gcc的包含路径和库路径：

```
export C_INCLUDE_PATH=$LOCAL/pcre/include:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$LOCAL/pcre/include:$CPLUS_INCLUDE_PATH
export LD_LIBRARY_PATH=$LOCAL/pcre/lib:$LD_LIBRARY_PATH
export PATH=$LOCAL/pcre/bin:$PATH
```

然后进入cppcheck源码目录按照如下方式编译：
```
make SRCDIR=build CFGDIR=$LOCAL/bin/cfg HAVE_RULES=yes
make install PREFIX=$LOCAL CFGDIR=$LOCAL/bin/cfg
```

然后将`$LOCAL/bin`添加到`.bashrc`的环境变量中：

```
export PATH=$LOCAL/bin:$PATH
```

之后ale便可使用cppcheck进行语法检查，比如C++11语法下的资源泄漏检查：

![ale检查资源泄漏](ale检查资源泄漏.jpg)

除此之外还有其他功能，可参考[cppcheck官网](http://cppcheck.sourceforge.net/)的Undefined behaviour一栏。

## 4. 其他插件

其实最重要的补全、跳转插件都有了，其余的插件我也没详细研究，但也都配置了，使用开箱即用的功能，没有特别定制。

### 4.1 Ultisnips

```
Plug 'SirVer/ultisnips'
Plug 'honza/vim-snippets'
```

很多IDE都有自动生成代码的功能，比如输入main然后按回车或TAB键会自动生成main函数的模板代码，输入for按回车生成可供选择的多种for循环的模板代码。这里第1个插件提供了代码生成的功能，第2个插件则提供了各自语言的模板代码。

```
" Do not use <tab> if you use YouCompleteMe
let g:UltiSnipsExpandTrigger="<tab>"
let g:UltiSnipsJumpForwardTrigger="<c-b>"
let g:UltiSnipsJumpBackwardTrigger="<c-z>"

" If you want :UltiSnipsEdit to split your window.
let g:UltiSnipsEditSplit="vertical"
```

暂时我还只停留在简单地使用现成模板上，比如`cl`生成类，`ns`生成命名空间，`main`生成main函数，当然，这些都会用TAB键触发，因此之前YCM的配置中禁止了TAB补全防止与其冲突。

C++的snippets在`~/.vim/plugged/vim-snippets/snippets/cpp.snippets`文件定义，可以照着修改该文件来修改代码生成模板。

### 4.2 vim-clang-format

```
Plug 'rhysd/vim-clang-format'
```

之前安装llvm中自带了`clang-format`，`vim-clang-format`是对它的集成。无需配置，安装好后，vim命令模式下`:ClangFormat`便可对当前文件自动格式化。当然也可以`nmap`设置快捷键，不过我也习惯了输入`:Cl`+TAB键格式化当前代码。

代码格式化首先需要`clang-format`路径在环境变量`PATH`中（之前安装clang时已经配置过），然后从当前目录往上寻找`.clang-format`文件，我是直接用谷歌开源代码风格，比如我生成该文件在HOME目录下，命令如下：

```
clang-format --style=Google --dump-config >~/.clang-format
```

如果需要定制代码格式化文件，只需修改`.clang-format`即可，网上有各种资料。

### 4.3 C++头文件、源文件切换

```
Plug 'vim-scripts/a.vim'
```

开箱即用，据说是很早的插件了，`:A`命令切换头文件和源文件，比如`xxx.h`和`xxx.cc`。`:AV`和`:AS`命令则是打开竖直、水平切分窗口来切换头文件和源文件。

### 4.4 asyncrun

```
Plug 'skywind3000/asyncrun.vim'
```

[skywind](http://www.skywind.me/blog/)大佬（知乎账号为*韦易笑*，vim大神)写的插件，我的vim配置也大部分是参考他的[Vim 8下C/C++开发环境搭建](http://www.skywind.me/blog/archives/2084)。这里也基本照搬配置了，使用说明可参考他的博客：

```
" 自动打开 quickfix window ，高度为 6
let g:asyncrun_open = 6

" 任务结束时候响铃提醒
let g:asyncrun_bell = 1

" 设置 F10 打开/关闭 Quickfix 窗口
nnoremap <F10> :call asyncrun#quickfix_toggle(6)<cr>

" 设置 F9 编译单个文件
nnoremap <silent> <F9> :AsyncRun g++ -std=c++11 -Wall -O2 "$(VIM_FILEPATH)" -o "$(VIM_FILEDIR)/$(VIM_FILENOEXT)" <cr>

" 递归查找包含该目录的目录作为根目录，若找不到则将文件所在目录作为当前目录
let g:asyncrun_rootmarks = ['.svn', '.git', '.root', '_darcs', 'build.xml'] 
" 设置 F7 编译项目
nnoremap <silent> <F7> :AsyncRun -cwd=<root> make <cr>

" 设置 F5 运行当前程序
"nnoremap <silent> <F5> :AsyncRun -raw -cwd=$(VIM_FILEDIR) "$(VIM_FILEDIR)/$(VIM_FILENOEXT)" <cr>
nnoremap <silent> <F5> :AsyncRun g++ -std=c++11 -Wall "$(VIM_FILEPATH)" && ./a.out <cr>"
```

由于vim 8才支持了异步执行终端命令，因此该插件需要vim版本至少为8。我的vim版本为8.1，还可以直接在vim中横向打开终端窗口（命令为`terminal`，默认横向打开，`vert term`纵向打开）。

### 4.5 LeaderF

```
Plug 'Yggdroot/LeaderF'
```

其配置照抄了skywind的

```
"" LeaderF
" Ctrl+P在当前项目目录打开文件搜索, Ctrl+N打开MRU搜索，搜索最近打开的文件
" Alt+P打开函数搜索，Alt+N打开Buffer搜索
let g:Lf_ShortcutF = '<c-p>'
let g:Lf_ShortcutB = '<m-n>'
noremap <c-n> :LeaderfMru<cr>
noremap <m-p> :LeaderfFunction!<cr>
noremap <m-n> :LeaderfBuffer<cr>
noremap <m-m> :LeaderfTag<cr>
let g:Lf_StlSeparator = { 'left': '', 'right': '', 'font': '' }

let g:Lf_RootMarkers = ['.project', '.root', '.svn', '.git']
let g:Lf_WorkingDirectoryMode = 'Ac'
let g:Lf_WindowHeight = 0.30
let g:Lf_CacheDirectory = expand('~/.vim/cache')
let g:Lf_ShowRelativePath = 0
let g:Lf_HideHelp = 1
let g:Lf_StlColorscheme = 'powerline'
let g:Lf_PreviewResult = {'Function':0, 'BufTag':0}
```

各种`Leaderf`开头的命令，可以`:LeaderfFunction`查找函数、`:LeaderfFile`查找文件，等等。我用得不是很多，所以配置基本上也没用。

### 4.6 vim-signify

```
Plug 'mhinz/vim-signify'
```

配置仅需

```
set signcolumn=yes  " 强制显示侧边栏，防止时有时无
```

可以实时显示git项目的修改状态，如下图所示：

![signify显示git项目修改情况](signify显示git项目修改情况.jpg)

### 4.7 vim页面美化插件

```
Plug 'sickill/vim-monokai'               " monokai主题
Plug 'vim-airline/vim-airline'           " 美化状态栏
Plug 'vim-airline/vim-airline-themes'
Plug 'plasticboy/vim-markdown'           " markdown高亮
Plug 'octol/vim-cpp-enhanced-highlight'  " C++代码高亮
```

配置如下：

```
"" airline
let laststatus = 2
let g:airline_powerline_fonts = 1
let g:airline_theme = "dark"
let g:airline#extensions#tabline#enabled = 1

"" vim-monokai
colorscheme monokai

"" vim-markdown
" Github风格markdown语法
let g:vim_markdown_no_extensions_in_markdown = 1

"" vim-cpp-enhanced-highlight
let g:cpp_class_scope_highlight = 1
let g:cpp_member_variable_highlight = 1
let g:cpp_class_decl_highlight = 1
let g:cpp_experimental_template_highlight = 1
```
## 5. 我自己写的简单代码模板

之前简单学了下vimscripts，简单写了点代码模板，即新建`.h`、`.c`、`.cc`、`.cpp`文件会自动生成相应代码。

```
" 代码模板
autocmd BufNewFile *.cc,*.cpp,*.c exec ":call SetTitle()"
function SetTitle()
    if &filetype == 'cpp'
        call setline(1, "#include <iostream>")
        call setline(2, "using namespace std;")
        call setline(3, "")
        call setline(4, "int main(int argc, char* argv[]) {")
        call setline(5, "  return 0;")
        call setline(6, "}")
    endif
    if &filetype == 'c'
        call setline(1, "#include <stdio.h>")
        call setline(2, "#include <stdlib.h>")
        call setline(3, "")
        call setline(4, "int main(int argc, char* argv[]) {")
        call setline(5, "  return 0;")
        call setline(6, "}")
    endif
    " 新建文件后自动定位到文件末尾
    autocmd BufNewFile * normal G
endfunction

autocmd BufNewFile *.h exec ":call SetTitleForH()"
function SetTitleForH()
    let name = toupper("".expand("%"))
    let name = join(split(name, '\.'), '_')
    let name = join(split(name, '-'), '_')
    call setline(1, join(['#ifndef', name]))
    call setline(2, join(['#define', name]))
    call setline(3, join(['#endif  //', name]))
    exec ":2"
endfunction
```

## 6. 总结

本篇介绍了搭建C/C++开发环境中的一些插件，虽然对于一般人我可以算vim狂热者了，但是对于真正熟悉vim的人来说，我还有很多需要学习的，比如很多插件我也是浅尝辄止，但毕竟vim不同于IDE的一点就是其配置高度个人化，真正觉得需要提高效率了再去看看是不是需要详细了解下插件怎么用。

我的vim-plug管理的插件如下：

```
call plug#begin('~/.vim/plugged')

Plug 'Valloric/YouCompleteMe'
Plug 'scrooloose/nerdtree', { 'on': 'NERDTreeToggle' }
Plug 'w0rp/ale'
Plug 'SirVer/ultisnips'
Plug 'honza/vim-snippets'
Plug 'rhysd/vim-clang-format'
Plug 'vim-scripts/a.vim'
Plug 'skywind3000/asyncrun.vim'
Plug 'Yggdroot/LeaderF'
Plug 'mhinz/vim-signify'
Plug 'sickill/vim-monokai'               " monokai主题
Plug 'vim-airline/vim-airline'           " 美化状态栏
Plug 'vim-airline/vim-airline-themes'
Plug 'plasticboy/vim-markdown'           " markdown高亮
Plug 'octol/vim-cpp-enhanced-highlight'  " C++代码高亮

call plug#end()
```

如果vim编辑比较卡的话，可以试着注释掉某些插件，比如我实测C++代码高亮有时候会拖慢速度。YCM如果配置于某些较为庞大的项目可能也有点问题。

`~/.vim/plugged`目录下各插件占用磁盘大小：

```
$ du -h --max-depth=1 . | grep -v "\.$"
512K    ./vim-clang-format
1.8M    ./vim-snippets
240K    ./a.vim
8.8M    ./ultisnips
1.4M    ./LeaderF
436K    ./vim-cpp-enhanced-highlight
908K    ./nerdtree
1.9M    ./asyncrun.vim
341M    ./YouCompleteMe
992K    ./vim-markdown
2.3M    ./vim-signify
820K    ./vim-airline-themes
1.2M    ./vim-airline
24M     ./ale
224K    ./vim-monokai
```

YCM目录包含了编译的文件，所以从200多M涨到了341M。
