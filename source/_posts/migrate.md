---
title: 迁移通知
date: 2024/12/18 18:35:00
updated: 2024/12/18 18:35:00
tags:
    - not-ctf
thumbnail: /assets/migrate.png
excerpt: 本站已正式迁移至 https://rocketma.dev 。原先网站托管在 GitHub Pages 上，在国内部分地区直接无法访问，每次和别人分享博客地址， 需要别人开梯子又不太好意思，于是我将博客迁移到了 Cloudflare Pages，再怎么说 Cloudflare 在国内至少能打开。
---

**本站已正式迁移至https://rocketma.dev**

原先网站托管在GitHub Pages上，在国内部分地区直接无法访问，每次和别人分享博客地址，
需要别人开梯子又不太好意思，于是我将博客迁移到了Cloudflare Pages，再怎么说Cloudflare
在国内至少能打开。考虑到原来的网址还挺长的，我还在cf买了一个域名，这样一方面和我的GitHub
ID直接关联，另一方面短了也好输。

Cloudflare Pages部署起来还是很方便的，甚至比GitHub还方便。Cloudflare买域名也很方便，
价格也不是很高（虽然比国内高，但至少方便）。

{% note green fa-wheelchair-move %}
`<head>`中插入了js代码，检测到是GitHub的域名就自动重定向到新域名，不用担心原域名失效。
{% endnote %}
