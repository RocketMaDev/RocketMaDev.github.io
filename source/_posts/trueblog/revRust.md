---
title: 逆向操作系统实验课的rust评测程序
date: 2025/04/14 14:35:00
updated: 2025/04/22 14:21:00
excerpt: |-
    我们操作系统实验课的配套任务有一个专门的评测程序来判断是否通过，
    就像oj一样。然而，他是在本地评测的，测完需要我们复制结果到平台上，
    整个程序是用rust写的，不仅需要权限，而且报错也很抽象，让我用得很恼火。
    `file`一看，竟然没剥符号！直接逆向启动，还原处理逻辑，手写注册机实现任意题目
    token输出...
tags:
    - non-ctf
    - rust
thumbnail: /assets/trueblog/ehtoken.png
---

我们操作系统实验课基于[tatakOS](https://github.com/yztz/tatakOS)，
在它的基础上做一些修改。具体的任务基于头歌平台分发，要拉取头歌平台的git仓库，
做完任务后使用老师分发的`eh`程序运行评测，通过就会输出一串Token ~~flag~~，
然后上传到头歌平台上通过评测。

{% note blue fa-square-check %}
原先以为是学院里写的粗劣操作系统，让我们补全代码，没想到这个 *tatakOS*
竟然是操作系统赛获奖的系统。我粗略地看了一眼代码，没有明显瑕疵，
感觉可以做做。
{% endnote %}

## 逆向

`eh`有个“安装程序”，会把评测用镜像文件释放到`/var/local/eh/testcases/`文件夹下，
然而，这个目录只有root能创建，因此还要求root权限。你一个评测程序还需要root？？
这对我这个日用Linux的人来说完全不能理解。然而，只是这样，还不能体现出`eh`的抽象。
它的报错写得根本不可读，只是抛出了一个rust异常，编译tatakOS出错了也不把信息打印全，
异常的名字也奇奇怪怪，找不到git仓库的用户竟然是`UnrecognizedUser`，
不看文档根本不知道怎么解决。

忍不了了，一看这个程序，竟然没剥符号！这下不得不逆向了。

有了符号，就能分清哪些函数来自库，哪些是程序自己的。在成功通过第一个评测后，
下断点在输出token的位置，向上追踪，发现一串json字符串：

```json
{
    "meta": {
        "edu_coder_uid": "phmuan9zc",
        "timestamp": "2025-04-08 16:40:35.689451041 +08:00",
        "lab_id": 1
    },
    "score": "22/22",
    "token": "8yP3tR6wQ9zX4vL2kM",
    "pass": true
}
```

但是输出的并不是json，而是一串base64。继续向下追踪，发现另一个json，
随后被`serde`反序列化后，被 *chacha20* 解密，解密结果是一个rsa公钥pem字符串。
随后使用`RustCrypto/RSA`模块 *PKCS#1 v1.5* 加密后转换成base64
字符串打印出来。知道流程之后，只需要找到所有题目的token，就可以写一个注册机，
直接上交平台，通过评测。

{% note green fa-lightbulb %}
为了验证`eh`的流程和我说的是否一致，一开始我使用CyberChef来验证，
结果它的RSA实现有问题，即使我截获了rng做重放，使用raw加密，
结果也和程序的对不上，换用了python的模块才对上。为了追踪题目
token，还动用了`rr`，碰巧遇到了[pwndbg的bug](https://github.com/pwndbg/pwndbg/pull/2850)，
顺手修了一下，花费了我不少时间。rust的字符串是没有`'\0'`截断的，
和go类似，在函数调用时将长度一起传入，再加上静态编译，
`strings`直接输出一大坨，这也给还原代码带来了一些困扰。
{% endnote %}

通过头歌平台，还可以泄露评测脚本和私钥，具体怎么做就不说了，留给读者自己探索。
最后写出的注册机大概是这样：

```python get_flag.py
#!/usr/bin/python3
import argparse
import sys
import json
from datetime import datetime
from random import randint
from base64 import b64encode

from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5

PUBLIC_PEM = '[REDACTED]'
TOKENS     = '[REDACTED]'

parser = argparse.ArgumentParser(
    usage='python %(prog)s -u <uid> -i <lab_id>',
    description='摆脱 eh ，直接生成Token！'
)

parser.add_argument('-u', '--userid', type=str, required=True, help='用户UID（头歌平台个人主页）')
parser.add_argument('-l', '--lab-id', type=int, required=True, help='实验ID（1-17）')

if len(sys.argv) == 1:
    parser.print_help(sys.stderr)
    sys.exit(1)

args = parser.parse_args()
if args.lab_id < 1 or args.lab_id > 17:
    print('实验ID超出范围', file=sys.stderr)
    sys.exit(1)

timefmt = f'%Y-%m-%d %H:%M:%S.%f{randint(100, 1000)} %:z'
nowtime = datetime.now().astimezone().strftime(timefmt)
payload = {
    "meta": {
        "edu_coder_uid": args.userid,
        "timestamp": nowtime,
        "lab_id": args.userid
    },
    "score": "22/22" if args.lab_id != 4 else "1/1",
    "token": TOKENS[(args.lab_id - 1) * 0x12: args.lab_id * 0x12],
    "pass": True
}

cipher = PKCS1_v1_5.new(RSA.import_key(PUBLIC_PEM))
ciphertext = cipher.encrypt(json.dumps(payload, separators=(',', ':')).encode())
print('[+] 已生成该实验ID的Token：', file=sys.stderr)
print(b64encode(ciphertext).decode())
```

![register](/assets/trueblog/registerFlag.png)

## 后续

在我写出注册机后，潘博帮我联系到了出题人，让我帮忙提点建议改进。他说，
我们用的程序已经是老版了，新版改了不少。我问他要源码，他爽快地给了，
新版在编译时会剥符号，而且还加了反调试(~~虽然patch一下就没了~~)，
除此之外新版会将运行结果打包成“token”，交给云端来判断是否通过，
不再在本地判断是否通过了，这也意味着注册机不再能正常工作了，
相对安全了不少。

## 杂谈

在各种内外部因素影响下，不少人学习rust，并到处宣扬rust，这无疑是令人反感的。
尽管如此，rust出色的包管理以及严格的编译器拉高了编程者的下限，这也使它成为了C++
的直接竞争者，注定会在未来影响世界编程格局。

不少rust软件，例如`uv`、`ruff`、`neovide`等具有卓越的性能，能够完美替代它们的竞争品。
~~点名`pip`，安装个包太折磨了，`uv`一分钟的事`pip`半小时还解决不了依赖问题，~~
使用rust写软件固然舒适，但是在不同的场合要使用合适的语言，有功夫锈化所有工具，
不如先精进自己的代码技术，而不是用rust在用户面前班门弄斧。

## 参考

1. [yztz/tatakOS](https://github.com/yztz/tatakOS)
2. [fix: adjust to `rr`'s vFile reply](https://github.com/pwndbg/pwndbg/pull/2850)
