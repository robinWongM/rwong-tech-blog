---
title: "在不使用 VPS 的情况下部署你的 Hugo 博客"
description: "这是不是就叫 Serverless"
date: 2021-10-13T03:00:00+08:00
---

## 选型

之前我们用 Caddy 的 http.git 插件来实现这样的效果：Git 仓库更新 -> 触发 Caddy 在本地拉取 Git 仓库 -> 使用 Hugo 生成网站 -> 用 Caddy 作为静态文件服务器 serve 博客。然后我们就得到了一个以 GitHub 仓库为 source 的静态博客。

缺点在于，**你需要一个服务器**来做这个事情。结果就是，你每个月都要花好几十块大洋去养一个超低负载的云服务器，计算资源被浪费的同时，你可能也得不到一个好的访问速度（比如 1 Mbps 小水管）。

既然 Hugo 博客本质是一堆静态文件，于是我们可以考虑另外一种更快速也更省钱的静态文件托管方式：**对象存储 + CDN**。对象存储的存储成本要比云服务器的硬盘低很多，你也不需要为云服务器的 CPU、内存、带宽等资源付费。如果对象存储的流量费用让你有些犹豫，你还可以搭配 CDN 服务，在提速的同时又能降费（是不是很神奇）。

那没了计算资源，我们怎么跑 Hugo 呢？其实跑 Hugo 就是 CI/CD 的一种形式，于是我们可以用 GitHub Actions 不花一分钱来做这个事情（如果你的博客是 public repository 的话）。

## 准备对象存储与 CDN

这里我用的是腾讯云的对象存储（COS）和 CDN，你也可以随便选一个你喜欢的。具体过程不赘述，可以参考云服务提供商的文档，大概的思路是

1. 建一个存储桶，并且开启存储桶的网站访问能力
2. 建一个 CDN 加速域名，源站为存储桶
3. 建一个子用户（IAM 之类的），授权存储桶的读写权限，拿到 AccessKey 和 SecretKey
4. 配置 GitHub Actions

腾讯云 COS 有个坑点是，如果你想要用 CDN 搭配 COS 静态网站，那么你**不能**打开 COS 静态网站的自动 HTTPS 功能，而应该在 CDN 处配置自动 HTTPS 跳转，否则会得到一个白屏。

## 配置 GitHub Actions

又到了 yaml 工程师最喜欢的环节，配置 GitHub Actions 实际上就是写 workflow 的 yaml 文件，告诉它当 XXX 发生时，需要做 YYY 和 ZZZ。这里需要注意的是，访问存储桶需要用到子用户的 AccessKey 和 SecretKey，这些东西直接写在 yaml 里自然是不行的，所以我们把它们先配置到仓库的 Secrets 里。

这里有一份示例 yaml 文件，直接复制到你的仓库的 `.github/workflows/deploy-to-cos.yaml` 并配置好 Secrets 就可以开启 Hugo 博客部署到腾讯云 COS 的自动化流程了。

Secrets 列表：

- `TENCENT_CLOUD_SECRET_ID`: 子用户的用户名，或者叫 AccessKey，或者叫 SecretID
- `TENCENT_CLOUD_SECRET_KEY`: 子用户的密码，或者叫 SecretKey
- `COS_BUCKET`: 存储桶的 ID，比如 `rwong-tech-blog-xxxxxxxxxx` 之类的
- `COS_REGION`: 存储桶的地域，比如 `ap-guangzhou` 之类的

```yaml
name: Deploy to COS

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build-and-deploy:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2
        with:
          submodules: true

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: TencentCloud/cos-action@v1
        with:
          secret_id: ${{ secrets.TENCENT_CLOUD_SECRET_ID }}
          secret_key: ${{ secrets.TENCENT_CLOUD_SECRET_KEY }}
          cos_bucket: ${{ secrets.COS_BUCKET }}
          cos_region: ${{ secrets.COS_REGION }}
          local_path: public
          remote_path: /
          clean: "true"
```

## 效果

你看到的这个博客目前（2021 年 10 月）就是用这样的流程搭建的。

得益于 GitHub Codespaces，博客文章的撰写也不需要再用到本地能力了，浏览器里直接就有一个现成的 VS Code，还可以跑 `hugo server` 实时在线预览。撰写完成后 commit & push，就能将文章发布到网络上。

## 题外话

这篇文章的 slug 里说的是 `without VPS`，虽说我们已经很少看见 `VPS` 这个词语了，取而代之的是**云服务器**，什么 `EC2`, `ECS`, `CVM` 云云，但我们也都清楚，这些都是高级一点的 `VPS` 罢了。

当然他们的底层实现和以前的 VPS 还是有很大不同的，部署在云环境里的云服务器通常都有更灵活的资源升降配，具备在宿主机间快速转移的能力，多台主机之间也能组成云上的私有网络，规模上来之后成本可以降低不少。