---
created: 2024-09-28T11:16:00+00:00
categories:
  - 技术研究
tags:
  - 代码审计
updated: 2024-03-07T00:00:00+00:00
date: 2024-03-07T00:00:00+00:00
slug: fortify-sca-v232-installation
title: Fortify SCA v23.2.0破解版安装小记
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d9c94dd8-4e21-4af7-a16f-e2de36426efa/scanned.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=6cb8a45b47768e8dd83bbc5a425ed2a08fec11b46b515bb96a6f83f5bea5f9a3&X-Amz-SignedHeaders=host&x-id=GetObject
id: 10f906e1-7468-8023-bfba-d00b32c23243
---

## Fortify SCA 简介

Fortify SCA，全称为 Fortify Static Code Analyzer，是一款静态代码分析工具，可以帮助开发人员在软件开发过程中发现和修复安全漏洞。通过对源代码进行深度扫描，Fortify SCA 可以准确地识别出各种类型的安全隐患，如 SQL 注入、跨站脚本等漏洞。同时，它也提供了强大的报告和审计功能，使得开发团队可以有效地管理和跟踪安全问题。这款软件广泛应用于各种规模的软件开发项目中，被业界公认为静态代码分析领域的领导者。

## 资源寻找

利用 Google Hacking 手段找到了一个旧版本的破解包，并根据相关信息追查到破解包都是来自一个 Telegram 频道。

```text
 (intext:"pan.baidu.com" OR intext:"t.me")  AND intext:"Fortify_SCA_23"
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f5272e38-a55c-48b6-a45d-88d88cb5bf72/google-hacking.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=af39b534b09c31b7ac98b69cef1a1d2fa4b37b53061744c67850b631a3adc68a&X-Amz-SignedHeaders=host&x-id=GetObject)

这个电报频道在 24 年 1 月 14 日发布了较新版本的破解包。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/243da515-de30-459e-9338-8288f37fa03d/telegram.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=3ea384bbd00d698a0d7a3efab722e79be975d8da6d482908c6f3390751012035&X-Amz-SignedHeaders=host&x-id=GetObject)

相关文件下载链接如下，全部下载至本地并解压。

```text
 SCA:
 https://ponies.cloud/source_code_analysis/fortifySCA/win/Fortify_SCA_23.2.0_Windows.zip
 ​
 Tools:
 https://ponies.cloud/source_code_analysis/fortifySCA/win/Fortify_Tools_23.2.0_Windows.zip
 ​
 Crack & License file (Password: Pwn3rzs):
 https://ponies.cloud/source_code_analysis/fortifySCA/Fortify_SCA_23.2_Crack_pwn3rzs_cyberarsenal.7z
 ​
 Rules:
 https://ponies.cloud/source_code_analysis/fortifySCA/FortifyRules_2023.3.0.0006_en.zip
 FortifyRules_zh_CH_2023.1.1.0001(离线规则库): https://mega.nz/file/6rwggQJD#OKgMxNHTqCvbGYxDtPcq6XPEpkVnymSR9qvfvFo6QMk
```

## 安装过程

Fortify SCA v23.2.0 破解版的安装过程相对简单，由于破解包来历不明，所以为了安全考虑，建议将其安装在一个虚拟机中，做好隔离。

### Fortify SCA 安装

首先，先安装 Fortify_SCA_23.2.0_Windows 文件夹中的 Fortify_SCA_23.2.0_windows_x64.exe。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/1484baf1-ae2c-4403-9986-5b631733cbac/sca-installation.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=cb0329dd428958573e72b6ef07a46f00bc677d35e03fe356309ed6d1efab1b1f&X-Amz-SignedHeaders=host&x-id=GetObject)

点击下一步，并同意协议，再一路 Next，直到出现如下图所示选择 Fortify License 文件位置时，选择 Fortify_SCA_23.2_Crack_pwn3rzs_cyberarsenal 文件夹中的 fortify.license 文件，继续下一步。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/33662505-dd0d-47a5-b7a3-5c3f0724a773/sca-license.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=b405c90170d88dfd32271e6261f798b7c1c4abf967f73816f0fe923091752c23&X-Amz-SignedHeaders=host&x-id=GetObject)

然后就会出现 LIM License 界面，选择默认选项 No，后续也是一路 Next，最后等待安装。

如果在安装过程中出现如下报错，不用管就行，之后 Fortify_SCA_23.2.0 便将安装完毕。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/40295890-0187-431d-b0a6-d681720b4aee/update-failed.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=bc39516c99d323b9f76175ae03128ed5e352f69c18787863cd37a7411622a554&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/96262f09-fbb0-42f9-9275-bb65eb574fb0/sca-installed.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=ee87dda5b20dab720d162805e56dbdc8f9792c6f9fd131531197eaf00669b884&X-Amz-SignedHeaders=host&x-id=GetObject)

### Fortify Apps and Tools 安装

第二个安装的就是 Fortify_Tools_23.2.0_Windows 文件夹中的 Fortify_Apps_and_Tools_23.2.0_windows_x64.exe。

过程与 SCA 类似，一路默认下一步，到了选择 License 界面，就选择如上同样的 fortify.license 文件。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/8fb5c3fd-86e0-4eaa-b6fb-3cf824553fc2/apps-license.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=e40ea9c660e2ecaac6a79de6c4dd59bfae4e214156c23d0fcccc46bf14e6a39e&X-Amz-SignedHeaders=host&x-id=GetObject)

继续一路下一步，等待安装结束。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/2cb24d7e-eca3-4f36-ad81-f9c4c5779a85/apps-installed.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=c5cd050fd6c8065749a0e34e61f53ddb4f1dd77058ac9b492cdbbe365edf486f&X-Amz-SignedHeaders=host&x-id=GetObject)

### 替换 Jar 文件

现在，便需要将 Fortify_SCA_23.2_Crack_pwn3rzs_cyberarsenal 文件夹中的 fortify-common-23.2.0.0023.jar 文件复制至`C:\Program Files\Fortify\Fortify_Apps_and_Tools_23.2.0\Core\lib\`、`C:\Program Files\Fortify\Fortify_SCA_23.2.0\Core\lib\`两个目录中。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ea0bb00f-a583-4601-94a2-00131931875f/replace.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=0575f3f0ca79af636d2d522a73dfd52af155cc4d11b947e7d196182cad26d7fb&X-Amz-SignedHeaders=host&x-id=GetObject)

## 规则库

最后，将下载的规则库文件解压，再将其中的 ExternalMetadata 和 rules 目录复制至`C:\Program Files\Fortify\Fortify_SCA_23.1.0\Core\config\`中。

如果不习惯英文，则可以选择 FortifyRules_zh_CH_2023.1.1.0001(离线规则库)。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/da97d259-d6f4-4b83-b578-42555122eef4/rule.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=eee09fe3c23062e168311edcf00064d74d2790f43a6ec3fa7ce00fcdce911f91&X-Amz-SignedHeaders=host&x-id=GetObject)

## 使用

运行`C:\Program Files\Fortify\Fortify_Apps_and_Tools_23.2.0\bin\`目录中的 auditworkbench.cmd，即可开启图形化界面。

如下图，正在对一个 Java 项目进行扫描中。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/4c332183-f475-4acb-ab1e-bee37e6cabe6/scanning.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=03a7bb991d847b305e15571b2a9beb7003ec56d439fe4a01f99a1849bb9b67cf&X-Amz-SignedHeaders=host&x-id=GetObject)

扫描完成后如下图展示所示。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ba5ddf13-5c87-45d2-86bf-df04732048ac/scanned.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=be9f1cdc97b105274ce3bf8a50027de8e648ab8773cc88a277e0bbf2029ab611&X-Amz-SignedHeaders=host&x-id=GetObject)
