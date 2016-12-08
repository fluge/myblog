---
title: Java里的hasMap和golang里的map
date: 2016-12-05 14:29:56
categories: 
- Java和golang
tags: 
- golang
- Java
---
### 哈希表  
哈希表就是一种以`键-值(key-value)`存储数据的结构,我们只要输入待查找的键即Key,即可找到对应的值。  
使用哈希找查有两个步骤:
1. 使用哈希函数将被找查的键转换为数组的索引.在理想的情况下,不同的键会被装换为不同的索引值,但是在有些情况下我们需要处理多个键被哈希到同一个索引值得情况。所以哈希查找的第二个步骤是处理冲突。
2. 处理哈希碰撞冲突。一般处理哈希碰撞用拉链法和开放寻址法等方法。
> 开放地址法:当发生地址冲突时，按照某种方法继续探测哈希表中的其他存储单元，直到找到空位置为止。  
> 拉链法:当通过哈希函数把键转换为数组的索引时,如果索引重复,就在该位置用链表顺序 存储该键值对。  

![](http://ofa8x9gy9.bkt.clouddn.com/%E6%8B%89%E9%93%BE%E6%B3%95.png)  
### Java中的HasMap  
基本认识：基于Map接口,*允许null键/值,非同步,不保证有序*,也不保证顺序不随时间变化。 
<!--more--> 
两个重要参数:  
* 容量(Capacity)：Capacity就是bucket的大小
* 负载因子(Load factor)：Load factor就是bucket填满程度的最大比例。 
如果对迭代性能要求很高的话不要把`capacity`设置过大,也不要吧`load factor`设置过小，当bucket的entries的数目大于`capacity*load factor`是就需要调整bucket的大小为当前的2倍。
#### `put()`函数的基本思路:
1. 对key的`hashCode()`做hash,然后计算对应的index
2. 如果没有发生碰撞就直接放到桶里
3. 如果发生碰撞就采用拉链法,以链表的形式存储在该桶里
4. 如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD),就把链表转换为红黑树
5. 如果节点存在,就替换成新的value
6. 如果桶满了(超过load factor*current capacity),就要resize  

```java
public V put(K key, V value){
	return putVal(hash(key),key,value,false,true);
}
final V putVal(int hash,K key,V value,boolean onlyIfAbsent,boolean evict)
{
Node<K,V>[] tab; Node<K,V> p; int n, i;
    // tab为空则创建
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 计算index，并对null做处理
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 节点存在
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 该链为树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 该链为链表
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 写入
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 超过load factor*current capacity，resize
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
#### `get()`函数的基本思路
1. 桶里的第一个节点就直接命中
2. 如果桶里有冲突，就通过`equal()`来找查对应的entry，若为树时间复杂度为O(logN)，链表就是O(N)   

```java
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 直接命中
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // 未命中
            if ((e = first.next) != null) {
                // 在树中get
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 在链表中get
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
}
```  
//TODOHasMap的哈希算法  
注意:通过hash的方法，通过put和get存储和获取对象。存储对象时，我们将`K/V`传给put方法时,它调用hashCode计算hash从而得到bucket位置,进一步存储，`HashMap`会根据当前bucket的占用情况自动调整容量(超过Load Facotr则resize为原来的2倍)。获取对象时,我们将K传给get,它调用hashCode计算hash从而得到bucket位置,并进一步调用equals()方法确定键值对。如果发生碰撞的时候，Hashmap通过链表将产生碰撞冲突的元素组织起来,在Java 8中,如果一个bucket中碰撞冲突的元素超过某个限制(默认是8,则使用红黑树来替换链表,从而提高速度   
### golang中的map  
基本认识:在go中一个map就是一个哈希表的引用,map类型可以写为map[K]V,对K的类型要求是必须支持`==`比较运算符。但是不建议使用浮点型作为Key。

```go
struct Hmap{//map的关键数据
    uint8   B;    // 可以容纳2^B个项
    uint16  bucketsize;   // 每个桶的大小
    ....
    byte    *buckets;     // 2^B个Buckets的数组
    byte    *oldbuckets;  // 前一个buckets，只有当正在扩容时才不为空
};
//初始化的3种方式
ages:=make(map[string]int)// mapping from strings to ints
ages:=map[string]int{}
ages:=map[string]int{
    "alice":0,
    "charlie":33,
}
//取值
ages["alice"]=0
//赋值
ages["charlie"]=34
//删除
delete(ages, "alice") // remove element ages["alice"]
```  
上面这些都是安全的，及时失败也会返回对应value类型的零值。  
但是有时候需要想知道对应的元素是否真的在map之中。推荐写法：
```go
//map的下标语法将产生两个值；第二个是一个布尔值   
//用于报告元素是否真的存在。布尔变量一般命名为ok，特别适合马上用于if条件判断部分。
if age, ok := ages["alice"]; !ok { /* ... */ }
```   

#### 扩容
在golang中主要采用增量扩容--扩容因子为`6.5`。这个主要是为了缩短map容器的响应时间，因为在map桶里面数据很多事,直接复制进行扩容就会很卡，导致较长一段时间无法响应请求。不过具体时间复杂度还是采用的均摊法。具体做法:
> 扩容会建立一个大小是原来2倍的空表。将旧的bucket搬到新表中(复制),但是并不会将旧的bucket从oldbucket中删除，而是加上一个已删除的标记。
   
由于整个过程是逐渐完成的,这样就会导致一部分数据还没有完全复制到新表，所以会对insert，remove，get等操作产生影响。并且只有当所有复制操作完成后才会释放oldbucket。  
#### `insert`分析基本思路  
1. 根据key算出hash值，进而得出索引的位置
2. 如果bucket的位置在old table中，就重新hash到新表中
3. 查找对应的位置，如果在bucket中如果已经存在相应的key，就覆盖原来value，没有就插入
4. 根据table中元素的个数，判断是否扩容
5. 如果对应的bucket已经full，重新申请新的bucket作为overbucket(溢出桶链表)。
6. 将key/value pair插入到bucket中。  
#### `get`查找过程
1. 根据key算出hash值，进而得出索引的位置
2. 如果存在old table, 首先在old table中查找，如果找到的bucket已经扩容，转到步骤3。 反之，返回其对应的value。
3. 在new table中查找对应的value。  
```go
do { //对每个bucket
    //依次比较桶内的每一项存放的高位hash与所求的hash值高位是否相等
    for(i = 0, k = b->data, v = k + h->keysize * BUCKETSIZE; i < BUCKETSIZE; i++, k += h->keysize, v += h->valuesize) {
        if(b->tophash[i] == top) { 
            k2 = IK(h, k);
            t->key->alg->equal(&eq, t->key->size, key, k2);
            if(eq) { //相等的情况下再去做key比较...
                *keyp = k2;
                return IV(h, v);
            }
        }
    }
    b = b->overflow; //b设置为它的下一下溢出链
} while(b != nil);
```
这里一个细节需要注意一下。不认真看可能会以为低位用于定位bucket在数组的index，那么高位就是用于key/valule在bucket内部的offset。事实上高8位不是用作offset的，而是用于加快key的比较的。  
### 总结
在扩容过程中，oldbucket是被冻结的，查找时会在oldbucket中查找，但不会在oldbucket中插入数据。如果在oldbucket是找到了相应的key，做法是将它迁移到新bucket后加入扩容标记。
然后就是只要在某个bucket中找到第一个空位，就会将key/value插入到这个位置。也就是位置位于bucket前面的会覆盖后面的(类似于存储系统设计中做删除时的常用的技巧之一，直接用新数据追加方式写，新版本数据覆盖老版本数据)。找到了相同的key或者找到第一个空位就可以结束遍历了。不过这也意味着做删除时必须完全的遍历bucket所有溢出链，将所有的相同key数据都删除。所以目前map的设计是为插入而优化的，删除效率会比插入低一些。