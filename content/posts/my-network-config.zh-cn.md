---
title: "我是怎么上网的"
date: 2024-04-19
draft: false
categories:
    - 网络
---

如果你在南大或者中大的校园里见过我，也许你会问一个问题：

> 为什么你只需要 iPad 就能工作（包括写代码）?

这个问题的答案涉及到几个方面，最重要的当然还是 iPad 上面的软件生态。但是，这里我想讲的是另一个方面：我的网络配置。

## 校园网与 VPN

我首先推荐的解决方案是，**在宿舍或实验室里架设一台属于自己的服务器**。我们所有的仓库、开发环境、影音娱乐软件等等都部署在这台服务器上。那么下一个需要解决的问题是，在学校里，我们有几种需要 “移动办公” 的场景：

- 在图书馆里自习
- 在教室里上课
- 在宿舍里休息

可以看到，我们需要一种远程访问的手段来在学校内访问我们的服务器。理论上来说，我们可以直接暴露服务器的端口，然后在校园网里里直接访问。但是，这样做的安全问题非常严重。校内确实一般没有人会攻击我们的服务器，但是校方理论上监听了所有的流量。更重要的是，例如 Jellyfin 这样的服务，我们一般只会使用 http 来访问它，这样的话，密码以及任何其他信息都是明文传输的。 

所以，我推荐使用 VPN 进行访问。在需要移动办公时，首先让移动设备通过 VPN 连接到服务器，然后再直接访问服务器上的服务。这样的话，我们的流量就会被加密，校方也无法监听我们的流量。我所使用的 VPN 是基于 [ipsec-vpn-server](https://github.com/hwdsl2/docker-ipsec-vpn-server) 的，它搭建的 ikev2 VPN 必须通过私钥证书进行连接，非常安全。

我把这个服务跑在了 Docker 里面：

```bash
❯ docker container ls | rg ipsec
b13ca7eb8233   hwdsl2/ipsec-vpn-server   "/opt/src/run.sh"   2 months ago   Up 6 days   0.0.0.0:500->500/udp, :::500->500/udp, 0.0.0.0:4500->4500/udp, :::4500->4500/udp   
```

可以看到，它只占用了 500 和 4500 两个端口。这样一来，你只需要在 iptables 开放这两个端口即可，其他的都可以关闭。事实上，我没有直接把这台服务器连接到校园网，而是通过一个路由器连接到校园网，服务器连接在路由器创建的内网里。我只需要在路由器的端口转发里面把这两个端口转发到服务器上，就可以在校园网里访问 VPN 了。

## Cloudflare Warp

中国的网络环境，有一些比较复杂的问题。我们有些时候必须使用一些特殊的服务来加速国外网络的连接。我推荐的服务是 Cloudflare Warp。它是一个基于 WireGuard 的 VPN 服务。然而这是比较 “非主流” 的解决方案，它具有一个比较大的问题，那就是无法 “分流”。

所谓的 “分流”，简单来说就是国内的流量不走 Warp，国外的流量走 Warp. 这样的话，我们可以避免一些国内的服务因为走 Warp 而变得不可用。这个特性本质上是一种用户态的路由，一些 “主流” 的解决方案都支持这个特性。

有三种简单的解决方案来解决这个问题：

- 手动切换。这个方法最简单，但是也最不方便。每次需要访问国外的服务时，我们需要手动切换到 Warp. 显然，这是不太合适的。
- 使用 Warp 提供的 [Split Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-devices/warp/configure-warp/route-traffic/split-tunnels/) 功能。这个功能支持对于特定的 IP 地址或域名进行分流。然而，Cloudflare 没有提供任何 API 来自动化地配置这个功能。也就是说，如果你想让 bilibili 服务不走 Warp，你需要手工把 bilibili 的全部域名（大概可能有 30 个以上）一个个地用 Web 界面添加到 Split Tunnel 里面。
- 使用混合方案，开一台虚拟机，让虚拟机走 Warp，然后在主机里使用 “主流” 方案进行分流。这是比较可行的方案，然而，“主流” 的透明代理方案经常和 Docker 有冲突，我也不喜欢这种方案。

这三种方案似乎都不太合理。这里看起来必须要仔细研究一下 Warp 到底是如何工作的，才能找到一个更好的解决方案。

## Warp 的工作原理

Warp 作为一个 VPN，首先会创建一个虚拟网卡。然后，所有的流量都会被发送到这个虚拟网卡上。观察到它是很简单的：

```bash
❯ ifconfig | rg Cloudflare -A 10
CloudflareWARP: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1280
        inet 172.16.0.2  netmask 255.255.255.255  destination 172.16.0.2
        inet6 fe80::bcc8:7a58:8e3a:d8d8  prefixlen 64  scopeid 0x20<link>
        inet6 2606:4700:110:8dcf:a1af:2f13:73f3:426d  prefixlen 128  scopeid 0x0<global>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 2607433  bytes 790963841 (790.9 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6150386  bytes 4399939541 (4.3 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

一个直觉的问题是，数据包是怎么被 “劫持” 到这个虚拟网卡上的呢？

理论上来说，这种工作是由路由表完成的。我们可以通过 `ip route` 命令来查看路由表：

```bash
❯ ip route
default via 192.168.1.1 dev enp34s0 proto dhcp metric 20100 
10.10.0.0/24 dev docker0 proto kernel scope link src 10.10.0.1 
10.10.1.0/24 dev br-bb1b34f83079 proto kernel scope link src 10.10.1.1 
10.10.2.0/24 dev br-cbd63cef39db proto kernel scope link src 10.10.2.1 
10.10.3.0/24 dev br-fbf2794de605 proto kernel scope link src 10.10.3.1 
10.10.4.0/24 dev br-81377bc74af2 proto kernel scope link src 10.10.4.1 
10.10.5.0/24 dev br-6dba9d07a5e4 proto kernel scope link src 10.10.5.1 
10.10.12.0/24 dev br-1e11e9d90ce2 proto kernel scope link src 10.10.12.1 
10.10.19.0/24 dev br-d4a69960927a proto kernel scope link src 10.10.19.1 
10.10.21.0/24 dev br-e5a33351bb28 proto kernel scope link src 10.10.21.1 
10.10.100.0/24 dev br-9c1f570184a3 proto kernel scope link src 10.10.100.1 
169.254.0.0/16 dev br-fbf2794de605 scope link metric 1000 
192.168.1.0/24 dev enp34s0 proto kernel scope link src 192.168.1.185 metric 100
```

然而，这里似乎没有任何与 Warp 相关的路由。事实上，和大部分人想象的不同，Linux 的路由结构还是比较复杂的。

现代的 Linux 支持一种叫做 "policy routing" 的机制。传统的路由算法仅仅依照目的地址进行路由判定，但事实上，很多时候需要通过一些其他信息（源地址、端口、协议……）进行路由选择，所以 Linux 使用了一种 “两级” 的结构，

- 由 `ip route` 管理的路由表进行具体的路由选择
- 路由表可能有很多张，由 `ip rule` 根据策略规则选择使用什么路由表。路由表匹配可能失败，如果失败就几乎回到 `ip rule` 匹配下一个可能的规则。

```bash
❯ ip rule
0:      from all lookup local
32765:  not from all fwmark 0x100cf lookup 65743
32766:  from all lookup main
32767:  from all lookup default
```

可以看到，这里有四张路由表，分别是 `local`, `main`, `default` 和 `65743`，前面三张表是 Linux 在启动时自动创建的。而 `65743` 是 Warp 创建的。这条规则的意思很明确：如果这个数据包没有被打上 `0x100cf` 标记的话，那么使用 `65743` 路由表来处理它。自然地，我们会想到，Warp 的出口流量会打上 `0x100cf` 标记，以避免回环问题（Warp 的出口流量又被发送到 Warp，造成无限循环）。

而路由表 `65743` 可能会非常复杂，比如在我的服务器上：

```bash
❯ ip route show table 65743
0.0.0.0/5 dev CloudflareWARP proto static scope link 
8.0.0.0/7 dev CloudflareWARP proto static scope link 
11.0.0.0/8 dev CloudflareWARP proto static scope link 
12.0.0.0/6 dev CloudflareWARP proto static scope link 
16.0.0.0/4 dev CloudflareWARP proto static scope link 
32.0.0.0/3 dev CloudflareWARP proto static scope link 
64.0.0.0/3 dev CloudflareWARP proto static scope link 
96.0.0.0/4 dev CloudflareWARP proto static scope link 
112.0.0.0/7 dev CloudflareWARP proto static scope link 
114.0.0.0/9 dev CloudflareWARP proto static scope link 
114.128.0.0/10 dev CloudflareWARP proto static scope link 
114.192.0.0/12 dev CloudflareWARP proto static scope link 
114.208.0.0/14 dev CloudflareWARP proto static scope link 
114.213.0.0/16 dev CloudflareWARP proto static scope link 
114.214.0.0/15 dev CloudflareWARP proto static scope link 
114.216.0.0/13 dev CloudflareWARP proto static scope link 
114.224.0.0/11 dev CloudflareWARP proto static scope link 
... (about 200 lines)
```

为什么会这么多规则呢？

这就要说到 Split Tunnel 了。Split Tunnel 的功能是通过这些路由规则实现的。

具体来说，如果默认的只有这一条（所有的流量都发送到 Warp 的虚拟网卡）的话：

```bash
0.0.0.0/0 dev CloudflareWARP proto static scope link
```

现在我们添加了一条规则说，`1.0.0.0/8` 的流量不走 Warp，那么 Warp 就必须让这个路由表 “分裂”：

```bash
0.0.0.0/8 dev CloudflareWARP proto static scope link
2.0.0.0/7 dev CloudflareWARP proto static scope link
4.0.0.0/6 dev CloudflareWARP proto static scope link
8.0.0.0/5 dev CloudflareWARP proto static scope link
16.0.0.0/4 dev CloudflareWARP proto static scope link
32.0.0.0/3 dev CloudflareWARP proto static scope link
64.0.0.0/2 dev CloudflareWARP proto static scope link
128.0.0.0/1 dev CloudflareWARP proto static scope link
```

也就是说，`1.0.0.0/8` 被排除在了这张表里，所以它就不会被 Warp 所路由，会被重新分配给 `ip rule` 的下一条规则，也就是 `main` 表进行路由。可以看到，这样的减法每次都可能产生一堆 CIDR 块。

## 不可行的方案

根据以上的知识，一个直觉的解决方案是，添加一个优先级更高的路由规则，然后把大陆 IP 全部添加到这个规则指向的路由表里：

```bash
❯ ip rule
0:      from all lookup local
32764:  from all lookup 10000
32765:  not from all fwmark 0x100cf lookup 65743
32766:  from all lookup main
32767:  from all lookup default
```

然而这是不可行的。第一个原因是，Warp 在每次重连时会非常傲娇地覆盖我们设置的规则，重连以后的情况可能是：

```bash
❯ ip rule
0:      from all lookup local
32763:  not from all fwmark 0x100cf lookup 65743
32764:  from all lookup 10000
32765:  not from all fwmark 0x100cf lookup 65743
32766:  from all lookup main
32767:  from all lookup default
```

第二个原因是，在不知不觉之间，本机防火墙的 OUTPUT 链已经被 Warp 设下了非常强的规则。

```bash
❯ sudo nft list table inet cloudflare-warp
table inet cloudflare-warp {
        chain input {
        ...
        }

        chain output {
                type filter hook output priority filter; policy drop;
                oif "lo" accept
                oif "CloudflareWARP" goto tun
                ip saddr 0.0.0.0 ip daddr 255.255.255.255 udp sport 68 udp dport 67 accept
                meta nfproto ipv4 udp sport 67 udp dport 68 accept
                ip6 saddr fe80::/10 ip6 daddr ff02::1:2 udp sport 546 udp dport 547 accept
                ip6 saddr fe80::/10 ip6 daddr ff05::1:3 udp sport 546 udp dport 547 accept
                ip6 saddr fe80::/10 ip6 daddr fe80::/10 udp sport 547 udp dport 546 accept
                meta l4proto ipv6-icmp accept
                ip daddr 162.159.137.105 tcp dport 443 accept
                ...
                reject
        }

        chain tun {
                ip saddr 172.16.0.2 accept
                ip6 saddr 2606:4700:110:8dcf:a1af:2f13:73f3:426d accept
                ip6 saddr fe80::/10 accept
                ip protocol tcp reject with tcp reset
                reject
        }
}
```

这些规则的意思是，只有下列三种报文可以被发出:

- 目的地址属于 Split Tunnel 里定义的不走 Warp 的范围内
- 目标接口是 CloudflareWARP 接口
- 其他特殊情况，比如本机的流量、dhcp 等等

结果是，我们这样 “绕过” 的流量自然会被 reject 掉。我觉得最好还是不要和 Warp 对着干，所以也不倾向于修改这些规则。值得一提的是，这些规则通过通常的 `iptables` 命令是看不到的（iptables 不支持自定义表，而 nftables 支持），必须通过 `nft` 命令来查看。

## 最终方案

当我们的服务器作为其他设备的 VPN 服务器的时候，它本身承担了类似于路由器的功能。也就是说，VPN 容器的流量其实是被 “转发” 出去了。

换句话说，这些流量不会走 OUTPUT 链，因为 OUTPUT 链过滤的是本机运行的程序所发送的流量。这无疑给了我们一些暗示：至少可以在 iptables 内部对 VPN 容器发出的流量做分流。

还记得之前的 `0x100cf` 标记吗？只要这个标记被打上了，流量就不会走 Warp. 那么，只要在 PREROUTING 链里面给源地址在国内的流量打上标记，似乎就能完成任务了，例如：

```bash
iptables -D PREROUTING -t mangle -d 114.212.0.0/16 -j MARK --set-mark 0x100cf
```

然而国内的 IP 段实在是太多了，必须写一个程序来完成这些操作。另一个问题是，当你把这些 CIDR 全都加进去（大概有 10000 个左右），你会发现系统的网络性能劣化到了惊人的程度。

这是因为 iptables 的匹配是纯粹的线性匹配，它会一条条检查规则是否被满足，这样任何的数据包都会需要 10000 次匹配，这会严重影响吞吐量。

所以，应该使用 `ipset` 创建一个 CIDR 的集合，（基于哈希表的）集合匹配就相对快速了：

```bash
ipset create cn hash:net family inet
ipset add cn 114.212.0.0/16
...

iptables -D PREROUTING -t mangle -m --match-set cn dst -j MARK --set-mark 0x100cf
```

我写了一个 [Python 程序](https://gist.github.com/ayanamists/bb194f497b8f13fcced5d5c3550c2aa8)来执行创建和添加任务，以及一个 systemd 任务，使得每次开机自动执行该任务：

```service
[Unit]
Description=Init the network settings, will take quite long time
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/python3 /home/ayanamists/service/route/main.py

[Install]
WantedBy=multi-user.target
```

虽然这里有一些 “医者不能自医” 的尴尬，服务器本身还是没有实现分流，但是这也是一个不错的解决方案了。

## 总结

经过以上配置，我能够在校园的任何一个地方安全地访问到我的服务器，并且实现了比较流畅的网络体验。我一直有个想法，一个程序员单枪匹马改变世界的时代已经过去了，但至少，我们应该改变我们的生活，用技术让生活更加美好。

## 参考资料

- [ip rule 的文档](https://man7.org/linux/man-pages/man8/ip-rule.8.html)
- [iptables 的图解](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)

