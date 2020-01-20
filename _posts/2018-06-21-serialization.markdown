---

layout:     post
title:      "序列化相关问题"
subtitle:   ""
date:       2018-06-21
author:     "Chenxs"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Java
    - Serialization
---

# 序列化相关问题

### transient
```
transient Set<Map.Entry<K,V>> entrySet;
```
序列化有两种方式
* 1.实现Serializable接口(隐式序列化) 
* 2.实现Externalizable接口。(显式序列化)

transient, static 不会被序列化
使用场景：
1.信息敏感
2.不使用的信息，不占用空间（比如session 中生效的时间，只有读到的时候才生成）


### 定制序列化方式
重写writeObject,readObject
比如hashMap（原因：比如在不同虚拟机上，entry的排序等会不同，所以不序列化，另外写入和读取）

``` java
    private void writeObject(java.io.ObjectOutputStream s)
        throws IOException {
        int buckets = capacity();
        // Write out the threshold, loadfactor, and any hidden stuff
        s.defaultWriteObject();
        s.writeInt(buckets);
        s.writeInt(size);
        internalWriteEntries(s);
    }
```

``` java
    private void readObject(java.io.ObjectInputStream s)
        throws IOException, ClassNotFoundException {
        // Read in the threshold (ignored), loadfactor, and any hidden stuff
        s.defaultReadObject();
        reinitialize();
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new InvalidObjectException("Illegal load factor: " +
                                             loadFactor);
        s.readInt();                // Read and ignore number of buckets
        int mappings = s.readInt(); // Read number of mappings (size)
        if (mappings < 0)
            throw new InvalidObjectException("Illegal mappings count: " +
                                             mappings);
```


