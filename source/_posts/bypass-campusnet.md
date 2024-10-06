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
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/bb1221b4-cdf0-440a-b81d-a69752798cf9/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=e025fd4fee364475af11a938d83c7aa8424ec0c210aae7ad3a7519f5f341109e&X-Amz-SignedHeaders=host&x-id=GetObject
id: 10f906e1-7468-80da-91a5-cf400874eee0
---

> 本文在绝大数人眼里或许是篇福利文；在此文中介绍如何通过 DNS TUNNEL 的方式来绕过校园网认证，实现免认证免费上网；或许此招式并不是最优解，但对于绝大多数校园认证网确实能够成功实现。怎么说呢！其实我早盯上了校园网了。

## **场景分析**

### **吐槽**

在某所高校中，存在一家网络运营商，主要面向毫无收入的学生们，为我们提供日常上网冲浪。

其特点就是三字：**贵**、**差**、**抠**。每月 79 的高昂费用；网络质量差，打游戏经常 460，还只让三个设备使用。

虽然我打比赛有机房的免费 Wi-Fi 用，用不着此校园网，但我还是看不下去如此这般从学生身上赚钱的行为，不服这口气，遂有了本文。

### **信息收集**

在这所高校的网络中，统一使用的是 WiFi 热点客户端认证方式；当连上 WiFi 后，本机会向 DHCP 服务器获取一个内网 IP；关于这个 IP 地址，起初还让我很是疑惑，没想到在资源如此匮乏的大天朝，此运营商还会分一个公网 IP 给俺。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/504ccf9d-56a7-4ce5-a0cd-22ecd378fe17/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=efbebf995272c02c416662baf63feee36b00b9d90de215e9a6ecf42395ffc8d4&X-Amz-SignedHeaders=host&x-id=GetObject)

后来才知道这是个保留地址，详见其  [维基百科](https://en.wikipedia.org/wiki/Reserved_IP_addresses) 。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/808eedf0-b34d-427e-99ff-dc2e2964085c/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=bf2454be362cabdd27811896372f9f565e0790fee1edc5bbbfa186514cb55e08&X-Amz-SignedHeaders=host&x-id=GetObject)

在未认证前还会弹出一个下载认证客户端软件的页面，这里所用到的恶心技术就是利用 HTTP 协议的缺陷，当我们访问一个 HTTP 的网站时，网关会对这个响应报文劫持篡改，给我们 302 重定向到一个指定的下载认证客户端页面。而当我们打开一个 HTTPS 类型的网站是不可能被劫持的。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/29ee3668-e355-4a1e-9ddc-99a71c600656/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=2121f8c0a9380b9fa045126ce0e51696e6834d9e511246b9cbd69bd621dbd002&X-Amz-SignedHeaders=host&x-id=GetObject)

上图就是重定向后的客户端下载页面，让我匪夷所思的是最上面的那个位置本该是一个域名，为何是个公网 IP。既然没有使用域名，那何必需要 DNS，何不直接关闭 53 端口，为何让我如此这般有机可乘，实在让我百思不得其解 🤔。

由下图可得知，DNS 53 端口是开启的。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/beb0c944-13b5-4d36-8bd2-b5c07c67500b/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=00999f7f9b8c679faf11f394406f48d79b4c8659e6c8126a3e66275196fd9013&X-Amz-SignedHeaders=host&x-id=GetObject)

### **原理简述**

原理其实很简单，由上述信息得知，校园网认证过程一般需要放行 DNS 和 DHCP 报文，也就是 53 和 67/68 端口。53 端口既可以是 UDP，也可是 TCP；67/68 端口走的是 UDP 传输协议。

本文着重点是 DNS 53 端口，其实 UDP 67 也可以绕过认证；但本文将围绕 DNS 53 来实现绕过认证，不讨论后者。

而在这个 53 端口中，网关/防火墙如果不进行报文检查，那么就也将意味着，任何数据包都可以通过此端口传输；如果真的是这样的话，那就很简单了，直接 openVPN 架起，详见此文  [利用 openVPN 实现 udp53,67,68 端口绕过校园网认证上网](http://zgao.top/2019/03/03/%E5%88%A9%E7%94%A8openvpn%E5%AE%9E%E7%8E%B0udp536768%E7%AB%AF%E5%8F%A3%E7%BB%95%E8%BF%87%E6%A0%A1%E5%9B%AD%E7%BD%91%E8%AE%A4%E8%AF%81%E4%B8%8A%E7%BD%91/) 。

但是，恰巧不幸的是，这种情况是很少存在的，也就是说 53 端口仅只允许 DNS 报文通过。如果是这种情况，还是有办法的。办法就是，使用 DNS 隧道。

简单来讲，既然 53 端口的 DNS 数据包可以通过网关/防火墙，那么就可以在本机运行一个程序，用来将其他端口数据包伪装成 DNS 数据包，发送到本地 DNS 服务器，这样网关/防火墙也不会进行拦截。但是这样仅只是将数据发送出去，如何回来呢？回来需要两个东西，一个是 VPS ，另一个就是域名。还得在域名购买商那里做如下解析设置：

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f5040d5d-b162-4e6c-b9eb-4dffd58a2e59/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=33006d76587c86705d7bd3a4c9114b8ebfa24230f4dcdedac9060a84f6193a0d&X-Amz-SignedHeaders=host&x-id=GetObject)

以上，d2t 和 tunnel 可以随意命名；另外，VPS 公网 IP 为 47.73.228.119。还有一点就是 VPS 是某马家的学生云，在此文发布之后，或可能未续费而停掉。意思就是说，不要想着搞我服务器了，虽然公网 IP 暴露了。

然后步入正题做个假设，我们在本机 PC 上将数据包伪装成 DNS 数据，且向本地 DNS 服务器指定将要查询一个域名，而本地域名服务器收到数据后，并不能成功解析，便只能将此数据包进行转发，转发到哪里呢？请注意上表中的 NS 记录，就是用来指定一个域名由 VPS 来进行解析；所以毫无疑问，数据包顺利地到达服务器。接下来我们同样可以在 VPS 上运行一个同样的程序，用来对伪装的数据包来进行还原，然后再将还原的数据包发送到互联网中。再然后服务器就会收到回来的响应数据包，再对此响应包进行 伪装成 DNS 响应数据包，按照过来的路径，反向地将伪装好的 DNS 响应数据包发送到本机 PC，PC 收到 DNS 伪装响应包后，再对其进行还原，最终达到本机 PC 收到真正需要的数据包。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/02e3e8a4-ed5a-494c-9eb1-2caf1361fcbb/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=eae5cd143e0106551870655ba44063757a99466424b3057b87e931705d314ce8&X-Amz-SignedHeaders=host&x-id=GetObject)

## **开始实战**

### **所需**

- VPS
  - Ubuntu 16 serevr
  - 带宽 1 Mbps
  - IP 148.70.218.239
- Domain
  - 0xf4n9x.com

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/b149d150-5940-46a3-8963-69acaa79c7df/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=c83582cb482708bd0c9d0892624ff59bbe7fbb6fcbb9580a04c5eb7908639dba&X-Amz-SignedHeaders=host&x-id=GetObject)

- PC
  - Ubuntu 18 desktop

### **伪装程序**

前面谈原理的时候，说到需要一个对数据包做 DNS 伪装的程序。而这个实现这种功能的程序有很多。

就拿我用过的两款软件来说，第一个是 dns2tcp，第二个，也就是要说的主角就是 iodine。由于前者相较于后者较复杂，使用未成功，故弃之，主要说后者。

这个小工具可以通过 DNS 服务器对 IPv4 数据进行隧道传输。有时候防火墙禁止了其他类型的流量时，而 DNS 查询流量却未被禁用时，此时就可以用来传输正常 IPv4 流量。

这个工具其实是攻击者用来通过 DNS 隧道来反弹 shell 滴，不过我是拿来突破校园网认证。

Github：[https://github.com/yarrick/iodine](https://github.com/yarrick/iodine)

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

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/bb1221b4-cdf0-440a-b81d-a69752798cf9/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=f5bfc1d6eab8bf56abb8d487cdc1dd8fa2e5b39b6584b1f9a0e8ed5fc48a7d64&X-Amz-SignedHeaders=host&x-id=GetObject)

当出现上图标记的那段文字，即为成功。

### **代理**

开启系统自带代理。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/53c36842-fb18-46ce-98fe-b64e25ca387e/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=14def0cad33ffd3e24ec03c2629f1b9240be98b06c9b401e603930fd5a5b2565&X-Amz-SignedHeaders=host&x-id=GetObject)

或者使用浏览器插件  [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif)（墙裂推荐）

Github：[github.com/FelisCatus/SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/de4e819e-a3fc-4361-87df-32fb93e740df/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=9bdd313a17a75ac0a9bf144c4012406dde04d2bfac706100937b56223dc13fb8&X-Amz-SignedHeaders=host&x-id=GetObject)

代理服务器即本机，端口 9999。

### **测试**

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/dc9950da-34b6-4fe7-81da-2c1dcd3b2205/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=fba0eab4fc8c6af2874f0e6339e87f7d34b9db753ec4da22645bc6b685036bc9&X-Amz-SignedHeaders=host&x-id=GetObject)

## **质量**

### **关于网速**

我绕过认证次数总共两次，第一次是在凌晨接近 2 点左右，那时候网速还行；而第二次在在写这篇文章的白天下午，速度是出了奇的慢，打开个百度将近十秒钟。

另外，也和我的 VPS 出口带宽有莫大的关系；毕竟只有 1Mbps。

### **未遵循标准的结果**

TCP/IP 四层体系结构已明确规定各个协议的作用，如果非要在不该传输正常数据的端口中传输一切数据，那结果也可想而知。
