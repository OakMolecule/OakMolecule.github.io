---
title: Linux密码加密方法
tags:
  - Linux
  - 密码加密
url: 128.html
id: 128
comments: false
categories:
  - Linux
---

Linux密码加密方法
===========

Linux在`/etc/shadow`中记录了散列后的密码，本以为同意的密码散列后的结果却截然不同。 Python可以直接通过下面命令生成密码为`test`的散列。你可能会注意到你每次执行得到的散列结果不一样，不要认为是你的Python版本有问题或者执行方法有问题（其实是我写错了），最开始我也以为我了，最后发现其实他确实这样的。你可以直接将结果可以直接放到`/etc/shadow`文件中，用来当用户的密码，无论生成的哪个值都可以当用户密码。

    python3 -c 'import crypt; print(crypt.crypt("test", crypt.mksalt(crypt.METHOD_SHA512)))'

你也可以通过`passwd`命令修改相同的账户密码，同样最后生成的散列值都是不一样的。

密码加密
----

在[Linux用户信息详解](https://zhangpu-vp.cn/?p=64 "Linux用户信息详解")文章中，介绍了`/etc/shadow`文件中密码的几种散列方法。但后面更深入了解之后才发现密码加密方法并不是简单的散列之后就可以。其实一个散列后的密码是按照下面格式分布的。 `$<id>[$<param>=<value>(,<param>=<value>)*][$<salt>[$<hash>]]`

*   `id`: 表示了密码加密的方法，例如1是MD5, 5是SHA-256，具体请参考下表；
*   `param`及`value`: 哈希算法的复杂度；
*   `salt`: 可以理解为Base64编码的salt（随机字符串）；
*   `hash`: 密码和salt进行哈希处理后的结果。

Scheme id

Schema

Example

DES

`Kyq4bCxAXJkbg`

_

BSDi

`_EQ0.jzhSVeUyoSqLupI`

1

MD5

`$1$etNnh7FA$OlM7eljE/B7F1J4XYNnk81`

2, 2a, 2x, 2y

bcrypt

`$2a$10$VIhIOofSMqgdGlL4wzE//e.77dAQGqntF/1dT7bqCrVtquInWy2qi`

3

NTHASH

`$3$$8846f7eaee8fb117ad06bdd830b7586c`

5

SHA-256

`$5$9ks3nNEqv31FX.F$gdEoLFsCRsn/WRN3wxUnzfeZLoooVlzeF4WjLomTRFD`

6

SHA-512

`$6$qoE2letU$wWPRl.PVczjzeMVgjiA8LLy2nOyZbf7Amj3qLIL978o18gbMySdKZ7uepq9tmMQXxyTIrS12Pln.2Q/6Xscao0`

md5

Solaris MD5

`$md5,rounds=5000$GUBv0xjJ$$mSwgIswdjlTY0YxV7HBVm0`

sha1

PBKDF1 with SHA-1

`$sha1$40000$jtNX3nZ2$hBNaIXkt4wBI2o5rsi8KejSjNqIq`

Linux密码加密方法就像私钥和公钥一样再次颠覆思想，

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