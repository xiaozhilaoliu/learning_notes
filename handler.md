## 基本介绍

Handler是android提供线程通信框架，其中涉及到的主要类有Handler、Looper、MessageQueue！

## 源码解析

### 一、Handler源码

```java
    final Looper mLooper;               //用于轮询消息
    final MessageQueue mQueue;          //消息队列
    final Callback mCallback;
    final boolean mAsynchronous;
```



