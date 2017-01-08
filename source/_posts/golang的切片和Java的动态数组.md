---
title: golang的切片和Java的动态数组
date: 2016-11-29 17:44:44
categories: 
- Java和golang
tags: 
- golang
- Java
---
### Java里的动态数组---ArrayList  
ArrayList是实现List接口的动态数组，每个ArrayList实例都有一个容量，该容量是指用来存储列表元素的数组的大小。随着向ArrayList中不断添加元素，容量会自动增长，自动增长会带来数据向新数组的*重新拷贝*。同时需要注意的是这个实现不是同步的。如果多个线程同时访问一个ArrayList实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。(结构上的修改是指任何添加或删除一个或多个元素的的操作，或者显示调整底层数组的大小；仅仅设置元素的值不是结构上的修改)  
<!--more-->
#### Java里面的初始化和实现
```java
   public class ArrayList<E>   extends 	 AbstractList<E>  implements   List<E>, RandomAccess, Cloneable, java.io.Serializable{
 	
     //设置arrayList默认容量
     private static final int DEFAULT_CAPACITY = 10;
 
     //空数组，当调用无参数构造函数的时候默认给个空数组
     private static final Object[] EMPTY_ELEMENTDATA = {};
 
     //这才是真正保存数据的数组
     private transient Object[] elementData;
 
     //arrayList的实际元素数量
     private int size;
 
     //构造方法传入默认的capacity 设置默认数组大小
     public ArrayList(int initialCapacity) {
         super();
         if (initialCapacity < 0)
             throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
         this.elementData = new Object[initialCapacity];
     }
 
     //无参数构造方法默认为空数组
     public ArrayList() {
         super();
         this.elementData = EMPTY_ELEMENTDATA;
     }
 
     //构造方法传入一个Collection， 则将Collection里面的值copy到arrayList
     public ArrayList(Collection<? extends E> c) {
         elementData = c.toArray();
         size = elementData.length;
         if (elementData.getClass() != Object[].class)
             elementData = Arrays.copyOf(elementData, size, Object[].class);
     }
}
```
从上面的源码可以看出来，ArrayList的本质就是数组的，其中的add,get,set,remove等操作都是对数组的操作，所以ArrayList的特性基本都是源于数组:有序、元素可以重复、插入慢、获取快等特性。

#### ArrayList里面的将数组动态扩容实现add和remove
```java
    //在末尾增加元素，虽然有时需要扩容但是时间复杂度为O(1)
	public boolean add(E e) {
         ensureCapacityInternal(size + 1);  // Increments modCount!!
         elementData[size++] = e;
         return true;
     }
     //在数组中间增加元素，因为需要移动后面的元素，所以时间复杂度为O(n)
     public void add(int index, E element) {
         rangeCheckForAdd(index);
 
         ensureCapacityInternal(size + 1);  // Increments modCount!!
         System.arraycopy(elementData, index, elementData, index + 1,
                          size - index);
         elementData[index] = element;
     } 
     private void ensureCapacityInternal(int minCapacity) {
         if (elementData == EMPTY_ELEMENTDATA) {
              minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
         }
         ensureExplicitCapacity(minCapacity);
      }
      private void ensureExplicitCapacity(int minCapacity) {
          modCount++;
         //超出了数组可容纳的长度，需要进行动态扩展
         if (minCapacity - elementData.length > 0)
             grow(minCapacity);
     }
      //这才是动态扩展的核心
     private void grow(int minCapacity) {
         int oldCapacity = elementData.length;
         //设置新数组的容量扩展为原来数组的1.5倍
         int newCapacity = oldCapacity + (oldCapacity >> 1);
         //再判断一下新数组的容量够不够，够了就直接使用这个长度创建新数组， 不够就将数组长度设置为需要的长度
         if (newCapacity - minCapacity < 0)
             newCapacity = minCapacity;
         //判断有没超过最大限制
         if (newCapacity - MAX_ARRAY_SIZE > 0)
             newCapacity = hugeCapacity(minCapacity);
         //将原来数组的值copy新数组中去
         elementData = Arrays.copyOf(elementData, newCapacity);
     }
     private static int hugeCapacity(int minCapacity) {
         if (minCapacity < 0) // overflow
              throw new OutOfMemoryError();
         return (minCapacity > MAX_ARRAY_SIZE) ?
             Integer.MAX_VALUE :
             MAX_ARRAY_SIZE;
     }
```
从上面的ArrayList的源码就可以知道,整个ArrayList的动态实现就是在增加数据的时候判断数组的容量是否足够,不够就重新生成一个1.5倍的数组,然后进行复制。这就是整个ArrayList的核心。
### golang里面的动态数组---slice
#### Go中的数组定义
在Go中的数组和Java有点不一样。在golang中数组是内置类型,初始化后长度是固定的，没有办法修改其长度,数组的长度也是其类型的一部分。数组是值类型,通过从0开始的下标索引访问元素值。值得注意的是如果GO中的数组作为函数的参数，那么实际传递的参数是一份数组的拷贝,而不是数组的指针。  
![](http://ofa8x9gy9.bkt.clouddn.com/golang%E6%95%B0%E7%BB%84.png)  

```go
var b [5]int //没有初始值，会自动的给出默认值{0,0,0,0,0}
a:=[5]int{1,2,3,4,5}
b:=[...]int{1,2,3,4,5}
```
#### slice
数组的长度是不可改变的,在很多场景都不是很适用，但是slice不一样。slice是golang的内置类型。在slice中有两个概念,和数组一样，有两个内置的属性：一个是len长度，一个是cap容量。slice是应用类型,因此当传递切片将和应用同一指针，修改值会影响其他的对象。

```go
//一般建议的初始化是用make()来初始化
var a []int
```
上面就可以表示一个slice,和声明数组差不多。只是少了一个长度。
slice也可以从一个数组或者已经存在的`slice`中再次声明。`slice`通过`a[i:j]`来获取,其中i是数组的开始位置,j是结束位置(不包含),长度为j-i 

```go
// 声明一个含有10个元素元素类型为byte的数组
var arr = [10]byte {'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j'}

// 声明两个含有byte的slice
var a, b ,c ,d[]byte
// a指向数组的第3个元素开始，并到第五个元素结束，现在a含有的元素: arr[2]、arr[3]和ar[4]
a = arr[2:5]
// b是数组arr的另一个slice, b的元素是：arr[3]和arr[4]
b = arr[3:5]
//c是数组arr的另一个slice,c的元素师:arr[0],arr[1],arr[2]
c = arr [:3]
//slice的默认开始位置是0，arr[:n]等价于arr[0:n]
//slice的第二个序列默认是数组的长度，ar[n:]等价于ar[n:len(ar)]
//如果从一个数组里面直接获取slice，可以这样ar[:],因为默认第一个序列是0，第二个是数组的长度，即等价于ar[0:len(ar)]
```
基本结构如下：  
![](http://ofa8x9gy9.bkt.clouddn.com/slice.png)   
`slice`是引用类型,所以修改a中元素中的值，那么b中的值也会改变。
对于slice有几个有用的内置函数：
* `len()`获取slice的长度
* `cap()`获取slice的最大容量
* `append()` 向slice中追加一个或者多个元素，然后返回一个和slice一样类型的slice
* `copy()` 从源slice的src中复制元素到目标dst，并且返回复制的元素的个数
slice一般都是通过`make()`进行实例化操作,在进行扩容是用`append()`,如果直接加入的个数打入slice的初始容量会报错。

```go
//基本用法
slice := append([]int{1,2,3},4,5,6)
//合并两个slice
slice := append([]int{1,2,3},[]int{4,5,6}...)
//将字符串当作[]byte类型作为第二个参数传入
bytes := append([]byte("hello"),"world"...)
```
需要注意的是`append()`函数会改变slice的引用。cap不足时会按照cap的两倍进行扩容。
### 有意思的算法---扩容
首先有一个问题:在ArrayList中扩容是通过复制整个数组完成,每次当数组的容量满了，就会重新建一个长度是上次两倍的数组，然后进行复制操作，然后释放掉原来的数组。时间复杂度可以简单的看作使用for循环的嵌套，在复制数组的时候相当于用for循环来遍历了一遍数组。所以复制的时间复杂度应该是O(N)的。  
但是整个ArrayList在末尾插入的时候表现是很快的。这里就有一个均摊的思想。  
- 首先并不是每个元素的插入都会触发复制扩容这个操作。只有才数组长度不够的情况下，才会产生。然后均摊下来就是o(1)了。所以在某些情况下AarryList的性能会出现波动也是这个原因。




