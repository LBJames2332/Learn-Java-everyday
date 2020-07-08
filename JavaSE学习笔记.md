# JavaSE学习笔记

### java环境搭建

1.https://jdk.java.net下载jdk

2.以管理员身份运行命令行

3.cd到jdk文件目录

4.生成jre文件

> bin\jlink.exe --module-path jmods --add-modules java.desktop --output jre 

# 重点知识

## ThreadLocal源码剖析（小部分，有机会深究剩下的）

### 遗留问题

TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);

具体是怎样清除过期entry的呢？

* 关于怎么算hash的？

```java
private final int threadLocalHashCode = nextHashCode();
//ThreadLocal的成员变量threadLocalHashCode即这个ThreadLocal的hashcode
//在实例化的时候则调用nextHashCode()方法赋值完毕
//因为是final修饰的，所以不可更改，即唯一化了
private static AtomicInteger nextHashCode = new AtomicInteger();
//这个方法会返回当前的类变量(一个AtomicInteger)增加delta之后的值，所以理论上相当均匀
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

* get

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);//拿到thread的threadlocals
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        /**根据key(ThreadLocal)去threadlocals(一个ThreadLocalMap)中寻找value
          *计算index
          *若==则返回，否则向后找，直到为null or ==
          */
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;//找到，则返回entry的value
            return result;
        }
    }
    return setInitialValue();//
}
```

```java
private T setInitialValue() {
    T value = initialValue();//初始为null
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);//若threadlocals不为null，则初始化value为null
    } else {
        createMap(t, value);//若threadlocals为null则create(懒初始化)
    }
    //这一段存疑(不知道什么时候会走这里)
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}
```

* set

```java
private void set(ThreadLocal<?> key, Object value) {
    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);//计算index

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) 
    /**
      *循环向后找table的每个槽(slot)上的entry
      *若为null，表示这个位置是空，说明可以放在这儿
      *或者找到key==this的，则更新这个entry的value
      */
    {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);//替换过期项(Key为null的项)
            return;
        }
    }
	//到这儿的时候i位置slot为null，则我们选这个slot
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        /**
          *扫描log2(sz)次数，看看是否有value为null的不为null的slot(过期的家伙)
          *若不能清除任何，且长度超过阈值，则rehash
          */
        rehash();
}
```

* 下一个index

```java
/**
 * Increment i modulo len.
 */
private static int nextIndex(int i, int len) {//其实就是对index+1，若超过len则回0
    return ((i + 1 < len) ? i + 1 : 0);
}
```

* rehash

```java
/**
 * Re-pack and/or re-size the table. First scan the entire
 * table removing stale entries. If this doesn't sufficiently
 * shrink the size of the table, double the table size.
 */
private void rehash() {
    expungeStaleEntries();//清除所有过期的entry

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)//若size大于槽位的3/4，则resize
        resize();
}
```

