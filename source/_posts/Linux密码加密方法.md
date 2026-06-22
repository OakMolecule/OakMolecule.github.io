---
title: Linux密码加密方法
date: 2020-06-15 10:00:00
tags:
  - Linux
  - 密码加密
url: 128.html
id: 128
comments: false
categories:
  - Linux
---

Linux 在 `/etc/shadow` 第二列保存散列后的密码。同一个明文密码，每次生成的散列值往往不同——这不是 Python 或命令用错了，而是因为引入了 **salt（盐）**。

用 Python 的 `crypt` 模块可以快速生成 SHA-512 散列：

```bash
python3 -c 'import crypt; print(crypt.crypt("test", crypt.mksalt(crypt.METHOD_SHA512)))'
```

每次执行结果都不同，但都可以直接写入 `/etc/shadow` 作为该用户的密码。用 `passwd` 修改同一账户密码时，生成的散列值同样每次都不一样。

<!-- more -->

## 密码串的格式

在 {% post_link linux-user-info 'Linux用户信息详解' %} 一文中介绍了 `/etc/shadow` 里密码字段的含义。更深入一点：散列后的密码并不是「算法名 + 摘要」那么简单，而是按固定格式拼成的一串字符：

```
$<id>[$<param>=<value>(,<param>=<value>)*][$<salt>[$<hash>]]
```

各段含义如下：

- `id`：加密方案标识，例如 `1` 表示 MD5，`5` 表示 SHA-256，`6` 表示 SHA-512，详见下表；
- `param` / `value`：部分算法支持的复杂度参数（如 bcrypt 的 cost、PBKDF 的 rounds）；
- `salt`：随机盐，通常以类 Base64 字符集编码；
- `hash`：明文密码与 salt 经指定算法多次迭代后的结果。

## 常见加密方案

| Scheme id | 算法 | 示例 |
|-----------|------|------|
| （无） | DES | `Kyq4bCxAXJkbg` |
| `_` | BSDi | `_EQ0.jzhSVeUyoSqLupI` |
| `1` | MD5 | `$1$etNnh7FA$OlM7eljE/B7F1J4XYNnk81` |
| `2` / `2a` / `2x` / `2y` | bcrypt | `$2a$10$VIhIOofSMqgdGlL4wzE//e.77dAQGntF/1dT7bqCrVtquInWy2qi` |
| `3` | NTHASH | `$3$$8846f7eaee8fb117ad06bdd830b7586c` |
| `5` | SHA-256 | `$5$9ks3nNEqv31FX.F$gdEoLFsCRsn/WRN3wxUnzfeZLoooVlzeF4WjLomTRFD` |
| `6` | SHA-512 | `$6$qoE2letU$wWPRl.PVczjzeMVgjiA8LLy2nOyZbf7Amj3qLIL978o18gbMySdKZ7uepq9tmMQXxyTIrS12Pln.2Q/6Xscao0` |
| `md5` | Solaris MD5 | `$md5,rounds=5000$GUBv0xjJ$$mSwgIswdjlTY0YxV7HBVm0` |
| `sha1` | PBKDF1（SHA-1） | `$sha1$40000$jtNX3nZ2$hBNaIXkt4wBIo5rsi8KejSjNqIq` |

现代 Linux 发行版默认多用 `$6$`（SHA-512 crypt）或 `$5$`（SHA-256 crypt）。`$1$`（MD5）已不安全，不应在新系统上使用。

## 用 Bash 理解 SHA-512 crypt

`$6$` 格式的计算过程比「单次哈希」复杂得多：先混合密码与 salt，再迭代数千轮 SHA-512，最后按固定字节序重排并做 crypt 专用的 Base64 编码。光看格式字符串不容易建立直觉，下面是一段纯 Bash 实现的 sha512-crypt（作者 Vidar 'koala_man' Holen），保存为脚本后传入密码和可选 salt 即可对照 `crypt(3)` / `mkpasswd` 的输出：

```bash
#!/bin/bash
# sha512-crypt for GNU and Bash
# By Vidar 'koala_man' Holen

b64="./0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

stringToNumber() {
    expression=0
    for((i=0; i<${#1}; i++))
    do
        expression=$(printf '(%s)*256+%d' "$expression" "'${1:$i:1}")
    done
    bc <<< "$expression"
}

# Turn some string into a \xd4\x1d hex string
stringToHex() {
    for((i=0; i<${#1}; i++))
    do
        printf '\\x%x' "'${1:i:1}"
    done
}

# Turn stdin into a \xd4\x1d style sha512 hash
sha512hex() {
    sum=$(sha512sum)
    read sum rest <<< "$sum" # remove trailing dash
    hex=$(sed 's/../\\x&/g' <<< "$sum")
    echo "$hex"
}

# Turn an integer into a crypt base64 string with n characters
intToBase64() {
    number=$1
    n=$2
    for((j=0; j<n; j++))
    do
        digit=$(bc <<< "$number % 64")
        number=$(bc <<< "$number / 64")
        echo -n "${b64:digit:1}"
    done
}

base64Index() {
    for((i=0; i<64; i++))
    do
        if [[ ${b64:i:1} == $1 ]]
        then
            echo $i
            exit 0
        fi
    done
    exit 1
}

# From hex string $1, get the bytes indexed by $2, $3 ..
getBytes() {
    num=$1
    shift
    for i
    do
        echo -n "${num:$((i*4)):4}"
    done
}

hexToInt() {
    {
    echo 'ibase=16;'
    tr a-f A-F <<< "$1" | sed -e 's/\\x//g'
    } | bc
}

base64EncodeBytes() {
    n=$1
    shift
    bytes=$(getBytes "$@")
    int=$(hexToInt "$bytes")
    intToBase64 "$int" "$n"
}

doHash() {
    password="$1"
    passwordLength=$(printf "$password" | wc -c)
    salt="$2"
    saltLength=$(printf "$salt" | wc -c)
    magic="$3"
    [[ -z $magic ]] && magic='$6$'

    salt=${salt#$magic}
    salt=${salt:0:64} # 16 first bytes

    intermediate=$(
    {
        # Start intermediate result
        printf "$password$salt"

        # compute a separate sha512 sum
        alternate=$(printf "$password$salt$password" | sha512hex)

        # Add one byte from alternate for each character in the password. Wtf?
        while true; do printf "$alternate"; done | head -c "$passwordLength"

        # For every 1 bit in the key length, add the alternate sum
        # Otherwise add the entire key (unlike MD5-crypt)
        for ((i=$passwordLength; i != 0; i>>=1))
        do
            if (( i & 1 ))
            then
                printf "$alternate"
            else
                printf "$password"
            fi
        done

    } | sha512hex
    )
    firstByte=$(hexToInt $(getBytes "$intermediate" 0))

    p_bytes=$(for((i=0; i<$passwordLength; i++)); do printf "$password"; done | sha512hex | head -c $((passwordLength*4)) )
    s_bytes=$(for((i=0; i<16+${firstByte}; i++)); do printf "$salt"; done  | sha512hex | head -c $((saltLength*4)) )

    for((i=0; i<5000; i++))
    do
        intermediate=$({
            (( i & 1 )) && printf "$p_bytes" || printf "$intermediate"
            (( i % 3 )) && printf "$s_bytes"
            (( i % 7 )) && printf "$p_bytes"
            (( i & 1 )) && printf "$intermediate" || printf "$p_bytes"
        } | sha512hex)
    done

    # Rearrange the bytes and crypt-base64 encode them
    hex=$(base64EncodeBytes 86 "$intermediate" \
        63  62 20 41  40 61 19  18 39 60  59 17 38  37 58 16  15 36 57  56 14 35 \
            34 55 13  12 33 54  53 11 32  31 52 10   9 30 51  50  8 29  28 49  7 \
             6 27 48  47  5 26  25 46  4   3 24 45  44  2 23  22 43  1   0 21 42)

    printf "%s$salt\$%s\n" "$magic" "$hex"

}

if [[ $# < 1 ]]
then
    echo "Usage: $0 password [salt]" >&2
    exit 1
fi

password=$(stringToHex "$1")
salt=$(stringToHex "$2")
[[ -z $salt ]] && salt=$(tr -cd 'a-zA-Z0-9' < /dev/urandom | head -c 16)

doHash "$password" "$salt" '$6$'
```

用法示例：

```bash
chmod +x sha512-crypt.sh
./sha512-crypt.sh test
```

输出形如 `$6$<salt>$<hash>`，与 `mkpasswd -m sha-512 test` 或 Python `crypt.crypt()` 的结果一致（salt 不同则整串不同，但都能通过同一明文验证）。
