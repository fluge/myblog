---
title: 并发系列(一) Java7的ConcurrentHashMap 
date: 2017-1-18 18:30:18
categories: 
- Java
tags: 
- Java
---
从[HashMap并发的死循环](https://fluge.github.io/2016/12/15/HashMap%E5%B9%B6%E5%8F%91%E7%9A%84%E6%AD%BB%E5%BE%AA%E7%8E%AF/)可以知道,Hashmap是没办法在多线程的情况下使用的，为了解决这个问题，在Java4之前用的是hashtable,只是现在不推荐的。在Java5之后就比较推荐使用java.util.concurrent.ConcurrentHashMap，这个在多线程的情况下，也能有很好的性能。从这里引入了Java里面一类很重要的概念---并发。先解决完上一个问题。高并发下ConcurrentHashMap的结构。
### 并发的一些初步了解--synchronized和volatile  
在多线程的并发的情况下有安全的访问变量，为了解决这个问题引入一个机制---锁机制。让多线程不能同时访问一个共享变量。在并发过程中有需要简单的了解两个东西的含义。
#### Java中的synchronized的简单分析  
`synchronized`的用法要弄清晰一个问题:`synchronized`锁住的是代码还是对象？
首先是一个被`synchronized`修饰的代码块
<!--more-->

```java
 private static int count;
    public SyncThread() {
        count = 0;
    }
    public  void run() {
        synchronized(this) {
            for (int i = 0; i < 3; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ":" + (count++));
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```
在来看两段程序，这个概念可以清晰很多

```java
//第一段代码
SyncThread syncThread = new SyncThread();
Thread thread1 = new Thread(syncThread, "Thread A");
Thread thread2 = new Thread(syncThread, "Thread B");
thread1.start();
thread2.start();

//第二段代码
SyncThread syncThread1 = new SyncThread();
SyncThread syncThread2 = new SyncThread();
Thread thread1 = new Thread(syncThread1, "Thread A");
Thread thread2 = new Thread(syncThread1, "Thread B");
thread1.start();
thread2.start();
```
这两段代码执行的结果:

```java
//第一段代码的执行结果，两个线程依次顺序执行
Thread A:0 
Thread A:1 
Thread A:2  
Thread B:3 
Thread B:4 
Thread B:5 
//第二段代码的执行结果，两个线程轮流执行
Thread A:0 
Thread B:1 
Thread A:2 
Thread B:3 
Thread A:4 
Thread B:5 
```
第一个结果是A，B两个线程按照锁的方式，依次执行。第二个结果是两个线程不受锁的控制交替执行，为什么会出现这个情况呢？主要是因为第一段代码中`线程A和线程B`都是访问`syncThread`这个一个对象，必须按照获得锁的顺序执行。但是在第二段代码中`线程A`访问的是`syncThread1`,`线程B`访问的是`syncThread2`,`线程A执行的是syncThread1对象中的synchronized代码(run)`,线程B一样。这就可以知道`synchronized`锁住的是对象，这时会有两把锁分别锁定syncThread A对象和syncThread B对象，而这两把锁是互不干扰的，不形成互斥，所以两个线程可以同时执行。 
`synchronized`是一种同步锁它修饰的对象有以下几种： 
1. 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，上文的例子就是代码块，作用的对象是调用这个代码块的对象； 
2. 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象； 
3. 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个`类`。  
 
无论synchronized关键字加在方法上还是对象上，如果它作用的对象是非静态的，则它取得的锁是对象；如果synchronized作用的对象是一个静态方法或一个类，则它取得的锁是对类，该类所有的对象同一把锁。每个对象只有一个锁（lock）与之相关联，谁拿到这个锁谁就可以运行它所控制的那段代码
#### Java中的volatile的简单分析
Volatile是轻量级的synchronized，它在多处理器开发中保证了`共享变量`的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。
想要彻底的理解`volatile`就必须理解Java的内存模型这个会在下一篇里文章讲到。关于`volatile`要知道就是每当线程要访问一个被volatile修饰的变量时都会从内存中直接拉取，而不会从缓存中获取这个变量的值。
### Hashtable和----已经淘汰的遗留并发的HashMap  
简单说一下Hashtable和HashMap的区别：HashMap是非synchronized的适合在单线程下使用，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。Hashtable由于方法是由synchronized修饰的。可以在并发的情况下进行使用，只不过效率不高不建议使用。
#### Hashtable源码的简单分析  
Hashtable源码和HashMap差不多。先看`put()`方法

```java
 public synchronized V put(K key, V value) {
         // Hashtable中不能插入value为null的元素！！！    
        if (value == null) {
            throw new NullPointerException();
        }
        // “Hashtable中已存在键为key的键值对”,则用value替换    
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
        addEntry(hash, key, value, index);
        return null;
    }
private void addEntry(int hash, K key, V value, int index) {
        //若“Hashtable中不存在键为key的键值对”，将“修改统计数”+1    
        modCount++;
        //若“Hashtable实际容量” > “阈值”(阈值=总的容量 * 加载因子)则扩容   
        Entry<?,?> tab[] = table;
        if (count >= threshold) {
            rehash();
            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }
         //将新的key-value对插入到tab[index]处（即链表的头结点）.
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```
从源码看基本和HashMap差不多。解决哈希冲突的方法一样。但是不允许为null的键值对。`get()`都差不多就不分析了。  
#### Hashtable淘汰的原因
HashTable容器使用synchronized来保证线程安全,但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法时,其他线程访问HashTable的同步方法时,可能会进入阻塞或轮询状态。如线程1使用put进行添加元素,线程2不但不能使用put方法添加元素,并且也不能使用get方法来获取元素,所以竞争越激烈效率越低。  
HashTable容器在竞争激烈的并发环境下表现出效率低下的原因是所有访问HashTable的线程都必须竞争`同一把锁`,那假如容器里有多把锁,每一把锁用于锁容器其中一部分数据,那么当多线程访问容器里不同数据段的数据时,线程间就不会存在锁竞争,从而可以有效的提高并发访问效率,这就是ConcurrentHashMap所使用的`锁分段技术`,首先将数据分成一段一段的存储，然后给每一段数据配一把锁,当一个线程占用锁访问其中一个段数据的时候,其他段的数据也能被其他线程访问。  
### Java7的ConcurrentHashMap的结构分析和锁分段技术  
现在在Java8优化了Java7的的锁分段技术。取消了segment和Java8的Hashmap的优化一样。现在看Java7的锁分段技术，毕竟还是很有思考价值的。Java8的ConcurrentHashMap分析会在另一篇博文里面  
Java7的ConcurrentHashMap的基本结构图,这个可以很清晰的认识到ConcurrentHashMap得内部存储结构。这个和HashMap的结构还是有很大差距的。不过有一点不会变的是:`两者的本质都是数组和链表的结合`
![](http://ofa8x9gy9.bkt.clouddn.com/Java7%E7%9A%84ConcurrentHashMap.png)
Java7的ConcurrenHashMap类中有两个静态内部类`HashEntry`和`Segment`。HashEntry 用来封装映射表的键值对;Segment用来充当锁的角色,每个`Segment`对象守护整个散列映射表的若干个桶。每个桶是由若干个 `HashEntry对象链接起来的链表`。一个ConcurrentHashMap实例中包含由若干个Segment对象组成的数组。Segment 在某些意义上有点类似于HashMap了，都是包含了一个数组，而数组中的元素可以是一个链表。
#### HashEntry类  
```java
 static final class HashEntry<K,V> { 
        final K key;                 // 声明 key 为 final 型
        final int hash;              // 声明 hash 值为 final 型 
        volatile V value;            // 声明 value 为 volatile 型
        final HashEntry<K,V> next;   // 声明 next 为 final 型 

        HashEntry(K key, int hash, HashEntry<K,V> next, V value) { 
            this.key = key; 
            this.hash = hash; 
            this.next = next; 
            this.value = value; 
        } 
 }
```
在这个里面需要注意的是  
1. `key`,`hash`,`next节点`都被声明为final型,这就意味着,发生了哈希冲突后,新来的节点只能插在头结点。而且链表原来的结构也没有被改变，插入新健 / 值对到链表中的操作不会影响读线程正常遍历这个链表。
2. `value`被声明为volatile变量。用volatile来保证多个线程对数据的可见性。就为`get()`不加锁打下基础。  

#### Segment类
```java
 static final class Segment<K,V> extends ReentrantLock implements Serializable { 
        //在本 segment 范围内，包含的 HashEntry 元素的个数 
        transient volatile int count; 
        //table 被更新的次数 
        transient int modCount; 
        //当 table 中包含的 HashEntry 元素的个数超过本变量值时，触发 table 的再散列
        transient int threshold;
        /** 
         * table 是由 HashEntry 对象组成的数组
         * 如果散列时发生碰撞，碰撞的 HashEntry 对象就以链表的形式链接成一个链表
         * table 数组的数组成员代表散列映射表的一个桶
         * 每个 table 守护整个 ConcurrentHashMap 包含桶总数的一部分
         * 如果并发级别为 16，table 则守护 ConcurrentHashMap 包含的桶总数的 1/16 
         */ 
        transient volatile HashEntry<K,V>[] table; 
        //装载因子 
        final float loadFactor; 
        Segment(int initialCapacity, float lf) { 
            loadFactor = lf; 
            setTable(HashEntry.<K,V>newArray(initialCapacity)); 
        } 
        /** 
         * 设置 table 引用到这个新生成的 HashEntry 数组
         * 只能在持有锁或构造函数中调用本方法
         */ 
        void setTable(HashEntry<K,V>[] newTable) { 
            // 计算临界阀值为新数组的长度与装载因子的乘积
            threshold = (int)(newTable.length * loadFactor); 
            table = newTable; 
        } 
        //根据 key 的散列值，找到 table 中对应的那个桶（table 数组的某个数组成员） 
        HashEntry<K,V> getFirst(int hash) { 
            HashEntry<K,V>[] tab = table; 
        // 把散列值与 table 数组长度减 1 的值相“与”，得到散列值对应的 table 数组的下标然后返回 table 数组中此下标对应的 HashEntry 元素
            return tab[hash & (tab.length - 1)]; 
        } 
 }
```
在这个里面需要注意的是：
1. `Segment`类是继承ReentrantLock类,这就是为了让Segment对象可以充当锁。每个Segment对象用来守护其(成员对象 table中)包含的若干个桶。
2. `table`是一个由`HashEntry对象`组成的数组。table数组的每一个数组成员就是散列映射表的一个桶。
3. `count`变量是一个计数器，它表示每个`Segment对象`管理的table数组(若干个HashEntry组成的链表)包含的HashEntry对象的个数。并且是`volatile`变量,所以当需要更新计数器时,不用锁定整个ConcurrentHashMap
#### 分段锁实现并发下的put()操作
```java
    public V put(K key, V value) { 
        if (value == null)          //ConcurrentHashMap中不允许用null作为映射值
            throw new NullPointerException(); 
        int hash = hash(key.hashCode());        // 计算键对应的散列码
        // 根据散列码找到对应的 Segment 
        return segmentFor(hash).put(key, hash, value, false); 
 }

     //使用 key 的散列码来得到 segments 数组中对应的 Segment 
    final Segment<K,V> segmentFor(int hash) { 
    // 将散列值右移 segmentShift 个位，并在高位填充 0 
    // 然后把得到的值与 segmentMask 相“与”
    // 从而得到 hash 值对应的 segments 数组的下标值
    // 最后根据下标值返回散列码对应的 Segment 对象
        return segments[(hash >>> segmentShift) & segmentMask]; 
 }
    //在 Segment 中执行具体的 put 操作
    V put(K key, int hash, V value, boolean onlyIfAbsent) { 
            lock();  // 加锁，这里是锁定某个Segment对象而非整个ConcurrentHashMap 
            try { 
                int c = count; 
                if (c++ > threshold) {    // 如果超过再散列的阈值,执行再散列，table 数组的长度将扩充一倍
                    rehash();  
                }
                HashEntry<K,V>[] tab = table; 
                // 把散列码值与 table 数组的长度减 1 的值相“与”
                // 得到该散列码对应的 table 数组的下标值
                int index = hash & (tab.length - 1); 
                // 找到散列码对应的具体的那个桶
                HashEntry<K,V> first = tab[index]; 
                HashEntry<K,V> e = first; 
                while (e != null && (e.hash != hash || !key.equals(e.key))) {
                    e = e.next; 
                }
                V oldValue; 
                if (e != null) {            // 如果键 / 值对以经存在
                    oldValue = e.value; 
                    if (!onlyIfAbsent) {
                        e.value = value;    // 设置 value 值
                    }
                } 
                else {                        // 键 / 值对不存在 
                    oldValue = null; 
                    ++modCount;         // 要添加新节点到链表中，所以 modCont 要加 1  
                    // 创建新节点，并添加到链表的头部 
                    tab[index] = new HashEntry<K,V>(key, hash, first, value); 
                    count = c;               // 写 count 变量
                } 
                return oldValue; 
            } finally { 
                unlock();                     // 解锁
            } 
        }
```
在这个里面需要注意的是:这里的`加锁操作`是针对(键的hash值对应的)`某个具体的Segment`,锁定的是该Segment而不是整个ConcurrentHashMap。因为插入键值对的操作只是在这个Segment包含的某个桶中完成,不需要锁定整个ConcurrentHashMap。此时,其他写线程对另外15个Segment的加锁并不会因为当前线程对这个Segment的加锁而阻塞。同时,所有读线程几乎不会因本线程的加锁而阻塞,除非读线程刚好读到这个Segment中某个HashEntry的value域的值为null,此时需要加锁后重新读取该值。  
### 小小的总结  
这次的并发下的优化的具体方向是根据一些试用场景优化的:除了少数插入操作和删除操作外，绝大多数都是读取操作，而且读操作在大多数时候都是成功的。原来的Hashtable的`synchronized`直接加锁的方式,会在`put()`操作的时候同时会阻塞其他线程的`get()`操作。ConcurrentHashMap就多次采用`volatile`变量来解决变量在`JMM(Java内存模型)`中对其他线程可见性。这也可以使`volatile`对`synchronized`锁的优化。  
用分段锁去优化`synchronized`的`put()`操作的阻塞。就在于`减少多个线程对同一个锁的请求频率`和`减少线程对锁的持有时间`,就是减小锁的细粒度来优化锁的阻塞。


----  
参考:
[Java中Synchronized的用法](http://blog.csdn.net/luoweifu/article/details/46613015)
[探索 ConcurrentHashMap 高并发性的实现机制](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/)
[ConcurrentHashMap 的实现原理](http://wiki.jikexueyuan.com/project/java-collection/concurrenthashmap.html)