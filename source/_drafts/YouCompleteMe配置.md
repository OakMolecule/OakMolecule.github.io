---
title: YouCompleteMe配置
url: 148.html
id: 148
comments: false
categories:
  - vim
tags:
  - vim
  - YouCompleteMe
---

YouCompleteMe安装及配置
==================

记录一下几天YCM的配置记录，保存一下`.vimrc`，`.ycm_extra_conf.py`。顺便和大家分享一下我配置经验。

安装Vundle
--------

首先需要安装Vundle，这是vim的一个插件管理器。也很容易，需要安装git。 首先执行下面的命令：

    git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim

然后编辑`.vimrc`，插入一下内容

    set nocompatible              " be iMproved, required
    filetype off                  " required
    
    " set the runtime path to include Vundle and initialize
    set rtp+=~/.vim/bundle/Vundle.vim
    call vundle#begin()
    
    " let Vundle manage Vundle, required
    Plugin 'VundleVim/Vundle.vim'
    
    call vundle#end()            " required
    filetype plugin indent on    " required

保存并退出。 命令行安装执行：`vim +PluginInstall +qall` vim中安装执行`:PluginInstall` 这时你再打开vim会发现配色都没了，甚至`{Backspace}`都每办法使用。这里不用担心后面我会打开他们的。

安装YouCompleteMe
---------------

**确保您的Vim版本至少为7.4.1578，并支持Python 2或Python 3脚本。**

1.  执行命令`vim --version`检查vim版本是否高于7.4.1578+；
2.  检查vim是否有python或python3的支持，在命令模式中执行`echo has('python') || has('python3')`，输出如果为1则支持；0则不支持，意味着你可能要从源码编译vim了，并不难，参考[从源码编译vim](https://github.com/Valloric/YouCompleteMe/wiki/Building-Vim-from-source "编译vim")。
3.  编辑`.vimrc`，在`Plugin 'VundleVim/Vundle.vim'`后中添加一行`Plugin 'Valloric/YouCompleteMe'`。 关闭重新打开vim，然后执行`:PluginInstall`，这时插件就下载完成了。

### 编译`ycm_core`库

首先安装`clang`、`python-dev`或`python3-dev`、`cmake`。

    cd ~
    mkdir ycm_build
    cd ycm_build
    cmake -G "Unix Makefiles" . ~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp

到此YouCompleteMe的安装就结束了。

配置VIM
-----

YouCompleteMe的安装工作不算难，难的是配置工作。 以下是我vim的一些配置。 [https://github.com/OakMolecule/dot_vimrc](https://github.com/OakMolecule/dot_vimrc)

C/C++自动补全
---------

将补全的头文件默认配置复制到用户目录，`cp ~/.vim/bundle/YouCompleteMe/third_party/ycmd/examples/.ycm_extra_conf.py ~/.ycm_extra_conf.py` YCM会根据目录结构寻找.ycm\_extra\_conf.py的，首先从当前目录一直向上查找，直到找到根目录。如果一直没有则找用户目录的配置。 YCM也推荐了一个根据用户工程生成.ycm\_extra\_conf.py的工具[YCM-Generator](https://github.com/rdnetto/YCM-Generator "YCM-Generator")，但好像需要Makefile或者CMakeLists.txt。现在还不太会用，等回来再补上。

一篇知乎的vim配置分享[如何在 Linux 下利用 Vim 搭建 C/C++ 开发环境?--韦易笑的回答](https://www.zhihu.com/question/47691414/answer/373700711 "如何在 Linux 下利用 Vim 搭建 C/C++ 开发环境?--韦易笑的回答")