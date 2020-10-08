---
title: 在Linux C调用系统命令
date: 2020-03-26 15:42:16
tags:
categories:
  - Linux
  - C/C++
---

在开发中时不时会在代码中调用系统命令，Lnux中调用系统命令的方法比较多。是否需要获取命令的输出也需要考虑。

# exec函数系列
`exec`函数系列是指以`exec`为开头用来执行Shell命令的几个函数。

``` C
#include <unistd.h>

extern char **environ;

int execl(const char *pathname, const char *arg, ...
                /* (char  *) NULL */);
int execlp(const char *file, const char *arg, ...
                /* (char  *) NULL */);
int execle(const char *pathname, const char *arg, ...
                /*, (char *) NULL, char *const envp[] */);
int execv(const char *pathname, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[],
                char *const envp[]);
```

# system
函数定义如下：
``` C
#include <stdlib.h>
int system(const char *command);
```
在其中会使用`fork()`调用`execl()`函数，最后执行函数
## 返回值
- 如果参数`command`为NULL，则在shell可用的情况下为非零值，在没有shell可用的情况下为0。
- 如果无法`fork`子进程则返回-1，并且将errno设置为错误值。
- 如果命令无法执行则返回127
- 如果可以执行直接返回执行结果

# popen


