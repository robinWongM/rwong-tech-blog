---
title: "使用 IPv6 + WireGuard 实现三 网 融 合"
description: "如何在家里优雅地连接校园网"
date: 2020-02-19T23:00:00+08:00
---

先声明一下，本文介绍的内容均用于合法合规的用途。可能学校网络中心不太乐意大家自建隧道，因为这实质上允许了未经许可的外网访问，所以如果搭建了的话还请自用，不要分享给校外人员使用权。

## 背景

由于学校校园网 VPN 并发数只有 1，所以如果在家里有多台设备需要同时远程连入校园网的需求时就比较难办。起初是没有这种需求的，因为一般只需要手上这台性能一般的电脑连入校园网，再远程访问到我放在实验室里的另一台性能强劲的电脑，就可以做几乎所有的事情。

但是恰好在开发一个需要校园网的小程序，所以手机也需要连上校园网，而电脑也需要远程到校园网通过另一台电脑里操作微信开发者工具。于是问题就来了。

我曾经试过在一台 Windows 设备上开 AnyConnect 然后再开移动热点共享给其它设备，结果发现它卡在初始化移动热点上，不动了。后来我重启了一下电脑，先开移动热点后连 AnyConnect，这次倒是成功开启了，可是移动设备连上热点后并没有连入校园网。

还有另外一种方案。由于家里有路由器，而且刷了 Padavan，所以其实可以在路由器上安装 OpenConnect，这样路由器后边的设备都可以共享这个连接。但是，由于校园网 VPN 下发的路由里将所有的 IPv6 都导向了 VPN，所以家里的 IPv6 访问速度会从 150Mbps 下降到不到 30Mbps，若每次手动修改路由表又显得十分麻烦。

更重要的是，校园网 VPN 的带宽是有限的。而像我这种远程桌面重度用户，显然会挤占其他优先级更高的学术流量。那么答案很明显了——自建隧道“连入校园网”。

## 连连看

那么怎么自建隧道呢？隧道隧道，总是要有两端的，而且这两端要能够直接建立连接。放在我们的场景里很明显，一端在家里，一端在校园网里。于是我们可以用家里的路由器作为一端，以校园网里我们能够控制的一台设备作为另一端（也就是我放在实验室的电脑了）。

现在人在家里，第一个需要解决的问题就是如何连上校园网里的这台设备。那你就会问了，我要连上校园网里的这台设备，就需要连上校园网；而我要连上校园网，又得连上这台校园网里的设备，经典的 bootstrap 问题怎么解？

其实这个问题的前半部分是不成立的。因为这个“连”，它可以不是主动的。

每台连入校园网的设备都会获得一个 v4 的 IP 地址和 v6 的 IP 地址。以我放在实验室的 Windows 设备为例，连入有线校园网后得到的 IP 为 `172.18.xxx.xxx` 和 `2001:250:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx`。

我们来讨论一下 IPv4 的情况。`172.18.xxx.xxx`是内网地址，我们没法直接连上，需要“连上校园网”后才可以。这个设备经过校园网的对称型 NAT 网关后在公网上表现为 `120.xxx.xxx.xxx`，而显然，我们在家里去连这个 `120.xxx.xxx.xxx` 也是不会有结果的，更何况电信给我们分配的 IP 也是 `100.64.xxx.xxx` 的内网 IP。两个 NAT 后边的设备想要互访，有没有可能呢？有，除非双方都是对称型 NAT。好在电信的 NAT（疑似）是完全锥型 NAT，所以通过 UDP 打洞的方式是可以连上的。

故首先使用了 ZeroTier 的方案，借助官方的 Moon 确实可以实现互访，整个设置方法十分便捷。两边的设备输入一个 ID，然后分配一下 IP 就可以了。但是在实际使用的过程中（例如连接校内设备的远程桌面），大约十分钟就断 线一次，且需要很长一段时间才能恢复连接。我个人猜测是 NAT UDP 打洞不稳定（比如映射关系会一段时间后变化？），所以弃用了该方案。

IPv6 的世界没有 NAT，但是这也不代表我们就能直接连上 `2001:250:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx` 了。校园网边界防火墙的存在，使得默认情况下校园网内的设备是完全无法接受 IPv6 传入连接的。怎么办？禁止了传入连接并不代表我们无法向对方发送数据，只要对方向我们请求连接，那么之后与这条连接有关的数据包（不管是来的还是去的）就可以被防火墙放行。比如 iptables 里很常见的一条：

```
# iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED 
```

那对方能向我们请求连接么？由于电信并不像校园网那样给我们做安 全 防 护，所以我们家里的路由器本身是可以接受传入连接的。

那么这就开搞了！

## 现代的隧道方案

传统的 PPTP、L2TP 就不说了，虽然操作系统有原生支持，但是好像不支持下发路由（？没了解过），而且运营商似乎有限制之类的，连学校都已经弃用了。

前后试用过 SoftEther 与 OpenVPN。其中前者不支持下发路由（需要购买商业版才可以）；后者效果挺可，流畅使用了一下午后，由于 Padavan 给 Windows 设备下发的 IPv6 地址动不动就自己没掉，所以我就把 Padavan 换成 OpenWRT 了。这一换，原本 Padavan 原生提供的很好用的 OpenVPN 管理功能就没了，OpenWRT 提供的 luci-app-openvpn-server 就简陋很多，看起来是为了免流而设计的（自带特殊代码...）。简单试用之后，觉得有点奇怪，好像少了生成证书的功能，后来才发现是自带证书......安全性不保。

后来白白推荐了 WireGuard。官网宣传上说是 Fast, Modern, Secure。emmmmm配起来似乎确实挺简单的（相比于 OpenVPN 来说），但是也没那么傻瓜，Windows 上的图形界面就是给你一个配置文件的编辑器（

WireGuard 与之前我试用过的这些隧道软件不同，它没有服务器和客户端的概念，每个节点都是“平等的” Peer（也就是 C/S 变成了 P2P）。每个 Peer 会有一个 PublicKey 与 PrivateKey，通过 `wg genkey` 可以直接生成，然后填入配置文件即可。

来配置。首先确定子网，通常我们可以随便挑一个不冲突的 /24 段，比如 `192.168.2.0/24` 之类。这里由于某些原因，我挑了一个稍显奇怪的 /29 段。

OpenWRT 这端我用了自带 WireGuard 的固件，具体操作就是 Linux 端的 luci 复刻版：在网络接口里新建一个类型为 WireGuard 的接口（就叫 wg0 好了），然后填入PrivateKey、监听端口，以及我们要给接口分配的 IP（如 192.168.124.1）。注意这里是给接口分配的 IP，要以 CIDR 格式写上，才能让路由器知道你和哪些设备在同一个局域网。这里我填的是 `192.168.208.99/29`，意思是我的 IP 是 `192.168.208.99`，而且是处于一个 /29 的网段内。

之后设置对端 Peer 的 PublicKey、对端 Peer 允许的 IP 列表（也就是，要设置什么路由走向这个 Peer，通常都会包含我们给这个 Peer 分配的 IP，以及我们希望从那个 Peer 那里访问到什么网段）。在初步测试的情况下，我们只填对端 IP：`192.168.208.97/32`。嘿注意，这里我们写的是 `/32`，因为这里我们实际上写的是路由表项。我们只希望发往 `192.168.208.97/32` 的流量流向这个 Peer。

保存应用设置，重启一下这个接口，差不多就搞定了。

![20200219155424.png](https://cdn.rwong.tech/blog/20200219155424.png)

顺带一提，防火墙区域最好选择 wan；Peer 设置里勾选上“路由允许的 IP”，Endpoint 不填。

![20200219164508.png](https://cdn.rwong.tech/blog/20200219164508.png)

在另一端呢，使用 Windows 客户端，手动新建一个 Empty Tunnel，然后键入配置文件。配置文件的内容与刚才在 OpenWRT，同时在添加 Peer 时加上 Endpoint，也就是我们 OpenWRT 这端的地址。

![20200219160121.png](https://cdn.rwong.tech/blog/20200219160121.png)

这就是玄妙之处了：让校园网里的对端主动连上我们，所以我们不在我们这端填对端 Peer 的 Endpoint，而是在对端那边填上我们这端的 Endpoint，这样就能让对端向我们主动建立连接。

之后两端都激活之后，根据我们在 AllowedIPs 里的设置，两端都可以用对端设置的 IP 来 ping 通对端。

![20200219165403.png](https://cdn.rwong.tech/blog/20200219165403.png)

由于路由器里加入了 wan 区域，所以我们不仅可以在路由器上 ping 对端，还可以在路由器后边的设备上 ping 对端。

![20200219165939.png](https://cdn.rwong.tech/blog/20200219165939.png)

## 开始融合

现在我们已经成功建立了连接，两台机器看起来就像是在同一个局域网里一样（尽管实际上中间隔了不知道多少台路由器）。接下来的任务，就是要能够通过校园网里的这台设备，去访问校园网里的其它设备。

How it works? 我们只需要让发往校园网里其他设备的网络包，都发送到那台设备去，然后让那台设备代为转发；而为了能够“一来一回”，我们需要那台设备充当一个 NAT 网关的角色（毕竟我们这个自建的隧道并不会给我们家里的设备分配校园网 IP）。说了这么多，其实就是要让校园网里的那台设备，变成一台路由器。

那怎么让它变成一台路由器呢？

这台设备安装的是 Windows 10 Pro 系统。我在网上找了很久，只找到说要启用一个 RRAS 服务（Routing and Remote Access Service）。我就试着启用了一下，嘿，还真能转发包了。

不过后来我没有采用这种方案，原因待会儿再说。好的，那一旦我们这台设备变成路由器以后，我们就可以再配置一下 WireGuard，使得家里这边发往校园网其它设备的包，都先发送到隧道对端去。

配置很简单，只需要往 AllowedIPs 加入我们期望的路由表项即可。例如 `10.0.0.0/8`, `172.16.0.0/12` 之类。由于我们勾选了“路由允许的 IP”，所以从路由器上发往这些地址的包，都会从隧道走。

![20200219204236.png](https://cdn.rwong.tech/blog/20200219204236.png)

然后你就能愉快地从路由器后边的设备畅享校园网了！

![20200219204456.png](https://cdn.rwong.tech/blog/20200219204456.png)

## 三网融合

人们总是喜新厌旧，期望着更好的。

为了更加便于远程办公，我又有一个新需求：远程访问另一个内网。这个内网部署在校园网内，也是前边有个路由器，占用网段 `192.168.1.0/24`。可惜没有 IPv6。

在网上搜索到一篇文章：[通过 Wireguard 构建 NAT to NAT VPN](https://anyisalin.github.io/2018/11/21/fast-flexible-nat-to-nat-vpn-wireguard/)，刚好就可以解决我们的问题。正由于 WireGuard 这种 P2P 的设计，让这种 NAT to NAT VPN 变得十分易于部署。

以我放在校园网里的那台设备为 Gateway（因为它能访问到家里，它也能被那个内网所访问），之后的事情就是在 `192.168.1.0/24` 内网里挑一台设备作为 WireGuard Peer（分配地址 `192.168.208.98`），并加入到 Gateway 的 Peer 列表里。

事情很顺利，直到我从 `192.168.208.98` Ping 回家中的 `192.168.208.99` 时：

![20200219205601.png](https://cdn.rwong.tech/blog/20200219205601.png)

这就很奇怪了。意思好像是说，我应该换个网关？

之前在找 Windows 设备如何配置成为一台路由器时，大家都说要用 Windows Server。虽然误打误撞开启了 Windows 10 Pro 里的 RRAS，但是我没有任何办法去配置它。Windows Server 同样也是开启这个服务，不过它有一个服务器控制台可以控制相关的 NAT 与路由设置。后来好不容易得知 Windows 10 也可以下载相关组件来远程管理 Windows Server 上的这些服务器功能，装完之后发现这个组件确实检测到了本机，但是又写了一句“非服务器计算机”。同样没有给出任何操作面板。

![20200219210216.png](https://cdn.rwong.tech/blog/20200219210216.png)

虽然似乎这个 `Redirect Network` 的提示不影响网络访问，但我还是怕这个无法自行配置的 RRAS 会对网络造成什么影响（还是懂得太少造成的安全感缺失）。于是在本机上开了个 Linux 虚拟机 + 桥接网卡，来作为整个组网方案中的 Gateway。

为了尽量降低内存占用，这边采用 Alpine Linux 作为虚拟机的系统。Alpine Linux 官方有个 Wiki 页面介绍如何配置 WireGuard，但其实用 `wg-quick` 会比 Wiki 介绍的手动配置方法更便捷。我们只需要把之前 Windows 版的配置文件原样复制到 `/etc/wireguard/wg0.conf`，之后 `wg-quick up wg0` 就可以了。

当然还要记得设置 `net.ipv4.ip_forward = 1`，并且 iptables 配置为允许 FORWARD 相关的 packet。

而为了实现三网融合，我们需要在每台 peer（包括 Gateway）设置正确的 AllowedIPs。假设：

- Peer 1 在 `192.168.1.0/24` 中
- Gateway 在 `172.18.187.0/24` 中
- Peer 2 在 `192.168.123.0/24` 中

若是三个网段能够完完全全的互访，那么

- Peer 1 给 Gateway 设置的 AllowedIPs 应该为 `192.168.208.96/29, 192.168.123.0/24`（为啥不加 `172.18.187.0/24` 呢？因为本来就可以访问）
- Gateway 给 Peer 1 设置的 AllowedIPs 应该为 `192.168.208.98/32, 192.168.1.0/24`
- Gateway 给 Peer 2 设置的 AllowedIPs 应该为 `192.168.208.99/32, 192.168.123.0/24`
- Peer 2 给 Gateway 设置的 AllowedIPs 应该为 `192.168.208.96/29, 192.168.1.0/24`

这样就大功告成了。每个 Peer 只需要添加 Gateway 作为他们的唯一 Peer，并且设置其 AllowedIPs 为 WireGuard 局域网网段 + 他们想要访问的网段即可；而 Gateway 需要添加所有的 Peer，并且设置相应的 AllowedIPs 为 Peer 在 WireGuard 局域网网段的 IP + Peer 所在的 NAT 网段。

而由于我不需要完完全全的互访，比如我不需要从 Peer 1 访问 Peer 2 所在的网段，那我就不需要在 Peer 1 和 Gateway 上配置 `192.168.123.0/24`。

## 告一段落？

正当我想要愉快地使用自建隧道跑远程桌面时，我就发现这个隧道，好卡。

打开任务管理器 + 疯狂拖动窗口（这样可以瞬间提高视频码率至最高），看网络吞吐量，最多也就只有 3 Mbps。而平日在学校大内网，这个速度至少也有 30 Mbps。

那我就不用远程桌面好了，VSCode Remote 它不香么？之后我又一通操作，配好 SSH Remote，准备开始调试 Angular 应用。结果一打开，豁，白屏————————好久。

开 F12 看看，噢原来一直在下载 js 脚本。咱的资源大概有 2~30 MB，下载得好一会儿呢。算了，初次加载慢一点就慢一点，不能忍的是，它居然下载到一半就网络错误中断了！回头看 VSCode，远程终端无响应，过了一会儿自动重连了。

怎么会这么菜呢？难道上传带宽就这么点？而且还这么不稳定？我又跑去 [东北大学测速网站](https://speed.neu6.edu.cn/) 跑了一下测速，结果 IPv6 上传明明有 10 Mbps 呀！

然后我又在家里跑了校园网大文件测速，结果发现下载速度还是有 600 KB/s 左右，也没有出现中断的情况。虽然速度抖动的很厉害，但是这就显得远程桌面的 3 Mbps 吞吐量与 SSH 断连更加诡异了。

后来想了想，会不会是 UDP 的锅？或者更准确地说，是运营商 QoS 对 UDP （尤其是跨网的）限速了。不过我还不太能确认，毕竟跑大文件下载的速度和远程桌面的速度存在这么大的差异，还是十分令人匪夷所思。

WireGuard 使用 UDP 协议，且官方没有加入 TCP 支持。有没有什么办法用 TCP 呢？

OpenVPN 是支持 TCP 的，但是配置三网融合很麻烦（虽然如果速度太慢的话，三网融合也没有什么意义）。事实上，我确实又用 OpenVPN 配了个两点间的隧道，远程桌面的吞吐量可喜地上去了，可是 ping 和 jitter 却搞得飞起，原本 40+ ms 变成 80+ ms，不知为何。

突然想起以前在主机圈看过一些人对付 UDP QoS 的骚操作：伪装。

隆重介绍我们的好朋友：[udp2raw-tunnel](https://github.com/wangyu-/udp2raw-tunnel)，一个可以让 UDP 伪装成 TCP 甚至 ICMP 的隧道。

整个部署很简单，OpenWRT 上也有相应的 luci 包可以简化你的配置。在我这里只需要优化 Gateway 到 Peer 2 的链路就可以了，又由于 Gateway 处于主动的位置，所以在 Gateway 部署 udp2raw client，在 OpenWRT 上部署 udp2raw server 即可。然后我们设定一下 udp2raw，local 就是要监听的地址:端口，remote 就是要把包转发出去的地址:端口，这些 README 中都有相应介绍。

不过需要注意的是，由于 OpenWRT 的 luci 包默认将 udp2raw 运行在 client 模式，所以需要手动修改一下 `/etc/init.d/udp2raw`，将其中写入配置文件的地方里的 `-c` 改成 `-s`。此外，还需要更改 WireGuard 配置，将 MTU 调为 1280（最低值），避免 udp2raw 出现 huge packet 警告。

再试试？重新建立连接后，打开远程桌面，豁，丝般流畅，媲美（不卡的时候的）校网 VPN。

这个伪装会不会有点不道德？我觉得 udpspeeder 这种多倍发包的比较不道德，毕竟会更劣化网络质量。我们这种伪装，应该还好，毕竟我们的实际体验也证实我们无法接受那种程度的丢包。

## 还有一件事

我们的家宽每两天自动重拨号一次。

这一拨号不要紧，路由器拿到的 IPv6 地址就变了呀！这也意味着，每两天我就得上一次路由器后台，复制一下地址，连一下校网 VPN，更改 Gateway 上 Peer 的 Endpoint。

这种重复低效的劳动是应该避免的。解决问题的思路还是比较成熟的，那就是 DDNS。不过有两个问题：

- DNS 有 TTL，更新一个记录需要 TTL 之后才能生效，我在阿里云解析上最低能够设置为 10 分钟。不过更糟糕的是，我们的 DNS 服务器未必会遵守 TTL。
- WireGuard 在握手失败时，不会尝试重新解析域名。

在我们使用了 udp2raw 后，第二个问题迎刃而解，因为我们只需要让 udp2raw 更换远端地址就可以了。可惜，udp2raw 不支持 IPv6 域名，只能纯地址。

那我们就不用传统 DDNS 方案了，不如自己实现一个？

服务端部署在阿里云学生机上：[https://github.com/robinWongM/fast-ddns-server](https://github.com/robinWongM/fast-ddns-server)

客户端是一个简易 Python 脚本，负责在地址变动时重启 udp2raw。服务端在写的时候是支持多域名的，但客户端比较简单粗暴，只取拿到的第一条记录；且没有断线重连支持（似乎）。

```python
#!/usr/bin/env python3
import websocket
import json
import subprocess
from signal import SIGINT

peer = "YOUR PEER PUBLIC KEY"
udp2raw = None

def stop_udp2raw():
    if udp2raw != None:
        udp2raw.terminate()
        udp2raw.wait()
        print("Killed udp2raw")

def restart_udp2raw(endpoint):
    global udp2raw
    stop_udp2raw()
    udp2raw = subprocess.Popen(["udp2raw", "-c", "-a", "-l", "127.0.0.1:LISTEN_PORT", "-r", f"[{endpoint}]:ENDPOINT_PORT", "--raw-mode", "faketcp", "--cipher-mode", "xor", "--auth-mode", "md5", "--retry-on-error"])
    print("Started udp2raw")

def on_message(ws, message):
    msg = json.loads(message)
    print(message)
    if msg['event'] == 'list':
        entry = msg['data'][0]
    elif msg['event'] == 'update':
        entry = msg['data']
    else:
        return print(f"Unknown message occurred: {message}")
    print("Updating")
    try:
        restart_udp2raw(entry['value'])
    except OSError as e:
        print("Execution failed:", e, file=sys.stderr)
    print(f"Updated: {entry['value']}")

def on_error(ws, error):
    print(error)

def on_close(ws):
    print("### closed ###")

def on_open(ws):
    ws.send(json.dumps({'event': 'list'}))

if __name__ == "__main__":
    ws = websocket.WebSocketApp("ws://SERVER_ADDRESS/subscribe",
                              on_message = on_message,
                              on_error = on_error,
                              on_close = on_close)
    ws.on_open = on_open
    ws.run_forever()
    stop_udp2raw()
```

配合 `init.d` 脚本食用，works like a charm。

其实一开始写的是个 Windows 客户端，给 Windows 用，原理是动态修改 hosts 文件，支持 WebSocket 断线重连等高级功能。但是后来发现 WireGuard 在断线重连时不会重解析，遂作罢了。