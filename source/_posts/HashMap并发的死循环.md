 ---
title: HashMap并发的死循环
date: 2016-12-15 17:52:18
categories: 
- Java
tags: 
- Java
---
HashMap从设计上来说就不适合在并发的情况的下使用,因为HashMap每次在`put()`时，总会检查一遍对应桶的容量，如果桶满了，或者超过了设定的值，就会`reserve()`来进行扩容,然后通过`get()`来取出相应的值。这个过程在单线程下是没什么问题的，但是如果在并发的条件下，多个线程同时reserve桶，然后有线程这个时候执行`get()`就会产生死循环，具体等会看源码。在Java里面有一个很老的hashtable就是加了锁的HashMap。现在Java中多线程里一般使用ConcurrentHashMap，至于为什么。会在下一篇博文里分析。
<!--more-->
### HashMap的rehash源代码  
`put()`方法的Java8源码分析看我的[Java里的hashMap和golang里的map](https://fluge.github.io/2016/12/05/Java%E9%87%8C%E7%9A%84hasMap%E5%92%8Cgolang%E9%87%8C%E7%9A%84map/)  

```java
public V put(K key, V value){
    return putVal(hash(key),key,value,false,true);
}
final V putVal(int hash,K key,V value,boolean onlyIfAbsent,boolean evict)
{
    /
    .......
    /

    ++modCount;
    // 超过load factor*current capacity，resize
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```  

参考：
[疫苗：Java HashMap的死循环](http://coolshell.cn/articles/9606.html) 
---
