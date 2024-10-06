---
created: 2024-09-28T03:29:00+00:00
categories:
  - 技术研究
tags:
  - 逆向
updated: 2024-09-28T00:00:00+00:00
date: 2024-09-26T00:00:00+00:00
slug: x__p-cracking
title: X**p逆向破解分析
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/45037e07-1aa1-49c4-9a93-731184a207b8/s.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083949Z&X-Amz-Expires=3600&X-Amz-Signature=2a607052c028ee8472875600e293ab57920ee783034cbdf16a910dd366cba0a0&X-Amz-SignedHeaders=host&x-id=GetObject
id: 10f906e1-7468-8053-895d-ef51a6698ba1
---

> **开发不易，请支持正版，本文内容仅供逆向研究、学习与探讨，不提供任何破解程序。**

## APP 简介

X\*\*p 是 macOS 上一款体积小巧的知名截图软件，其亮点在于拥有滚动长截图等功能。该软件只上架了 App Store，没有提供其他下载方式。截止本文发出之时，最新版本为 2.2.4，此版本距离上一个版本 2.2.3 的发布，时间之差相隔两年之久。

## 目录结构

在 X\*\*p.app 包中，MacOS 文件夹中的 X\*\*p 为可执行程序文件，Frameworks 中则存放了软件所需的框架，是以.framework 结尾的 Bundle 结构，除此之外，Frameworks 目录中还有以.dylib 结尾的动态库文件。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/8b6279c3-7874-4725-9bd4-cdbf945b63c3/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=f50bcbcb93901aa2d6f46bef8c73df30c284a8e8587cf5d87713dbf218bbc4fe&X-Amz-SignedHeaders=host&x-id=GetObject)

根据文件的命名，当然是要着重留意 X\*\*p 二进制程序与 X\*\*pLibrary.framework 框架。

## 切入点探索

打开该软件，先熟悉下应用程序的功能与配置，发现在首选项中存在两种订阅购买方案，一种是按年订阅付费，另一种是一次付费终生订阅。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/933626d8-d150-4897-ad93-e5983d39cad7/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=c382683c6c57dd2236037c3085fe4bc10c09ed00468dae3c4c6f0471b8705174&X-Amz-SignedHeaders=host&x-id=GetObject)

原本开始的思路是，在网络层面做好 MITM，以对流量进行解密与监控，同时手工地去完成整个订阅购买流程，直到最终的付款确认那一步再取消。这个过程中，实时观察程序产生的网络请求、数据传输。

但未能如我所愿，抓不到明文数据包，这个过程产生的流量似乎走的不是常规的 HTTP 协议？又或者是苹果公司做了限制，不允许这个过程的流量被轻易解密分析，毕竟涉及到敏感的金融支付。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/95537601-86d8-420f-b4c7-7e3fd628e5a6/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=dd4774092d83935dac8acefdd80cc88a2f9843aefc19a928533aaf99d6cd2332&X-Amz-SignedHeaders=host&x-id=GetObject)

但偶然间，却在终端发现了一些面包屑，当我以命令行方式运行二进制 X\*\*p 程序，并手工地去完成整个订阅流程，直到最终的付款确认那一步再取消。此时，一些看似有价值的信息就会产生在终端。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/aeb8b1fe-f057-4275-ae15-70af62c5729e/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=196f17951f04325d3a798beeb31905d74e4722a5bd95ec300ca48a4c1bbef19d&X-Amz-SignedHeaders=host&x-id=GetObject)

如上图中的日志，是在进行按年订阅付费时产生的。下图中的日志则是在进行一次付费终生订阅时产生的。留意两张图中的 itemName，1year_x\*\*p_pro 和 function_pack_1，根据字面意思，前者所代表的含义大概就是一年的 X\*\*p Pro；后者根据实践所得经验，对应的应该是一次付费终生订阅。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/afae29bd-12ad-44b8-b19c-5a83bd807cc1/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=97ffe885adec01d39356dd78a5efedc2b199f1ae7e3c85fb509583b7c466d904&X-Amz-SignedHeaders=host&x-id=GetObject)

这样的信息能出现在终端，大概是开发者的疏忽大意，忘记关闭日志输出。用相同的手段回过头去测上一个版本 2.2.3 的 X\*\*p 程序，终端却是无任何日志产生，在上一个版本中，开发者倒是有关闭日志输出的。

文章开头我提到过，目前最新版 2.2.4 距离上一个版本的发布，时间之差相隔两年之久。那么我大胆推测，经过这两年时间，开发者或许忘了自己曾写的代码逻辑，遂在开发期间把日志输出打开了，最后却忘了关闭。这样便让我有了可趁之机，实属歪打正着、误打误撞。

## 静动结合分析

针对 macOS 应用程序的静态分析，我一般使用 Hopper，IDA Pro 也会用到，二者各有千秋，不妨结合使用、对比分析。对于此处的 X\*\*p 二进制程序，就先用 Hopper 分析好了，根据如上找到的切入点，直接在 Hopper 中搜索 1year_x\*\*p_pro 字符串。

```assembly
                     a1yearx**ppro:
00000001001ae813         db         "1year_x**p_pro", 0         ; DATA XREF=cfstring_1year_x**p_pro
                     aFunctionpack1:
00000001001ae822         db         "function_pack_1", 0        ; DATA XREF=cfstring_function_pack_1
                     aFunctionpack1c:
00000001001ae832         db         "function_pack_1_cn", 0     ; DATA XREF=cfstring_function_pack_1_cn
```

搜索结果如上，现在倒是如我所愿、意料之中了，不仅搜到了 1year_x\*\*p_pro，function_pack_1 也出现了。跟进 1year_x\*\*p_pro 的数据交叉引用，即 cfstring_1year_x\*\*p_pro，这里又有几个交叉引用，见下。

```assembly
                     cfstring_1year_x**p_pro:
00000001002092a8         dq         ___CFConstantStringClassReference, 0x7c8, a1yearx**ppro, 0xe ; "1year_x**p_pro", DATA XREF=sub_10000e588+156, -[XNPRCHelper verifyRCData]+408, -[XNPBuyHelper refreshProductInfoWithHandler:]+308, -[XNPBuySubscriptionViewController productGot:]+204, qword_value_4297101992
                     cfstring_function_pack_1:
00000001002092c8         dq         ___CFConstantStringClassReference, 0x7c8, aFunctionpack1, 0xf ; "function_pack_1", DATA XREF=-[XNPRCHelper verifyRCData]+420, -[XNPBuyHelper refreshProductInfoWithHandler:]+316, qword_value_4297102024
                     cfstring_function_pack_1_cn:
00000001002092e8         dq         ___CFConstantStringClassReference, 0x7c8, aFunctionpack1c, 0x12 ; "function_pack_1_cn", DATA XREF=-[XNPRCHelper verifyRCData]+436, -[XNPBuyHelper refreshProductInfoWithHandler:]+328, qword_value_4297102056
```

1year_x\*\*p_pro 存在着五个数据引用，其中有三个是类方法的引用，指向 Objective-C 方法中的特定偏移量，这意味着这些方法直接使用了该字符串常量进行某种操作。

又根据与 function_pack_1 和 function_pack_1_cn 的数据引用重叠对比，最终确认如下两个引用应特别注意。

```assembly
-[XNPRCHelper verifyRCData]+408
-[XNPBuyHelper refreshProductInfoWithHandler:]+308
```

逐一查看，首先是-[XNPRCHelper verifyRCData]+40，verifyRCData 这个方法的作用看起来是在订阅后对收据（Receipt）数据进行验证的。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/0520c582-e943-4e03-b4dd-8fcf8db83ab7/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=ea8dbbb3f3ad917ef9c98d34791fc13adad6be856e05f5d910fdc6d3920aba69&X-Amz-SignedHeaders=host&x-id=GetObject)

如下是 IDA Pro 9.0 生成的伪代码，论人类可读性，IDA Pro 9.0 更胜一筹。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/7fb705d4-db52-45ac-8671-faa521c42d21/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=9084237c777b14f080234a012e7aebb339e6ac5c4b13c7af47387bc82b29a600&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/46100628-ba0e-4205-9d1d-c0888e40f326/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=0da570b34fc0daacafc814ba8696e6e4d3c1994096e3ed846f64783fddd82916&X-Amz-SignedHeaders=host&x-id=GetObject)

其次是-[XNPBuyHelper refreshProductInfoWithHandler:]+308，这个类方法从名称上来看，应该是通过处理程序刷新产品信息。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/637fc864-dc61-47e1-9d82-5aa7e7c12309/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=339c3a6619c4c9172a1ba908040f7f320e0824d7c5c2017e3ce8a912d9cb5946&X-Amz-SignedHeaders=host&x-id=GetObject)

根据伪代码逻辑，产品信息除了未订阅的状态外，无外乎 1year_x\*\*p_pro（年订阅）、function_pack_1 与 function_pack_1_cn 三种类型，后两者应该都是终身订阅，只是最后一者对应的应该（或许？）是大陆区。

在静态分析一筹莫展，且对伪代码和汇编指令毫无眉目之时，不妨借助动态 Hook 的方式来辅助分析。接下来，将利用 Frida 来 Hook X\*\*p 程序，以动态的方式结合上面静态分析的结果做进一步深入地研究。

先把/Applications 目录中的 X\*\*p.app 复制至家目录某个文件夹中，这样便不必考虑文件权限问题。在 Hook 前还需对 X\*\*p.app 重签名，不然 Frida 无法对其进行 Hook 操作，会报如下错误。

```bash
sudo frida-trace -m "*[XNPRCHelper verifyRCData*]" X**p
Failed to attach: unable to access process with pid 87339 from the current user account
```

如下图，使用 codesign 命令对 X\*\*p.app 重签名后，便能正常 Hook 了。

```bash
codesign -f -s - --deep X**p.app
X**p.app: replacing existing signature
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/2180ec36-ce0a-4277-a5e5-16dff8d4bfa5/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=62d2d627f352a9a8e330e41d9428181038fa7170f144e22e4a7ae3e6f1930884&X-Amz-SignedHeaders=host&x-id=GetObject)

这里先对 XNPRCHelper verifyRCData 进行了 Hook，但该方法无论如何都触发不了调用。

此时改向 Hook XNPBuyHelper refreshProductInfoWithHandler，发现当进入到购买界面，该方法就会被调用一次。

```bash
frida-trace -m "*[XNPBuyHelper refreshProductInfoWithHandler*]" X**p
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/1b30ec26-2c22-4fd8-9dea-cc7964b7ba41/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=77c1ffb290972efeb46ac3b5f855f100376047e416436d8a3dbc05798971f962&X-Amz-SignedHeaders=host&x-id=GetObject)

据此，推测授权调用点仍在 XNPBuyHelper refreshProductInfoWithHandler 方法的上层。

继续回到 Hopper 中，按 x 快捷键快速查找 XNPBuyHelper refreshProductInfoWithHandler 的引用，发现有一个结构体。

```assembly
000000010017f3e4    struct __objc_relative_method {     ; "v24@0:8@?16",@selector(refreshProductInfoWithHandler:)
                        0x100217e68-0x10017f3e4,             // name (relative address)
                        aV240816_1001d2383-0x10017f3e8,      // signature (relative address)
                        -[XNPBuyHelper refreshProductInfoWithHandler:]-0x10017f3ec // implementation (relative address)
                    }
```

结构体中有一个 refreshProductInfoWithHandler 方法选择器，进一步查看引用，如下，直接进入 sub_10017c720+4 中。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/fb7c041f-7970-4ee5-aaf7-b0a773649b03/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=aaba29d638b6d0e43fad27b0d9b42e424a76991c22cacb2962f6e0aef1f2695a&X-Amz-SignedHeaders=host&x-id=GetObject)

```assembly
                 sub_10017c2a0:
000000010017c2a0     adrp    x1, #0x10021b000      ; 0x10021bfd8@PAGE, CODE XREF=-[XNPManageSubscriptionAlertController showAlertOnWindow:]+240, -[XNPBuyViewController viewDidAppear]+180
000000010017c2a4     ldr     x1, [x1, #0xfd8]      ; 0x10021bfd8@PAGEOFF, "refreshProductInfoWithHandler:",@selector(refreshProductInfoWithHandler:)
000000010017c2a8     adrp    x16, #0x1001e0000     ; 0x1001e0590@PAGE
000000010017c2ac     ldr     x16, [x16, #0x590]    ; 0x1001e0590@PAGEOFF, _objc_msgSend_1001e0590,_objc_msgSend
000000010017c2b0     br      x16                   ; _objc_msgSend
```

在 sub_10017c2a0 中存在两个代码引用。

```assembly
-[XNPManageSubscriptionAlertController showAlertOnWindow:]+240
-[XNPBuyViewController viewDidAppear]+180
```

第一个方法是在窗口上显示警报，对 showAlertOnWindow 方法以及 XNPManageSubscriptionAlertController 类下的所有方法进行 Hook，均没有任何反应，所以这里不必多看。

将注意力朝向-[XNPBuyViewController viewDidAppear]+180，对该方法进行 Hook，可发现当进入购买页面时，该方法同样被调用。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/eafa0b43-1102-4e4d-aab3-452b2df52cbf/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=6acb3d56fefd43d2fcea961a53ef38c22f2b3448eff099bb79ee15449b9e23bd&X-Amz-SignedHeaders=host&x-id=GetObject)

此时，又对 XNPBuyViewController 类中的所有方法进行了 Hook，根据观察到的调用栈，一路追踪到 setupUIByPurchase 方法。

```bash
frida-trace -m "*[XNPBuyViewController *]" X**p
           /* TID 0x103 */
 20718 ms  -[XNPBuyViewController viewIdentifier]
 20719 ms  -[XNPBuyViewController viewIdentifier]
 20736 ms  -[XNPBuyViewController viewIdentifier]
 20736 ms  -[XNPBuyViewController toolbarItemImage]
 20737 ms  -[XNPBuyViewController toolbarItemLabel]
 ......
 20748 ms  -[XNPBuyViewController setButtonPrivacy:0x139f336f0]
 20748 ms  -[XNPBuyViewController setButtonTerms:0x139f34800]
 20748 ms  -[XNPBuyViewController setBuyFeatureViewWrapperView:0x139f33230]
 20748 ms  -[XNPBuyViewController setManageSubscriptionButton:0x139f32b10]
 20748 ms  -[XNPBuyViewController setSubscriptionViewWrapperView:0x139f388d0]
 20749 ms  -[XNPBuyViewController viewDidLoad]
 20749 ms     | -[XNPBuyViewController setPackBackgroudViews:0x6000026230f0]
 20749 ms     | -[XNPBuyViewController setSubscriptionViewController:0x600001950500]
 20749 ms     | -[XNPBuyViewController subscriptionViewController]
 20749 ms     | -[XNPBuyViewController subscriptionViewWrapperView]
 20749 ms     | -[XNPBuyViewController subscriptionViewController]
 20753 ms     | +[XNPBuyViewController setupButton:0x13b012840]
 20755 ms     | -[XNPBuyViewController setupPackBackgroundView:0x13b011a40]
 20755 ms     |    | -[XNPBuyViewController packBackgroudViews]
 20755 ms     | -[XNPBuyViewController setBuyFeatureViewController:0x600001d41080]
 20755 ms     | -[XNPBuyViewController buyFeatureViewController]
 20755 ms     | -[XNPBuyViewController buyFeatureViewWrapperView]
 20755 ms     | -[XNPBuyViewController buyFeatureViewController]
 20757 ms     | +[XNPBuyViewController setupButton:0x13b174d10]
 20757 ms     | -[XNPBuyViewController setupPackBackgroundView:0x13b168300]
 20757 ms     |    | -[XNPBuyViewController packBackgroudViews]
 20757 ms     | -[XNPBuyViewController setupPackBackgroundView:0x13b170a60]
 20757 ms     |    | -[XNPBuyViewController packBackgroudViews]
 20757 ms     | -[XNPBuyViewController manageSubscriptionButton]
 20757 ms     | -[XNPBuyViewController buttonPrivacy]
 20757 ms     | -[XNPBuyViewController buttonTerms]
 20757 ms     | -[XNPBuyViewController appearanceDidChangeNotificaion:0x0]
 20757 ms     |    | -[XNPBuyViewController packBackgroudViews]
 20757 ms     | -[XNPBuyViewController setupUIByPurchase]
 20757 ms     |    | -[XNPBuyViewController buyFeatureViewController]
 ......
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/102bbf79-ccc1-4a9e-8ff7-f68d92072225/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=765bd8a35adc58949fc71bdfff03588293e20c2e0d171c47cbf16d74bc063558&X-Amz-SignedHeaders=host&x-id=GetObject)

```objective-c
void __cdecl -[XNPBuyViewController setupUIByPurchase](XNPBuyViewController *self, SEL a2)
{
  void *v3; // x20
  unsigned int v4; // w21
  NSView *v5; // x21
  NSArray *v6; // x20
  void *v7; // x0
  void *v8; // x21
  __int64 v9; // x22
  void *v10; // x23
  XNPBuyThankSubscriptionViewController *v11; // x20
  NSView *v12; // x20
  XNPBuyThankSubscriptionViewController *v13; // x21
  void *v14; // x22
  NSButton *v15; // x20
  XNPBuyFeatureViewController *v16; // x19
  __int128 v17; // [xsp+0h] [xbp-100h] BYREF
  __int128 v18; // [xsp+10h] [xbp-F0h]
  __int128 v19; // [xsp+20h] [xbp-E0h]
  __int128 v20; // [xsp+30h] [xbp-D0h]
  _BYTE v21[128]; // [xsp+48h] [xbp-B8h] BYREF

  v3 = objc_retainAutoreleasedReturnValue(+[XNPSystemStatus sharedSystemStatus](&OBJC_CLASS___XNPSystemStatus, "sharedSystemStatus"));
  v4 = (unsigned int)objc_msgSend(v3, "p");
  objc_release(v3);
  if ( v4 )
  {
    v19 = 0u;
    v20 = 0u;
    v17 = 0u;
    v18 = 0u;
    v5 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController subscriptionViewWrapperView](self, "subscriptionViewWrapperView", 0LL));
    v6 = objc_retainAutoreleasedReturnValue(-[NSView subviews](v5, "subviews"));
    objc_release(v5);
    v7 = -[NSArray countByEnumeratingWithState:objects:count:](
           v6,
           "countByEnumeratingWithState:objects:count:",
           &v17,
           v21,
           16LL);
    if ( v7 )
    {
      v8 = v7;
      v9 = *(_QWORD *)v18;
      do
      {
        v10 = 0LL;
        do
        {
          if ( *(_QWORD *)v18 != v9 )
            objc_enumerationMutation(v6);
          objc_msgSend(*(id *)(*((_QWORD *)&v17 + 1) + 8LL * (_QWORD)v10), "removeFromSuperview");
          v10 = (char *)v10 + 1;
        }
        while ( v8 != v10 );
        v8 = -[NSArray countByEnumeratingWithState:objects:count:](
               v6,
               "countByEnumeratingWithState:objects:count:",
               &v17,
               v21,
               16LL);
      }
      while ( v8 );
    }
    objc_release(v6);
    v11 = -[XNPBuyThankSubscriptionViewController init](
            objc_alloc(&OBJC_CLASS___XNPBuyThankSubscriptionViewController),
            "init");
    -[XNPBuyViewController setThankSubscriptionViewController:](self, "setThankSubscriptionViewController:", v11);
    objc_release(v11);
    v12 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController subscriptionViewWrapperView](self, "subscriptionViewWrapperView"));
    v13 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController thankSubscriptionViewController](self, "thankSubscriptionViewController"));
    v14 = objc_retainAutoreleasedReturnValue(-[XNPBuyThankSubscriptionViewController view](v13, "view"));
    -[NSView addSubview:](v12, "addSubview:", v14);
    objc_release(v14);
    objc_release(v13);
    objc_release(v12);
    v15 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController manageSubscriptionButton](self, "manageSubscriptionButton"));
    -[NSButton setHidden:](v15, "setHidden:", 0LL);
    objc_release(v15);
  }
  v16 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController buyFeatureViewController](self, "buyFeatureViewController"));
  -[XNPBuyFeatureViewController setupUIByPurchase](v16, "setupUIByPurchase");
  objc_release(v16);
}
```

setupUIByPurchase 方法的伪代码如上，在其中调用了 sharedSystemStatus 方法，获取 XNPSystemStatus 类的共享实例，将其保存到 v3，随后又调用了 p 方法，将其返回值保存至 v4。

根据如上观察到的调用栈（已将关键调用栈放置在下面），能够表明在 setupUIByPurchase 方法中，程序未能通过第一个 if 判断，直接到达了对 v16 的赋值以及对 XNPBuyViewController buyFeatureViewController 方法的调用；否则，如果通过了第一个`if (v4)`的判断，那下一个该调用的方法应当是 XNPBuyViewController subscriptionViewWrapperView 方法。关键代码已放置下边，显而易见，一看即明。

```bash
// 关键调用栈
 20749 ms  -[XNPBuyViewController viewDidLoad]
							......
 20757 ms     | -[XNPBuyViewController setupUIByPurchase]
 20757 ms     |    | -[XNPBuyViewController buyFeatureViewController]
					 ......
```

```objective-c
// 关键代码
  v3 = objc_retainAutoreleasedReturnValue(+[XNPSystemStatus sharedSystemStatus](&OBJC_CLASS___XNPSystemStatus, "sharedSystemStatus"));
  v4 = (unsigned int)objc_msgSend(v3, "p");
  objc_release(v3);
  if ( v4 )
  {
    // ......
    v5 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController subscriptionViewWrapperView](self, "subscriptionViewWrapperView", 0LL));
    // ......
  }
  v16 = objc_retainAutoreleasedReturnValue(-[XNPBuyViewController buyFeatureViewController](self, "buyFeatureViewController"));
  -[XNPBuyFeatureViewController setupUIByPurchase](v16, "setupUIByPurchase");
  objc_release(v16);
```

那么这里如果能够通过第一个 if 判断，或许就能完成授权，从而达到破解目的。根据代码的判断逻辑，只需要求 v4 非零即可通过判断，这样看来，p 方法的返回值就是关键所在。

通过进一步地搜索，发现 XNPSystemStatus 类存在于 X\*\*pLibrary.framework/Versions/A/X\*\*pLibrary 中。

```assembly
    _OBJC_CLASS_$_XNPSystemStatus:
0000000100260a48     extern function code     ; in @rpath/X**pLibrary.framework/Versions/A/X**pLibrary, DATA XREF=sub_100007424+152, -[XNPMenuletMenu updateProItem]+24, -[XNPFileNameWindowController windowDidLoad]+704, -[XNPFileNameWindowController windowDidChangeOcclusionState:]+56, -[XNPFileNameWindowController okButtonClick:]+28, -[XNPBuyViewController setupUIByPurchase]+48, -[XNPBuyFeatureViewController setupUIByPurchase]+24, -[XNPBuyFeatureViewController pack1ButtonClick:]+32, -[XNPMainMenu updateProItem]+24, -[XNPDistributeNotificationManager sendPStatus]+24, -[XNPBuySubscriptionViewController yearlySubscribeButtonClick:]+32
```

接着便对 X\*\*pLibrary 进行反汇编，以找到 XNPSystemStatus 类。

首先看 p 方法，其地址为 0x1\*\*dc。

```assembly
            -[XNPSystemStatus p]:
000000000001**dc    ldrb    w0, [x0, #0x9]    ; Objective C Implementation defined at 0xed1c4 (instance method), DATA XREF=0xed1c4
000000000001**e0    ret
```

伪代码如下。

```objective-c
bool __cdecl -[XNPSystemStatus p](XNPSystemStatus *self, SEL a2)
{
  return self->_p;
}
```

到此，事情就变得简单多了。我们还是先用 Frida Hook 这个方法，先看看其正常的返回值，如下图所示，在正常未订阅状态下，返回的是 0x0。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/bd13bd6d-9d13-4e23-9b91-dea75b7c1204/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=574983aee763e5ecdf6ed9fb204ddd590201dbcc00fbcb9dadcdfcf178fb8304&X-Amz-SignedHeaders=host&x-id=GetObject)

那不妨劫持 p 方法，将其返回值修改为 0x1，看看会有何发生？

```javascript
// Path: __handlers__/XNPSystemStatus/p.js

defineHandler({
  onEnter(log, args, state) {},

  onLeave(log, retval, state) {
    log(
      `-[XNPSystemStatus p] return: ` + retval + " onLeave at the beginning."
    );
    retval.replace(0x1);
    log(`-[XNPSystemStatus p] return: ` + retval + " onLeave at the end.");
  },
});
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/40045cf4-8c8e-4f1b-96fd-d3b212187383/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=e2887b0e7a9cc1ff3c0871944130d820493b20eef5fc59466bf88d41d29bb96e&X-Amz-SignedHeaders=host&x-id=GetObject)

破解了？但似乎不够完美。

既然劫持篡改了 p 方法的返回值能够达到年订阅的效果，那便继续寻找能够控制达到终生订阅的方法。从何找起？显然是遵循就近优先原则，从 XNPSystemStatus 类中找起，不难发现有一个与 p 很相似的 p1 方法，地址为 0x1\*\*ec。

```assembly
            -[XNPSystemStatus p1]:
000000000001**ec    ldrb    w0, [x0, #0xa]    ; Objective C Implementation defined at 0xed1dc (instance method), DATA XREF=0xed1dc
000000000001**f0    ret
```

```objective-c
bool __cdecl -[XNPSystemStatus p1](XNPSystemStatus *self, SEL a2)
{
  return self->_p1;
}
```

由于 p1 与 p 的相似性，猜测这可能就是控制另一种终生订阅授权方式的方法。遂继续回到 X\*\*p 中查找相关调用，发现在 XNPBuyFeatureViewController setupUIByPurchase 中存在其调用，且 XNPBuyFeatureViewController setupUIByPurchase 与 XNPBuyViewController 中的 setupUIByPurchase 方法也非常相似，这也印证了 p1 就是控制另一种终生订阅授权的方法的猜想。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/0cfb2e31-458d-4532-8076-f84ded5e7fa8/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=739d3955a0d316e7ab2f35add5d86e15117f491b675029ad61c3a89cc53ecc3f&X-Amz-SignedHeaders=host&x-id=GetObject)

既然如此，此处也利用 Frida Hook p1 方法，将其返回值由 0x0 改为 0x1，效果图如下，完全符合预期。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/55229b41-5a2e-462e-a62b-a670c185c573/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083950Z&X-Amz-Expires=3600&X-Amz-Signature=f6ad21fe1763926de8fa0cb3a749ef1eb1f9bfebf2e729798df7814a49636870&X-Amz-SignedHeaders=host&x-id=GetObject)

## 动态库注入

以上虽说是破解了，但在应用程序的使用上，显然是不够便捷的。于是我们便可以想到将 Frida 连同 Hook 脚本作为一个动态依赖库注入至 X\*\*p.app 中，Frida 官方也提供了对应的 Gadget dylib，我们只需编写配置文件加载相应的 Hook 脚本即可。

这个过程还需要使用到 insert_dylib，这是一个将 dylib 动态依赖库插入至 Mach-O 二进制文件的命令行程序，可以利用它将 Frida Gadget dylib 快速注入至 X\*\*p 程序中。

首先，使用 GCC 将 insert_dylib 的 C 代码编译成二进制文件。

```bash
git clone https://github.com/tyilo/insert_dylib
gcc insert_dylib/insert_dylib/main.c -o insert_dylib
```

然后在 Frida 的 GitHub 仓库（[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)）中找到以 dylib 结尾的 Frida Gadget dylib 文件并下载。参考 Frida 官方文档，编写如下配置文件，命名为 frida.config，配置文件的文件名需要与 Frida Gadget dylib 的文件名保持一致。

```json
{
  "interaction": {
    "type": "script",
    "path": "./frida.js"
  }
}
```

最终的 Hook 脚本 frida.js 文件内容如下。

```javascript
var appBaseAddr = Module.findBaseAddress("X**pLibrary");

var func = appBaseAddr.add(0x1 ** dc);
Interceptor.attach(func, {
  onEnter(this1, args) {},
  onLeave(retval) {
    console.log(
      `-[XNPSystemStatus p] return: ` + retval + " onLeave at the beginning."
    );
    retval.replace(0x1);
    console.log(
      `-[XNPSystemStatus p] return: ` + retval + " onLeave at the end."
    );
  },
});

var func = appBaseAddr.add(0x1 ** ec);
Interceptor.attach(func, {
  onEnter(this1, args) {},
  onLeave(retval) {
    console.log(
      `-[XNPSystemStatus p1] return: ` + retval + " onLeave at the beginning."
    );
    retval.replace(0x1);
    consolelog(
      `-[XNPSystemStatus p1] return: ` + retval + " onLeave at the end."
    );
  },
});
```

将所有相关文件复制至 X\*\*p.app 中，包含 insert_dylib 二进制程序、Frida Gadget dylib、Frida 配置文件及 Hook 脚本文件。

```bash
cp frida-gadget-16.5.2-macos-universal.dylib 224/X**p.app/Contents/MacOS/frida.dylib
cp insert_dylib frida.config frida.js 224/X**p.app/Contents/MacOS/
```

再利用 insert_dylib 将 Frida Gadget dylib 注入至 X\*\*p 二进制程序中。

```bash
cd 224/X**p.app/Contents/MacOS && ./insert_dylib @executable_path/frida.dylib ./X**p
Binary is a fat binary with 2 archs.
LC_CODE_SIGNATURE load command found. Remove it? [y/n] y
LC_CODE_SIGNATURE load command found. Remove it? [y/n] y
Added LC_LOAD_DYLIB to all archs in ./X**p_patched

mv X**p X**p_bak && mv X**p_patched X**p
```

最后对 X\*\*p.app 再次重签，并移至/Applications 中，即完成最后的破解。

```bash
codesign -f -s - --deep X**p.app
```

破解后的效果图如下，长截图功能正常使用，且无任何水印。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ff6b08fd-ffca-4b05-8312-5c9ac7479fa2/s.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083951Z&X-Amz-Expires=3600&X-Amz-Signature=6ab4405cd1f002ac9e8446bcb2e1ce865e44545db2d3bd93175e86273890c16a&X-Amz-SignedHeaders=host&x-id=GetObject)

## 参考链接

- [https://frida.re/docs/gadget/](https://frida.re/docs/gadget/)
