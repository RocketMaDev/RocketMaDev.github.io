---
title: moectf2023 - int overflow
date: 2023/9/24 12:00:00
tags:
    - noob
excerpt: 通过输入特定的long long数值4294852782以溢出int，成功获取flag。
---

## 文件分析

下载`int_overflow`, 保护全开  
~~ghidra分析为??位程序~~

## 解题思路

经过分析，没有漏洞可钻，自然也就没必要知道是几位的程序了

这道题突破口在于输入long long数值，溢出int  
在vuln里指出，要输入-114514(~~好臭的数字~~)，但不能带-号  
ghidra直接大方给出：4294852782

nc连接，输入该数即拿到flag

Done.