---
title: Google Protocol Buffer
url: 57.html
id: 57
categories:
  - 未分类
tags:
---

Google Protocol Buffer
======================

简介
--

[Protocol Buffer](https://developers.google.cn/protocol-buffers "Protocol Buffer")是Google的与语言无关，与平台无关，可扩展的机制，用于对结构化数据进行序列化（例如XML），但更小，更快，更简单。 您定义要一次构造数据的方式，然后可以使用生成的特殊源代码轻松地使用各种语言在各种数据流中写入和读取结构化数据。

安装
--

首先安装autogen，make

    $ sudo apt-get install autogen make

然后执行以下命令进行编译安装

    # 直接通过git下载
    git clone https://github.com/protocolbuffers/protobuf.git
    cd protobuf
    ./autogen.sh
    ./configure
    # 编译
    make
    # 安装
    sudo make install

通过git安装安装的是最新版本，不安装最新版本可通过下载releases版本代码再进行编译。

使用
--