---
created: 2024-10-08T01:53:00+00:00
categories:
  - 技术研究
description: 在未使用反破解技术的前提下，规范化开发虽能够为未来的开发与维护带来极大的便利，但同时也可能无意间为破解者提供更多可利用的线索与机会。
tags:
  - 逆向
updated: 2024-10-09T00:00:00+00:00
date: 2024-10-06T00:00:00+00:00
slug: t__o-cracking
title: T**O逆向破解：从无限试用到永久订阅
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/b3654e07-1a1c-4bf6-b95c-50292b520a1f/to.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=44b92dfd329a1ac81438e8f98ecde52a90e231a17ea60102566bbd21845b09c3&X-Amz-SignedHeaders=host&x-id=GetObject
id: 119906e1-7468-804d-b34c-e11ab6295c89
---

## AI 自动摘要

本文详细介绍了对 macOS 上一款休息提醒软件 T\*\*O 进行逆向破解的过程，从无限试用到实现永久订阅。主要内容包括：

- 分析软件的订阅机制和试用功能。
- 使用 IDA Pro 反汇编程序，寻找关键代码。
- 通过 Hook 技术实现无限试用。
- 深入分析关键方法，实现永久订阅。

文章展示了逆向工程的基本步骤，包括代码分析、Hook 注入等技术，为读者提供了一个实际的逆向破解案例。

## 引子与动机

有时，我们太过于专注工作和学习，不知不觉间在电脑面前一坐就是好几个小时。久坐的危害自不必多说，并且这种方式的效率未必就如我们想象的那般高效。长时间坐在电脑面前，思维往往会变得迟钝，灵感也容易枯竭，身体的疲惫与大脑的倦怠相互交织，反而会使工作和学习进展得缓慢。

于是，我便找到了 macOS 上这款休息提醒软件 T\*\*O，利用它每隔一段时间就提醒我休息几分钟，起身走一走接杯水，或眺望远方缓解眼疲劳等等。当然，这个休息的频率是可以按照自己的习惯自定义设置的。

原本，在用这款软件的期间根本就没想过破解它，当时觉得作者很良心、免费功能已足够使用。但后来买了手表后不再用这款软件了，倒是动起了破解该软件的念头，遂在假期闲来无事时实现这个想法。

## 订阅与试用

在正式破解前，还是先对此应用程序多一些了解。打开 APP 可以看到 T\*\*O 有三种档次的订阅制，分别是$3.99/季度、$7.99/半年、$14.99/一年。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/9e23904d-ce92-4d33-9f53-8787282fe2fe/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=a26f2cbde95ef7822d8db030d724a3ea89ae1017bc44e0a1385d97c23e7680dd&X-Amz-SignedHeaders=host&x-id=GetObject)

除了付费订阅，软件开发者还提供了免费试用，每次可试用一小时高级功能，次数不限，从这点上能看出开发者其实很良心了。

如下图所示，当红心 ❤️ 被点亮时，即为此处的高级功能正在被使用，也意味着当前处于试用期。但过一小时后，这些高级功能就会自动关闭，需手动再次开启，前面提到过，一次只可试用一小时。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/942da945-924b-4871-ba59-803fd71de45f/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=d286ee241e410929cb7e69a277aea1abbb37cae70ca5c336a7c203d3dbf0b14b&X-Amz-SignedHeaders=host&x-id=GetObject)

## 无限试用

开始，还是先对 APP 进行重签名，然后利用 IDA Pro 反汇编 T**O 二进制程序，并在终端以命令行方式运行 T**O 二进制程序，此时产生了如下日志，出现了产品清单，都是未购买的。

```bash
$ ./T**O
2024-10-06 17:50:17.194 T**O[16002:61422285] Started T**O version 0.0.0 (0000) Mac App Store edition from file:///Applications/T**O.app/
loading 2024-10-06 04:00:00 +0000 took 0.008906006813049316 seconds
2024-10-06 17:50:17.251 T**O[16002:61422285] accessibility trusted: 0
2024-10-06 17:50:17.803 T**O[16002:61422285] App was not auto-launched
2024-10-06 17:50:17.932 T**O[16002:61422285] IAP: addTransactionObserver: <InAppPurchaseManager: 0x600002c46940>
2024-10-06 17:50:17.950 T**O[16002:61422285] accessibility trusted: 0
2024-10-06 17:50:23.601 T**O[16002:61422291] IAP: Not purchased: com.d****.t**o.base.3m
2024-10-06 17:50:23.601 T**O[16002:61422291] IAP: Not purchased: com.d****.t**o.base.6m
2024-10-06 17:50:23.601 T**O[16002:61422291] IAP: Not purchased: com.d****.t**o.base.12m
2024-10-06 17:50:23.638 T**O[16002:61422291] IAP: Loaded list of products...
2024-10-06 17:50:23.638 T**O[16002:61422291] IAP: Not purchased: com.d****.t**o.base.3m
2024-10-06 17:50:23.638 T**O[16002:61422291] IAP: Not purchased: com.d****.t**o.base.6m
2024-10-06 17:50:23.638 T**O[16002:61422291] IAP: Not purchased: com.d****.t**o.base.12m
```

显然，3m、6m 和 12m 分别代表着三种不同档次的订阅制。根据这个微小的线索，尝试在 IDA Pro 中搜索 3m、6m 或 12m 关键词。

```assembly
__cfstring:000000010011F3B8 cfstr_3m    __CFString <___CFConstantStringClassReference, 0x7C8, a3m, 2>
__cfstring:000000010011F3B8                                     ; DATA XREF: -[D****LicenseController stage]+424↑o
__cfstring:000000010011F3B8                                     ; -[D****LicenseController kindForIdentifier:]+18↑o ...
__cfstring:000000010011F3B8                                     ; "3m"
__cfstring:000000010011F3D8 cfstr_6m    __CFString <___CFConstantStringClassReference, 0x7C8, a6m, 2>
__cfstring:000000010011F3D8                                     ; DATA XREF: -[D****LicenseController stage]+440↑o
__cfstring:000000010011F3D8                                     ; -[D****LicenseController kindForIdentifier:]:loc_100038F34↑o ...
__cfstring:000000010011F3D8                                     ; "6m"
__cfstring:000000010011F3F8 cfstr_12m   __CFString <___CFConstantStringClassReference, 0x7C8, a12m, 3>
__cfstring:000000010011F3F8                                     ; DATA XREF: -[D****LicenseController stage]+5D0↑o
__cfstring:000000010011F3F8                                     ; -[D****LicenseController kindForIdentifier:]:loc_100038F50↑o ...
__cfstring:000000010011F3F8                                     ; "12m"
```

如上，三者存在两个相同的引用，分别是-[D****LicenseController stage]和-[D****LicenseController kindForIdentifier:]，先看前者。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/32a83e80-d436-44a2-a09e-8ef75e54e911/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=d3430b6960ba5c00ced705f31a7ce9a68f29e56f5f49a670fa497089b14461a7&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/8112a861-f685-4023-be98-093ff615d5c7/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=2798e8bf5cf9f4516de91783f50d103cddfbc9f2ccdc5b23880c714c15a85a2f&X-Amz-SignedHeaders=host&x-id=GetObject)

这个类方法的代码量有点小大，根据此类方法的签名可知它返回的类型为整数型，那暂时也不着急去详细分析代码逻辑，继续查看这个类方法的引用，可发现在-[D****LicenseController stageDescription]中存在其调用。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/505df047-e2fa-4c62-a7f2-5dc61e47ae22/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=49ff83ded522518676c4ffc7f71a4762dd8fdf60721b961fb76e1fc11549a61c&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/32b235bb-2695-4543-8164-fb5be2928799/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=41cda49dd9ee77c5f93d6212048099d62b9b055cc78e9f503f76d9b0f1db9fc1&X-Amz-SignedHeaders=host&x-id=GetObject)

```objective-c
id __cdecl -[D****LicenseController stageDescription](D****LicenseController *self, SEL a2)
{
  __int64 v2; // x0

  v2 = -[D****LicenseController stage](self, "stage");
  if ( v2 <= 539 )
  {
    if ( v2 == 500 )
      return CFSTR("Free");
    if ( v2 == 510 )

      return CFSTR("Trial");
  }
  // ...
}
```

-[D****LicenseController stageDescription]的伪代码如上，其中调用了-[D****LicenseController stage]，将其返回值赋给 v2，随后对 v2 进行判断，如果等于 500 返回 Free，如果等于 510 则返回 Trial。

基于此，还是先看看-[D****LicenseController stage]的返回值，将其打印出来看看。

```bash
$ frida-trace -m "*[D****LicenseController stage*]" "T**O"
393760 ms  -[D****LicenseController stage] return: 0x1f4 onLeave at the beginning.
```

结果如上，正常情况下，该类方法会返回 0x1f4，转换成十进制等于 500，正好对应上了代码中的 Free。

```objective-c
if ( v2 == 500 )
  return CFSTR("Free");
```

既然如此，继续 Hook -[D****LicenseController stage]，并将其返回值改成 510 看看，对应的十六进制为 0x1fe，Hook 脚本内容如下。

```javascript
defineHandler({
  onEnter(log, args, state) {
    // log(`-[D****LicenseController stage]`);
  },

  onLeave(log, retval, state) {
    log(
      `-[D****LicenseController stage] return: ` +
        retval +
        " onLeave at the beginning."
    );
    retval.replace(0x1fe);
    log(
      `-[D****LicenseController stage] return: ` +
        retval +
        " onLeave at the end."
    );
  },
});
```

```bash
393760 ms  -[D****LicenseController stage] return: 0x1f4 onLeave at the beginning.
393760 ms  -[D****LicenseController stage] return: 0x1fe onLeave at the end.
```

通过长时间的测试，经过这样的改动后，确实能够一直永久试用下去，中间再无中断，无需手动再次开启试用。

本文原本到此就该结束了的，但到了第二天，我发现这种破解方式不够完美，无法创建无限个数的 Breaks，唯独这一个高级功能是需要付费订阅才可拥有。此外，本文到此为止篇幅还比较短，无论是写起来还是读起来，总让人觉得意犹未尽。

## 永久订阅

为此，我们继续看先前未详细分析的-[D****LicenseController stage]，由于这个类方法的代码量有点小大，先从开头看起。IDA Pro 反汇编出的部分伪代码如下，在最后，这个方法会返回整型变量 v61，这一点牢记。另外，还可看出一些数据在反汇编的过程中出现了丢失，这意味着不可过于依赖反汇编后的伪代码。

```objective-c
signed __int64 __cdecl -[D****LicenseController stage](D****LicenseController *self, SEL a2)
{
  D****LicenseController *v2; // x26

	// ......

	v2 = self;
	v3 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastValidation](self, "lastValidation"));
	-[NSDate timeIntervalSinceNow](v3, "timeIntervalSinceNow");
	v5 = v4;
	objc_release(v3);
	v6 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastTrialExpiryDate](v2, "lastTrialExpiryDate"));
	-[NSDate timeIntervalSinceNow](v6, "timeIntervalSinceNow");
	v8 = v7;
	objc_release(v6);
	v9 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastPurchaseExpiryDate](v2, "lastPurchaseExpiryDate"));
	-[NSDate timeIntervalSinceNow](v9, "timeIntervalSinceNow");
	v11 = v10;
	objc_release(v9);
	v12 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastValidation](v2, "lastValidation"));
	objc_release(v12);
	v13 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastTrialExpiryDate](v2, "lastTrialExpiryDate"));
	v15 = v8 <= 0.0 && v13 != 0LL;
	objc_release(v13);
	v16 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastPurchaseExpiryDate](v2, "lastPurchaseExpiryDate"));
	v18 = v11 <= 0.0 && v16 != 0LL;
	objc_release(v16);
	if ( v12 && v5 <= 0.0 && v5 > -15.0 && (!v15 || v8 <= -15.0) && (!v18 || v11 <= -15.0) )
	  return -[D****LicenseController lastStage](v2, "lastStage");
  v20 = objc_retainAutoreleasedReturnValue(-[D****LicenseController licenses](v2, "licenses"));
  v21 = objc_retainAutoreleasedReturnValue(-[NSMutableDictionary objectForKeyedSubscript:](v20, "objectForKeyedSubscript:", CFSTR("LicenseDetails")));
  v22 = objc_retainAutoreleasedReturnValue(objc_msgSend(v21, "sortedArrayUsingComparator:", &stru_100118D98));
  objc_release(v21);
  objc_release(v20);
  v23 = objc_retainAutoreleasedReturnValue(-[D****LicenseController unregistered](v2, "unregistered"));
  if ( v23 )
  {
    v24 = objc_retainAutoreleasedReturnValue(objc_msgSend(v22, "arrayByAddingObject:", v23));
    objc_release(v22);
    v22 = v24;
  }

	// ......

  return v61;
}
```

对于 T\*\*O 这个应用程序，用户在使用该 APP 的过程中，会处于不同的阶段，例如免费使用阶段、试用阶段和付费订阅阶段。而 stage 翻译成中文是“阶段”，意味着此方法的作用很可能就是通过某些检查，以确定应用程序当前所处的阶段。

根据如上代码可看出，开头先调用了 lastValidation、lastTrialExpiryDate 和 lastPurchaseExpiryDate 三个方法来获取最后一次验证时间、试用期过期时间和订阅过期时间。然后对它们调用 timeIntervalSinceNow 方法，以计算这些时间与当前时间的时间差，相当于计算出验证、试用和订阅的剩余时间。随后对这三个时间间隔进行条件判断，小于等于 0.0 表示时间已过，大于-15.0 意味着有 15 秒的时间窗口，而小于等于-15.0 则意味着这个时间窗口已过。

```objective-c
if ( v12 && v5 <= 0.0 && v5 > -15.0 && (!v15 || v8 <= -15.0) && (!v18 || v11 <= -15.0) )
  return -[D****LicenseController lastStage](v2, "lastStage");
```

如上判断如果通过则会返回-[D****LicenseController lastStage]，这个方法可能代表应用程序在最后一次有效状态下的阶段。

这时能想到的是，先观察 lastValidation、lastTrialExpiryDate 和 lastPurchaseExpiryDate 三个方法的正常返回值，有必要时可尝试进行篡改。通过如下 Hook 代码，便能打印出人类可读的时间格式，否则直接打印出来会是一个指向 NSDate 类型的地址。除 lastValidation 外，其他两个方法也一样做如下操作。

```javascript
defineHandler({
  onEnter(log, args, state) {},
  onLeave(log, retval, state) {
    var retval0 = new ObjC.Object(retval);
    log(
      `-[D****LicenseController lastValidation] return: ` +
        retval0 +
        " onLeave at the beginning."
    );
  },
});
```

结果如下，当前由于我既没有试用也没有订阅应用程序，所以 lastTrialExpiryDate 和 lastPurchaseExpiryDate 均为 nil，这是理所当然的。

```bash
-[D****LicenseController lastValidation] return:  2024-10-08 09:07:46 +0000 onLeave at the beginning.
-[D****LicenseController lastTrialExpiryDate] return: nil onLeave at the beginning.
-[D****LicenseController lastPurchaseExpiryDate] return: nil onLeave at the beginning.
-[D****LicenseController lastValidation] return:  2024-10-08 09:07:46 +0000 onLeave at the beginning.
-[D****LicenseController lastTrialExpiryDate] return: nil onLeave at the beginning.
-[D****LicenseController lastPurchaseExpiryDate] return: nil onLeave at the beginning.
-[D****LicenseController lastStage] return: 0x1f4 onLeave at the beginning.
```

当进入应用程序开启试用后，会发现 lastTrialExpiryDate 的时间值会是在当前时刻的一个小时后。继续 Hook lastPurchaseExpiryDate 方法，并将其返回值篡改为一个未来几十年后的时间值。

```javascript
// __handlers__/D****LicenseController/lastPurchaseExpiryDate.js

defineHandler({
  onEnter(log, args, state) {},

  onLeave(log, retval, state) {
    var NSDate = ObjC.classes.NSDate;
    var timestamp = 4092599349; // 2099-09-09 09:09:09
    var endDate = NSDate.dateWithTimeIntervalSince1970_(timestamp);

    retval.replace(endDate);

    var retval1 = new ObjC.Object(retval);
    log(`-[D****LicenseController lastPurchaseExpiryDate] return: ` + retval1);
  },
});
```

打开程序观察了下，发现这个操作对 APP 没任何影响。那没办法，只能继续分析-[D****LicenseController stage]后续的操作。不过，鉴于代码量小大，就不逐行一一分析了。

往下看到 LABEL_75，其中对 lastTrialExpiryDate 进行了试用期过期判断，如果如果还在试用期内，则将 510LL 赋给 v61，而 stage 最终返回的就是 v61。这个应该不会陌生，因为在上面恰恰就是这么操作以达到无限试用的。

```objective-c
LABEL_75:
    v62 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastTrialExpiryDate](v2, "lastTrialExpiryDate"));
    if ( v62
      && (v63 = v62,
          v64 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastTrialExpiryDate](v2, "lastTrialExpiryDate")),
          -[NSDate timeIntervalSinceNow](v64, "timeIntervalSinceNow"),
          v66 = v65,
          objc_release(v64),
          objc_release(v63),
          v66 >= 0.0) )
    {
      v67 = 0LL;
      v61 = 510LL;
    }
    else
    {
      v67 = 0LL;
      v61 = 500LL;
    }
    goto LABEL_83;

	  // ...
```

继续跳转到 LABEL_83，其中有对 v61 进行数值判断，如果等于 560，会进一步计算 lastPurchaseExpiryDate 是否大于 604800.0 秒（7 天）。

```objective-c
LABEL_83:
  v68 = objc_retainAutoreleasedReturnValue(+[NSUserDefaults standardUserDefaults](&OBJC_CLASS___NSUserDefaults, "standardUserDefaults"));
  v69 = objc_retainAutoreleasedReturnValue(-[NSUserDefaults objectForKey:](v68, "objectForKey:", CFSTR("D****LicenseMessageEndDate")));
  v70 = -[D****LicenseController lastStage](v2, "lastStage");
  -[D****LicenseController setLastStage:](v2, "setLastStage:", v61);
  v71 = objc_retainAutoreleasedReturnValue(+[NSDate date](&OBJC_CLASS___NSDate, "date"));
  -[D****LicenseController setLastValidation:](v2, "setLastValidation:", v71);
  objc_release(v71);
  if ( v61 == 560 )
  {
    v72 = objc_retainAutoreleasedReturnValue(-[D****LicenseController lastPurchaseExpiryDate](v2, "lastPurchaseExpiryDate"));
    -[NSDate timeIntervalSinceNow](v72, "timeIntervalSinceNow");
    if ( v73 >= 604800.0 )
    {
      objc_release(v72);
      goto LABEL_91;
    }
    // ...
```

这里的逻辑似乎是在订阅临近到期前 7 天提醒用户，不过这并不是重点。重点在于，我们能够确定当处于订阅期（或阶段）时，-[D****LicenseController stage]的返回值 v61 会等于 560。

前面通过篡改-[D****LicenseController stage]的返回值为 510 以达到无限试用，在那个时候所根据的提示是-[D****LicenseController stageDescription]这个方法中的内容。

```objective-c
id __cdecl -[D****LicenseController stageDescription](D****LicenseController *self, SEL a2)
{
  __int64 v2; // x0

  v2 = -[D****LicenseController stage](self, "stage");
  if ( v2 <= 539 )
  {
    if ( v2 == 500 )
      return CFSTR("Free");
    if ( v2 == 510 )
      return CFSTR("Trial");
  }
  else
  {
    switch ( v2 )
    {
      case 540LL:
        return CFSTR("Past Trial");
      case 550LL:
        return CFSTR("Past");
      case 560LL:
        return CFSTR("Current");
    }
  }
  return CFSTR("Unknown");
}
```

在-[D****LicenseController stageDescription]中确实是有个 560LL 的，但当时不理解 Current 所代表的寓意是什么，这个单词翻译成中文只是“当前”的意思，与订阅完全谈不上搭边，所以那个时候并未留意 560LL。现在根据-[D****LicenseController stage]中的代码逻辑，才判断出当-[D****LicenseController stage]返回 560 时，即为应用程序处于订阅阶段。

对于-[D****LicenseController stageDescription]这个类方法，我认为它不过只是一个辅助性的方法，不参与核心业务逻辑，其存在的作用大概只是为了提升代码的可读性与可维护性以及调试便利性。之前觉得开发者很良心，现在从这里还能看出开发者在规范化开发上也很用心，但由于开发者的这种“用心”，使得这款软件极容易被破解。然而，对于一般的 APP，破解起来未必就有这么好的运气，大量需要深入分析的代码或汇编依旧是逃不开的，该碰的壁、该踩的坑也是躲不过的。

最终破解后的效果图，所有高级功能均可正常使用，创建无限个数的 Breaks 也是没问题的，在应用程序左下角也显示了「奖励现已永久可用」。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/cf515a91-a87e-4349-9b73-36aa5f0a44f6/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241021%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241021T023341Z&X-Amz-Expires=3600&X-Amz-Signature=8daae399a80525c63491bb9eda74557829cdc677bdc75e18e520d43f76ac877b&X-Amz-SignedHeaders=host&x-id=GetObject)
