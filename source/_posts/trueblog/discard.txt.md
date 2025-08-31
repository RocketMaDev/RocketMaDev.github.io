---
title: 从前端解决下载无后缀文件出现后缀的问题.txt
date: 2025/08/31 21:14:00
updated: 2025/08/31 23:22:00
tags:
    - non-ctf
    - tricks
thumbnail: /assets/trueblog/chromium/doc.png
excerpt: >-
    最近我们的新生训练赛就要开始了，题目陆陆续续到了测试阶段，对于pwn题，
    一些简单题就只需要提供一个binary文件，没有后缀，下载下来就能直接打。结果上传一看，
    限制了文件类型。好不容易能上传任意文件了，下载下来却发现多了一个`.txt`后缀。
    ELF怎么可能有后缀呢...
---

# 发现bug.txt

最近我们的新生训练赛就要开始了，题目陆陆续续到了测试阶段，对于pwn题，
一些简单题就只需要提供一个binary文件，没有后缀，下载下来就能直接打。结果上传一看，
限制了文件类型。好不容易能上传任意文件了，下载下来却发现多了一个`.txt`后缀。
ELF怎么可能有后缀呢？检查http响应，没有问题：

```http
HTTP/1.1 200 OK
Date: Sun, 31 Aug 2025 13:20:35 GMT
Server: Apache/2.4.52 (Ubuntu)
Accept-Ranges: bytes
Content-Disposition: attachment; filename="shellcode"
Content-Length: 16552
Content-Type: application/zip
Last-Modified: Sun, 31 Aug 2025 13:13:15 GMT
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
```

然而实际下载的文件却变成了`shellcode.txt`。

于是我跑到[我自己的网站](/2025/02/13/RocketCup2025)上，
有一个`putenv`文件，尝试下载，http响应是这样的：

```http
HTTP/1.1 200 OK
Date: Sun, 31 Aug 2025 13:24:26 GMT
Server: openresty
Content-Disposition: inline; filename=putenv
Content-Length: 16552
Content-Type: application/octet-stream
Last-Modified: Sat, 08 Feb 2025 06:30:59 GMT
```

下载下来是没有后缀名的。这不是都正确声明`Content-Disposition`了吗，
为什么还是不能下载无后缀名的文件呢？

# 尝试复现.txt

尝试修改自己的python服务，使响应和平台的一致，并没有任何效果。既不是因为`Content-Type`，
又不是因为`Content-Disposition`，真正的原因是什么呢？

# 硬调chromium.txt

为了深入理解其中发生了什么，只能调试浏览器了，看看下载文件中发生了什么。由于chromium
是开源的，可以就着源码，拿符号对着看。问题是，浏览器这么多进程，怎么调试呢？
阅读[官方的调试指南](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/linux/debugging.md)，
里面提到可以关闭沙箱，并限制renderer进程，然而这仍然不够，还需要搭配上gdb调试的一些设置，
它们是：

```plaintext stub
set breakpoint always-inserted on
set detach-on-fork off
set non-stop on
```

这样就能让所有进程同时运行，同时将断点下在所有进程中，并且监控所有进程。

那么方法有了，调试什么函数呢？直接使用copilot读取仓库并寻找下载时指定文件名的函数，
定位到了`net::GenerateFileNameImpl`函数，然后直接跟到`GetSuggestedFilenameImpl`，
其中的重要内容大致是

```cpp /net/base/filename_util_internal.cc
std::u16string GetSuggestedFilenameImpl(
    const GURL& url,
    const std::string& content_disposition,
    const std::string& referrer_charset,
    const std::string& suggested_name,
    const std::string& mime_type,
    const std::string& default_name,
    bool should_replace_extension,
    ReplaceIllegalCharactersFunction replace_illegal_characters_function) {

  ...
  std::string filename;  // In UTF-8
  bool overwrite_extension = false;
  bool is_name_from_content_disposition = false;
  // Try to extract a filename from content-disposition first.
  if (!content_disposition.empty()) {
    HttpContentDisposition header(content_disposition, referrer_charset);
    filename = header.filename();
    if (!filename.empty())
      is_name_from_content_disposition = true;
  }
  // Then try to use the suggested name.
  if (filename.empty() && !suggested_name.empty())
    filename = suggested_name;

  ...

  // extension should not appended to filename derived from
  // content-disposition, if it does not have one.
  // Hence mimetype and overwrite_extension values are not used.
  if (is_name_from_content_disposition)
    GenerateSafeFileName("", false, &result);
  else
    GenerateSafeFileName(mime_type, overwrite_extension, &result);

  std::u16string result16;
  if (!FilePathToString16(result, &result16)) {
    result = base::FilePath(default_name_str);
    if (!FilePathToString16(result, &result16)) {
      result = base::FilePath(kFinalFallbackName);
      FilePathToString16(result, &result16);
    }
  }
  return result16;
}
```

接着我们使用`gdb -x stub -args /usr/lib/chromium/chromium --no-sandbox --single-process`
启动浏览器调试，把断点下在`net::GenerateFileNameImpl`，下载调试符号，让浏览器跑起来。
启动完成以后尝试下载一下文件，观察到此时进入调试态，注意寄存器的状态是什么样的。

{% note blue fa-box %}
虽然Arch提供了chromium的调试符号包，但是调试符号中只有函数名，这意味着我们没有源码可以看，只能盲调。
通过pwndbg的`nextcall`命令，可以很方便地观察调用了哪些函数，就着源码看也还行。
{% endnote %}

例如以下是下载我网站上文件时寄存器的状态，根据源码，rdx是指向`Content-Disposition`的字符串，
rsi是指向url的字符串，r9是指向mime_type的字符串，即`application/octet-stream`。

![regs](/assets/trueblog/chromium/putenv_regs.png)

接下来不断使用`nextcall`去找我们感兴趣的函数，以此侧面观察控制流（记得在
`GetSuggestedFilenameImpl`步入）。在这里我们就能观察到解析了`Content-Disposition`。

![header](/assets/trueblog/chromium/parse_header.png)

之后调用了`net::EnsureSafeExtension`函数，把参数对应过去，mime_type是空字符串，
后缀名被`Content-Disposition`强制指定了，因此没有添加后缀名。(#L32)

![extension](/assets/trueblog/chromium/putenv_ext.png)

接着看看平台上下载文件的例子，首先观察mime_type是`text/plain`，而且链接前面是`blob`，
并且没有`Content-Disposition`头的信息。

![regs](/assets/trueblog/chromium/blob_regs.png)

继续执行，直接到了`EnsureSafeExtension`的地方，此时我们发现走了另一个分支（
`!is_name_from_content_disposition`），阅读源码可知在这里由于传入的mime_type是`text/plain`，
因此后续发现prefered extension是`.txt`，导致被强制加上了.txt后缀。(#L34)

![extension](/assets/trueblog/chromium/blob_ext.png)

{% note green fa-lightbulb %}
在chromium查找扩展名的过程中，由于`application/octet-stream`是“通用二进制数据”，
因此检查直接返回了，不要求后缀名。
{% endnote %}

# 错误设置的下载方式

原先平台上下载方式是把请求响应包在blob里，然后模拟链接点击下载，
mime_type是在链接中设置的，查阅
[Mozilla HTTP 文档](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/a#download)
发现需要在blob中设置才行。

<img src="/assets/trueblog/chromium/doc.png" width="60%">

# 修复这个小bug

给blob加上mime_type就行。

```diff
@@ -418,14 +418,13 @@
       }
 
       // 创建 blob URL
-      const blob = new Blob([response.data]);
+      const blob = new Blob([response.data], { type: 'application/octet-stream' });
       const url = window.URL.createObjectURL(blob);
 
       // 创建临时下载链接
       const link = document.createElement('a');
       link.href = url;
       link.download = filename;
-      link.type = 'application/octet-stream';
       document.body.appendChild(link);
       link.click();
```

# 参考

1. [2025 新年火箭杯 | RocketDevlog](/2025/02/13/RocketCup2025)
2. [Chromium Docs - Tips for debugging on Linux](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/linux/debugging.md)
3. [net::GetSuggestedFilenameImpl | GitHub](https://github.com/chromium/chromium/blob/main/net/base/filename_util_internal.cc#L219)
4. [&lt;a&gt;：锚元素](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/a#download)
