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

### 二、Handler关键函数

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

    //构建函数中，获取当前线程的mLooper对象
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

    //发送消息的函数
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);     //这里将消息发送到线程队列里面
    }
```

### 三、Looper的关键字段

```java
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();    //当前线程存储对象，唯一
    private static Looper sMainLooper;  // guarded by Looper.class      //主线程对象，全局唯一，静态

    final MessageQueue mQueue;                                            //消息队列
    final Thread mThread;                                                 //当前线程
```

### 四、Looper关键函数

```java
 //每个线程都需要调运这个函数，来创建一个Looper对象。冰倩存储为当前线程对象
  private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
 //ActivityThread的main函数中调用，创建主线程关联的looper
  public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```



