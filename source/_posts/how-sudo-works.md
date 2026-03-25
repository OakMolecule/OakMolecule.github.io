---
title: sudo的工作原理
tags:
  - Linux
url: 113.html
id: 113
comments: false
categories:
  - Linux
date: 2019-12-17 12:27:40
---

# sudo的工作原理
此文不介绍sudoers文件的配置方法，只是简单介绍sudo工作流程。
## 工作流程
工作流程如下图，第一次用markdown写流程图，不太精细。
``` mermaid
flowchart TD
    st([用户执行sudo])
    is_have_cookie{检查 /var/run/sudo/ts 文件夹下是否有认证文件}
    cookie_is_expire{检查认证文件是否有效}
    is_in_sudoers{是否在 /etc/sudoers 文件中}
    input_passwd[输入密码]
    op[执行并返回结果]
    e([退出])

    st --> is_have_cookie

    is_have_cookie -- yes --> cookie_is_expire
    is_have_cookie -- no --> input_passwd

    cookie_is_expire -- yes --> is_in_sudoers
    cookie_is_expire -- no --> input_passwd

    input_passwd --> is_in_sudoers

    is_in_sudoers -- yes --> op
    is_in_sudoers -- no --> e

    op --> e
```

## 认证文件简介
认证文件可以理解为cookie，里面记录认证时间，在终端的PID，用户ID，终端的ttydev。
解析代码参考[Github](https://github.com/nongiach/sudo_inject/ "Github")。
认证文件以二进制保存，每个用户会有一个自己的cookie文件，放在`/var/run/sudo/ts`文件夹下，文件名为用户名。
文件保存结构如下：
``` C
struct timestamp_entry_v1 {
    unsigned short version;	/* version number */
    unsigned short size;	/* entry size */
    unsigned short type;	/* TS_GLOBAL, TS_TTY, TS_PPID */
    unsigned short flags;	/* TS_DISABLED, TS_ANYUID */
    uid_t auth_uid;		/* uid to authenticate as */
    pid_t sid;			/* session ID associated with tty/ppid */
    struct timespec ts;		/* time stamp (CLOCK_MONOTONIC) */
    union {
    dev_t ttydev;		/* tty device number */
    pid_t ppid;         /* parent pid */
    } u;
};

struct timestamp_entry {
    unsigned short version;	/* version number */
    unsigned short size;	/* entry size */
    unsigned short type;	/* TS_GLOBAL, TS_TTY, TS_PPID */
    unsigned short flags;	/* TS_DISABLED, TS_ANYUID */
    uid_t auth_uid;		/* uid to authenticate as */
    pid_t sid;			/* session ID associated with tty/ppid */
    struct timespec start_time;	/* session/ppid start time */
    struct timespec ts;		/* time stamp (CLOCK_MONOTONIC) */
    union {
	dev_t ttydev;		/* tty device number */
	pid_t ppid;		/* parent pid */
    } u;
};
```

有两种保存方式，通过`version`进行区分，当`version`值为1时使用上面的结构体；剩下的使用下面的结构体。
`type`字段表示这下方的`union`中使用哪个字段，`type`会使用以下几个值：
``` C
#define TS_GLOBAL               0x01    /* not restricted by tty or ppid */
#define TS_TTY                  0x02    /* restricted by tty */
#define TS_PPID                 0x03    /* restricted by ppid */
#define TS_LOCKEXCL             0x04    /* special lock record */
```

`flags`字段自行研究吧，没有太关注。
`auth_uid`这个是授权用户的UID。
`sid`这个表示所在终端的PID，但是这个有一个东西需要注意，如果这个账户是通过别的账户`su`过来的，那么他的值为最上层终端的PID，不太清楚原因。


