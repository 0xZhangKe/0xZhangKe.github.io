---
layout: post
category: computer
title: "Handler 机制深入理解（Handler、Looper、MessageQueue）"
author: ZhangKe
date:   2017-07-31 22:54:03 +0800
---

## Handler
Handler 主要用于在不同的线程中相互通信，使用场景最多的应该就是在子线程中更新 UI。
与 Handler 相关的类：
**Handler**：处理与发送消息（Message）
**Message**：消息的包装类
**Looper**：整个 Handler 通信的核心，接受 Handler 发送的 Message，循环遍历 MessageQueue 中的 Message，并发送至 Handler 处理
**MessageQueue**：保存 Message
**LocalThread**：存储 Looper 与 Looper所在线程的信息

关系流程图：

![](/assets/img/post/handler/1.webp)

上图简单概括为：**Handler 将 Message post 或者 send 到 MessageQueue 中，Looper 不断循环从 MessageQueue 中取出 Message 交由 Handler 来处理。**
### Handler 消息处理
Handler 处理消息的方法为 dispatchMessage：
```
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
```
**①判断 Message.callback 是否为空，不为空则交给 callback 处理；
②判断 Handler.callback 是否为空，不为空则调用mCallback.handleMessage(msg) 处理；
③如果前面都为空，则调用 Handler.handleMessage(msg) 方法处理。**

所以我们可以通过重写 dispatchMessage方法来改变这种默认的消息分发顺序。

一般来说，我们都是按照如下的方式使用 Message
```
Message message = mHandler.obtainMessage();
message.what = 0;
message.obj = "123";
mHandler.sendMessage(message);
```
我们很少在 Message 中使用 callback，那么 这个 callback究竟又是什么？
### Handler 消息发送

#### post 方式

刚刚说到，Handler 发送 Message 有两类方法：post 和 send 。其中最常用的就是 send 方法了，下面来看一个使用 post 的例子：
```
mHandler.post(new Runnable() {
    @Override
    public void run() {
        //do something
    }
});
```
打开 post(Runnable) 方法：
```
public final boolean post(Runnable r){
    return  sendMessageDelayed(getPostMessage(r), 0);
}
```
其中把  Runnable 参数通过 getPostMessage() 方法包装成了 Message 消息，然后继续使用 send 方式发送到 MessageQueue 中去。
那么继续打开 getPostMessage 方法：
```
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
到这里就很明白了，先获取一个 Message 对象，然后把 Runnable 赋值给 m.callback，继而使用 send 方式发送。也就是说，post 方法本质上也是 send 方法，只不过对 Runnable 做了一层包装。
*PS：我们平时自定义 View 时，如果想要在主线程做某些操作一般会使用 post(Runnable r) 方法，其实这个方法也是调用了 Handler 的 post 方法，将该 Runnable send 到主线程的 MessageQueue 中。*
#### send 方式
通过各种调用之后，最终会调用 sendMessageAtTime 发送消息
```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
先获取到 MessageQueue ，然后判断是否为空，不为空则调用 enqueueMessage 方法
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
这里其实**最终就调用了 MessageQueue 的 enqueueMessage 方法了，这个方法就是使 Message 对象入列。**
## MessageQueue
MessageQueue 就是一个队列，具有一个队列的一般特征：
①元素入队：
```
boolean enqueueMessage(Message msg, long when)
```
②元素出队：
```
Message next()
```
③删除元素：
```
void removeMessages(Handler h, int what, Object object)
void removeMessages(Handler h, Runnable r, Object object)
```
## Looper
在整个 Handler 通信机制中，Looper 才是重头戏，Looper 就好比计算机的 CPU，控制整个通信系统的正常运转。
通过上面的介绍我们知道，Handler 负责发送与处理消息，MessageQueue 负责存储消息，Message 是消息的载体，那么他们之间到底是怎样的结合起来的？当然靠的是 Looper，Looper 通过循环遍历 MessageQueue ，当有消息时就发送到 Handler 中处理。 

道理很简单，但实际可能要稍微复杂点，首先这里需要明确几个概念：
**① 每个 Thread 对应一个 Looper
② 每个 Looper 对应一个 MessageQueue
③ 每个 MessageQueue 对应多个 Message
④ 每个 Message 最多只能指定一个 Handler 处理**

当我们在普通线程中使用 Handler 时，一般方式如下：
```
class LooperThread extends Thread{
    public Handler handler;
    @Override
    public void run() {
        super.run();
        Looper.prepare();//准备工作
        handler = new Handler(){
            @Override
            public void dispatchMessage(Message msg) {
                super.dispatchMessage(msg);
                //处理消息
            }
        };
        Looper.loop();//进入主循环
    }
}
```
到了这里我们大概会有一个疑问，无论是 Handler 的创建还是 Handler.sendMessage() 都没有和 Looper 产生联系，那么当我们使用 Handler 发送消息时，是如何确保这个消息与 Looper 关联起来的呢？

看上面的代码，首先通过 Looper 的静态方法 Looper.prepare(); 开始准备工作，这个方法的核心代码如下：
```
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
先不管 sThreadLocal 是个什么东西，首先判断 sThreadLocal.get() 是否为空，如果为空则抛出一个运行时异常：Only one Looper may be created per thread ，这个异常很常见，当你在一个线程中多次调用 Looper.prepare(); 时就会抛出这个异常，这里保证每个线程只有一个 Looper 对象。如果 sThreadLocal.get() 返回空，则调用 sThreadLocal.set(new Looper(quitAllowed)) 方法，将 Looper 对象放入 sThreadLocal 中，挺简单的几行代码，逻辑也很清晰，那在看看 sThreadLocal 到底是个什么东西。

其在 Looper 中的定义为：
```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
其实到这里我们大概能判断出来 ThreadLocal 是一个用来存储 Looper 的东西。
我们只需要关心上面用到的 sThreadLocal.get 和 sThreadLocal.set 方法，先看 get：
```
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
public T get() {
    Thread t = Thread.currentThread();//先获取当前线程的对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
//把当前线程的对象当做 key，获取 Looper，判断当前线程的 Looper 是否已经存在
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}
```
这个方法主要就是获取当前线程的 Looper 对象，再来看 set 方法：
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
大意就是把当前线程的信息 Looper 保存到 ThreadLocal 中。

到了这我们大概就能明白了，**Looper.prepare 就是在当前线程中创建一个 Looper 对象，并且保证只有一个 Looper 存在。**
可以看到，sThreadLocal 与 Looper.prepare 都是静态方法，Looper 对象也是保存在静态变量中，那么也就是说，当我们的 App 启动时，sThreadLocal 变量就已经存在并且保存了主线程的 Looper，当后面我们在子线程中使用 Handler 时，就会继续把该线程的 Looper对象保存到 sThreadLocal 中。

罗里吧嗦扯了半天，那么回过头来看看刚刚的那个问题，Handler  如何与 Looper 关联起来？
我们来看看 Handler 的默认构造方法，最终调用的构造器代码如下：
```
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
主要在于 11 行和 16 行，调用 Looper.myLooper() 方法将 Looper 对象赋值给当前 Handler 中的 mLooper，在调用 mLooper.mQueue 将 Looper 中的 MessageQueue 赋值给 mQueue。<br>

到了这里整个 Handler 通信的机制就已经很明确了，再来个大概的总结：
**① 使用 Looper.prepare(); 做初始化操作，该方法会在当前线程中创建一个唯一的 Looper 对象；
② 创建 Handler 对象，Handler 在创建时就会与当前线程中的 Looper 对象与 MessageQueue 对象绑定；
③ Looper.loop() 启动循环。**
## ActivityThread 中的 Handler
ActivityThread 与普通线程使用 Looper.prepare(); 不同，这里使用 Looper.prepareMainLooper(); 来初始化：
```
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
这个函数除了会调用 prepare 之外，主要的功能就是将当前线程（也就是主线程）的 Looper 赋值给 sMainLooper 变量，这样就可以使用 Looper.getMainLooper() 获取主线程的 Looper。
另外 Handler 的创建跟普通线程也有一定的区别：
```
if (sMainThreadHandler == null) {
    sMainThreadHandler = thread.getHandler();
}
```
在 ActivityThread 的 mian 函数中，会先实例化一个 ActivityThread 对象，然后通过这个对象的 getHandler 方法获取 Handler。

这个 Handler 用于处理主线程中的各种消息，其中已经定义了很多处理消息的操作，例如启动 Activity 、内存管理等操作，只要发送通过这个 Handler 发送相关的 Message 即可，而 ActivityThread 之所以要讲这个 Handler 赋值给其中的静态变量  sMainThreadHandler 也是因为要通过这个 Handler来发送相关的消息。

初始化结束后就开始进入消息循环（ Looper.loop ）的阶段，这个方法就是用于不停地接受 Handler 发送过来的各种消息，并交给 Handler 来处理：
```
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }
        msg.recycleUnchecked();
    }
}
```
这里就是开始一个死循环，看其中的 36 行，接受到消息后交给 Handler 的 dispatchMessage 方法来处理。

END
## 参考文献
[1]林学森. 深入理解Android内核设计思想[M]. 北京, 2014.