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
##### Java里面的初始化和实现
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

##### ArrayList里面的将数组动态扩容实现add和remove
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
在Go中的数组和Java有点不一样。在golang中数组是内置类型
```go

```
