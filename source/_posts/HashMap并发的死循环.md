 ---
title: HashMap并发的死循环
date: 2016-12-15 17:52:18
categories: 
- Java
tags: 
- Java
---
HashMap从设计上来说就不适合在并发的情况的下使用,因为HashMap每次在`put()`时，总会检查一遍对应桶的容量，如果桶满了，或者超过了设定的值，就会`reserve()`来进行扩容,然后通过`get()`来取出相应的值。这个过程在单线程下是没什么问题的，但是如果在并发的条件下，多个线程同时reserve桶，然后有线程这个时候执行`get()`就有可能产生死循环，造成CPU的100%占用，具体等会看源码。在Java里面有一个很老的hashtable就是加了锁的HashMap。现在Java中多线程里一般使用ConcurrentHashMap，至于为什么。会在下一篇博文里分析。
### HashMap的rehash源代码  
`put()`方法的Java8源码分析看我的[Java里的hashMap和golang里的map](https://fluge.github.io/2016/12/05/Java%E9%87%8C%E7%9A%84hasMap%E5%92%8Cgolang%E9%87%8C%E7%9A%84map/)，在Java8中优化了扩容的hash算法，更加高效。在这分析死循环用的是Java7的源码。更加清晰点。
<!--more-->

```java
void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
         //创建一个新的Hash Table
        Entry[] newTable = new Entry[newCapacity];
        //将Old Hash Table上的数据迁移到New Hash Table上
        transfer(newTable, initHashSeedAsNeeded(newCapacity))
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
/**
 * 将当前table的Entry转移到新的table中
 */
void transfer(Entry[] newTable){
    Entry[] src = table;
    int newCapacity = newTable.length;
    //下面这段代码的意思是:从OldTable里摘一个元素出来，然后放到NewTable中
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                //在新的table 中求得适合插入的位置
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);// 可能导致死循环
        }
    }
}
```
在单线程中的执行流程其实是很直观的:先对要插入的元素进行哈希，在数组中找到相应的位置，如果发生冲突就变成链表存储，在看桶有没有满。有就进`resize()`扩容操作。但是在多线程的时候由于扩容操作产生`环形链`，会造成`get()`方法命中时----Infinite Loop,然后CPU爆炸。  
#### 举例分析(网上一个很经典的例子--引用自酷壳)  
假设我们的hash算法是简单的key mod一下表的大小(即数组的长度),现在有两个线程：一个蓝色标注，一个红色标注。
关键代码在`transfer()`中把旧的table的Entry转移到新的table中的时候

```java
do {
     Entry<K,V> next = e.next;<--假设红色线程执行到这里就被调度挂起了,蓝色线程全部执行
     int i = indexFor(e.hash, newCapacity);
     e.next = newTable[i];
     newTable[i] = e;
     e = next;
} while (e != null);
```
两个线程在蓝色线程执行完后的情况。这个时候红色线程中的`e指向了key(3)`,而`next指向了key(7)`,但是在蓝色线程中`链表已经扩容完成`,并且`链表的顺序被反转` ,这个时候就有了环链的征兆了。e的下一个节点本来是next,经过蓝色线程扩容后变成了next下一个节点是e。就有很大的几率产生环形链。  
![](http://ofa8x9gy9.bkt.clouddn.com/JAVA%20HASHMAP%E7%9A%84%E6%AD%BB%E5%BE%AA%E7%8E%AF.jpg)  
接着看,这个时候红色线程得到了执行的机会.被调度回来进行执行，
先是执行`newTalbe[i] = e;`,然后是`e = next，导致了e指向了key(7)`,而下一次循环的`next = e.next导致了next指向了key(3)`。
![](http://ofa8x9gy9.bkt.clouddn.com/HashMap03.jpg)  
然后红色线程接着工作,把key(7)摘下来，放到newTable[i]的第一个,然后把`e和next往下移`。
![](http://ofa8x9gy9.bkt.clouddn.com/HashMap04.jpg)   
然后重点来了：`e.next = newTable[i]`导致`key(3).next指向了key(7)`。但是此时的`key(7).next 已经指向了key(3)`,环形链表就这样出现了。
![](http://ofa8x9gy9.bkt.clouddn.com/HashMap05.jpg)     
于是,当我们的线程一调用到,HashTable.get(11)时,悲剧就出现了——Infinite Loop。

---
参考：
[疫苗：Java HashMap的死循环](http://coolshell.cn/articles/9606.html)  
[不正当使用HashMap导致cpu 100%的问题追究](http://ifeve.com/hashmap-infinite-loop/) 