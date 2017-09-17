## 介绍

LinkedList是一个继承于AbstractSequentialList的双向列表；它也可以被当作堆栈、队列或双端队列进行操作；List 接口，能对它进行队列操作。实现 Deque 接口，即能将LinkedList当作双端队列使用；**LinkedList和ArrayList一样都是线程不安全的**！

## 特性 {#xiaozhi}

1. LinkedList中的头结点不保存值，但是LinkedHashMap是保存值的
2. 实现了Deque和List接口，可以既可以进行队列操作，又可以当做双端列表操作

## 源码分析

### 1、类的继承关系

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

### 2、代码解析

实现AbstractSequentialList，支持队列的操作；

实现Deque接口，支持双端队列的操作

