## 介绍

LinkedList是一个继承于AbstractSequentialList的双向列表；它也可以被当作堆栈、队列或双端队列进行操作；List 接口，能对它进行队列操作。实现 Deque 接口，即能将LinkedList当作双端队列使用；**LinkedList和ArrayList一样都是线程不安全的**！

## 特性 {#xiaozhi}

1. LinkedList中的头结点不保存值，但是LinkedHashMap是保存值的
2. 实现了Deque和List接口，可以既可以进行队列操作，又可以当做双端列表操作

## 源码分析

### 1、类的继承关系

#### 代码片段

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

```java
/**
 * Returns a shallow copy of this {@code LinkedList}. (The elements
 * themselves are not cloned.)
 *
 * @return a shallow copy of this {@code LinkedList} instance
 */
public Object clone() {
    LinkedList<E> clone = superClone();

    // Put clone into "virgin" state
    clone.first = clone.last = null;
    clone.size = 0;
    clone.modCount = 0;

    // Initialize clone with our elements
    for (Node<E> x = first; x != null; x = x.next)
        clone.add(x.item);

    return clone;
}
```

#### 代码解析

```auto
1、实现AbstractSequentialList，支持队列的操作；
2、实现Deque接口，支持双端队列的操作
3、实现Cloneable支持，支持clone操作;通过代码，可以知道，clone是进行的浅拷贝；
4、实现Serializable，支持序列化的操作！
```

### 2、节点数据结构

#### 代码片段

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

#### 代码解析

```auto
1、这个类是LinkedList的一个静态内部类;
2、这个类有一个前驱，也有后驱，是一个双向列表
3、节点中的item是每个元素item的引用对象
```

### 3、数据元素操作

#### 代码片段

```java
/**
 * Inserts the specified element at the beginning of this list.
 *
 * @param e the element to add
 */
public void addFirst(E e) {
    linkFirst(e);
}

/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #add}.
 *
 * @param e the element to add
 */
public void addLast(E e) {
    linkLast(e);
}
```



