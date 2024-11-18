---
created: 2024-11-03T12:20:00+00:00
categories:
  - 技术研究
description: 利用网络层面的中间人攻击劫持应用与服务器端之间的通信，通过篡改数据包、伪造验证响应来实现破解，这一招式在技术水平上虽见不得有多高，但却十分实用，且在当前iOS上用的相当普遍，尤其是在现今越狱变得越来越困难的情况下。
tags:
  - 破解
updated: 2024-11-03T00:00:00+00:00
date: 2024-11-02T00:00:00+00:00
slug: cracking-ai-app-by-mitm-attack
title: 利用中间人攻击破解某基于ChatGPT的第三方AI应用
cover: /img/post/cracking-ai-app-by-mitm-attack/cover.png
id: 133906e1-7468-8011-8df1-c9615dc528f4
---

## 前言

最近一段时期，利用 AI 来辅助工作与学习变得越来越频繁，确切地说，使用 ChatGPT GPT-4o 模型的次数越来越多了。由此也出现了两个问题，一是 OpenAI 官方对于普通用户每日可用 GPT-4o 模型的次数有限。第二个问题也是最主要的问题，则是国内用户访问 ChatGPT 极受网络环境的影响，一般的代理根本没法访问，即使是优质的、自建的代理，在对话几次后，后续每对话一次都需要刷新一次网页来验证自身身份，使得在使用上用户体验感极差。

终于，以上限制我无法继续忍受，尤其是受网络环境所导致的极差用户体验。于是就找到了一款基于 ChatGPT 并集成了 GPT-3.5、GPT-4o mini 和 GPT-4o 模型的第三方 AI 应用，据此应用官网的介绍，其用户数量超过六百万，那功能和用户体验感想必自有保证。在使用过程中，发现该应用的会员订阅验证机制是基于第三方的应用内订阅平台，而非苹果官方平台。如此一来，该应用便被我顺利破解。虽然破解过程十分简单，按理说本不足为道，但鉴于许多应用采用的都是这种验证机制，都可基于类似的手段来一一破解，因此此应用的破解便颇具代表性，遂将这一过程记录成文。

## 免责声明

本文所有内容仅用于学术研究、技术探讨及教育学习的目的，旨在帮助读者加深对软件安全及逆向工程的理解。文中描述的技术与方法并非鼓励或支持任何未经授权的行为，亦不应将本文内容用于任何未经授权的目的。软件破解及相关技术的使用需遵守各国法律法规，任何个人或机构在实践时应对自身行为负责，作者不承担因不当使用本文内容所引发的任何责任。

## 订阅验证通信流程分析

在正式分析前，不妨先想想，当我们对应用的高级功能付完费完成订阅后，后续每次打开应用，应用是如何得知高级功能已经订阅了的？基于这个疑问，再作出进一步的思考，应用是否会在网络层面进行订阅状态的验证？以及具体是如何验证的？整个通信流程又是如何的？

有了猜想，于是我们就可以想到通过类似 Proxifier 这样的代理链工具，将此 AI 应用的全部 HTTP 流量强制转到 BurpSuite，以监控该应用在完整通信流程中所产生的全部 HTTP 流量。同时，在 BurpSuite 端开启请求与响应的劫持，后续可能需要篡改响应数据。

![](/img/post/cracking-ai-app-by-mitm-attack/image.png)

此时，再打开这个 AI 应用，回到 BurpSuite 中便可观察到如下 GET 请求，请求的域名为 api.revenuecat.com。

```text
GET /v1/subscribers/$RCAnonymousID%3A697911394ea640868b8a5d99e47f1266/offerings HTTP/1.1
Host: api.revenuecat.com
X-Platform-Flavor: native
User-Agent: *%20*/1046 CFNetwork/1404.0.5 Darwin/22.3.0
X-Revenuecat-Etag: 6145a6a0eaaba93a
X-Client-Build-Version: 1046
X-Client-Bundle-Id: com.*.*
X-Version: 4.41.2
X-Client-Version: 1.3
X-Platform-Version: Version 13.2.1 (Build 22D68)
X-Platform: macOS
X-Observer-Mode-Enabled: false
Authorization: Bearer appl_gQpRDGQpplyrTBswHvTlWdSsPMU
X-Storefront: USA
Accept-Language: en-US,en;q=0.9
Accept: */*
Content-Type: application/json
X-Is-Sandbox: false
X-Storekit2-Enabled: false
Accept-Encoding: gzip, deflate, br
Connection: keep-alive


```

用 Google 搜一下 RevenueCat，就能得知这其实是一个用于管理应用内订阅的移动 SDK 和 API**，**为 Apple App Store 和 Google Play Store 复杂的订阅系统提供支持。也就是我在前面提到的第三方应用内订阅平台。

将如上面的请求放行，看看会收到什么响应。如下，响应状态码为 304 未修改，似乎与缓存有关。经过测试，如果这里不作处理，原封不动的将响应返回给 AI 应用程序，那后续将什么都不会发生。

```text
HTTP/1.1 304 Not Modified
Date: Sun, 03 Nov 2024 07:19:20 GMT
Connection: keep-alive
server: envoy
access-control-allow-origin: *
access-control-expose-headers: X-Request-Id
x-revenuecat-etag: 6145a6a0eaaba93a
x-revenuecat-request-time: 1730618360809
x-signature: EIXmUGI5fWDSlDvDjpbSoJJiF3JsCZLtMZ/NUtwEat92TgAAqe6Ra7vFrglbnzvqC/uPP6D4697CxKTbwUAR+tQqVxKSm8/ZUCl88V/fBNjXgaMbGUuWtjI4WW5HLkzEgp1DBsTEBxHqURnNi8dW6PSHfzMTnvRkdTyU6jpjVUnPN8TKqAdfLJSd20mctOXP5n0QThXTrXZqlYZdd5VuQiaJowKbkyugWwQV+xLWyNeceUEL
x-amzn-trace-id: Root=1-672723f8-11fa0d5f46ca89c4611ca65a
x-envoy-upstream-service-time: 16
vary: Accept-Encoding
x-request-id: badbd6c7-0439-447c-af24-1637931cbf08


```

所以，这里需要篡改删除响应中的 x-revenuecat-etag 标头（相当于清除缓存）并放行，使 AI 应用程序接收到该响应，这样才会有后续的请求发生。

![](/img/post/cracking-ai-app-by-mitm-attack/image%201.png)

不出所料，AI 应用程序发起了一个同样的请求，但却接收到了不一样的响应数据。仔细观察请求包中的 X-Revenuecat-Etag 标头，可推断出当这个 HTTP 标头的值为空时，才不会受缓存的影响，才能收到正常的响应数据。

![](/img/post/cracking-ai-app-by-mitm-attack/image%202.png)

```json
{
  "current_offering_id": "default",
  "offerings": [
    {
      "description": "AI Plus standard offer",
      "identifier": "default",
      "metadata": null,
      "packages": [
        {
          "identifier": "$rc_monthly",
          "platform_product_identifier": "aiplus_monthly"
        },
        {
          "identifier": "$rc_annual",
          "platform_product_identifier": "ai_plus_gpt_yearly"
        },
        {
          "identifier": "$rc_weekly",
          "platform_product_identifier": "ai_plus_chatgpt"
        }
      ]
    }
    // ...省略部分无关紧要的字段
  ],
  "placements": {
    "fallback_offering_id": "default"
  }
}
```

进一步地，查阅 RevenueCat 官方 API 文档，得知/v1/subscribers/{app_user_id}/offerings 这个 API 接口是用于获取应用程序的报价信息。例如，如上响应中的 platform_product_identifier 字段表示商店中报价的标识符。根据字面意思来推测，ai_plus_chatgpt 可能代表按周订阅，aiplus_monthly 或许代表按月订阅，ai_plus_gpt_yearly 应该代表按年订阅。

继续回到 BurpSuite，观察后续的通信流量，发现应用程序又发起了一个新请求，且其中存在 X-Revenuecat-Etag 标头，尝试先放行请求。

```text
GET /v1/subscribers/$RCAnonymousID%3A697911394ea640868b8a5d99e47f1266 HTTP/1.1
Host: api.revenuecat.com
X-Platform-Flavor: native
User-Agent: *%20*/1046 CFNetwork/1404.0.5 Darwin/22.3.0
X-Revenuecat-Etag: 1db63f2915741c25
X-Client-Build-Version: 1046
X-Client-Bundle-Id: com.*.*
X-Version: 4.41.2
X-Client-Version: 1.3
X-Platform-Version: Version 13.2.1 (Build 22D68)
X-Observer-Mode-Enabled: false
X-Platform: macOS
Authorization: Bearer appl_gQpRDGQpplyrTBswHvTlWdSsPMU
X-Storefront: USA
Accept-Language: en-US,en;q=0.9
Accept: */*
Content-Type: application/json
X-Is-Sandbox: false
X-Storekit2-Enabled: false
Accept-Encoding: gzip, deflate, br
Connection: keep-alive


```

随后，又收到了一个 304 状态码且带 x-revenuecat-etag 标头的空响应，依旧按照上面的处理方式，去除 x-revenuecat-etag 标头并放行，使 AI 应用程序接收到经过篡改的响应。

![](/img/post/cracking-ai-app-by-mitm-attack/image%203.png)

不出意料，AI 应用程序又发起了一个同样的请求，但响应有所不同了，见如下。

```json
{
  "request_date": "2024-11-03T07:20:36Z",
  "request_date_ms": 1730618436398,
  "subscriber": {
    "entitlements": {},
    "first_seen": "2024-11-01T08:32:51Z",
    "last_seen": "2024-11-03T06:04:46Z",
    "management_url": null,
    "non_subscriptions": {},
    "original_app_user_id": "$RCAnonymousID:697911394ea640868b8a5d99e47f1266",
    "original_application_version": null,
    "original_purchase_date": "2024-11-01T07:41:16Z",
    "other_purchases": {},
    "subscriptions": {}
  }
}
```

放行此响应后，发现后续再无相关请求出现，这表明经过此请求与相应后，订阅验证流程已结束，如上几个请求与响应便是订阅验证的一个全流程。

## 中间人攻击客户端验证

经过了上面对订阅验证流程的流量分析，大概就能够推断出最后一个请求响应就是关键所在。

继续查阅 RevenueCat 官方 API 文档，见如下链接：

[https://www.revenuecat.com/docs/api-v1#tag/customers/operation/subscribers](https://www.revenuecat.com/docs/api-v1#tag/customers/operation/subscribers)

![](/img/post/cracking-ai-app-by-mitm-attack/image%204.png)

根据文档所述，/v1/subscribers/{app_user_id}接口是一个获取或创建客户的 API，作用是使用给定的 App User ID 获取客户的最新客户信息，如果不存在，则创建新客户。同时，官方还给出了一个响应示例，见如下 JSON。

```json
{
  "request_date": "2019-07-26T17:40:10Z",
  "request_date_ms": 1564162810884,
  "subscriber": {
    "entitlements": {
      "pro_cat": {
        "expires_date": null,
        "grace_period_expires_date": null,
        "product_identifier": "onetime",
        "purchase_date": "2019-04-05T21:52:45Z"
      }
    },
    "first_seen": "2019-02-21T00:08:41Z",
    "management_url": "https://apps.apple.com/account/subscriptions",
    "non_subscriptions": {},
    "original_app_user_id": "XXX-XXXXX-XXXXX-XX",
    "original_application_version": "1.0",
    "original_purchase_date": "2019-01-30T23:54:10Z",
    "other_purchases": {},
    "subscriptions": {
      "annual": {
        "auto_resume_date": null,
        "billing_issues_detected_at": null,
        "expires_date": "2019-08-14T21:07:40Z",
        "grace_period_expires_date": null,
        "is_sandbox": true,
        "original_purchase_date": "2019-02-21T00:42:05Z",
        "ownership_type": "PURCHASED",
        "period_type": "normal",
        "purchase_date": "2019-07-14T20:07:40Z",
        "refunded_at": null,
        "store": "app_store",
        "unsubscribe_detected_at": "2019-07-17T22:48:38Z"
      }
    }
  }
}
```

将官方提供的这份响应示例与我们的响应相对比，可发现最主要的差别在于后者缺失了 subscriber.entitlements 和 subscriber.subscriptions 字段内容。

![](/img/post/cracking-ai-app-by-mitm-attack/image%205.png)

参照官方提供的响应示例，将缺失的相关字段补充上，部分字段的值可参考请求 offerings 接口所获取的报价信息。然后重启应用以重来一遍订阅验证流程，直到接收到最后一个请求的响应时，劫持并篡改替换为如下内容。

```json
{
  "request_date": "2024-11-03T07:20:36Z",
  "request_date_ms": 1730618436398,
  "subscriber": {
    "entitlements": {
      "AI Plus": {
        "product_identifier": "ai_plus_gpt_yearly",
        "purchase_date": "2024-11-01T00:00:00Z",
        "expires_date": "2099-09-09T00:00:00Z"
      }
    },
    "first_seen": "2024-11-01T08:32:51Z",
    "last_seen": "2024-11-03T06:04:46Z",
    "management_url": null,
    "non_subscriptions": {},
    "original_app_user_id": "$RCAnonymousID:697911394ea640868b8a5d99e47f1266",
    "original_application_version": null,
    "original_purchase_date": "2024-11-01T07:41:16Z",
    "other_purchases": {},
    "subscriptions": {
      "ai_plus_gpt_yearly": {
        "expires_date": "2099-09-09T00:00:00Z",
        "original_purchase_date": "2024-11-01T00:00:00Z",
        "purchase_date": "2024-11-01T00:00:00Z",
        "ownership_type": "PURCHASED",
        "store": "app_store"
      }
    }
  }
}
```

![](/img/post/cracking-ai-app-by-mitm-attack/image%206.png)

此刻，回到 AI 应用中，可发现应用中已显示“享受无限制的使用”，即表明破解成功。

![](/img/post/cracking-ai-app-by-mitm-attack/image%207.png)

总结下破解的两个关键。一是，当请求包中出现了 X-Revenuecat-Etag 标头值，需要去除，否则服务器端会认定客户端存在缓存，从而不返回任何响应数据。第二个关键，在服务器端返回客户的订阅信息到客户端之前，需要将此数据篡改为已订阅的数据，从而欺骗到位于客户端中的订阅验证机制。

## iOS 端应用流量重写

以上破解针对的是 macOS 平台上的 AI 应用程序，而在 iOS 上恰好有此应用的同款应用，且 iOS 上的一些常见代理程序（如小火箭等）也都是支持 MITM 以及脚本重写流量的。

```javascript
/*******************************
[Script]
ai* = type=http-response, pattern=^https:\/\/api\.revenuecat\.com\/.+\/(receipts$|subscribers\/?(.*?)*$), script-path=ai*.js, requires-body=true, timeout=60
ai* = type=http-request, pattern=^https:\/\/api\.revenuecat\.com\/.+\/(receipts$|subscribers\/?(.*?)*$), script-path=ai*.js, timeout=60

[MITM]
hostname = %APPEND% api.revenuecat.com
*******************************/
let obj = {};

if (typeof $response == "undefined") {
  delete $request.headers["X-RevenueCat-ETag"];
  obj.headers = $request.headers;
} else {
  let body = JSON.parse(
    (typeof $response != "undefined" && $response.body) || null
  );

  if (body && body.subscriber) {
    let subscriber = body.subscriber;

    subscriber.entitlements["AI Plus"] = {
      product_identifier: "ai_plus_gpt_yearly",
      expires_date: "2999-01-01T00:00:00Z",
      purchase_date: "2024-11-01T00:00:00Z",
    };
    subscriber.subscriptions["ai_plus_gpt_yearly"] = {
      expires_date: "2999-01-01T00:00:00Z",
      original_purchase_date: "2024-11-01T00:00:00Z",
      purchase_date: "2024-11-01T00:00:00Z",
      ownership_type: "PURCHASED",
      store: "app_store",
    };

    obj.body = JSON.stringify(body);
  }
}

$done(obj);
```

那么，根据前面总结的破解的两个关键，编写如上 JavaScript 脚本，并将脚本导入至小火箭，配置如下两条规则，以及开启 HTTPS 解密。

![](/img/post/cracking-ai-app-by-mitm-attack/image%208.png)

现在，打开 iOS 端应用，即可发现应用被成功破解。

![](/img/post/cracking-ai-app-by-mitm-attack/image%209.jpeg)

## 漏洞修复

经过相关查阅最终确认下来，这个问题的造成主要责任在于 RevenueCat，其在设计之初未优先考虑到安全风险，没有想到会受到 MITM 的攻击。好在 2023 年 7 月份，RevenueCat 对这一问题做出了解决。因此，对于此类攻击的防范，首先需要升级 SDK 至 4.25.0 版本及以上（iOS 平台）以启用 Trusted Entitlements 功能；其次还需正确地配置此功能的模式，因为至少就目前而言此功能即使启用了，但在默认配置下依旧是形同虚设，并不能真正防范中间人攻击，具体详细操作请参见下面这篇文章。

<div style="width: 100%; margin-top: 4px; margin-bottom: 4px;"><div style="display: flex; background:white;border-radius:5px"><a href="https://www.revenuecat.com/blog/engineering/trusted-entitlements/"target="_blank"rel="noopener noreferrer"style="display: flex; color: inherit; text-decoration: none; user-select: none; transition: background 20ms ease-in 0s; cursor: pointer; flex-grow: 1; min-width: 0px; flex-wrap: wrap-reverse; align-items: stretch; text-align: left; overflow: hidden; border: 1px solid rgba(55, 53, 47, 0.16); border-radius: 5px; position: relative; fill: inherit;"><div style="flex: 4 1 180px; padding: 12px 14px 14px; overflow: hidden; text-align: left;"><div style="font-size: 14px; line-height: 20px; color: rgb(55, 53, 47); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; min-height: 24px; margin-bottom: 2px;">Trusted Entitlements Secures Your App's IAPs from MiTM Piracy</div><div style="font-size: 12px; line-height: 16px; color: rgba(55, 53, 47, 0.65); height: 32px; overflow: hidden;">Explore how RevenueCat's Trusted Entitlements fortify your app's in-app purchases against MiTM attacks using cryptography.</div><div style="display: flex; margin-top: 6px; height: 16px;"><img src="https://www.revenuecat.com/favicon-32x32.png?v=983d81cd061310b2eb379250c70bcb88"style="width: 16px; height: 16px; min-width: 16px; margin-right: 6px;"><div style="font-size: 12px; line-height: 16px; color: rgb(55, 53, 47); white-space: nowrap; overflow: hidden; text-overflow: ellipsis;">https://www.revenuecat.com/blog/engineering/trusted-entitlements/</div></div></div><div style="flex: 1 1 180px; display: block; position: relative;"><div style="position: absolute; inset: 0px;"><div style="width: 100%; height: 100%;"><img src="https://www.revenuecat.com/static/b5b6e09bc44183007d8c919b3c4c6046/9585e/trusted-entitlements.jpg" referrerpolicy="no-referrer" style="display: block; object-fit: cover; border-radius: 3px; width: 100%; height: 100%;"></div></div></div></a></div></div>

## 写在最后

原本，没有什么是花钱不能解决的，开头那两个问题，钱花够了自然就能轻松解决。但对于我这种不爱走寻常路的人来说，显然是不会选择常规手段来解决问题。倒不是本人抠抠搜搜不舍得花钱，遇到好用的东西和该支持的开源项目，我都是毫不吝啬地选择付费赞助支持的。只是我觉得比起常规手段，不如让人过过破解的瘾、享受享受 Hacking 的乐趣。

不过，虽破解了这款应用，但快乐也仅在破解成功后的那一瞬间，在未来我不会继续非法使用这款应用，主要是有两方面的考虑。一是如果长期利用破解手段来非法使用这款应用，所花的真金白银会由该应用所属的公司来承担，这对这家初创公司的伤害比较大。第二个原因则是在学习方面，AI 虽能帮助使用者快速获取“What”型问题的答案，但欲速则不达，对于深入学习、深层次理解“How”与“Why”型问题，仍需要系统性训练和自主思考，基础知识也特别重要。更何况，足够复杂的问题，AI 也无能为力，甚至还可能会出错。
