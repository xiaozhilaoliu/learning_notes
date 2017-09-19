## 基本介绍

Handler是android提供线程通信框架，其中涉及到的主要类有Handler、Looper、MessageQueue！

## 源码解析

### 一、Handler关键字段

```java
    final Looper mLooper;               //用于轮询消息
    final MessageQueue mQueue;          //消息队列
    final Callback mCallback;           //消息处理回调
    final boolean mAsynchronous;        //消息发送是否阻塞
```

### 二、关键函数

```java
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
    
    //侯建函数中，获取当前线程的mLooper对象
      public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```



