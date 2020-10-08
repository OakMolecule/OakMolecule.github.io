---
title: Linux进程隐藏及发现
url: 89.html
id: 89
comments: false
categories:
  - Linux
tags:
---

Linux进程隐藏及发现
============

挖矿病毒攻击主机往往会产生高IO，CPU高利用率，造成电脑卡顿，但通过`top`或`ps`命令可能无法找到这些应用。但是这时操作系统确实很卡，这时进程很有可能被隐藏了。

进程隐藏的方法
-------

说到进程隐藏就必须要提到动态链接和静态链接（虽然我也不太懂这个吧），动态链接就是使用动态库，编译的时候很多系统的同文件或者自己写的头文件不编译进去，而静态编译就都编译进去。在Linux中，会有一些默认的动态链接库的保存路径，也可以自己通过设置`LD_LIBRARY_PATH`环境变量来设置。使用GCC编译的过程中可以使用`-L`参数指定。在程序执行时会首先加载在`/etc/ld.so.preload`文件中配置的动态链接库。你的电脑中可能没有，可以通过`touch`命令建立，如果`touch`的时候保错，当然不是`Permission denied`了，你的系统很有可能中招了。 在程序运行中可以通过`strace`来查看所有的动态链接库的调用过程。

![strace ps执行结果](https://zhangpu-vp.cn/wp-content/uploads/2019/11/image-1574071969889.png)

我们可以发现在第4行，程序首先读取配置文件`/etc/ld.so.preload`，读取完成后在第8行打开`/usr/local/lib/libprocesshider.so`文件。此文件源码请参考[GitHub](https://github.com/gianlucaborello/libprocesshider "gianlucaborello/libprocesshider")，按照Readme的方法进行操作即可。

隐藏病毒发现
------

通过观察发现，PS首先加载配置文件，然后加载so文件。这时我们可以直接删除`/etc/ld.so.preload`（但建议使用mv命令，进行重命名，万一里面有正经东西呢），但是这个开源隐藏工具并未隐藏配置文件及so文件，有的工具会直接隐藏这些文件。所以可以直接使用系统调用对文件进行重命名。

    int test_rename(const char *oldFile, const char *newFile)
    {
        if (rename(oldFile, newFile) < 0 )
            printf("error! %s -> %s\n", oldFile, newFile);
        else
            printf("ok! %s -> %s\n", oldFile, newFile);
        return 0;
    }