## 一、介绍

> 一个继承与HashMap的类；具有访问有序性，也就是，可以按照插入顺序或者访问顺序排列；

## 二、特性

* 和HashMap一样，基于散列表实现！
* 同样不是线程安全的
* **区别是其内部维护了一个双向循环链表，该链表是有序的，可以按元素插入顺序或元素最近访问顺序\(LRU\)排列**

## 三、源码解析

1. 有连个列表节点，head、tail；tail指向最后一个节点，head指向头结点！

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;     //注意，很明显这里的每个节点，都有一个前驱和后驱
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

/**
 * The head (eldest) of the doubly linked list.
 */
transient LinkedHashMap.Entry<K,V> head;  //最久或者最先插入的节点

/**
 * The tail (youngest) of the doubly linked list.
 */
transient LinkedHashMap.Entry<K,V> tail;  //最新或者最近访问的节点

// link at the end of list
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;      //如果头结点为空，设置head=tail=p
    else {
        p.before = last;  //如果已经有节点，那么将新节点插入当前列表的尾部，并且tail变为新加入的节点
        last.after = p;
    }
}
```

* 可以根据参数指定列表的排序

```java
/**
  * The iteration ordering method for this linked hash map: <tt>true</tt>
  * for access-order, <tt>false</tt> for insertion-order.
  *
  * @serial
  */
 final boolean accessOrder;

 按照字面意思就可以理解，是否按照访问顺序操作！也就是，若果值为true，那么在put一个新key的value时候，除了将其添加到hash位置中，
 还需要将其插入（移动）到列表的尾部！


 //当节点被操作过后，将节点移动尾部
 void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

* 


