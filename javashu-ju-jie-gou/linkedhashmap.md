## 一、介绍

> 一个继承与HashMap的类；具有访问有序性，也就是，可以按照插入顺序或者访问顺序排列；

## 二、特性

* 和HashMap一样，基于散列表实现！
* 同样不是线程安全的
* **区别是其内部维护了一个双向循环链表，该链表是有序的，可以按元素插入顺序或元素最近访问顺序\(LRU\)排列**



```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;     //注意，很明显这里的每个节点，都有一个前驱和后驱
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```



