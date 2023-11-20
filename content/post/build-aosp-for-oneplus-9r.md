---
title: "从零开始为一加 9R 编译 AOSP"
description: ""
date: 2023-11-19T00:00:00+08:00
draft: true
---

1. 从镜像站下载 AOSP 镜像

```
curl 'http://mirrors.bfsu.edu.cn/aosp-monthly/aosp-latest.tar' -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8' -H 'Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2' -H 'Accept-Encoding: gzip, deflate, br' -H 'Connection: keep-alive' -H 'Referer: https://mirrors.bfsu.edu.cn/help/AOSP/' -H 'Upgrade-Insecure-Requests: 1' -H 'Sec-Fetch-Dest: document' -H 'Sec-Fetch
-Mode: navigate' -H 'Sec-Fetch-Site: same-origin' -H 'Sec-Fetch-User: ?1' -H 'Pragma: no-cache' -H 'Cache-Control: no-cache' -H 'TE: trailers' | tar xf -
```

```
cd aosp
repo sync
```

2. Android 13

```
repo init -b android-13.0.0_r78
repo sync
```

3. Kernel

```
git clone https://github.com/OnePlusOSS/android_kernel_oneplus_sm8250 -b oneplus/sm8250_t_13.1_op9r --depth 1
```