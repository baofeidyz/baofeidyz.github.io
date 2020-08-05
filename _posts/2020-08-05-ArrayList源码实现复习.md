---
layout: post
title: ArrayList源码实现复习
date: 2020-08-05
Author: baofeidyz
categories: java
tags: [java]
comments: true
---

```shell
java version "1.8.0_251"
Java(TM) SE Runtime Environment (build 1.8.0_251-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.251-b08, mixed mode)
```

基于数组实现
```java
transient Object[] elementData;
```

## add(E e)

```java
/**
  * Appends the specified element to the end of this list.
  *
  * @param e element to be appended to this list
  * @return <tt>true</tt> (as specified by {@link Collection#add})
  */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

`size`是当前数组实际使用的大小，如果当前所需要的数组地址已经大于了当前数组的容量，则对数组进行扩容操作，即调用`grow`方法。

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
得到所需要扩容的大小以后，调用`navite`方法对集合进行扩容。
扩容结束以后，将`size`（代表着实际占用的变量）进行自增，同时将这个数组下标进行赋值。

## remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

`remove`方法实际上就是遍历数组所有的元素，然后找到数组下标，再根据数组下标进行删除

```java
/*
  * Private remove method that skips bounds checking and does not
  * return the value removed.
  */
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                          numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

定位到具体需要删除的数组下标以后，将下标后的数据往前移动，并将最后一个元素设为`null`，便于GC回收内存。

## 迭代器

私有内部类，通过“游标”去操作数组

## 为什么不能在循环里面增加或删除元素

### foreach

foreach本质上就是迭代器的实现，但在删除的时候，使用的是集合的`remove`方法，而不是迭代器提供的`remove`方法，这就导致在迭代器遍历中，有一个校验是否并发修改的方法无法通过验证，抛出异常。

```java

@SuppressWarnings("unchecked")
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}

// modCount是ArrayList中的属性值，是集合添加元素、删除元素的次数，expectedModCount是迭代器中的属性值，是预期的修改次数。实际修改值与期望值不同
final void checkForComodification() {
  if (modCount != expectedModCount)
      throw new ConcurrentModificationException();
}

```

### for循环

for循环本质上是使用数组下标遍历数组，通过前文中提到的`remove(Object o)`方法的实现，可以了解到，实际上是将需要删除的元素后的数组向前移动，并将最后一个元素设为null，便于GC回收。

那么当我们使用for循环操作数组，并对其进行删除操作的时候

```java
for (int i = 0; i < list.size(); i++){
  if(list[i] == 'xxx'){
    list.remove(i);
  }
}
```

假设数组中，第X个元素满足条件，并将其进行删除。 此时X后的数据元素全部向前移动，那么第X个元素，已经是移动前X+1，如果此时i++自增，那么你取到的是未移动前的X+2个元素。

所以我们只需要修正一下遍历的数组下标即可解决

```java
for (int i = 0; i < list.size(); i++){
  if(list[i] == 'xxx'){
    list.remove(i);
    i--;
  }
}
```

部分参考： https://blog.csdn.net/wangjun5159/article/details/61415358