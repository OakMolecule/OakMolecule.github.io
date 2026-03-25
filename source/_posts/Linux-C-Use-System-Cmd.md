---
title: 在Linux C调用系统命令
date: 2020-03-26 15:42:16
tags:
categories:
  - Linux
  - C/C++
  - 操作系统
---

在开发中时不时会在代码中调用系统命令，Linux 中调用系统命令的方法比较多。是否需要获取命令的输出也需要考虑。

# exec函数系列

`exec` 函数系列是指以 `exec` 为开头用来执行Shell命令的几个函数。

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

在其中会使用 `fork()` 调用 `execl()` 函数，最后执行函数

## 返回值

- 如果参数 `command` 为 `NULL`，则在 shell 可用的情况下为非零值，在没有 shell 可用的情况下为 `0`。
- 如果无法`fork`子进程则返回 `-1`，并且将 `errno` 设置为错误值。
- 如果命令无法执行则返回 `127`
- 如果可以执行直接返回执行结果

# popen

`popen()` 是一种更方便的“运行命令并拿到输出”的方式。

## popen 函数

函数定义如下：

``` C
#include <stdio.h>
FILE *popen(const char *command, const char *type);
int pclose(FILE *stream);
```

它做了两件事：

- 启动一个子进程去执行你传进来的 `command`
- 根据 `type` 把子进程和当前进程连起来，让你可以像读写文件一样读/写命令的标准输入/标准输出

### type 常见取值

- `r`：你从 `popen()` 返回的 `FILE *` 里读取命令的输出（子进程的 `stdout`）
- `w`：你向 `FILE *` 写入内容（子进程的 `stdin`）

### 返回值怎么理解

- `popen()` 成功：返回一个 `FILE *`
- `popen()` 失败：返回 `NULL`，并设置 `errno` 表示错误原因
- `pclose()`：关闭这个“管道”，并等待子进程结束；一般情况下返回子进程的结束状态；如果 `pclose()` 本身失败（例如参数不合法等），通常会返回 `-1` 并设置 `errno`

如果你只是需要“能不能跑完”，一般判断 `pclose()` 是否为非零也够用；如果你想拿到更精确的退出码，再结合 `sys/wait.h` 里的宏做进一步判断。

## 示例：读取命令输出

下面这个例子会执行 `ls -l`，并把输出逐行打印出来：

``` C
#include <stdio.h>

int main(void) {
    FILE *fp = popen("ls -l", "r");
    if (!fp) {
        return 1;
    }

    char buf[256];
    while (fgets(buf, sizeof(buf), fp)) {
        fputs(buf, stdout);
    }

    pclose(fp);
    return 0;
}
```

## 普通选择建议

- 只要“执行一下命令”但不关心输出：优先用 `system()`
- 想“读取命令输出”（例如把 `command` 的结果拿来解析）：用 `popen(..., "r")`
- 你的命令参数里包含用户输入、或者不希望走 shell：考虑用 `exec*` 系列配合 `fork()`，自己处理参数和管道（更稳、更安全）

顺便提醒一句：`system()` 和 `popen()` 都会走 shell（对很多场景来说方便，但也更容易受到转义/注入问题影响）。如果 `command` 来自不可信输入，尽量不要直接拼接字符串。

