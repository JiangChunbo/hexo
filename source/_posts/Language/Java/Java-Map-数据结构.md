---
title: Java Map 数据结构
date: 2022-11-09 22:00:49
tags:
---


# HashMap


## 数据结构


底层数据结构：数组 + 单向链表

```java
Node<K,V>[] table;
```

```java
static class Node<K,V> implements Map.Entry<K,V> {
    // 该 Node 的 hash 值
    final int hash;
    final K key;
    V value;
    // 下一个节点
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```


> 解决哈希冲突的方法: 开放定址法、再哈希法、链地址法等。参见[这里](http://c.biancheng.net/view/3437.html)



## 如何计算 Node.hash

将 `key.hashCode()` 的返回值经过特定的扰动得到 hash 值。

JDK 7 的 hash 计算:

```java
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
JDK 8 的 hash 计算: 
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```


## 如何得到元素位置

```java
(n - 1) & hash
```

## 扩容的时机

1. 构造之后，第一次 `put`。由于 `table` 是 `null` 所以会进行第一次 `resize()`

```java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

2. `put` 之后如果当前 `size` 大于 `threshold`，进行 `resize()`


3. 调用 `treeifyBin()`，如果 `table` 的长度小于 `MIN_TREEIFY_CAPACITY`，即 64，则放弃


> tips: bin 表示大容器。


## HashMap 的 table 长度为什么是 2 的幂次方

一般考虑要用 hash 值做对 `table.size` 的取模运算得到余数，该余数才是存放的索引。

取余(%)运算，如果除数是 2 的幂次方，等价于其除数减一的与(&)运算。也就是:

$hash \% length == hash\&(length-1)$

考虑到采用 `&` 运算，相对于 `%` 效率更高，因此 HashMap 的长度是 2 的幂次方。