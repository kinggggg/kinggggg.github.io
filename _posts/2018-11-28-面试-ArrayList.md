---
layout:     post
title:      面试-ArrayList基础
subtitle:   ArrayList源码窥探
date:       2018-11-28
author:     李伟博
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java 
    - 面试 
    - 容器 
    - ArrayList
---

> 在通常的情况下，如果你去面试一个Java工程师的话，ArrayList是肯定会问到的问题。下面就从一些很基础的方面来仔细研究下ArrayList相关的问题

#### ArrayList的底层实现是通过数组的方式


```
/**
 * 默认的数组长度为10
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * 这个数组就是ArrayList底层实现用于保存元素的数组
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * 当前数组中包含的真实有效的元素的个数
 */
private int size;
```

#### 当向ArrayList中添加元素时，若超过了默认的数组长度大小（为10），则会动态创建出一个新的数组

流程如下：

```
 //开始代码调用
 List<String> list = new ArrayList<>();
 list.add("a");
 
 //ArrayList源码：
 //首先看看add方法
 public boolean add(E e) {
    // 首先确保数组长度满足元素的存储的要求，传递的参数值为当前数组长度+1
    ensureCapacityInternal(size + 1);  
    elementData[size++] = e; //将新加入的元素
    return true;
 }
 
 private void ensureCapacityInternal(int minCapacity) {
    //若默认的数组此时为空数组的话，计算是应该创建默认大小的数组还是创建当前数组长度+1长度的数组
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
 }
 
 private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 若当前为了存储a而需要的数组的长度大于当前数组的真实长度话，说明数组长度已经不够用，需要动态生成一个新的数组
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
 }
 
 //动态生成新数组
 private void grow(int minCapacity) {
    // 旧数组的长度
    int oldCapacity = elementData.length;
    // 新数组的长度为旧数组长度的1.5倍。右移运算符相当于除以2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 采用数组拷贝的方式动态创建出一个新数组，原来的数组的内容不变；新数组中未赋值的元素值为null
    elementData = Arrays.copyOf(elementData, newCapacity);
 }
```

#### ArrayList的删除单个元素public E remove(int index)方法考察


```
// 整体来说，删除元素的思路为
// 加入原始数组存在五个元素 a b c d e ，当调用remove(2)时，
// 需要将index为3和4的对应的值，依次向左移动 此时数组中元素的值为a b d e null
// Java是如何移动元素的呢，就是调用了一个native方法System.arraycopy()
public E remove(int index) {
    // 判断要删除index是否越界
    rangeCheck(index);

    modCount++;
    // 取出要删除的元素，主要是为了返回个调用者，其他的作用没有
    E oldValue = elementData(index);

    // 要移动的元素个数
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}

/* 数组元素移动的native方法签名
 * @param      src      原数组
 * @param      srcPos   从元数据的起始位置开始
 * @param      dest     目标数组
 * @param      destPos  目标数组的开始起始位置
 * @param      length   要copy的数组的长度
 */
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

#### 线程安全性
1. ArrayList不是线程安全的；而在早期JDK提供了和ArrayList功能相似的容器类Vector，其实线程安全的
2. 若需要将线程不安全的ArrayList变为线程安全的话，可以通过Collections提供的方法

```
Collections.synchronizedList(list);
```
再深入看下为啥调用这个方法后ArrayList就变成线程安全的呢？

```
public static <T> List<T> synchronizedList(List<T> list) {
    return (list instanceof RandomAccess ?
            new SynchronizedRandomAccessList<>(list) :
            // 其实返回的这个对象，而SynchronizedList对象提供的方法其实和ArrayList是一样的，不同的是其
            // 对每个方法的访问使用的同步锁
            new SynchronizedList<>(list));
}

// 可以看到SynchronizedList使用的锁mutex就是一个Object对象

// 省略不关注的代码...
final Object mutex;     // Object on which to synchronize
// 省略不关注的代码...

static class SynchronizedList<E>
    extends SynchronizedCollection<E>
    implements List<E> {
    private static final long serialVersionUID = -7754090372962971524L;

    final List<E> list;

    SynchronizedList(List<E> list) {
        super(list);
        this.list = list;
    }
    SynchronizedList(List<E> list, Object mutex) {
        super(list, mutex);
        this.list = list;
    }

    // 省略不关注的代码...

    public E get(int index) {
        synchronized (mutex) {return list.get(index);}
    }
    public E set(int index, E element) {
        synchronized (mutex) {return list.set(index, element);}
    }
    
    // 省略不关注的代码...
}
```

#### 2、什么情况下你会使用ArrayList？什么时候你会选择LinkedList？

若是不经常访问特定元素，而经常进行增加或者删除元素应该用LinkedList

若是经常访问特定元素，而不经常增加或则删除元素应该用ArrayList

这是由ArrayList和LinkedList底层数据结构决定的。在经常增加或者删除元素时，ArrayList底层采用的是数组拷贝的方式，性能损耗非常大；而LinkedList底层采用链表最为数据结构，因此其增加和删除元素的代价会非常小，不涉及元素的拷贝只涉及地址的修改






 

