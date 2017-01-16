---
title: 并发系列(一) synchronized和volatile 
date: 2016-12-29 17:52:18
categories: 
- Java
tags: 
- Java
---
从[HashMap并发的死循环](https://fluge.github.io/2016/12/15/HashMap%E5%B9%B6%E5%8F%91%E7%9A%84%E6%AD%BB%E5%BE%AA%E7%8E%AF/)可以知道,Hashmap是没办法在多线程的情况下使用的，为了解决这个问题，在Java4之前用的是hashtable,只是现在不推荐的。在Java5之后就比较推荐使用java.util.concurrent.ConcurrentHashMap，这个在多线程的情况下，也能有很好的性能。从这里引入了Java里面一类很重要的概念---并发。先解决完上一个问题。高并发下ConcurrentHashMap的结构。
### 并发的一些初步了解--synchronized和volatile  
在多线程的并发的情况下有安全的访问变量，为了解决这个问题引入一个机制---锁机制。让多线程不能同时访问一个变量。在并发过程中有需要简单的了解两个东西的含义。
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
3. 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个`类的所有对象`； 
4. 修改一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个`类的所有对象`。  
无论synchronized关键字加在方法上还是对象上，如果它作用的对象是非静态的，则它取得的锁是对象；如果synchronized作用的对象是一个静态方法或一个类，则它取得的锁是对类，该类所有的对象同一把锁。 每个对象只有一个锁（lock）与之相关联，谁拿到这个锁谁就可以运行它所控制的那段代码
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
        @SuppressWarnings("unchecked")
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
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```
从源码看基本和HashMap差不多。解决哈希冲突的方法一样。但是不允许为null的键值对。`get()`都差不多就不分析了。  
#### Hashtable淘汰的原因


  
----  
参考:[Java中Synchronized的用法](http://blog.csdn.net/luoweifu/article/details/46613015)