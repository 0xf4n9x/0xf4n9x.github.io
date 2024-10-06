---
created: 2024-10-02T13:38:00+00:00
categories:
  - 技术研究
tags:
  - Java安全
updated: 2023-01-24T00:00:00+00:00
date: 2022-10-15T00:00:00+00:00
slug: java-apps-remote-debug
title: Java程序远程调试
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ef36867b-fcd0-414e-85ff-ed1b0066a71d/idea-debug.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=34ddd3fab4acddca5b5a153fd617416b0d90714f08203ce947f9a7ab3f7093f3&X-Amz-SignedHeaders=host&x-id=GetObject
id: 113906e1-7468-80c2-b92d-fe765b25375c
---

## Jar 包

针对 Jar 包的远程调试，就拿 Behinder 举例，首先 IDEA 新建一个项目，并在项目中创建一个 lib 文件夹，将 Behinder 的 Jar 包放入其中，右击该 Jar 包选择作为库添加（Add as Library...），对弹出的窗口点击 OK。现在就能看到 Behinder Jar 包中反编译后的源码。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ce0bdee8-26cc-4ffc-bdef-ed24a8f89dbe/behinder-source-code.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=fc77062f3c955592daa260b2230aabc792ccf22e3c10a39a06f9d97da028d746&X-Amz-SignedHeaders=host&x-id=GetObject)

下一步，点击上图右上边的编辑配置（Add Configurations...），进入到如下窗口，单击左上角的+，选择 Remote JVM Debug，修改下名称，其他默认保持不变，不过需要注意端口冲突，如下第二张图所示。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/89a6fdd4-48c6-4ffd-8761-2feb7ac7b44b/add-remote-jvm-debug.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=e6b940d27f14e1ab08f0458d55a20c75e5b88808fd3b1bb4eedc61ff6dfb7c09&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/81447636-4e17-4c27-95e4-bc084580da1b/behinder-debug-conf.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=09a3753b614d6689cec4c6408145ab5d152e92cd8b231c20855e355ec774139a&X-Amz-SignedHeaders=host&x-id=GetObject)

最后，将上图中的参数添加至运行命令中，不过需要注意的一点是将 suspend 参数值修改为 y，它表示是否暂停程序等待调试器的连接。最终，使用如下命令将 Behinder 启动起来。

```shell
# ls                                                                  ~/SecTools/Behinder_v3.0_Beta_11.t00ls
Behinder.jar     data.db          server           更新日志.txt
# java -jar -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 Behinder.jar
Listening for transport dt_socket at address: 5005
```

回到 IDEA 中，将断点打至如下行，再点击右上角 Debug 绿色按钮，便可对 Behinder 进行远程调试了。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/cacf83cf-1e2d-4c6c-a224-4c3d4bff3194/behinder-debug.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=16beb64b46d998c452ce7dc7dd7cea54f6013c8db903b56251d32a30d02d9a83&X-Amz-SignedHeaders=host&x-id=GetObject)

## 虚拟机漏洞环境

一些漏洞环境需要我们自行搭建，将其安装在 VMware 虚拟机中，以方便本地对靶场进行漏洞复现。在这种情况下又该如何对其调试，下文以泛微 Ecology 为例，演示如何对虚拟机中的漏洞环境进行远程调试。

首先打开虚拟机，Ecology 默认安装在`C:\\WEAVER\\`路径下，Ecology 使用的 Web 服务器为 Resin，其配置文件路径位于`C:\\WEAVER\\Resin\\conf\\resin.properties`，打开该文件并找到`jvm_args`，添加如下内容，其中 5005 表示的是端口。

```text
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/52e519b7-753d-4080-bc19-91d38fc0977d/resin.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=e3245527f062cce15232c780a567cc4285e7e0569c1a35419c23f144b7f4d103&X-Amz-SignedHeaders=host&x-id=GetObject)

> 注意：如果 Web 服务器使用的是 Tomcat，那么只需将如下行添加至 Tomcat 配置文件中，配置文件位于 tomcat/bin/catalina.sh，对于 Linux，可能在/usr/local/目录下；对于 Windows，取决于用户自行存放的位置。

    ```text
    Java_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005"
    ```

保存更变，重启 Resin 服务。

接着，将虚拟机中 Ecology 的整个安装目录（`C:\\WEAVER\\`）全部拷贝到宿主机系统中，并使用 IDEA 打开 Ecology 源码项目（`C:\\WEAVER\\ecology`），同时 IDEA 中的 Java 版本要与虚拟机中的保持一致，初次打开项目会更新索引，需要等待一段时间。

在新增 Run/Debug Configurations 前，先安装[Resin 插件](https://plugins.jetbrains.com/plugin/14598-resin)，之后在窗口左上角进行新增，选择 Resin 下的 Remote，便出现如下配置界面，先修改 Host 为虚拟机 IP 地址，Port 为 5005 调试端口。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/562b4ccd-520a-4ad0-b9fc-1bb26fdb06e9/resin-debug-conf.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=3f60d9c356553b415b60c1adaad3d9dd6f26fa89ed5ab355fcf34863822681db&X-Amz-SignedHeaders=host&x-id=GetObject)

对 Application server 进行如下配置。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/651d63f6-2b0a-4054-83e8-573799dc99f6/resin-version.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=4c93c0c20838b6ac0d29550104d458575f8cb518ca7f320dcbfa71ac37b58eee&X-Amz-SignedHeaders=host&x-id=GetObject)

转到 Startup/Connection，将 Debug 下方的 Port 修改为 5005 端口，最后点击 OK。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c907bc6a-5235-411b-abd6-e5319afadf83/resin-debug-conf-startup.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=7474242e2f58861ce93a899cc531f4ed8d4460a990694c25fda0f2abba8e7dc1&X-Amz-SignedHeaders=host&x-id=GetObject)

如上配置完毕后，便可以打断点了，将断点打到`weaver.security.filter#SecurityMain`方法，使用浏览器访问一个 URL，就可以进行调试了。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/b7e2ecaf-b543-4305-9dd0-9943b83a1509/ecology-debug.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=82d7fad61b2eeaab7e41e3ec3656b88e34b8413786395fbdc998210e4fddaaa4&X-Amz-SignedHeaders=host&x-id=GetObject)

## Docker 容器靶场

譬如 Vulhub 这样的开源漏洞靶场，利用 Docker 一键启动漏洞靶场环境，省去了手动搭建环境的繁琐，极其地方便。下文以其中的 Shiro CVE-2016-4437 漏洞环境为例，介绍如何对 Java 类型的 Docker 容器靶场进行远程调试。

首先，该漏洞环境的源码其实就存放于 Vulhub 项目中的 base 目录，用 IDEA 打开`base/shiro/1.2.4/code`目录中的项目，先在 Project Structure 中将 JDK 版本设置为 8，接着新增 Run/Debug Configurations。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/16ef61e1-e5f2-44ec-a821-49bf4cd744d8/add-debug-configurations.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=05c9fe30135c77c49508e02a4e4a5c21eccbe1680f27ae4630938d6c6c5d8138&X-Amz-SignedHeaders=host&x-id=GetObject)

Host 填写靶场的 IP，但不过由于是 Docker 容器，所以 localhost 也是可行的，Port 填写一个不被占用的端口。JDK 版本选择 8，如上命令行参数在后续将会用到。

下一步，先在 docker-compose.yml 中新增一组暴露端口的配置，该端口将用于后续的远程调试通信。

```shell
# cat docker-compose.yml
version: '2'
services:
 web:
   image: vulhub/shiro:1.2.4
   ports:
    - "8080:8080"
    - "5005:5005"
```

然后，启动容器。

```shell
# docker compose up -d
```

进入容器，查看靶场环境的启动命令，如下 PID 为 1 的进程，正如开头对 Behinder 进行远程调试一样，都是一个 Jar 包启动的。

```shell
# docker exec -it 5fa8722e9d13 /bin/bash
root@5fa8722e9d13:/# ps -ax
  PID TTY      STAT   TIME COMMAND
    1 ?        Ssl    0:52 /usr/bin/java java -jar /shirodemo-1.0-SNAPSHOT.jar
   40 pts/0    Ssl    0:00 /bin/bash /bin/bash
   47 ?        Rl+    0:00 ps -ax
```

这样我们可以继续修改 docker-compose.yml 文件，添加上边 IDEA 中的命令行参数（suspend 参数值修改为 y），在启动容器时替换默认的命令。修改后，重新启动容器即可。

```shell
# cat docker-compose.yml
version: '2'
services:
 web:
   image: vulhub/shiro:1.2.4
   ports:
    - "8080:8080"
    - "5005:5005"
   command: /usr/bin/java -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 -jar /shirodemo-1.0-SNAPSHOT.jar
# docker compose up -d
```

接下来，还需配置依赖，先将容器中的 Jar 拷贝出来，并作为库添加。

```shell
# docker cp c0475028d3b2:/shirodemo-1.0-SNAPSHOT.jar ../../base/shiro/1.2.4/code
Successfully copied 22.3MB to /Users/r00t/SEC/vulnEnv/vulhub/base/shiro/1.2.4/code
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/64566a36-f8d9-4885-9990-c0f619667dca/idea-add-as-lib.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=a49b3e2d37d5859581cde55d7ad16538c004b74db91f7098b91057c4bbf3d84b&X-Amz-SignedHeaders=host&x-id=GetObject)

最后，在 IDEA 项目中创建一个 lib 目录，将 shirodemo-1.0-SNAPSHOT.jar 里 BOOT-INF 中的 lib 下所有文件拷贝至 IDEA 项目中的 lib 目录里面，并将拷贝出来的所有 jar 作为库添加。

至此，就可以打断点进行远程调试了。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/e42fdc10-445e-4c7c-8f3c-a55f0fa0a9fc/idea-debug.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=dbfea914ca8fe5367e94d2c4b69b015ef6c1d99070ea0194a47362bce226ee40&X-Amz-SignedHeaders=host&x-id=GetObject)

## WebLogic

WebLogic 是一个 Jave2E 应用服务器，不同于 Tomcat 服务器那么地轻量。下文将以 Vulhub 中的 CVE-2017-10271 漏洞环境为例，展示如何对 WebLogic 进行配置以达到远程调试。

首先还是修改 docker-compose.yml 文件，新增一组暴露端口的配置，该 8453 端口为 WebLogic 默认的调试端口号。

```shell
# cat docker-compose.yml
version: '2'
services:
 weblogic:
   image: vulhub/weblogic:10.3.6.0-2017
   ports:
    - "7001:7001"
    - "8453:8453"
```

启动并进入到容器中。

```shell
# docker compose up -d
# docker ps
CONTAINER ID   IMAGE                           COMMAND              CREATED          STATUS          PORTS                                                                                            NAMES
2cec5c91ab51   vulhub/weblogic:10.3.6.0-2017   "startWebLogic.sh"   47 minutes ago   Up 47 minutes   0.0.0.0:5005->5005/tcp, :::5005->5005/tcp, 0.0.0.0:7001->7001/tcp, :::7001->7001/tcp, 5556/tcp   cve-2017-10271_weblogic_1
# docker exec -it 2cec5c91ab51 /bin/bash
root@2cec5c91ab51:~/Oracle/Middleware#
```

修改 WebLogic 的 setDomainEnv.sh 文件，该文件位于`/root/Oracle/Middleware/user_projects/domains/base_domain/bin/`目录下，在如下图中的位置添加如下两行内容。

```text
debugFlag="true"
export debugFlag
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/97b13c52-4f79-47b6-9408-4d03f298ee69/weblogic-debugflag.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=8580c88439fe07ca3b8de1fb248dbbc46b15a40a6bc2a3a77f07e81cc3885a81&X-Amz-SignedHeaders=host&x-id=GetObject)

退出容器，并重新启动容器。

```shell
# docker restart 2cec5c91ab51
2cec5c91ab51
```

再次进入容器，将/root/Oracle/Middleware 目录下的 modules 文件夹和 wlserver_10.3 文件夹复制出来到一个新目录。

```shell
# docker exec -it 2cec5c91ab51 /bin/bash
root@2cec5c91ab51:~/Oracle/Middleware# tar -czvf modules.tar.gz modules/ && tar -czvf wlserver_10.3.tar.gz wlserver_10.3/
root@2cec5c91ab51:~/Oracle/Middleware# exit
# docker cp 2cec5c91ab51:/root/Oracle/Middleware/modules.tar.gz source/
# docker cp 2cec5c91ab51:/root/Oracle/Middleware/wlserver_10.3.tar.gz source/
# ls source/ && cd source/
modules.tar.gz  wlserver_10.3.tar.gz
# tar -zxf modules.tar.gz && tar -zxf wlserver_10.3.tar.gz
```

使用 IDEA 打开 source 目录，并将 modules 文件夹与 w1server_10.3/server/lib 文件夹作为库添加。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c9e0d68c-6356-49e6-b58b-6f90cd83f52f/weblogic-addaslib.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=e116a11cd6b350e2efbea75801f878598e59c5f54fb820e00058bbe908d3af93&X-Amz-SignedHeaders=host&x-id=GetObject)

新增如下 Debug 配置。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/f1502f8e-7be8-4759-b551-2c2dc6d07a5a/weblogic-debug-conf.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=c77d487b0bff04c34cc869ba4347bab147c9e92b99eff023ad3573c4fb2f61d4&X-Amz-SignedHeaders=host&x-id=GetObject)

将断点打在`wlserver_10.3/server/lib/weblogic.jar!/weblogic/wsee/jaxws/WLSServletAdapter#handle`方法，并点击 Debug 绿色按钮，使用浏览器访问http://127.0.0.1:7001/wls-wsat/CoordinatorPortType，如下图，成功进行调试。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/57e94ab8-5fe9-4190-a39f-0afb0f92c550/weblogic-debuging.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=65600c6c6cb91b6a1997bb1f5a1de588a75350dadcf5809b604948f00225bc39&X-Amz-SignedHeaders=host&x-id=GetObject)
