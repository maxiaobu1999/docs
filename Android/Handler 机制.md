[TOC]





# 概述

![handler架构图](/Users/v_maqinglong/Documents/IdeaProjects/docs/sources/handler架构图.jpg)

- Message、handler、Looper、MessageQueue四个对象
- MessageQueue，阻塞队列，排序，数据结构是msg的单链表，
- 挂起唤醒，由native的Looper实现。管道（pipe）和io多路复用实现（epoll）实现。UI线程读、handler线程写
- 同步消息、IdelHandler、Looper（ThreadLocal，日志回调），消息池。

##handler类图native层

![handler类图native层](../sources/handler类图native层.png)

## java与native层关系

![handler中java与native层关系图](../sources/handler中java与native层关系图.png)







# 知识点

## ThreadLocal

 线程本地存储区（Thread Local Storage，简称为TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。TLS常用的操作方法：

```java
ThreadLocal<String> threadLocal1 = new ThreadLocal<>();
threadLocal1.set("1");
threadLocal1.get();// 结果1
ThreadLocal<String> threadLocal2 = new ThreadLocal<>();
threadLocal1.set("2");
threadLocal1.get();// 结果2
// 在其他线程中执行，threadLocal1.get();// 结果为null
```

Thread对象 里 map<threadLocal的hash值，值>

map里数组 数组 角标hash与运算，存值，扩容

##  同步消息、屏障

背景：系统的msg需要优先执行

同步消息：需要优先执行的msg，eg：系统渲染、事件分发相关的消息

异步消息：普通的msg。应用层普通使用的都是。

同步屏障：把一个msg.target==null的msg，放在同步消息前面。这个msg就是同步屏障，若屏障到了队列头部，则说明队列中同步消息需要执行。遍历链表，根据！msg.isAsynchronous()找到，优先执行。


## 唤醒挂起



## 检测UI卡顿

looper每次处理消息的回调，使用如下：

```java
 Printer printer = new Printer() {
            @Override
            public void println(String x) {
                Log.e("+++++++", x);
            }
        };
 Looper.getMainLooper().setMessageLogging(printer);
```

日志打印如下

```shell
# handleMeassage()前触发一次
2020-06-23 14:06:07.808 25962-25962/com.norman.myapplication E/+++++++: >>>>> Dispatching to Handler (android.app.ActivityThread$H) {c316516} null: 159
# handleMeassage()后触发一次
2020-06-23 14:06:07.810 25962-25962/com.norman.myapplication E/+++++++: <<<<< Finished to Handler (android.app.ActivityThread$H) {c316516} null
```


## 消息阻塞

管道。内部是使用了Linux的 多路复用/epoll 实现的

https://blog.csdn.net/kisty_yao/article/details/71191175



## IdleHandler

- 存放在Message Queue中 `ArrayList<IdleHandler> mIdleHandlers `。

- 就是个接口

```java
    public static interface IdleHandler {
        boolean queueIdle();
    }
```

- 执行时机：

当 MessageQueue 中的消息都被响应后，MessageQueue 进入 Idle 状态，此时执行 mIdleHandlers 的函数

- 看源码搜索： 【// 此时消息队列处在空闲状态，执行IdleHandler.】

## epoll使用

```c++
// 创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。
int epoll_create(int size);
// 设置监听的事件：可读，可写等
 int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 挂起，等待事件的发生。当事件发生，被内核唤醒
 int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```



## 文件描述符&文件句柄

每个进程有张表(fdtable),记录打开的文件信息；

表的每一项(struct file)记录一个打开的文件的属性，eg：偏移量，读写访问模式等。

「文件句柄」：这个结构体

「文件描述（fd）」：结构体在表中的索引，指向句柄。即非负整形的索引号；

# TIP

msg时间戳用的是SystemClock.uptimeMillis()，即系统开机时间 



msg的runnbale ==> Handler构造传的Callback ==> Handler#handleMessage()，优先级如序。三者谁执行谁不执行，看【Handler# void handleMessage(@NonNull Message msg)】

Message的对象池，数据结构是单链表，表头是Message#next；



# todo

FileDescriptor 

io多路复用

# 源码



## Handler

```java
// 工具类，能发消息和处理消息。保存在Message#target中
public class Handler {
    /* 检测内存泄漏用的 */
    private static final boolean FIND_POTENTIAL_LEAKS = false;
    private static final String TAG = "Handler";// LOG标签
    private static Handler MAIN_THREAD_HANDLER = null;// 主线程Handler
    /** Handler处理消息的接口 */
    public interface Callback {
        /**
         * 处理消息的方法，具体看下面 【Handler#handleMessage(@NonNull Message msg)】
         * @param msg 触发函数执行的msg，这个msg肯定没有runnable
         * @return True 这个msg不再触发Handler#handleMessage()
         */
        boolean handleMessage(@NonNull Message msg);
    }
/***************************** 公共方法 BENGIN *******************************/    
    /** 处理消息的方法，子类复写实现 */
    public void handleMessage(@NonNull Message msg) {
    }
    /** 派发消息 */
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);// msg里面有runnable，优先级1
        } else {
            if (mCallback != null) {// handler的构造传了Callback，优先级2
                if (mCallback.handleMessage(msg)) {
                    return;//  True 这个msg不再触发Handler#handleMessage()
                }
            }
            handleMessage(msg);// 触发回调，优先级3
        }
    }
/*========================== 创建 BENGIN =============================*/
    /** 构造1 */
    public Handler() {
        this(null, false);
    }
    /**构造2, @param callback 处理消息的接口,看上面半屏.*/
    public Handler(@Nullable Callback callback) {
        this(callback, false);
    }
    /** 构造3，指定looper.工作线程使用，eg：HandlerThread */
    public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }
    /** 构造4 */
    public Handler(@NonNull Looper looper, @Nullable Callback callback) {
        this(looper, callback, false);
    }
    /**  构造5 APP隐藏； @param async true, 异步消息. */
    @UnsupportedAppUsage
    public Handler(boolean async) {
        this(null, async);
    }
    /**  构造6 APP隐藏 */
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) && (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    /** 构造7隐藏  @param async  true异步消息 */
    @UnsupportedAppUsage
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    /** 创建1，异步消息 */
    @NonNull
    public static Handler createAsync(@NonNull Looper looper) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        return new Handler(looper, null, true);
    }
    /** 创建2，异步消息 */
    @NonNull
    public static Handler createAsync(@NonNull Looper looper, @NonNull Callback callback) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        if (callback == null) throw new NullPointerException("callback must not be null");
        return new Handler(looper, callback, true);
    }
    /** 创建3，主线程Handler，隐藏 */
    @UnsupportedAppUsage
    @NonNull
    public static Handler getMain() {
        if (MAIN_THREAD_HANDLER == null) {
            MAIN_THREAD_HANDLER = new Handler(Looper.getMainLooper());
        }
        return MAIN_THREAD_HANDLER;
    }
    /** 创建4，空则返回主线程Handler，隐藏 */
    @NonNull
    public static Handler mainIfNull(@Nullable Handler handler) {
        return handler == null ? getMain() : handler;
    }
/*========================== 创建 END =============================*/
/*========================== 获取Message BEGIN =============================*/
    /** 消息的对象池里获取Message */
    @NonNull
    public final Message obtainMessage(){
        return Message.obtain(this);
    }

    /** 指定what值 */
    @NonNull
    public final Message obtainMessage(int what){
        return Message.obtain(this, what);
    }
    
    /** @param obj 对应 Message.obj . */
    @NonNull
    public final Message obtainMessage(int what, @Nullable Object obj) {
        return Message.obtain(this, what, obj);
    }
    /** 
     * @param arg1 对应 Message.arg1 .
     * @param arg2  对应 Message.arg2  */
    @NonNull
    public final Message obtainMessage(int what, int arg1, int arg2){
        return Message.obtain(this, what, arg1, arg2);
    }
    @NonNull
    public final Message obtainMessage(int what, int arg1, int arg2, @Nullable Object obj) {
        return Message.obtain(this, what, arg1, arg2, obj);
    }
/*========================== 获取Message END =============================*/
/*========================== 发送Message的方法 BEGIN =============================*/  

    public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    /** 指定时间执行任务  */
    public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }
    /** @param token 移除消息用的索引{@link #removeCallbacksAndMessages}. */
    public final boolean postAtTime(
            @NonNull Runnable r, @Nullable Object token, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }
    /** 延时执行 */
    public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }
    /** @hide */
    public final boolean postDelayed(Runnable r, int what, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r).setWhat(what), delayMillis);
    }
    public final boolean postDelayed(
            @NonNull Runnable r, @Nullable Object token, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r, token), delayMillis);
    }

    /** 将任务加入到队列前端 */
    public final boolean postAtFrontOfQueue(@NonNull Runnable r) {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }

    /**
     * 调用线程会，阻塞执行，直到Runnable执行完毕，run()通过handler.post()执行
     * @param timeout 超时毫秒. @hide 隐藏
     */
    public final boolean runWithScissors(@NonNull Runnable r, long timeout) {
        if (r == null) {
            throw new IllegalArgumentException("runnable must not be null");
        }
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout must be non-negative");
        }

        if (Looper.myLooper() == mLooper) {
            r.run();
            return true;
        }

        BlockingRunnable br = new BlockingRunnable(r);
        return br.postAndWait(this, timeout);
    }
    /**
     * 添加消息进消息队列.
     * @return Returns true，添加成功。false，添加失败，eg：已经添加过了
     */
    public final boolean sendMessage(@NonNull Message msg) {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what){
        return sendEmptyMessageDelayed(what, 0);
    }
    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
		public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    /**
     * 发消息的核心方法.sendXxx(),postXxx()最终都会调用此方法
     * @param uptimeMillis 时间戳.
     */
    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;// 消息队列
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
  	// 该消息会排在消息队列的队首
    public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }
    /** Handler的looper==当前线程的Looper？直接handleMessage():发消息； @hide 隐藏  */
    public final boolean executeOrSendMessage(@NonNull Message msg) {
        if (mLooper == Looper.myLooper()) {
            dispatchMessage(msg);
            return true;
        }
        return sendMessage(msg);
    }
 /*========================== 发送Message的方法 END =============================*/ 
  	// 添加消息进消息队列
    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
/*========================== 删除Message的方法 BEGIN =============================*/ 
    public final void removeCallbacks(@NonNull Runnable r) {
        mQueue.removeMessages(this, r, null);
    }

    /** token！=null？移除token对应消息：所有的callback都会被移除。 */
    public final void removeCallbacks(@NonNull Runnable r, @Nullable Object token) {
        mQueue.removeMessages(this, r, token);
    }
    /** what是这个的消息被移除 */
    public final void removeMessages(int what) {
        mQueue.removeMessages(this, what, null);
    }
    public final void removeMessages(int what, @Nullable Object object) {
        mQueue.removeMessages(this, what, object);
    }
    public final void removeCallbacksAndMessages(@Nullable Object token) {
        mQueue.removeCallbacksAndMessages(this, token);
    }
   /*========================== 删除Message的方法 END =============================*/ 
    /** {@hide} 返回 eg：XxxHandler:#what 或 XxxHandler:XxxCallback */
    @NonNull
    public String getTraceName(@NonNull Message message) {
        final StringBuilder sb = new StringBuilder();
        sb.append(getClass().getName()).append(": ");
        if (message.callback != null) {
            sb.append(message.callback.getClass().getName());
        } else {
            sb.append("#").append(message.what);
        }
        return sb.toString();
    }
    /** Returns 字符串： Runnable类名 | 0x+"16进制的message.what" */
    @NonNull
    public String getMessageName(@NonNull Message message) {
        if (message.callback != null) {
            return message.callback.getClass().getName();
        }
        return "0x" + Integer.toHexString(message.what);
    }
  	/** 获取有what为这个值方法 */
    public final boolean hasMessages(int what) {
        return mQueue.hasMessages(this, what, null);
    }
    /** 有没有消息 @hide 隐藏 */
    public final boolean hasMessagesOrCallbacks() {
        return mQueue.hasMessages(this);
    }
    public final boolean hasMessages(int what, @Nullable Object object) {
        return mQueue.hasMessages(this, what, object);
    }
    public final boolean hasCallbacks(@NonNull Runnable r) {
        return mQueue.hasMessages(this, r, null);
    }
		// 获取handler的looper
    @NonNull
    public final Looper getLooper() {
        return mLooper;
    }
		// 打印的方法
    public final void dump(@NonNull Printer pw, @NonNull String prefix) {
        pw.println(prefix + this + " @ " + SystemClock.uptimeMillis());
        if (mLooper == null) {
            pw.println(prefix + "looper uninitialized");
        } else {
            mLooper.dump(pw, prefix + "  ");
        }
    }
    /** 打印的方法 @hide */
    public final void dumpMine(@NonNull Printer pw, @NonNull String prefix) {
        pw.println(prefix + this + " @ " + SystemClock.uptimeMillis());
        if (mLooper == null) {
            pw.println(prefix + "looper uninitialized");
        } else {
            mLooper.dump(pw, prefix + "  ", this);
        }
    }
    @Override
    public String toString() {
        return "Handler (" + getClass().getName() + ") {"
        + Integer.toHexString(System.identityHashCode(this))
        + "}";
    }
/***************************** 公共方法 END *******************************/    
    // Messenger：IPC传递Message，底层是基于Handler发送的。这段与消息机制无关
  	@UnsupportedAppUsage
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }
		// 
    private final class MessengerImpl extends IMessenger.Stub {
        // IPC发送消息
      	public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();// 获取调用方Uid
            Handler.this.sendMessage(msg);
        }
    }
		// 把Runnable包装成Message
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
		// 包装成Message
    @UnsupportedAppUsage
    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }
		// 直接处理消息
    private static void handleCallback(Message message) {
        message.callback.run();
    }

    @UnsupportedAppUsage
    final Looper mLooper;// 从ThreadLocal里能拿到
    final MessageQueue mQueue;// Looper里有
    @UnsupportedAppUsage
    final Callback mCallback;// 处理的接口，构造赋值，返回值影响handlerMessage()触发
  	// 同步消息、异步消息
    final boolean mAsynchronous;
    @UnsupportedAppUsage
    IMessenger mMessenger; // 进程通信 Messenger的实现会用到Handler
  	// 用于阻塞的处理message，
    private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }
        @Override
        public void run() {
            try {
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;
                    notifyAll();
                }
            }
        }
				// 发消息-》wait阻塞-〉处理完notify
        public boolean postAndWait(Handler handler, long timeout) {
            if (!handler.post(this)) {
                return false;
            }

            synchronized (this) {
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {
                            return false; // timeout
                        }
                        try {
                            wait(delay);
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    while (!mDone) {
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
}

```



## MessageQueue

```java
/**
 * 消息队列,单向链表，存放 Message 的容器，msg#when排序，阻塞队列
 * MessageQueue messageQueue = Looper.myQueue();App可以这么获取.
 */
public final class MessageQueue {
    private static final String TAG = "MessageQueue";
    private static final boolean DEBUG = false;
    // True 队列可以退出.MainLooper为不可退出，其他Looper默认可退出
    @UnsupportedAppUsage
    private final boolean mQuitAllowed;
    @UnsupportedAppUsage
    @SuppressWarnings("unused")
    private long mPtr; //native层的 MessageQueue的指针
    @UnsupportedAppUsage
    Message mMessages;// 消息链表(有序，时间顺序)
  	// IdleHandler容器，添加时，先放在这里
    @UnsupportedAppUsage
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    private SparseArray<FileDescriptorRecord> mFileDescriptorRecords;
    private IdleHandler[] mPendingIdleHandlers;// 临时IdleHandler容器，处理事深拷贝到这里
    private boolean mQuitting;// false时，回结束循环
    // 是否唤醒的标识，true则表示native层正在挂起
    private boolean mBlocked;
    // 同步消息屏障数量，同时也是序号，msg。arg1保存了对应的序号。删除屏障时使用
    @UnsupportedAppUsage
    private int mNextBarrierToken;
//=======================Native 方法 BEGIN=====================
    private native static long nativeInit();// 创建native层的消息队列
    private native static void nativeDestroy(long ptr);// 销毁
    @UnsupportedAppUsage// 唤醒一次，ptr指针；timeoutMillis隔多久唤醒
    private native void nativePollOnce(long ptr, int timeoutMillis); 
    private native static void nativeWake(long ptr);// 唤醒
    private native static boolean nativeIsPolling(long ptr);// 判断正在挂起？
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
//=======================Native 方法 END=====================
//=======================核心方法 BENGIN=====================  
  	// 构造包权限，随Looper一同创建，quitAllowed true不能退出，eg：UI线程
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
		// 在 Looper 的 loop() 方法中被 循环调用。
    @UnsupportedAppUsage
    Message next() {
        // 当loop已经结束的时候，直接返回null
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int pendingIdleHandlerCount = -1; // IdleHandler的数量，先置为-1
        int nextPollTimeoutMillis = 0;// 阻塞多久，给native层用的，即多久后唤醒
        for (;;) {
         	 // 将当前线程中挂起的所有绑定器命令刷新到内核驱动程序。
        	 // 这对于在执行可能会阻塞很长时间的操作之前进行调用很有用，
        	 // 以确保已释放任何挂起的对象引用，以防止进程将对象保留的时间超过需要的时间。
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
						// 获取一个msg，ptr：消息队列在native层的指针
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                final long now = SystemClock.uptimeMillis();// 当前时间戳
                Message prevMsg = null;// 临时保存同步消息的前一个msg
                Message msg = mMessages;// 链表头msg
              	// 表头的handlers是null。即为同步屏障，说明队列中有同步消息
                if (msg != null && msg.target == null) {
                    do { // 如果有同步消息屏障，找到链表中第一个异步的Message
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());// false则是同步消息
                }// 找到第一个同步消息会结束遍历
                if (msg != null) {
                  	// 当前的msg是最优先处理的msg
                    if (now < msg.when) {
                      // 如果消息没到时间，则取时间差去休眠，然后进入IdleHandler逻辑
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 返回链表头的Message.
                        mBlocked = false;
                        if (prevMsg != null) {// 说明处理的是同步消息
                            prevMsg.next = msg.next;// 同步msg的坑连上
                        } else {// 说明处理的是普通msg
                            mMessages = msg.next;// 链表头换成下一个
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();// 标记msg正在使用
                        return msg;// 返回msg，因为looper在循环，所以next()还会接着循环
                    }
                } else {// msg都处理完了
                    nextPollTimeoutMillis = -1;
                }

                // 如果需要关闭队列，停止阻塞返回null，looper的xun huan
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // 此时消息队列处在空闲状态，开始执行IdleHandler.
                if (pendingIdleHandlerCount < 0// 循环开始前置为-1，
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();// 待处理IdleHandler的数量
                }
                if (pendingIdleHandlerCount <= 0) {// 没有idle handlers需要处理
                    // 阻塞.
                    mBlocked = true;
                    continue;
                }
								// null检测
                if (mPendingIdleHandlers == null) {
               mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
              	// 在同步代码块中,将mIdleHandlers（list）深拷贝为mPendingIdleHandlers（Array）
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }
						// 然后循环执行 Array 中的 IdleHandler，执行后从 List 中移除
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // 移除
                boolean keep = false;// idler回调的返回值
                try {
                    keep = idler.queueIdle();// **触发回调**，
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);// 移除
                    }
                }// keep=ture则会再次调用
            }
            // 重置数量.
            pendingIdleHandlerCount = 0;
            // 如有idlerHandler被处理，马上从native队列中取下一msg，不等.
            nextPollTimeoutMillis = 0;
          // 循环内
        }
      // 循环外
    }
  	// Handler 中被调用，向消息队列中添加 Message。when：msg执行的绝对时间（时间戳）
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {//所有msg必须指定Handler。同步屏障是特例，且不用此方法入队
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {// 判断Message是否正在被使用
            throw new IllegalStateException(msg + " This message is already in use.");
        }
        synchronized (this) {
            if (mQuitting) {// 判断looper停止，即Looper#quict()调用
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
            msg.markInUse(); // 标记消息为被使用
            msg.when = when;// 消息什么时间执行
            Message p = mMessages;// 链表头msg
            boolean needWake;// 标识需要唤醒？
         		//如果消息队列中没有消息 || 这消息需要立马执行 || 该消息需要执行的时间早于链表头执行的时间
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;// 将该消息至于链表头，优先执行
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // 把数据加到链表的中间，通常不需要主动的通知唤醒,除非
            		// 1.在消息头存在同步消息屏障，即该消息的handler为空
           			// 2.这条消息是异步消息
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) { // 从消息头开始遍历，找同步消息
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break; // 当到达消息尾或发现当前消息的时间早于遍历到的消息时间时
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false; // 如发现链表中有早于该消息的异步消息时，不用通知唤醒
                    }
                }
                msg.next = p;
                prev.next = msg;
            }
            // 因为 mQuitting是false, 所以我们可以认为mPtr != 0
            if (needWake) {
                nativeWake(mPtr);// 通知native函数需要唤醒
            }
        }
        return true;
    }
		// 参数 safe 表示是否要移除当前队列中已到时间但没有来得及处理的消息
		// true 表示不移除，false 表示全部移除
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
    // 释放native资源，
    private void dispose() {
        if (mPtr != 0) {
            nativeDestroy(mPtr);
            mPtr = 0;
        }
    }
//=======================核心方法 END=====================  
  
//=======================IdleHandle BEGIN=====================  
    /** 队列是否空闲：消息队列null|| 链表头的msg执行时间在未来 */
    public boolean isIdle() {
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            return mMessages == null || now < mMessages.when;
        }
    }

    /** 添加IdleHandler  */
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);// 添加到数组里
        }
    }

    /** 移除 IdleHandler  */
    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);// 从数组中移除
        }
    }
      /** MessageQueue空闲时，会执行 mIdleHandlers 中的消息 */
    public static interface IdleHandler {
      	 // 返回false则从 mIdleHandlers 中移除；true则不会，即当再次空闲，还会再次调用
        boolean queueIdle();
    }
//=======================IdleHandle END=====================  
    // 判断 queue 是否处于轮询中(是否被阻塞)
    public boolean isPolling() {
        synchronized (this) {
            return isPollingLocked();
        }
    }
		// 判断 queue 是否处于轮询中(是否被阻塞)
    private boolean isPollingLocked() {
        return !mQuitting && nativeIsPolling(mPtr);
    }
		// finalize是Java提供的回收方法，当对象被GC回收时调用，做一些native释放
    @Override
    protected void finalize() throws Throwable {
        try {
            dispose();
        } finally {
            super.finalize();
        }
    }
    /**
     * Adds a file descriptor listener to receive notification when file descriptor
     * related events occur.
     * <p>
     * If the file descriptor has already been registered, the specified events
     * and listener will replace any that were previously associated with it.
     * It is not possible to set more than one listener per file descriptor.
     * </p><p>
     * It is important to always unregister the listener when the file descriptor
     * is no longer of use.
     * </p>
     *
     * @param fd The file descriptor for which a listener will be registered.
     * @param events The set of events to receive: a combination of the
     * {@link OnFileDescriptorEventListener#EVENT_INPUT},
     * {@link OnFileDescriptorEventListener#EVENT_OUTPUT}, and
     * {@link OnFileDescriptorEventListener#EVENT_ERROR} event masks.  If the requested
     * set of events is zero, then the listener is unregistered.
     * @param listener The listener to invoke when file descriptor events occur.
     *
     * @see OnFileDescriptorEventListener
     * @see #removeOnFileDescriptorEventListener
     */
    public void addOnFileDescriptorEventListener(@NonNull FileDescriptor fd,
            @OnFileDescriptorEventListener.Events int events,
            @NonNull OnFileDescriptorEventListener listener) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, events, listener);
        }
    }

    /**
     * Removes a file descriptor listener.
     * <p>
     * This method does nothing if no listener has been registered for the
     * specified file descriptor.
     * </p>
     *
     * @param fd The file descriptor whose listener will be unregistered.
     *
     * @see OnFileDescriptorEventListener
     * @see #addOnFileDescriptorEventListener
     */
    public void removeOnFileDescriptorEventListener(@NonNull FileDescriptor fd) {
        if (fd == null) {
            throw new IllegalArgumentException("fd must not be null");
        }

        synchronized (this) {
            updateOnFileDescriptorEventListenerLocked(fd, 0, null);
        }
    }

    private void updateOnFileDescriptorEventListenerLocked(FileDescriptor fd, int events,
            OnFileDescriptorEventListener listener) {
        final int fdNum = fd.getInt$();

        int index = -1;
        FileDescriptorRecord record = null;
        if (mFileDescriptorRecords != null) {
            index = mFileDescriptorRecords.indexOfKey(fdNum);
            if (index >= 0) {
                record = mFileDescriptorRecords.valueAt(index);
                if (record != null && record.mEvents == events) {
                    return;
                }
            }
        }

        if (events != 0) {
            events |= OnFileDescriptorEventListener.EVENT_ERROR;
            if (record == null) {
                if (mFileDescriptorRecords == null) {
                    mFileDescriptorRecords = new SparseArray<FileDescriptorRecord>();
                }
                record = new FileDescriptorRecord(fd, events, listener);
                mFileDescriptorRecords.put(fdNum, record);
            } else {
                record.mListener = listener;
                record.mEvents = events;
                record.mSeq += 1;
            }
            nativeSetFileDescriptorEvents(mPtr, fdNum, events);
        } else if (record != null) {
            record.mEvents = 0;
            mFileDescriptorRecords.removeAt(index);
            nativeSetFileDescriptorEvents(mPtr, fdNum, 0);
        }
    }

    // Called from native code.
    @UnsupportedAppUsage
    private int dispatchEvents(int fd, int events) {
        // Get the file descriptor record and any state that might change.
        final FileDescriptorRecord record;
        final int oldWatchedEvents;
        final OnFileDescriptorEventListener listener;
        final int seq;
        synchronized (this) {
            record = mFileDescriptorRecords.get(fd);
            if (record == null) {
                return 0; // spurious, no listener registered
            }

            oldWatchedEvents = record.mEvents;
            events &= oldWatchedEvents; // filter events based on current watched set
            if (events == 0) {
                return oldWatchedEvents; // spurious, watched events changed
            }

            listener = record.mListener;
            seq = record.mSeq;
        }

        // Invoke the listener outside of the lock.
        int newWatchedEvents = listener.onFileDescriptorEvents(
                record.mDescriptor, events);
        if (newWatchedEvents != 0) {
            newWatchedEvents |= OnFileDescriptorEventListener.EVENT_ERROR;
        }

        // Update the file descriptor record if the listener changed the set of
        // events to watch and the listener itself hasn't been updated since.
        if (newWatchedEvents != oldWatchedEvents) {
            synchronized (this) {
                int index = mFileDescriptorRecords.indexOfKey(fd);
                if (index >= 0 && mFileDescriptorRecords.valueAt(index) == record
                        && record.mSeq == seq) {
                    record.mEvents = newWatchedEvents;
                    if (newWatchedEvents == 0) {
                        mFileDescriptorRecords.removeAt(index);
                    }
                }
            }
        }

        // Return the new set of events to watch for native code to take care of.
        return newWatchedEvents;
    }

// --------------------同步屏障相关 BEGIN--------------------------------
    /** 添加同步屏障 */
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());// 时间戳用的都是系统开时长
    }
		// 同设置步屏障，向消息队列中插入一个handler为空的Message
  	// when：触发屏障的时间，触发就会去找同步消息执行
    private int postSyncBarrier(long when) {
        synchronized (this) {
            final int token = mNextBarrierToken++;// 
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;// 将arg1 属性赋值为token，移除的时候用的
            Message prev = null;
            Message p = mMessages;
            if (when != 0) {// 用when排序放入链表中
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) {
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }

    /**
     * 移除同步消息屏障.
     * @param token 根据arg1与token是否相等来移除同步消息屏障,重新nativeWake
     * @hide  APP隐藏
     */
    @TestApi
    public void removeSyncBarrier(int token) {
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization " + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
// --------------------同步屏障相关 END--------------------------------

// ================== 根据传值的不同，判断是否包含（不同的）消息 BEGIN===============
		// 判断消息队列中是否含有指定what和object的消息
    boolean hasMessages(Handler h, int what, Object object) {
        if (h == null) {
            return false;
        }
      	// 单链表遍历
        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.what == what && (object == null || p.obj == object)){
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
    @UnsupportedAppUsage
    boolean hasMessages(Handler h, Runnable r, Object object) {
        if (h == null) {
            return false;
        }
        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h && p.callback == r && (object == null || p.obj == object){
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }

    boolean hasMessages(Handler h) {
        if (h == null) {
            return false;
        }
        synchronized (this) {
            Message p = mMessages;
            while (p != null) {
                if (p.target == h) {
                    return true;
                }
                p = p.next;
            }
            return false;
        }
    }
// ================== 根据传值的不同，判断是否包含（不同的）消息 END===============
// ================== 移除特定消息 BENGIN===============                    
    void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

           	// 先确定链表头
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // 再过滤其他位置
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

    void removeMessages(Handler h, Runnable r, Object object) {
        if (h == null || r == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h && p.callback == r
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.callback == r
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }

    void removeCallbacksAndMessages(Handler h, Object object) {
        if (h == null) {
            return;
        }

        synchronized (this) {
            Message p = mMessages;

            // Remove all messages at front.
            while (p != null && p.target == h
                    && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
		// 遍历回收所有的Message
    private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }
		// 使loop()方法在处理完消息队列中所有要传递的剩余消息后立即终止。
    // 但是，在循环终止之前，将不会传递将来具有到期时间的挂起延迟消息。
    private void removeAllFutureMessagesLocked() {
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;
        if (p != null) {
            if (p.when > now) {// 如果链表头的时间为未来时
                removeAllMessagesLocked();// 正常移除所有消息
            } else {
                Message n;
                for (;;) {
                    n = p.next;// 获取下一次需要传递的消息
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {// 下一次为未来时
                        break;
                    }
                    p = n;// 如果为过去时，则把它赋值给p
                }
               // 循环结束则找到了所有过去时 的最后一个消息
           		 // 此时，p为倒数第二个过去时消息,n为最后一个过去时消息
                p.next = null;
               // 移除所有未来时的消息
                do {
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }
// ==================  移除特定消息 END===============       
    // 打印当前MessageQueue状态
    void dump(Printer pw, String prefix, Handler h) {
        synchronized (this) {
            long now = SystemClock.uptimeMillis();
            int n = 0;
            for (Message msg = mMessages; msg != null; msg = msg.next) {
                if (h == null || h == msg.target) {
                    pw.println(prefix + "Message " + n + ": " + msg.toString(now));
                }
                n++;
            }
            pw.println(prefix + "(Total messages: " + n + ", polling=" + isPollingLocked()
                    + ", quitting=" + mQuitting + ")");
        }
    }

    void writeToProto(ProtoOutputStream proto, long fieldId) {
        final long messageQueueToken = proto.start(fieldId);
        synchronized (this) {
            for (Message msg = mMessages; msg != null; msg = msg.next) {
                msg.writeToProto(proto, MessageQueueProto.MESSAGES);
            }
            proto.write(MessageQueueProto.IS_POLLING_LOCKED, isPollingLocked());
            proto.write(MessageQueueProto.IS_QUITTING, mQuitting);
        }
        proto.end(messageQueueToken);
    }



    /**
     * A listener which is invoked when file descriptor related events occur.
     */
    public interface OnFileDescriptorEventListener {
        /**
         * File descriptor event: Indicates that the file descriptor is ready for input
         * operations, such as reading.
         * <p>
         * The listener should read all available data from the file descriptor
         * then return <code>true</code> to keep the listener active or <code>false</code>
         * to remove the listener.
         * </p><p>
         * In the case of a socket, this event may be generated to indicate
         * that there is at least one incoming connection that the listener
         * should accept.
         * </p><p>
         * This event will only be generated if the {@link #EVENT_INPUT} event mask was
         * specified when the listener was added.
         * </p>
         */
        public static final int EVENT_INPUT = 1 << 0;

        /**
         * File descriptor event: Indicates that the file descriptor is ready for output
         * operations, such as writing.
         * <p>
         * The listener should write as much data as it needs.  If it could not
         * write everything at once, then it should return <code>true</code> to
         * keep the listener active.  Otherwise, it should return <code>false</code>
         * to remove the listener then re-register it later when it needs to write
         * something else.
         * </p><p>
         * This event will only be generated if the {@link #EVENT_OUTPUT} event mask was
         * specified when the listener was added.
         * </p>
         */
        public static final int EVENT_OUTPUT = 1 << 1;

        /**
         * File descriptor event: Indicates that the file descriptor encountered a
         * fatal error.
         * <p>
         * File descriptor errors can occur for various reasons.  One common error
         * is when the remote peer of a socket or pipe closes its end of the connection.
         * </p><p>
         * This event may be generated at any time regardless of whether the
         * {@link #EVENT_ERROR} event mask was specified when the listener was added.
         * </p>
         */
        public static final int EVENT_ERROR = 1 << 2;

        /** @hide */
        @Retention(RetentionPolicy.SOURCE)
        @IntDef(flag = true, prefix = { "EVENT_" }, value = {
                EVENT_INPUT,
                EVENT_OUTPUT,
                EVENT_ERROR
        })
        public @interface Events {}

        /**
         * Called when a file descriptor receives events.
         *
         * @param fd The file descriptor.
         * @param events The set of events that occurred: a combination of the
         * {@link #EVENT_INPUT}, {@link #EVENT_OUTPUT}, and {@link #EVENT_ERROR} event masks.
         * @return The new set of events to watch, or 0 to unregister the listener.
         *
         * @see #EVENT_INPUT
         * @see #EVENT_OUTPUT
         * @see #EVENT_ERROR
         */
        @Events int onFileDescriptorEvents(@NonNull FileDescriptor fd, @Events int events);
    }

    private static final class FileDescriptorRecord {
        public final FileDescriptor mDescriptor;
        public int mEvents;
        public OnFileDescriptorEventListener mListener;
        public int mSeq;

        public FileDescriptorRecord(FileDescriptor descriptor,
                int events, OnFileDescriptorEventListener listener) {
            mDescriptor = descriptor;
            mEvents = events;
            mListener = listener;
        }
    }
}
```





## Looper



```java
/**
 * 线程的死循环，保存在ThreadLocal里。构造函数中创建消息队列，持有其引用
 */
public final class Looper {
    private static final String TAG = "Looper";
    // 执行prepare()后，通过sThreadLocal.get() ，可以获取Looper实例.
    @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    @UnsupportedAppUsage
    private static Looper sMainLooper;  // 主线程的Looper
    private static Observer sObserver;  // 监听消息处理，APP隐藏
    @UnsupportedAppUsage
    final MessageQueue mQueue;// 消息队列引用，构造时创建
    final Thread mThread;// 所属线程
    @UnsupportedAppUsage
    private Printer mLogging;// 打印的接口,setMessageLogging()设置，null不打印
    private long mTraceTag;// systrace相关，用于收集系统级数据，可用于性能调试
    /** 如果设置，如果消息发送时间超过此阀值，则Looper将显示警告日志。 */
    private long mSlowDispatchThresholdMs;
    /**  如果设置了，则如果消息传递（实际传递时间-发布时间）比此时间长，则Looper将显示警告日志。 */
    private long mSlowDeliveryThresholdMs;

    // 准备（创建looper）；@params quitAllowed 能不能退出，主线程不能退出
  	private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {// prepare()只能执行一次
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));// 通过ThreadLocal保存
    }
  	// 构造
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);// 创建队列
        mThread = Thread.currentThread();// 保存所属线程
    }  
   /**
    * 开始循环，轮消息队列
    * 调用{@link #quit()} 可停止循环
    * 1、从消息队列中循环阻塞拿msg，触发handleMessage()
    * 2、自定义日志输出回调&系统的消息处理超时日志打印
    * 3、systrace相关，用于收集系统级数据，可用于性能调试
    * 4、触发Observer观察者的回调，对APP隐藏，todo这东西干什么用的？
    */
    public static void loop() {
        final Looper me = myLooper();// 当前线程的looper
        if (me == null) {// 没有调用prepare()抛异常
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;// 消息队列
        // 清空远程调用端的uid和pid，用当前本地进程的uid和pid替代；todo？
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
      	// 主线程阻塞logcat会有警告日志，这个是时间阀值（猜到）
        // 下面adb命令是源码注释，作用修改/system/build.prop 属性，对手机一些配置进行设置.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;
				// 死循环
        for (;;) {
            Message msg = queue.next(); // 1、从MessageQueue阻塞中获取消息
            if (msg == null) {
                // 没有消息，说明消息队列退出了，结束死循环.
                return;
            }
            // 开发者自定义的打印接口
            final Printer logging = me.mLogging;
            if (logging != null) {// 设置了才会打印
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);// 拿到一个消息时会打印
            }
            // Make sure the observer won't change while processing a transaction.
            final Observer observer = sObserver;
						// systrace相关，用于收集系统级数据
            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;// 消息发送超时
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;// 消息交货超时
            if (thresholdOverride > 0) {// 上面adb获取的阀值
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);
						// 搜不搜集派发开始时间
            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;// 搜不搜集派发结束时间
						// systrace相关，用于收集系统级数据
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
						// msg开始派发的时间
            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;	
            final long dispatchEnd;// 派发结束的时间
            Object token = null;
            if (observer != null) {
                token = observer.messageDispatchStarting();// 触发回调，处理消息前
            }
            long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
          	try {
               // 2、使用Message中存的Handler处理消息，即调用Handler.handlerMeassage(msg)
                msg.target.dispatchMessage(msg);
                if (observer != null) {
                    observer.messageDispatched(token, msg);// 触发回调，处理消息后
                }
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;// 派发结束了
            } catch (Exception exception) {
                if (observer != null) {
                  // 触发回调，处理消息发生异常时调用
                    observer.dispatchingThrewException(token, msg, exception);
                }
                throw exception;
            } finally {
                ThreadLocalWorkSource.restore(origWorkSource);
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
          	// 超时日志输出
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery", msg)) {
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }
						// 触发日志输出
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            // 清空远程调用端的uid和pid，用当前本地进程的uid和pid替代；.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
						// 3、回收消息 , 放回对象池
            msg.recycleUnchecked();
        }
    }
   	/** 
   	 *退出Looper，队列中未处理的消息不处理。
   	 *会使MesageQueue的 mQuitting 置为 true， next()返回null，从而结束循环 
   	 */
    public void quit() {
        mQueue.quit(false);
    }
 		/** 关闭Looper，未处理且到期的消息会处理 */
    public void quitSafely() {
        mQueue.quit(true);
    }
/*============get相关 BEGIN==============*/
    /** 获取当前线程的Looper */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
    /** 获取当前Looper的消息队列（静态）  */
    public static @NonNull MessageQueue myQueue() {
        return myLooper().mQueue;
    }
    /** 获取当前Looper的消息队列  */
    public @NonNull MessageQueue getQueue() {
        return mQueue;
    }
    /** 调用线程？=Looper的线程 */
    public boolean isCurrentThread() {
        return Thread.currentThread() == mThread;
    }
    /** 获取Looper的线程 */
    public @NonNull Thread getThread() {
        return mThread;
    }
/*============get相关 END==============*/  
    /**
     * 控制分发消息时的日志打印，handleMessage()前后各触发一次
     * @param printer 接口，自定义如何打印.
     */
    public void setMessageLogging(@Nullable Printer printer) {
        mLogging = printer;
    }
  	// 消息处理超时，日志输出信息
    private static boolean showSlowLog(long threshold, long measureStart, long measureEnd,
            String what, Message msg) {
        final long actualTime = measureEnd - measureStart;
        if (actualTime < threshold) {
            return false;
        }
        Slog.w(TAG, "Slow " + what + " took " + actualTime + "ms "
                + Thread.currentThread().getName() + " h="
                + msg.target.getClass().getName() + " c=" + msg.callback + " m=" +msg.what);
        return true;
    }

    /**设置消息超时阀值，app隐藏*/
    public void setSlowLogThresholdMs(long slowDispatchThresholdMs, long slowDeliveryThresholdMs) {
        mSlowDispatchThresholdMs = slowDispatchThresholdMs;
        mSlowDeliveryThresholdMs = slowDeliveryThresholdMs;
    }

   /** 初始化主线程Looper，已经被系统调用过了，基本用不到 */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
              throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
    /** 获取主线程Looper */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
    /** 初始化Looper */
    public static void prepare() {
        prepare(true);
    }
    /** 打印  */
    public void dump(@NonNull Printer pw, @NonNull String prefix) {
        pw.println(prefix + toString());
        mQueue.dump(pw, prefix + "  ", null);
    }

    /** 打印日志 @hide */
    public void dump(@NonNull Printer pw, @NonNull String prefix, Handler handler) {
        pw.println(prefix + toString());
        mQueue.dump(pw, prefix + "  ", handler);
    }
    @Override
    public String toString() {
        return "Looper (" + mThread.getName() + ", tid " + mThread.getId()
                + ") {" + Integer.toHexString(System.identityHashCode(this)) + "}";
    }
//--------------------下面的看不懂-------------------  
  	// systrace相关，用于收集系统级数据，可用于性能调试{@hide} 
    @UnsupportedAppUsage
    public void setTraceTag(long traceTag) {
        mTraceTag = traceTag;
    }
    /** @hide */
    public void writeToProto(ProtoOutputStream proto, long fieldId) {
        final long looperToken = proto.start(fieldId);
        proto.write(LooperProto.THREAD_NAME, mThread.getName());
        proto.write(LooperProto.THREAD_ID, mThread.getId());
        if (mQueue != null) {
            mQueue.writeToProto(proto, LooperProto.QUEUE);
        }
        proto.end(looperToken);
    }  
    /**
     * Set the transaction observer for all Loopers in this process.
     *
     * @hide 隐藏
     */
    public static void setObserver(@Nullable Observer observer) {
        sObserver = observer;
    }
    /** {@hide}APP隐藏 ，消息处理的监听 */
    public interface Observer {
        /** handlerMessage()前调用  */
        Object messageDispatchStarting();

         /** handlerMessage()后调用  */
        void messageDispatched(Object token, Message msg);

        /** handlerMessage()发生异常调用 */
        void dispatchingThrewException(Object token, Message msg, Exception exception);
    }
}
```



## Message

```java
// 消息实体类，时间戳用的是开机时间
public final class Message implements Parcelable {
    /** 叫message code，应该是消息的标识，存整数用arg1、arg2比较合适 */
    public int what;
    public int arg1;//存个整数1
    public int arg2;// 存个整数2
    public Object obj;// 存个对象
    /** IPC的Messager用的，大概保存是send给谁   */
    public Messenger replyTo;
    /**  表示没有设置uid; @hide 隐藏，只有系统的服务用到.  */
    public static final int UID_NONE = -1;
    /** 正在使用的标识，第一位是1。加入队列后--回收之前 */
    /*package*/ static final int FLAG_IN_USE = 1 << 0;
    /** 异步消息标识，第二位是1 */
    /*package*/ static final int FLAG_ASYNCHRONOUS = 1 << 1;
    @UnsupportedAppUsage
    /*package*/ int flags;// 用1个整数记录标识，类似view的flags
    @UnsupportedAppUsage
    @VisibleForTesting(visibility = VisibleForTesting.Visibility.PACKAGE)
    public long when;// msg在什么时间执行，绝对时间，用的是开机时长的时间戳
    /*package*/ Bundle data;// 存个bundle
    @UnsupportedAppUsage
    /*package*/ Handler target;// msg对应的target
    @UnsupportedAppUsage
    /*package*/ Runnable callback; // Handler#post(Runnable)保存在这里
    @UnsupportedAppUsage
    /*package*/ Message next;// 对象池数据结构为单链表，单链表的索引
    private static Message sPool;// msg的对象池，单链表
    private static int sPoolSize = 0;// 对象池当前大小
    private static final int MAX_POOL_SIZE = 50;// 对象池大小阀值

/*============从对象池里获取消息相关方法 BEGIN==============*/  
    /** 从msg的对象池里获取msg */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    /**
     * 深拷贝消息.
     * @param 原始消息.
     * @return 对象池里的msg，内容来自原始msg深拷贝.
     */
    public static Message obtain(Message orig) {
        Message m = obtain();
        m.what = orig.what;
        m.arg1 = orig.arg1;
        m.arg2 = orig.arg2;
        m.obj = orig.obj;
        m.replyTo = orig.replyTo;
        m.sendingUid = orig.sendingUid;
        m.workSourceUid = orig.workSourceUid;
        if (orig.data != null) {
            m.data = new Bundle(orig.data);
        }
        m.target = orig.target;
        m.callback = orig.callback;
        return m;
    }
    /** 获取，并指定handler  */
    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;
        return m;
    }
    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        m.callback = callback;
        return m;
    }
    public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        return m;
    }
    public static Message obtain(Handler h, int what, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.obj = obj;
        return m;
    }
    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        return m;
    }
    public static Message obtain(Handler h, int what,
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;
        return m;
    }
/*============从对象池里获取消息相关方法 END==============*/  
    /** 把msg放回对象池，msg用完了才能使用  */
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "+ "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    /** 回收msg放入对象池，并清楚其数据  */
    @UnsupportedAppUsage
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
    /** 将o深拷贝到this  */
    public void copyFrom(Message o) {
        this.flags = o.flags & ~FLAGS_TO_CLEAR_ON_COPY_FROM;
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;
        this.sendingUid = o.sendingUid;
        this.workSourceUid = o.workSourceUid;
        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }
    /** 获取msg的执行时间  */
    public long getWhen() {
        return when;
    }
		// 设置handler
    public void setTarget(Handler target) {
        this.target = target;
    }
    /** 获取handler  */
    public Handler getTarget() {
        return target;
    }
    /** 获取Runnable   */
    public Runnable getCallback() {
        return callback;
    }
    /** 设置Runnbale @hide隐藏 */
    @UnsupportedAppUsage
    public Message setCallback(Runnable r) {
        callback = r;
        return this;
    }
    /** 获取保存的bundle数据，没有会new一个  */
    public Bundle getData() {
        if (data == null) {
            data = new Bundle();
        }
        return data;
    }
    /** 获取保存的bundle数据，没有会不new一个  */
    public Bundle peekData() {
        return data;
    }
    /**设置bundle数据  */
    public void setData(Bundle data) {
        this.data = data;
    }
    /** 设置what，隐藏，可使用Message#what=xxx */
    public Message setWhat(int what) {
        this.what = what;
        return this;
    }
    /** 使用handler发送msg  */
    public void sendToTarget() {
        target.sendMessage(this);
    }
    /** ture异步消息；false同步消息 */
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
    }
    /**  ture异步消息；false同步消息 */
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
		// 判断msg正在使用
    /*package*/ boolean isInUse() {
        return ((flags & FLAG_IN_USE) == FLAG_IN_USE);
    }
		// 标记为正在使用
    @UnsupportedAppUsage
    /*package*/ void markInUse() {
        flags |= FLAG_IN_USE;
    }
    /** 构造，推荐使用{@link #obtain() Message.obtain()}). */
    public Message() {
    }

// -----------------------看不懂 | 不重要--------------------------------
    /** Flags to clear in the copyFrom method */
    /*package*/ static final int FLAGS_TO_CLEAR_ON_COPY_FROM = FLAG_IN_USE;
    /** @hide */
    public static final Object sPoolSync = new Object();
    private static boolean gCheckRecycle = true;
  	/**
     * Optional field indicating the uid that sent the message.  This is
     * only valid for messages posted by a {@link Messenger}; otherwise,
     * it will be -1.
     */
    public int sendingUid = UID_NONE;

    /**
     * Optional field indicating the uid that caused this message to be enqueued.
     *
     * @hide Only for use within the system server.
     */
    public int workSourceUid = UID_NONE;
    /** @hide */
    public static void updateCheckRecycle(int targetSdkVersion) {
        if (targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            gCheckRecycle = false;
        }
    }
    @Override
    public String toString() {
        return toString(SystemClock.uptimeMillis());
    }

    @UnsupportedAppUsage
    String toString(long now) {
        StringBuilder b = new StringBuilder();
        b.append("{ when=");
        TimeUtils.formatDuration(when - now, b);

        if (target != null) {
            if (callback != null) {
                b.append(" callback=");
                b.append(callback.getClass().getName());
            } else {
                b.append(" what=");
                b.append(what);
            }

            if (arg1 != 0) {
                b.append(" arg1=");
                b.append(arg1);
            }

            if (arg2 != 0) {
                b.append(" arg2=");
                b.append(arg2);
            }

            if (obj != null) {
                b.append(" obj=");
                b.append(obj);
            }

            b.append(" target=");
            b.append(target.getClass().getName());
        } else {
            b.append(" barrier=");
            b.append(arg1);
        }

        b.append(" }");
        return b.toString();
    }

    void writeToProto(ProtoOutputStream proto, long fieldId) {
        final long messageToken = proto.start(fieldId);
        proto.write(MessageProto.WHEN, when);

        if (target != null) {
            if (callback != null) {
                proto.write(MessageProto.CALLBACK, callback.getClass().getName());
            } else {
                proto.write(MessageProto.WHAT, what);
            }

            if (arg1 != 0) {
                proto.write(MessageProto.ARG1, arg1);
            }

            if (arg2 != 0) {
                proto.write(MessageProto.ARG2, arg2);
            }

            if (obj != null) {
                proto.write(MessageProto.OBJ, obj.toString());
            }

            proto.write(MessageProto.TARGET, target.getClass().getName());
        } else {
            proto.write(MessageProto.BARRIER, arg1);
        }

        proto.end(messageToken);
    }
		// ========================序列化 BEGIN==========================
    public static final @android.annotation.NonNull Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }

        public Message[] newArray(int size) {
            return new Message[size];
        }
    };
		
    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) {
            throw new RuntimeException(
                "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable)obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                    "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
        dest.writeLong(when);
        dest.writeBundle(data);
        Messenger.writeMessengerOrNullToParcel(replyTo, dest);
        dest.writeInt(sendingUid);
        dest.writeInt(workSourceUid);
    }

    private void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
        when = source.readLong();
        data = source.readBundle();
        replyTo = Messenger.readMessengerOrNullFromParcel(source);
        sendingUid = source.readInt();
        workSourceUid = source.readInt();
    }
	// ========================序列化 END==========================  
}
```



##  android_os_MessageQueue.cpp



```java
#include "android_os_MessageQueue.h"
//构造
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}  
// 对应：private native static long nativeInit(); 创建消息队列
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();// 创建消息队列
    if (!nativeMessageQueue) {// 创建失败抛异常
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }
    nativeMessageQueue->incStrong(env);// 内存计数+
    return reinterpret_cast<jlong>(nativeMessageQueue);// 返回消息队列的指针
}  
//  销毁队列：private native static void nativeDestroy(long ptr);
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
  	// 通过指针获取消息队列对象
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env);// 内存计数-
}
//  private native void nativePollOnce(long ptr, int timeoutMillis);
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);// 调用native的looper的pollOnce
    mPollObj = NULL;
    mPollEnv = NULL;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
// 唤醒：private native static void nativeWake(long ptr);
void NativeMessageQueue::wake() {
    mLooper->wake();
}

//  private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
void NativeMessageQueue::setFileDescriptorEvents(int fd, int events) {
    if (events) {
        int looperEvents = 0;
        if (events & CALLBACK_EVENT_INPUT) {
            looperEvents |= Looper::EVENT_INPUT;
        }
        if (events & CALLBACK_EVENT_OUTPUT) {
            looperEvents |= Looper::EVENT_OUTPUT;
        }
        mLooper->addFd(fd, Looper::POLL_CALLBACK, looperEvents, this,
                reinterpret_cast<void*>(events));
    } else {
        mLooper->removeFd(fd);
    }
}




namespace android {
static struct {
    jfieldID mPtr;   // native object attached to the DVM MessageQueue
    jmethodID dispatchEvents;
} gMessageQueueClassInfo;
// Must be kept in sync with the constants in Looper.FileDescriptorCallback
static const int CALLBACK_EVENT_INPUT = 1 << 0;
static const int CALLBACK_EVENT_OUTPUT = 1 << 1;
static const int CALLBACK_EVENT_ERROR = 1 << 2;
class NativeMessageQueue : public MessageQueue, public LooperCallback {
public:
    NativeMessageQueue();
    virtual ~NativeMessageQueue();
    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj);
    void pollOnce(JNIEnv* env, jobject obj, int timeoutMillis);
    void wake();
    void setFileDescriptorEvents(int fd, int events);
    virtual int handleEvent(int fd, int events, void* data);
private:
    JNIEnv* mPollEnv;
    jobject mPollObj;
    jthrowable mExceptionObj;
};
MessageQueue::MessageQueue() {
}
MessageQueue::~MessageQueue() {
}
bool MessageQueue::raiseAndClearException(JNIEnv* env, const char* msg) {
    if (env->ExceptionCheck()) {
        jthrowable exceptionObj = env->ExceptionOccurred();
        env->ExceptionClear();
        raiseException(env, msg, exceptionObj);
        env->DeleteLocalRef(exceptionObj);
        return true;
    }
    return false;
}

NativeMessageQueue::~NativeMessageQueue() {
}
void NativeMessageQueue::raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj) {
    if (exceptionObj) {
        if (mPollEnv == env) {
            if (mExceptionObj) {
                env->DeleteLocalRef(mExceptionObj);
            }
            mExceptionObj = jthrowable(env->NewLocalRef(exceptionObj));
            ALOGE("Exception in MessageQueue callback: %s", msg);
            jniLogException(env, ANDROID_LOG_ERROR, LOG_TAG, exceptionObj);
        } else {
            ALOGE("Exception: %s", msg);
            jniLogException(env, ANDROID_LOG_ERROR, LOG_TAG, exceptionObj);
            LOG_ALWAYS_FATAL("raiseException() was called when not in a callback, exiting.");
        }
    }
}



int NativeMessageQueue::handleEvent(int fd, int looperEvents, void* data) {
    int events = 0;
    if (looperEvents & Looper::EVENT_INPUT) {
        events |= CALLBACK_EVENT_INPUT;
    }
    if (looperEvents & Looper::EVENT_OUTPUT) {
        events |= CALLBACK_EVENT_OUTPUT;
    }
    if (looperEvents & (Looper::EVENT_ERROR | Looper::EVENT_HANGUP | Looper::EVENT_INVALID)) {
        events |= CALLBACK_EVENT_ERROR;
    }
    int oldWatchedEvents = reinterpret_cast<intptr_t>(data);
    int newWatchedEvents = mPollEnv->CallIntMethod(mPollObj,
            gMessageQueueClassInfo.dispatchEvents, fd, events);
    if (!newWatchedEvents) {
        return 0; // unregister the fd
    }
    if (newWatchedEvents != oldWatchedEvents) {
        setFileDescriptorEvents(fd, newWatchedEvents);
    }
    return 1;
}
// ----------------------------------------------------------------------------
sp<MessageQueue> android_os_MessageQueue_getMessageQueue(JNIEnv* env, jobject messageQueueObj) {
    jlong ptr = env->GetLongField(messageQueueObj, gMessageQueueClassInfo.mPtr);
    return reinterpret_cast<NativeMessageQueue*>(ptr);
}


static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
static jboolean android_os_MessageQueue_nativeIsPolling(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->getLooper()->isPolling();
}
static void android_os_MessageQueue_nativeSetFileDescriptorEvents(JNIEnv* env, jclass clazz,
        jlong ptr, jint fd, jint events) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->setFileDescriptorEvents(fd, events);
}
// ----------------------------------------------------------------------------
static const JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()J", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "(J)V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(JI)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(J)V", (void*)android_os_MessageQueue_nativeWake },
    { "nativeIsPolling", "(J)Z", (void*)android_os_MessageQueue_nativeIsPolling },
    { "nativeSetFileDescriptorEvents", "(JII)V",
            (void*)android_os_MessageQueue_nativeSetFileDescriptorEvents },
};
int register_android_os_MessageQueue(JNIEnv* env) {
    int res = RegisterMethodsOrDie(env, "android/os/MessageQueue", gMessageQueueMethods,
                                   NELEM(gMessageQueueMethods));
    jclass clazz = FindClassOrDie(env, "android/os/MessageQueue");
    gMessageQueueClassInfo.mPtr = GetFieldIDOrDie(env, clazz, "mPtr", "J");
    gMessageQueueClassInfo.dispatchEvents = GetMethodIDOrDie(env, clazz,
            "dispatchEvents", "(II)I");
    return res;
}
} // namespace android
```



## Looper.cpp

```java
namespace android {
// 唤醒  nativeWake
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif
    ssize_t nWrite;// 适于计量内存中可容纳的数据项目个数的无符号整数类型
    do {
      	// 向管道的写端写入一个字符“W”，这样管道的读端就会因为有数据可读，从等待状态中醒来
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);
    if (nWrite != 1) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}  
// nativePollOnce
// @params timeoutMillis 超时等待时间。如-1无限等待至有事件发生。如0无需等待立即返回
// @params outFd 用来储存发生事件的那个文件描述符（句柄）。
// @params outEvents 用来存储在该文件描述符1上发生的事件，支持可读、可写、错误、中断4种状态。【epoll】
// @params outData 存储上下文数据。添加监听句柄时传递的，用来传递用户自定义的数据
// @return ALOOPER_POLL_WAKE 表示由wake()触发，即管道写端的那次写事件触发
//				 ALOOPER_POLL_TIMEOUT 表示等到超时
//				 ALOOPER_POLL_ERROR 表示等待过程中发生错误
//				 ALOOPER_POLL_CALLBACK 表示某个被监听的句柄因某种原因被触发
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {// 又一个死循环
        while (mResponseIndex < mResponses.size()) {// mResponses是一个Vector
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        result = pollInner(timeoutMillis);
    }
}
  
  
  
// --- WeakMessageHandler ---
WeakMessageHandler::WeakMessageHandler(const wp<MessageHandler>& handler) :
        mHandler(handler) {
}
WeakMessageHandler::~WeakMessageHandler() {
}
void WeakMessageHandler::handleMessage(const Message& message) {
    sp<MessageHandler> handler = mHandler.promote();
    if (handler != NULL) {
        handler->handleMessage(message);
    }
}
// --- SimpleLooperCallback ---
SimpleLooperCallback::SimpleLooperCallback(ALooper_callbackFunc callback) :
        mCallback(callback) {
}
SimpleLooperCallback::~SimpleLooperCallback() {
}
int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    return mCallback(fd, events, data);
}
// --- Looper ---
// Hint for number of file descriptors to be associated with the epoll instance.
static const int EPOLL_SIZE_HINT = 8;
// Maximum number of file descriptors for which to retrieve poll events each iteration.
static const int EPOLL_MAX_EVENTS = 16;
static pthread_once_t gTLSOnce = PTHREAD_ONCE_INIT;
static pthread_key_t gTLSKey = 0;
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    int result = pipe(wakeFds);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);
    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);
    // Allocate the epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;
    eventItem.data.fd = mWakeReadPipeFd;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);
}
Looper::~Looper() {
    close(mWakeReadPipeFd);
    close(mWakeWritePipeFd);
    close(mEpollFd);
}
void Looper::initTLSKey() {
    int result = pthread_key_create(& gTLSKey, threadDestructor);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not allocate TLS key.");
}
void Looper::threadDestructor(void *st) {
    Looper* const self = static_cast<Looper*>(st);
    if (self != NULL) {
        self->decStrong((void*)threadDestructor);
    }
}
void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS
    if (looper != NULL) {
        looper->incStrong((void*)threadDestructor);
    }
    pthread_setspecific(gTLSKey, looper.get());
    if (old != NULL) {
        old->decStrong((void*)threadDestructor);
    }
}
sp<Looper> Looper::getForThread() {
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");
    return (Looper*)pthread_getspecific(gTLSKey);
}
sp<Looper> Looper::prepare(int opts) {
    bool allowNonCallbacks = opts & ALOOPER_PREPARE_ALLOW_NON_CALLBACKS;
    sp<Looper> looper = Looper::getForThread();
    if (looper == NULL) {
        looper = new Looper(allowNonCallbacks);
        Looper::setForThread(looper);
    }
    if (looper->getAllowNonCallbacks() != allowNonCallbacks) {
        ALOGW("Looper already prepared for this thread with a different value for the "
                "ALOOPER_PREPARE_ALLOW_NON_CALLBACKS option.");
    }
    return looper;
}
bool Looper::getAllowNonCallbacks() const {
    return mAllowNonCallbacks;
}

int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif
    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %lldns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }
    // Poll.
    int result = ALOOPER_POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    // Acquire lock.
    mLock.lock();
    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = ALOOPER_POLL_ERROR;
        goto Done;
    }
    // Check for poll timeout.
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = ALOOPER_POLL_TIMEOUT;
        goto Done;
    }
    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= ALOOPER_EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= ALOOPER_EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= ALOOPER_EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= ALOOPER_EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;
    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                handler->handleMessage(message);
            } // release handler
            mLock.lock();
            mSendingMessage = false;
            result = ALOOPER_POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    // Release lock.
    mLock.unlock();
    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == ALOOPER_POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd);
            }
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = ALOOPER_POLL_CALLBACK;
        }
    }
    return result;
}
int Looper::pollAll(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    if (timeoutMillis <= 0) {
        int result;
        do {
            result = pollOnce(timeoutMillis, outFd, outEvents, outData);
        } while (result == ALOOPER_POLL_CALLBACK);
        return result;
    } else {
        nsecs_t endTime = systemTime(SYSTEM_TIME_MONOTONIC)
                + milliseconds_to_nanoseconds(timeoutMillis);
        for (;;) {
            int result = pollOnce(timeoutMillis, outFd, outEvents, outData);
            if (result != ALOOPER_POLL_CALLBACK) {
                return result;
            }
            nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
            timeoutMillis = toMillisecondTimeoutDelay(now, endTime);
            if (timeoutMillis == 0) {
                return ALOOPER_POLL_TIMEOUT;
            }
        }
    }
}

void Looper::awoken() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ awoken", this);
#endif
    char buffer[16];
    ssize_t nRead;
    do {
        nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
    } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
}
void Looper::pushResponse(int events, const Request& request) {
    Response response;
    response.events = events;
    response.request = request;
    mResponses.push(response);
}
int Looper::addFd(int fd, int ident, int events, ALooper_callbackFunc callback, void* data) {
    return addFd(fd, ident, events, callback ? new SimpleLooperCallback(callback) : NULL, data);
}
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ addFd - fd=%d, ident=%d, events=0x%x, callback=%p, data=%p", this, fd, ident,
            events, callback.get(), data);
#endif
    if (!callback.get()) {
        if (! mAllowNonCallbacks) {
            ALOGE("Invalid attempt to set NULL callback but not allowed for this looper.");
            return -1;
        }
        if (ident < 0) {
            ALOGE("Invalid attempt to set NULL callback with ident < 0.");
            return -1;
        }
    } else {
        ident = ALOOPER_POLL_CALLBACK;
    }
    int epollEvents = 0;
    if (events & ALOOPER_EVENT_INPUT) epollEvents |= EPOLLIN;
    if (events & ALOOPER_EVENT_OUTPUT) epollEvents |= EPOLLOUT;
    { // acquire lock
        AutoMutex _l(mLock);
        Request request;
        request.fd = fd;
        request.ident = ident;
        request.callback = callback;
        request.data = data;
        struct epoll_event eventItem;
        memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
        eventItem.events = epollEvents;
        eventItem.data.fd = fd;
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
            if (epollResult < 0) {
                ALOGE("Error adding epoll events for fd %d, errno=%d", fd, errno);
                return -1;
            }
            mRequests.add(fd, request);
        } else {
            int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem);
            if (epollResult < 0) {
                ALOGE("Error modifying epoll events for fd %d, errno=%d", fd, errno);
                return -1;
            }
            mRequests.replaceValueAt(requestIndex, request);
        }
    } // release lock
    return 1;
}
int Looper::removeFd(int fd) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ removeFd - fd=%d", this, fd);
#endif
    { // acquire lock
        AutoMutex _l(mLock);
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            return 0;
        }
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_DEL, fd, NULL);
        if (epollResult < 0) {
            ALOGE("Error removing epoll events for fd %d, errno=%d", fd, errno);
            return -1;
        }
        mRequests.removeItemsAt(requestIndex);
    } // release lock
    return 1;
}
void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now, handler, message);
}
void Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,
        const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now + uptimeDelay, handler, message);
}
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ sendMessageAtTime - uptime=%lld, handler=%p, what=%d",
            this, uptime, handler.get(), message.what);
#endif
    size_t i = 0;
    { // acquire lock
        AutoMutex _l(mLock);
        size_t messageCount = mMessageEnvelopes.size();
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }
        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);
        // Optimization: If the Looper is currently sending a message, then we can skip
        // the call to wake() because the next thing the Looper will do after processing
        // messages is to decide when the next wakeup time should be.  In fact, it does
        // not even matter whether this code is running on the Looper thread.
        if (mSendingMessage) {
            return;
        }
    } // release lock
    // Wake the poll loop only when we enqueue a new message at the head.
    if (i == 0) {
        wake();
    }
}
void Looper::removeMessages(const sp<MessageHandler>& handler) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ removeMessages - handler=%p", this, handler.get());
#endif
    { // acquire lock
        AutoMutex _l(mLock);
        for (size_t i = mMessageEnvelopes.size(); i != 0; ) {
            const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(--i);
            if (messageEnvelope.handler == handler) {
                mMessageEnvelopes.removeAt(i);
            }
        }
    } // release lock
}
void Looper::removeMessages(const sp<MessageHandler>& handler, int what) {
#if DEBUG_CALLBACKS
    ALOGD("%p ~ removeMessages - handler=%p, what=%d", this, handler.get(), what);
#endif
    { // acquire lock
        AutoMutex _l(mLock);
        for (size_t i = mMessageEnvelopes.size(); i != 0; ) {
            const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(--i);
            if (messageEnvelope.handler == handler
                    && messageEnvelope.message.what == what) {
                mMessageEnvelopes.removeAt(i);
            }
        }
    } // release lock
}
} // namespace android

```

