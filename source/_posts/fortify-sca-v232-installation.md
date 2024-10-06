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
cover: /img/post/fortify-sca-v232-installation/scanned.png
id: 10f906e1-7468-8023-bfba-d00b32c23243
---

## Fortify SCA 简介

Fortify SCA，全称为 Fortify Static Code Analyzer，是一款静态代码分析工具，可以帮助开发人员在软件开发过程中发现和修复安全漏洞。通过对源代码进行深度扫描，Fortify SCA 可以准确地识别出各种类型的安全隐患，如 SQL 注入、跨站脚本等漏洞。同时，它也提供了强大的报告和审计功能，使得开发团队可以有效地管理和跟踪安全问题。这款软件广泛应用于各种规模的软件开发项目中，被业界公认为静态代码分析领域的领导者。

## 资源寻找

利用 Google Hacking 手段找到了一个旧版本的破解包，并根据相关信息追查到破解包都是来自一个 Telegram 频道。

```text
(intext:"pan.baidu.com" OR intext:"t.me")  AND intext:"Fortify_SCA_23"
```

![](/img/post/fortify-sca-v232-installation/google-hacking.png)

这个电报频道在 24 年 1 月 14 日发布了较新版本的破解包。

![](/img/post/fortify-sca-v232-installation/telegram.png)

相关文件下载链接如下，全部下载至本地并解压。

```
SCA: 
https://ponies.cloud/source_code_analysis/fortifySCA/win/Fortify_SCA_23.2.0_Windows.zip

Tools: 
https://ponies.cloud/source_code_analysis/fortifySCA/win/Fortify_Tools_23.2.0_Windows.zip

Crack & License file (Password: Pwn3rzs):
https://ponies.cloud/source_code_analysis/fortifySCA/Fortify_SCA_23.2_Crack_pwn3rzs_cyberarsenal.7z

Rules:
https://ponies.cloud/source_code_analysis/fortifySCA/FortifyRules_2023.3.0.0006_en.zip
FortifyRules_zh_CH_2023.1.1.0001(离线规则库): https://mega.nz/file/6rwggQJD#OKgMxNHTqCvbGYxDtPcq6XPEpkVnymSR9qvfvFo6QMk
```

## 安装过程

Fortify SCA v23.2.0 破解版的安装过程相对简单，由于破解包来历不明，所以为了安全考虑，建议将其安装在一个虚拟机中，做好隔离。

### Fortify SCA 安装

首先，先安装 Fortify_SCA_23.2.0_Windows 文件夹中的 Fortify_SCA_23.2.0_windows_x64.exe。

![](/img/post/fortify-sca-v232-installation/sca-installation.png)

点击下一步，并同意协议，再一路 Next，直到出现如下图所示选择 Fortify License 文件位置时，选择 Fortify_SCA_23.2_Crack_pwn3rzs_cyberarsenal 文件夹中的 fortify.license 文件，继续下一步。

![](/img/post/fortify-sca-v232-installation/sca-license.png)

然后就会出现 LIM License 界面，选择默认选项 No，后续也是一路 Next，最后等待安装。

如果在安装过程中出现如下报错，不用管就行，之后 Fortify_SCA_23.2.0 便将安装完毕。

![](/img/post/fortify-sca-v232-installation/update-failed.png)

![](/img/post/fortify-sca-v232-installation/sca-installed.png)

### Fortify Apps and Tools 安装

第二个安装的就是 Fortify_Tools_23.2.0_Windows 文件夹中的 Fortify_Apps_and_Tools_23.2.0_windows_x64.exe。

过程与 SCA 类似，一路默认下一步，到了选择 License 界面，就选择如上同样的 fortify.license 文件。

![](/img/post/fortify-sca-v232-installation/apps-license.png)

继续一路下一步，等待安装结束。

![](/img/post/fortify-sca-v232-installation/apps-installed.png)

### 替换 Jar 文件

现在，便需要将 Fortify_SCA_23.2_Crack_pwn3rzs_cyberarsenal 文件夹中的 fortify-common-23.2.0.0023.jar 文件复制至`C:\Program Files\Fortify\Fortify_Apps_and_Tools_23.2.0\Core\lib\`、`C:\Program Files\Fortify\Fortify_SCA_23.2.0\Core\lib\`两个目录中。

![](/img/post/fortify-sca-v232-installation/replace.png)

## 规则库

最后，将下载的规则库文件解压，再将其中的 ExternalMetadata 和 rules 目录复制至`C:\Program Files\Fortify\Fortify_SCA_23.1.0\Core\config\`中。

![](/img/post/fortify-sca-v232-installation/rule.png)

如果不习惯英文，则可以选择 FortifyRules_zh_CH_2023.1.1.0001(离线规则库)。

## 使用

运行`C:\Program Files\Fortify\Fortify_Apps_and_Tools_23.2.0\bin\`目录中的 auditworkbench.cmd，即可开启图形化界面。

如下图，正在对一个 Java 项目进行扫描中。

![](/img/post/fortify-sca-v232-installation/scanning.png)

扫描完成后如下图展示所示。

![](/img/post/fortify-sca-v232-installation/scanned.png)