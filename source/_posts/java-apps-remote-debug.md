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
cover: /img/post/java-apps-remote-debug/idea-debug.png
id: 113906e1-7468-80c2-b92d-fe765b25375c
---

## Jar 包

针对 Jar 包的远程调试，就拿 Behinder 举例，首先 IDEA 新建一个项目，并在项目中创建一个 lib 文件夹，将 Behinder 的 Jar 包放入其中，右击该 Jar 包选择作为库添加（Add as Library...），对弹出的窗口点击 OK。现在就能看到 Behinder Jar 包中反编译后的源码。

![](/img/post/java-apps-remote-debug/behinder-source-code.png)

下一步，点击上图右上边的编辑配置（Add Configurations...），进入到如下窗口，单击左上角的+，选择 Remote JVM Debug，修改下名称，其他默认保持不变，不过需要注意端口冲突，如下第二张图所示。

![](/img/post/java-apps-remote-debug/add-remote-jvm-debug.png)

![](/img/post/java-apps-remote-debug/behinder-debug-conf.png)

最后，将上图中的参数添加至运行命令中，不过需要注意的一点是将 suspend 参数值修改为 y，它表示是否暂停程序等待调试器的连接。最终，使用如下命令将 Behinder 启动起来。

```shell
# ls
Behinder.jar     data.db          server           更新日志.txt
# java -jar -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 Behinder.jar
Listening for transport dt_socket at address: 5005
```

回到 IDEA 中，将断点打至如下行，再点击右上角 Debug 绿色按钮，便可对 Behinder 进行远程调试了。

![](/img/post/java-apps-remote-debug/behinder-debug.png)

## 虚拟机漏洞环境

一些漏洞环境需要我们自行搭建，将其安装在 VMware 虚拟机中，以方便本地对靶场进行漏洞复现。在这种情况下又该如何对其调试，下文以泛微 Ecology 为例，演示如何对虚拟机中的漏洞环境进行远程调试。

首先打开虚拟机，Ecology默认安装在`C:\WEAVER\`路径下，Ecology使用的Web服务器为Resin，其配置文件路径位于`C:\WEAVER\Resin\conf\resin.properties`，打开该文件并找到`jvm_args`，添加如下内容，其中5005表示的是端口。

```
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=5005
```

![](/img/post/java-apps-remote-debug/resin.png)

> 注意：如果 Web 服务器使用的是 Tomcat，那么只需将如下行添加至 Tomcat 配置文件中，配置文件位于 tomcat/bin/catalina.sh，对于 Linux，可能在/usr/local/目录下；对于 Windows，取决于用户自行存放的位置。
>
> ```
> Java_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005"
> ```

保存更变，重启 Resin 服务。

接着，将虚拟机中Ecology的整个安装目录（`C:\WEAVER\`）全部拷贝到宿主机系统中，并使用IDEA打开Ecology源码项目（`C:\WEAVER\ecology`），同时IDEA中的Java版本要与虚拟机中的保持一致，初次打开项目会更新索引，需要等待一段时间。

在新增 Run/Debug Configurations 前，先安装[Resin 插件](https://plugins.jetbrains.com/plugin/14598-resin)，之后在窗口左上角进行新增，选择 Resin 下的 Remote，便出现如下配置界面，先修改 Host 为虚拟机 IP 地址，Port 为 5005 调试端口。

![](/img/post/java-apps-remote-debug/resin-debug-conf.png)

对 Application server 进行如下配置。

![](/img/post/java-apps-remote-debug/resin-version.png)

转到 Startup/Connection，将 Debug 下方的 Port 修改为 5005 端口，最后点击 OK。

![](/img/post/java-apps-remote-debug/resin-debug-conf-startup.png)

如上配置完毕后，便可以打断点了，将断点打到`weaver.security.filter#SecurityMain`方法，使用浏览器访问一个 URL，就可以进行调试了。

![](/img/post/java-apps-remote-debug/ecology-debug.png)

## Docker 容器靶场

譬如 Vulhub 这样的开源漏洞靶场，利用 Docker 一键启动漏洞靶场环境，省去了手动搭建环境的繁琐，极其地方便。下文以其中的 Shiro CVE-2016-4437 漏洞环境为例，介绍如何对 Java 类型的 Docker 容器靶场进行远程调试。

首先，该漏洞环境的源码其实就存放于 Vulhub 项目中的 base 目录，用 IDEA 打开`base/shiro/1.2.4/code`目录中的项目，先在 Project Structure 中将 JDK 版本设置为 8，接着新增 Run/Debug Configurations。

![](/img/post/java-apps-remote-debug/add-debug-configurations.png)

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

![](/img/post/java-apps-remote-debug/idea-add-as-lib.png)

最后，在 IDEA 项目中创建一个 lib 目录，将 shirodemo-1.0-SNAPSHOT.jar 里 BOOT-INF 中的 lib 下所有文件拷贝至 IDEA 项目中的 lib 目录里面，并将拷贝出来的所有 jar 作为库添加。

至此，就可以打断点进行远程调试了。

![](/img/post/java-apps-remote-debug/idea-debug.png)

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

```
debugFlag="true"
export debugFlag
```

![](/img/post/java-apps-remote-debug/weblogic-debugflag.png)

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

![](/img/post/java-apps-remote-debug/weblogic-addaslib.png)

新增如下 Debug 配置。

![](/img/post/java-apps-remote-debug/weblogic-debug-conf.png)

将断点打在`wlserver_10.3/server/lib/weblogic.jar!/weblogic/wsee/jaxws/WLSServletAdapter#handle`方法，并点击 Debug 绿色按钮，使用浏览器访问http://127.0.0.1:7001/wls-wsat/CoordinatorPortType，如下图，成功进行调试。

![](/img/post/java-apps-remote-debug/weblogic-debuging.png)
