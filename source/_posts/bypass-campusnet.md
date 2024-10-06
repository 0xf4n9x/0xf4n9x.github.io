---
created: 2024-09-28T03:17:00+00:00
categories:
  - 技术研究
tags:
  - Bypass
updated: 2019-05-04T00:00:00+00:00
date: 2019-03-20T00:00:00+00:00
slug: bypass-campusnet
title: DNS隧道绕过校园网认证
cover: /img/post/Bypass-campusNet/campus.png
id: 10f906e1-7468-80da-91a5-cf400874eee0
---

> 本文在绝大数人眼里或许是篇福利文；在此文中介绍如何通过 DNS TUNNEL 的方式来绕过校园网认证，实现免认证~~免费~~上网；或许此招式并不是最优解，但对于绝大多数校园认证网确实能够成功实现。
>
> 怎么说呢！其实我早盯上了校园网了。

## **场景分析**

### **吐槽**

在某所高校中，存在一家网络运营商，主要面向毫无收入的学生们，为我们提供日常上网冲浪。

其特点就是三字：**贵**、**差**、**抠**。每月 79 的高昂费用；网络质量差，打游戏经常 460，还只让三个设备使用。

虽然我打比赛有机房的免费 Wi-Fi 用，用不着此校园网，但我还是看不下去如此这般从学生身上赚钱的行为，不服这口气，遂有了本文。

### **信息收集**

在这所高校的网络中，统一使用的是 WiFi 热点客户端认证方式；当连上 WiFi 后，本机会向 DHCP 服务器获取一个内网 IP；关于这个 IP 地址，起初还让我很是疑惑，没想到在资源如此匮乏的大天朝，此运营商还会分一个公网 IP 给俺。

![ip-a](/img/post/Bypass-campusNet/ipa.jpg)

后来才知道这是个保留地址，详见其  [维基百科](https://en.wikipedia.org/wiki/Reserved_IP_addresses) 。

| Address block | Scope    | Description                                                  |
| ------------- | -------- | ------------------------------------------------------------ |
| 100.64.0.0/10 | 私有网络 | [共享地址空间](https://en.wikipedia.org/wiki/IPv4_shared_address_space) |

在未认证前还会弹出一个下载认证客户端软件的页面，这里所用到的恶心技术就是利用 HTTP 协议的缺陷，当我们访问一个 HTTP 的网站时，网关会对这个响应报文劫持篡改，给我们 302 重定向到一个指定的下载认证客户端页面。而当我们打开一个 HTTPS 类型的网站是不可能被劫持的。

![](/img/post/Bypass-campusNet/campus.png)

上图就是重定向后的客户端下载页面，让我匪夷所思的是最上面的那个位置本该是一个域名，为何是个公网 IP。既然没有使用域名，那何必需要 DNS，何不直接关闭 53 端口，为何让我如此这般有机可乘，实在让我百思不得其解 🤔。

由下图可得知，DNS 53 端口是开启的。

![nslookup](/img/post/Bypass-campusNet/nslookup.jpg)

### **原理简述**

原理其实很简单，由上述信息得知，校园网认证过程一般需要放行 DNS 和 DHCP 报文，也就是 53 和 67/68 端口。53 端口既可以是 UDP，也可是 TCP；67/68 端口走的是 UDP 传输协议。

本文着重点是 DNS 53 端口，其实 UDP 67 也可以绕过认证；但本文将围绕 DNS 53 来实现绕过认证，不讨论后者。

而在这个 53 端口中，网关/防火墙如果不进行报文检查，那么就也将意味着，任何数据包都可以通过此端口传输；如果真的是这样的话，那就很简单了，直接 openVPN 架起，详见此文  [利用 openVPN 实现 udp53,67,68 端口绕过校园网认证上网](http://zgao.top/2019/03/03/%E5%88%A9%E7%94%A8openvpn%E5%AE%9E%E7%8E%B0udp536768%E7%AB%AF%E5%8F%A3%E7%BB%95%E8%BF%87%E6%A0%A1%E5%9B%AD%E7%BD%91%E8%AE%A4%E8%AF%81%E4%B8%8A%E7%BD%91/) 。

但是，恰巧不幸的是，这种情况是很少存在的，也就是说 53 端口仅只允许 DNS 报文通过。如果是这种情况，还是有办法的。办法就是，使用 DNS 隧道。

简单来讲，既然 53 端口的 DNS 数据包可以通过网关/防火墙，那么就可以在本机运行一个程序，用来将其他端口数据包伪装成 DNS 数据包，发送到本地 DNS 服务器，这样网关/防火墙也不会进行拦截。但是这样仅只是将数据发送出去，如何回来呢？回来需要两个东西，一个是 VPS ，另一个就是域名。还得在域名购买商那里做如下解析设置：

| 主机记录 | 类型   | 值                 |
| -------- | ------ | ------------------ |
| NS       | d2t    | tunnel.0xf4n9x.com |
| A        | tunnel | 47.73.228.119      |

以上，d2t 和 tunnel 可以随意命名；另外，VPS 公网 IP 为 47.73.228.119。还有一点就是 VPS 是某马家的学生云，在此文发布之后，或可能未续费而停掉。意思就是说，不要想着搞我服务器了，虽然公网 IP 暴露了。

然后步入正题做个假设，我们在本机 PC 上将数据包伪装成 DNS 数据，且向本地 DNS 服务器指定将要查询一个域名，而本地域名服务器收到数据后，并不能成功解析，便只能将此数据包进行转发，转发到哪里呢？请注意上表中的 NS 记录，就是用来指定一个域名由 VPS 来进行解析；所以毫无疑问，数据包顺利地到达服务器。接下来我们同样可以在 VPS 上运行一个同样的程序，用来对伪装的数据包来进行还原，然后再将还原的数据包发送到互联网中。再然后服务器就会收到回来的响应数据包，再对此响应包进行 伪装成 DNS 响应数据包，按照过来的路径，反向地将伪装好的 DNS 响应数据包发送到本机 PC，PC 收到 DNS 伪装响应包后，再对其进行还原，最终达到本机 PC 收到真正需要的数据包。

![flow chart](http://i1.wp.com/ww1.sinaimg.cn/large/006V665tgy1g19hlzc5fkj313t0h8dg6.jpg)

## **开始实战**

### **所需**

- VPS
  - Ubuntu 16 serevr
  - 带宽 1 Mbps
  - IP 148.70.218.239
- Domain
  - 0xf4n9x.com

| 主机记录 | 记录   | 值                 |
| -------- | ------ | ------------------ |
| NS       | d2t    | tunnel.0xf4n9x.com |
| A        | tunnel | 47.73.228.119      |

- PC
  - Ubuntu 18 desktop

### **伪装程序**

前面谈原理的时候，说到需要一个对数据包做 DNS 伪装的程序。而这个实现这种功能的程序有很多。

就拿我用过的两款软件来说，第一个是 dns2tcp，第二个，也就是要说的主角就是 iodine。由于前者相较于后者较复杂，使用未成功，故弃之，主要说后者。

这个小工具可以通过 DNS 服务器对 IPv4 数据进行隧道传输。有时候防火墙禁止了其他类型的流量时，而 DNS 查询流量却未被禁用时，此时就可以用来传输正常 IPv4 流量。

这个工具其实是攻击者用来通过 DNS 隧道来反弹 shell 滴，不过我是拿来突破校园网认证。

Github：https://github.com/yarrick/iodine

### **服务器**

由于是 Debian 系，所以安装特简单。

```bash
sudo apt-get install iodine
```

然后运行起来

```bash
sudo iodined -f -c -P password 10.0.0.1 d2t.0xf4n9x.com
```

参数解释：

```bash
-f 　前台运行
-c 　禁用检查所有传入请求的客户端IP地址；默认情况，来自不匹配IP请求将被拒绝。
-P　 设置认证密码
```

后面那个 IP 得是一个保留地址，再然后跟一个所要查询的域名。就这样让程序在 VPS 后台运行着。

### **客户端**

同样是 Debian 系，安装方法同样。

```bash
sudo apt-get install iodine
```

然后运行着，不要停止。

```bash
sudo iodine -f -P password d2t.0xf4n9x.com
```

再然后，通过 ssh 服务器，使用 9999 端口来作为转发端口。

```bash
ssh ubuntu@10.0.0.1 -D 9999
```

不用很久，就会登录到服务器。

![iodined](/img/post/Bypass-campusNet/iodined.jpg)

当出现上图标记的那段文字，即为成功。

### **代理**

开启系统自带代理。

![](http://i1.wp.com/ww1.sinaimg.cn/large/006V665tgy1g19ho5btdmj30j90dljrj.jpg)

或者使用浏览器插件  [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif)（墙裂推荐）

Github：[github.com/FelisCatus/SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega)

![](http://i1.wp.com/ww1.sinaimg.cn/large/006V665tgy1g19hohv40aj30vk0i7mxr.jpg)

代理服务器即本机，端口 9999。

### **测试**

![](/img/post/Bypass-campusNet/baidu.jpg)

## **质量**

### **关于网速**

我绕过认证次数总共两次，第一次是在凌晨接近 2 点左右，那时候网速还行；而第二次在在写这篇文章的白天下午，速度是出了奇的慢，打开个百度将近十秒钟。

另外，也和我的 VPS 出口带宽有莫大的关系；毕竟只有 1Mbps。

### **未遵循标准的结果**

TCP/IP 四层体系结构已明确规定各个协议的作用，如果非要在不该传输正常数据的端口中传输一切数据，那结果也可想而知。
