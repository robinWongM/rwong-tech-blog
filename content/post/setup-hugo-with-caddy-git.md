---
title: "使用 Caddy http.git 自动部署 Hugo 博客仓库"
description: "Let Git be the single source of truth"
date: 2020-01-31T17:15:00+08:00
---

本来是想用 Ghost 的（不知道是因为喜欢它的 Node 技术栈，还是因为觉得 WordPress 太笨重/古老），但是那个编辑器对 CJK 用户太不友好了...这个问题从 2018 年到现在[都没有解决](https://github.com/TryGhost/Ghost/issues/9801)。虽说就在最近，有人研究了[修复的方法](https://soulteary.com/2020/01/19/bugfix-for-ghost-editor-cjk-input.html)，但是又要用 Docker 的 bind mount，而我懒得改了（之前 Ghost 部署的时候直接用的 Volume），而且又不知道每次 Ghost 更新后对应的资源文件会不会有变动，整个维护成本变得相当大。Say Goodbye to Ghost👋

于是打算在 GitHub 上建一个仓库存放博客内容（`.md` 等），然后国内服务器上运行 Caddy 和 Hugo，当仓库更新时自动 Pull 并生成页面。省去每次手工部署的麻烦。

记录一下部署过程。

## 服务器环境

阿里云**安 全 加 固**版 Ubuntu Server 18.04.3 LTS

### 安装 Go

因为不想自己手动 cp 到 /usr/local/bin，所以用了 PPA。来源：https://github.com/golang/go/wiki/Ubuntu

```sh
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt update # 我觉得是多余的
sudo apt install golang-go
```

因为没有 `add-apt-repository`，所以还得

```sh
sudo apt install software-properties-common
```

配置一下 GOPROXY。（来源：https://segmentfault.com/a/1190000020293616）

```sh
export GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

## 安装 Caddy

由于需要一些官方网站上没有列出来的插件（如`caddy-alidns`），所以需要自己手动 build Caddy。来源：https://github.com/caddyserver/caddy/wiki/Plugging-in-Plugins-Yourself

```sh
mkdir caddy
cd caddy
nano caddy.go # 见下方 caddy.go 内容

go mod init
go build # 之后当前目录下会有一个 caddy 文件
```

`caddy.go` 文件内容：

```go
package main

import (
        "github.com/caddyserver/caddy/caddy/caddymain"

        // plug in plugins here, for example:
        // _ "import/path/here"
        _ "github.com/colachg/caddy-alidns"

        // hook.service 插件不是必要的
        _ "github.com/hacdias/caddy-service"

        // 下面这个插件似乎有点问题，build 不过去，所以取消掉了
        // _ "github.com/pieterlouw/caddy-net/caddynet"

        _ "github.com/captncraig/caddy-realip"
        _ "github.com/abiosoft/caddy-git"
        _ "github.com/epicagency/caddy-expires"
        _ "github.com/caddyserver/dnsproviders/cloudflare"
)

func main() {
        // optional: disable telemetry
        // caddymain.EnableTelemetry = false
        caddymain.Run()
}
```

然后加入 systemd 豪华大套餐。

来源：

- https://github.com/caddyserver/caddy/wiki/Caddy-as-a-service-examples
- https://github.com/caddyserver/caddy/tree/master/dist/init/linux-systemd

```sh
# 复制 caddy 文件到 /usr/local/bin + 处理权限
sudo cp caddy /usr/local/bin
sudo chown root:root /usr/local/bin/caddy
sudo chmod 755 /usr/local/bin/caddy

# 让非 root 权限下的 caddy 也可以监听 80, 443
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy

# www-data 用户
sudo groupadd -g 33 www-data
sudo useradd \
  -g www-data --no-user-group \
  --home-dir /var/www --no-create-home \
  --shell /usr/sbin/nologin \
  --system --uid 33 www-data

# 在 /etc 下存放配置与 SSL 证书
sudo mkdir /etc/caddy
sudo chown -R root:root /etc/caddy
sudo mkdir /etc/ssl/caddy
sudo chown -R root:www-data /etc/ssl/caddy
sudo chmod 0770 /etc/ssl/caddy

# Caddyfile
sudo touch /etc/caddy/Caddyfile
sudo chown root:root /etc/caddy/Caddyfile
sudo chmod 644 /etc/caddy/Caddyfile

# /var/www 存放网页
sudo mkdir /var/www
sudo chown www-data:www-data /var/www
sudo chmod 555 /var/www

# systemd service unit
wget https://raw.githubusercontent.com/caddyserver/caddy/master/dist/init/linux-systemd/caddy.service # 可能下载不下来，这时得手动新建一下，文件内容见后
sudo cp caddy.service /etc/systemd/system/
sudo chown root:root /etc/systemd/system/caddy.service
sudo chmod 644 /etc/systemd/system/caddy.service
sudo systemctl daemon-reload
sudo systemctl start caddy.service

# 开机自启
sudo systemctl enable caddy.service
```

`caddy.service` 文件内容：

```plaintext
[Unit]
Description=Caddy HTTP/2 web server
Documentation=https://caddyserver.com/docs
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

; Do not allow the process to be restarted in a tight loop. If the
; process fails to start, something critical needs to be fixed.
StartLimitIntervalSec=14400
StartLimitBurst=10

[Service]
Restart=on-abnormal

; User and group the process will run as.
User=www-data
Group=www-data

; Letsencrypt-issued certificates will be written to this directory.
Environment=CADDYPATH=/etc/ssl/caddy

; Always set "-root" to something safe in case it gets forgotten in the Caddyfile.
ExecStart=/usr/local/bin/caddy -log stdout -log-timestamps=false -agree=true -conf=/etc/caddy/Caddyfile -root=/var/tmp
ExecReload=/bin/kill -USR1 $MAINPID

; Use graceful shutdown with a reasonable timeout
KillMode=mixed
KillSignal=SIGQUIT
TimeoutStopSec=5s

; Limit the number of file descriptors; see `man systemd.exec` for more limit settings.
LimitNOFILE=1048576
; Unmodified caddy is not expected to use more than that.
LimitNPROC=512

; Use private /tmp and /var/tmp, which are discarded after caddy stops.
PrivateTmp=true
; Use a minimal /dev (May bring additional security if switched to 'true', but it may not work on Raspberry Pi's or other devices, so it has been disabled in this dist.)
PrivateDevices=false
; Hide /home, /root, and /run/user. Nobody will steal your SSH-keys.
ProtectHome=true
; Make /usr, /boot, /etc and possibly some more folders read-only.
ProtectSystem=full
; … except /etc/ssl/caddy, because we want Letsencrypt-certificates there.
;   This merely retains r/w access rights, it does not add any new. Must still be writable on the host!
ReadWritePaths=/etc/ssl/caddy
ReadWriteDirectories=/etc/ssl/caddy

; The following additional security directives only work with systemd v229 or later.
; They further restrict privileges that can be gained by caddy. Uncomment if you like.
; Note that you may have to add capabilities required by any plugins in use.
;CapabilityBoundingSet=CAP_NET_BIND_SERVICE
;AmbientCapabilities=CAP_NET_BIND_SERVICE
;NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

### 安装 Hugo

本来想用 GitHub Actions 的，这样的话可以解决一部分国内的网络问题。但是后来发现步骤有点繁琐，不如简单的 WebHook，所以就只好在 Caddy 所在的服务器上也安装 Hugo。

安装很简单，直接到 https://github.com/gohugoio/hugo/releases 下载一个 deb 包然后 `sudo dpkg -i` 即可。

官方推荐用 HomeBrew，但是 HomeBrew for Linux 的安装又需要用到 raw.githubusercontent.com，服务器直接装不上，于是作罢。

## GitHub 仓库

### 初始化 

建一个仓库，例如 `https://github.com/robinWongM/rwong-tech-blog`，然后 clone 到本地。

根据 Hugo 的 [QuickStart](https://gohugo.io/getting-started/quick-start/)，在仓库根目录建一个 config.toml。目前这边用的是：

```toml
baseURL = "https://rwong.tech/"
languageCode = "zh-CN"
title = "蛙声一片"
theme = "even"

[permalinks]
  posts = "/:year/:month/:title/"
```

使用 Git Submodule 添加主题文件。比如我 fork 了一个叫 `even` 的主题后：

```
git submodule add git@github.com:robinWongM/hugo-theme-even.git themes/even
```

然后 commit 即可。

### 新建 WebHook

在仓库的 Settings 里配置一个 WebHook。URL 与 secret 都是自定义的，比如目前我的 WebHook URL 就是 `https://rwong.tech/github/webhook`。

## 配置 Caddy + Git WebHook

HTTPS 证书由 Caddy 自动到 Let's Encrypt 申请。这边用的是 DNS Challenge，搭配阿里云云解析的 API 食用。

使用 alidns 的时候，需要设置环境变量 `ALICLOUD_ACCESS_KEY` 和 `ALICLOUD_SECRET_KEY`。（来源：https://github.com/colachg/caddy-alidns）

这边采用的方法是直接修改 `caddy.service`，在 `[Service]` 一节下增加：

```
Environment=ALICLOUD_ACCESS_KEY=*accesskey_id*
Environment=ALICLOUD_ALICLOUD_SECRET_KEY=*accesskey_secret*
```

在阿里云 RAM 访问控制下新建用户并赋予“编程访问”权限后，可以获得这两个值。其中 AccessKey Secret 只出现一次，所以需要妥善保存。

对于权限问题，目前给这个子用户赋予的权限是云解析的全部只读权限 + 云解析 `rwong.tech` 域名的写权限。

根据之前的配置，这边使用的 `Caddyfile` 如下：

```Caddyfile
(alitls) {
  tls {
    dns alidns
  }
}

rwong.tech {
  import alitls
  root /var/www/rwong-tech-blog/public
  git {
    repo https://github.com/robinWongM/rwong-tech-blog.git
    path ../
    branch master
    interval -1
    hook /github/webhook **secret**
    then hugo --destination=/var/www/rwong-tech-blog/public
  }
}

ddns.rwong.tech {
  import alitls
}
```

## 后记

之后挑个时间叛逃到 Treafik（