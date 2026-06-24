---
title: Linux 进程隐藏及发现
date: 2026-03-25 12:00:00
comments: false
categories:
  - Linux
tags:
  - Linux
  - 进程隐藏
---

{% note warning %}
# 声明

1. 本文内容仅用于安全研究与技术学习，旨在分析隐藏进程相关实现及检测思路。
2. 文中涉及的技术与代码具有潜在滥用风险，请勿用于未授权场景或任何非法用途。
3. 相关实验与验证应在隔离的测试环境中进行，避免对真实系统造成影响。
4. 作者不对因使用本文内容而产生的任何直接或间接后果负责。
{% endnote %}

# 什么是应用层隐藏进程

应用层隐藏进程本质不是让进程真的从内核消失，而是：

> **让“观察者看不到”或“看到被篡改后的结果”**

换句话说：

- 内核里的进程仍然存在
- 只是用户态工具（`ps`、`top`等）被“喂了假数据”

因此，应用层隐藏的核心是：

> **拦截信息获取路径 + 篡改结果**

<!-- more -->

# 应用层隐藏的常见实现路径

在 Linux 用户态，进程信息主要来源于：

- `/proc` 文件系统（如 `/proc/[pid]/stat`）

而读取文件大多依赖标准库接口，如 `readdir`、`open`、`read`。上层工具`ps`、`top` 本质也是用上述函数读 `/proc`。

所以实现隐藏的关键点通常是：

## Hook libc 接口

思路：

- 通过动态链接劫持（如 `LD_PRELOAD`）
- 替换标准库函数
- 在返回结果前做过滤

典型拦截点：

- `readdir` / `readdir64`
- `open` / `openat`
- `read`
- `fopen`

## 如何实现

下面用更容易读的方式解释一下的核心逻辑：

{% note info %}
下面代码为伪代码，仅用于理解原理。
{% endnote %}


```c
static struct dirent *(*original_readdir)(DIR *) = NULL;

// 这是一个重定义的 `readdir`，会替代 libc 默认版本。
struct dirent *readdir(DIR *dirp) {
    if (original_readdir == NULL) {
        // 找到 libc 中真正的 readdir 函数地址
        original_readdir = &libc_readdir;
    }

    while (1) {
        struct dirent *dir = original_readdir(dirp);
        if (dir == NULL) {
            return NULL;
        }

        if (当前遍历的是 /proc &&
            当前目录名是数字 PID &&
            该 PID 对应的进程名命中隐藏规则) {
            // 跳过该进程，实现隐藏
            continue;
        }

        return dir;
    }
}
```

### 为什么要用找真实的 `readdir` 函数

因为共享库自己实现了一个同名 `readdir`。如果它在内部再次直接调用 `readdir`，就会递归回自己，进而死循环。

### 为什么盯着 `/proc`

Linux 里每个进程通常对应 `/proc/<pid>` 下的一个目录。很多进程枚举工具本质上就是：

- 打开 `/proc`
- 逐个读取目录项
- 挑出纯数字目录名
- 再去读 `/proc/<pid>/stat`、`cmdline`、`status` 等文件

因此，只要在“读取 `/proc` 目录项”这个点上做过滤，很多上层工具都会一起失真。

## 扩展点

Hook `readdir` 这种方式只能保证在直接执行 `ps`，`top` 等工具时，隐藏进程。但如果换成 `top -p <pid>`，`ps -p <pid>` 等命令时，就无法隐藏。因为这类命令不会再调用 `readdir` 函数遍历 `/proc` 目录。

实际的样本会 Hook 更多的函数，以保证在更多场景下隐藏进程。

# 这种隐藏方式的本质特点

从安全研发角度：

1. **仅影响查看，不影响实体**

   * 内核中的 task_struct 仍存在
   * 调度、信号、资源占用不受影响

   👉 本质是“观测欺骗”，而不是“抹除进程”

2. **强依赖注入机制**

   * 常用方式：`LD_PRELOAD` 或 `/etc/ld.so.preload`
   * 一旦注入失效，隐藏立即失效

3. **路径依赖**

   * Hook 只影响通过被替换接口读取 `/proc` 的工具
   * 静态链接程序或直接系统调用可能绕过

# 检测与防御思路

核心原则：

> **绕过用户态 hook + 多路径交叉验证**

1. **使用静态链接程序直接读 `/proc`**

   * 工具如 `busybox`、自己用 Go/C 静态编译的程序
   * 避免 libc hook 干扰

2. **检查注入机制**

   * 查看 `LD_PRELOAD` 环境变量
   * 检查 `/etc/ld.so.preload`
   * 异常共享库加载也可能提示隐藏行为

3. **`strace` `ps` 进程

   * 查看是否加载了 `/etc/ld.so.preload`，一般系统不会有这个文件
   * 查看是否价值了奇怪的库（当然样本经常会把自己伪装的很真实），例如下图
   ![](/uploads/2019/11/image-1574071969889.png)
