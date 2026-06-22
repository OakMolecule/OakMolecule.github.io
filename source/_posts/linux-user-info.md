---
title: Linux 用户信息详解
categories:
  - Linux
date: 2019-11-01 21:10:21
tags:
  - Linux
  - 用户信息
---

# Linux 用户信息详解
Linux用户信息很丰富，存储在了多个文件中，其中包括 `/etc/passwd`，`/etc/shadow`，`/var/log/wtmp` 等文件中，因为在项目中用到了，所以有些了解。下面对这几个文件进行一个详细的介绍。

## 账户基本信息

### 文件各字段含有

`/etc/passwd` 存储了账户的基本信息，默认任何用户都可以读取，也是早期存储账户密码的文件，后因任何用户都可读取，造成安全问题，故后期不再用于存储。
`/etc/passwd` 文件采用`:`分割，共分为了7个字段，下面是文件中的一行。

```
root:x:0:0:root:/root:/usr/bin/bash
```

<!-- more -->

- 第一列：用户名；
- 第二列：为用户密码Flag，具体信息没有找到，根据个人推测是为了兼容早期版本，早期此字段用于存储用户密码，后期为了安全将密码放至 `/etc/shadow` 文件中。推测为了兼容，字段仍然保留。大部分用户此字段值为`x`，表示密码保存在 `/etc/shadow` 文件中，不是 `x` 则就是加密后的密码；
- 第三列：用户 ID，这个好理解，每个用户对应一个唯一的 ID，就像身份证号一样；
- 第四列：组 ID，也就是用户所属组的 ID 了。Linux 中每个用户只能在一个组中，但可以有多个附加组；
- 第五列：用户的一些描述，可能是姓名等，可以为空；
- 第六列：用户的 Home 目录，用于设置环境变量 `$HOME` ；
- 第七列：用户的默认终端，如果用户被禁止登录了此字段为`/usr/bin/nologin`，只要登录就会得到`This account is currently not available.`的提示。

### C 语言通过函数读取

在头文件 `pwd.h` 中的函数可以获取到 `/etc/passwd` 文件中的账户信息。
在 `pwd.h` 中定义了结构体`passwd`，结构体内容如下：

```c
struct passwd {
    char   *pw_name;       /* username */
    char   *pw_passwd;     /* user password */
    uid_t   pw_uid;        /* user ID */
    gid_t   pw_gid;        /* group ID */
    char   *pw_gecos;      /* user information */
    char   *pw_dir;        /* home directory */
    char   *pw_shell;      /* shell program */
};
```

结构体中的内容与 `/etc/passwd` 文件中的内容对应，就不详细介绍了。
头文件中也定义了一些相关的函数
。
``` C
struct   passwd    *getpwuid(uid_t);
int                 getpwnam_r(const char *, struct passwd *, char *, size_t, struct passwd **);
int                 getpwuid_r(uid_t, struct passwd *, char *, size_t, struct passwd **);
void                endpwent(void);
struct   passwd    *getpwent(void);
void                setpwent(void);
```

下面将介绍几个函数的作用：
`getpwuid`通过UID查找用户信息，返回`passwd`的结构体；
`getpwnam_r`和`getpwuid_r`不知道是做啥的。
`setpwent`、`getpwent`和`endpwent`是一套函数，`getpwent`是用来获取一行数据，每调用一次读取指针就向下走一行，直至读完返回NULL，`setpwent`将读取指针返回文件头，`endpwent`将文件指针返回尾部。

## 账户密码信息

### 文件各字段含有

`/etc/shadow`文件存储了账户的密码相关信息，内容还是很丰富的，与`passwd`相同，使用`:`分隔字段，`/etc/shadow`共分成了9个字段，其中一个为保留字段。`/etc/shadow`中储存了用户的密码，只用拥有root权限的用户才可以读写。

```
root:$6$K9DgMAqih5hXEJgh$fK3s53mo/YqeZe8v1NKuKHpUPdyuHjFT/d9pM0u2xFxMVAOwjX51cCr5BEqmMaAoISd35PA/cow1q.YoiOLTH0:18193:0:99999:7:::
```

上面就是`/etc/shadow`文件的一行，下面我将详细介绍各个字段的含义：
- 第一列：用户名；
- 第二列：加密后的密码，这里我的密码是1。貌似没有办法破解，但是可以通过字典的方式获得；

```
星号代表帐号被锁定，但是还是可以通过其他方式切换进去的，不同理解到底有啥用
双叹号表示这个密码已经过期了。
$6$ 开头的，表明是用SHA-512加密的，
$1$ 表明是用MD5加密的
$2$ 是用Blowfish加密的
$5$ 是用SHA-256加密的。
```

- 第三列：账户最后一次修改密码的时间，是自1970年1月1日以来天数；
- 第四列：自上次密码修改以来多少天内不能修改；
- 第五列：口令保持有效的最大天数，自上次密码修改以来多少天之后必须修改密码的时间；
- 第六列：警告时间，密码过期前几天提醒用户修改；
- 第七列：密码无效期，自过期以来的数天之内，密码仍然有效；
- 第八列：账户过期时间；
- 第九列：保留字段。

### C 语言通过函数读取

在头文件`shadow.h`中的函数可以获取到`/etc/shadow`文件中的账户信息。
在`shadow.h`中定义了结构体`spwd`，结构体内容如下：

``` C
struct spwd {
    char *sp_namp;          /* Login name */
    char *sp_pwdp;          /* Encrypted password */
    long  sp_lstchg;        /* Date of last change
                               (measured in days since
                               1970-01-01 00:00:00 +0000 (UTC)) */
    long  sp_min;           /* Min # of days between changes */
    long  sp_max;           /* Max # of days between changes */
    long  sp_warn;          /* # of days before password expires
                               to warn user to change it */
    long  sp_inact;         /* # of days after password expires
                               until account is disabled */
    long  sp_expire;        /* Date when account expires
                               (measured in days since
                               1970-01-01 00:00:00 +0000 (UTC)) */
    unsigned long sp_flag;  /* Reserved */
};
```

结构体中的内容与`/etc/shadow`文件中的内容对应，就不详细介绍了。
头文件中也定义了一些相关的函数。

``` C
struct spwd *getspnam(const char *name);
struct spwd *getspent(void);
void setspent(void);
void endspent(void);
struct spwd *fgetspent(FILE *stream);
struct spwd *sgetspent(const char *s);
int putspent(const struct spwd *p, FILE *stream);
int lckpwdf(void);
int ulckpwdf(void);
```

`getspnam`函数返回一个指向结构的指针，该结构包含影子密码数据库中与用户名匹配的记录的细分字段。
`getspent`函数返回一个指向影子密码数据库中下一个条目的指针。输入流中的位置由`setspent`初始化。完成阅读后，程序可以调用`endsend`，以便可以释放资源。
`fgetspent`函数类似于`getspent`，但使用提供的流，而不是由`setpent`隐式打开的流。
`sgetspent`函数将提供的字符串`s`解析为结构`spwd`。
`putspent`函数以阴影密码文件格式将提供的`struct spwd * p`的内容作为文本行写入流中。值为NULL的字符串条目和值为-1的数字条目被写为空字符串。
`lckpwdf`函数旨在防止影子密码数据库同时访问。它尝试获取锁，并在成功时返回0，或在失败时返回-1（在15秒内未获得锁）。`ulckpwdf`函数再次释放锁定。请注意，没有针对直接访问影子密码文件的保护措施。只有使用`lckpwdf`的程序才会注意到该锁定。

## 账户登录信息
账户登录信息保存在`/var/log/wtmp`和`/var/log/btmp`文件中，这个文件是一二进制文件，直接通过`cat`打印，无过的有用信息，通过命令`last`可以获取登录，通过`lastb`可以获取未成功登录的信息。

### C 语言通过函数读取
同样C语言也可以获取此信息。在`utmp.h`文件中，定义了以下结构体吗，和一些常量：
``` C
struct utmp {
     char ut_user[256];             /* User login name */
     char ut_id[14];                /* /etc/inittab id */
     char ut_line[64];              /* device name (console, lnxx) */
     pid_t ut_pid;                  /* process id */
     short ut_type;                 /* type of entry */
#if !defined(__64BIT__)
     int __time_t_space;            /* for 32vs64-bit time_t PPC */
#endif
     time_t ut_time;                /* time entry was made */
     struct exit_status
     {
          short e_termination;      /* Process termination status */
          short e_exit;             /* Process exit status */
     }
     ut_exit;                       /* The exit status of a process
                                     * marked as DEAD_PROCESS.
                                     */
     char ut_host[256];             /* host name */
     int __dbl_word_pad;            /* for double word alignment */
     int __reservedA[2];
     int __reservedV[6];
};

#define EMPTY              0
#define RUN_LVL            1
#define BOOT_TIME          2
#define OLD_TIME           3
#define NEW_TIME           4
#define INIT_PROCESS       5        /* Process spawned by "init"                 */
#define LOGIN_PROCESS      6        /* A "getty" process                         */

                                    /* waitingforlogin                           */
#define USER_PROCESS       7        /* A user process                            */
#define DEAD_PROCESS       8
#define ACCOUNTING         9
#define UTMAXTYPE ACCOUNTING        /* Largest legal value                        */
                                    /* of ut_type                                 */
```
这个函数我也还有很多参数不了解，后面有机会再补上。
