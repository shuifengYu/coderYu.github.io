---
layout: post
title:  "Android消息分发及多线程切换之Handler、Message的细枝末节（二）"
date:   2016-06-06 23:10:00 +0800
categories: Blog
---
之前说了Android消息分发和多线程切换的核心知识点，这次来说一下消息传递的整个过程，好像上一篇内容不看貌似也可以直接看这篇，不过建议还是看一下可以了解的更透彻一点吧~~

上一篇请看：http://www.jianshu.com/p/e914cda1b5fe

下面开始本篇的主题吧，我将按照Handler和Message的一般使用流程，跟着代码调用链一步步往下走，所以，看代码吧！


首先在主线程定义一个handler：

```
	Handler handler = new Handler(){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what){
                    case TEST:
                        handleTest(msg);
                        break;
                    default:
                        handleDefault(msg);
                        break;
                }
            }
       };
```
然后在工作线程中使用拿到handler的引用，直接使用：

```
	new Thread(){
            @Override
            public void run() {
                super.run();
                Message message = Message.obtain();
                message.what = TEST;
                Bundle bundle =new Bundle();
                bundle.putString("test key","test value");
                message.setData(bundle);
       			handler.sendMessage(msg);                
            }
        }.start();
```
以上即为一个简单使用场景，在工作线程中通过将Message发送到主线程的handler，让handler在主线程中处理UI相关的操作。

我们从第一段代码开始，先看一下Handler的创建过程。很简单，直接new一个Handler对象，然后实现它的**handleMessage(Message msg)**方法，但是这个handler可不是随便哪里都能创建的，相信很多同学肯定也碰到过在别的线程中创建handler报错的情况，类似`Can't create handler inside thread that has not called Looper.prepare()`这样的错误提示，为啥呢？

```
	/**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }
    
   
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

最终调用到的构造器中的第一处判断是用来检测内存泄露相关的，我们不管，看**mLooper**变量的赋值：

```
	/**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
**sThreadLocal.get()**方法上一篇我们讲过了，**sThreadLocal**是一个全局的静态变量，保存着线程对应的**looper**对象，**get()**方法用来获取当前线程对应的**looper**对象，由于我们这个**handler**是在主线程中创建的，而主线程在**ActivityThread**的**main()**方法中已经为我们创建了对应的**looper**对象，因此我们这里去get的话是能取到值的，所以下面一行的判断不会抛异常。但是一般的工作线程，我们如果没用为它创建**Looper**对象，它是没有对应的**Looper**对象保存在**sThreadLocal**中的，因此**Looper.myLooper()**就会返回null，接下去就会报错，也就是上面说的在工作线程中创建handler会报错的原因。取到**mLooper**之后，将**mLooper**中的保存**Message**的**MessageQueue**也取出保存在**handler**的属性**mQueue**中，然后将构造器的参数**callback**和**async**赋值，我们这里都为null。

这里大致介绍下**Handler**的几个属性：

* **boolean mAsynchronous：**

	这个是用来表明处理**Message**的时候是否需要保证顺序，默认为false，而且我们其实也没办法去将他设置为true，因为这些构造器都是`@hide`的，所以我们可以不管它。
* **Callback mCallback：**

	这个是Handler的内部接口**Callback**类型的一个属性，它就一个方法，也叫
**handleMessage(Message msg)**,只能在构造器中传参，如果我们在构造器中给了这个参数，那么发送给这个**handler**的消息将优先会由这个**mCallback**来执行，不过目前为止我并不清楚哪些场景适用这种方式，知道的同学欢迎指教一下~
* **Looper mLooper:**

	**handler**是用来处理**Message**的，因此我们必须要有**Message**来源，而**Message**的来源**MessageQueue**是在**Looper**中的，因此**handler**需要一个**Looper**属性。一般情况下**mLooper**即为主线程或者说是创建它的线程所对应的**Looper**,但是我们也可以传一个Looper对象给它，关于这个可以看看`HandlerThread`的相关知识。
* **MessageQueue mQueue：**

	没啥好说的，这个就是**handler**要处理的消息的来源了,它和**Looper**中的**mQueue**指向同一个对象。
	
在看一下**Handler**的消息分发方法：

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
    
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```
还记得上一篇中最后线程停留的那个死循环吗，里面有一行代码` msg.target.dispatchMessage(msg);`,这里每次取出的**Message**都会被它对应的**target**也就是**Handler**对象分发，也就是上面这段代码。

可以看到首先会判断**Message**对象自己的**callback**是否为空，它是一个**Runnable**对象，如果不为空直接调用**callback**的**run()**方法，否则判断**handler**的属性**mCallback**是否为空，属性介绍的时候已经讲过这个了，如果不为空则则调用**mCallback.handleMessage(msg)**方法，这个方法返回true就直接return，否则调用**handler**的**handlerMessage(msg)**方法，也就是我们要继承**Handler**要实现的方法，这个是不是有点类似于Android系统的屏幕事件传递机制呢？哈哈~

话说第一篇里有个坑还没填，就是如何在工作线程创建使用**Handler**，不过相信看到这里的同学应该已经心里有底了吧，参考上一篇中**ActivityThread**中的**main()**方法，很容易创建自己的无线循环工作线程了，下面直接给出代码：

```
	new Thread(){
            Handler handler;
            @Override
            public void run() {
                super.run();
                Looper.prepare();
                 handler = new Handler(){
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                        //handler the message
                    }
                };
                Looper.loop();
            }
        };
```
以上，即在工作线程中创建了**handler**,然后将**handler**的引用丢给别的线程，别的线程就可以通过**handler**发消息到这个工作线程来处理。


**Handler**对于消息的分发和处理的逻辑总算说完了，接下去看看**Message**是如何被送到**Handler**的。
先看下**Message**的基本信息吧：

* **int what:**

	表明消息的类别，**handler**根据这个字段来判断如何处理这个**Message**
* **Bundle data：**

	用于存放数据，可以存放一些相对复杂的内容，因此开销稍微大一点

* **int arg1,arg2:**

	两个基本类型的参数，用于传递一些简单的**Message**信息，可以的话尽量用这两个参数来传递信息，相对于**Bundle data**它性能消耗会小很多
	
* **int flags:**

	当前**Message**状态的一个标志位，类似异步，使用中这些状态
* **long when：**
	
	表示这个**Message**应该在何时被执行，**MessageQueue**中**Message**就是以这个字段为来排序的
* **Handler target：**

	**Message**将要被发送的对象，也就是由哪个target来接受处理这个**Message**，**这个字段必须不为null**
* **Runnable callback：**

	这个讲**handler**的时候说到过了，如果有这个字段，那么这个**Message**将由此**callback**来处理，注意下，它是一个**Runnable**
* **Message next:**

	由于**Message**在消息池中是链表的形式维护的，所以这个字段表示下一个**Message**对象
*  **static Message sPool：**

	这个是静态变量，指向当前消息池的第一个空闲**Message**
* **static int sPoolSize:**

	消息池的大小，也就是剩余空闲**Message**的数量
* **static final int MAX_POOL_SIZE = 50;**

	不用解释了吧？

再贴一下例子中的第二段代码：


```
	new Thread(){
            @Override
            public void run() {
                super.run();
                Message message = Message.obtain();
                message.what = TEST;
                Bundle bundle =new Bundle();
                bundle.putString("test key","test value");
                message.setData(bundle);
       			handler.sendMessage(msg);                
            }
        }.start();
        
```


**Message**的获取方式特别多，除了自己new一个外，其他基本都大同小异，最终都从**Message**消息池取一个空闲的**Message**使用，因为Android系统中到处都用到了**Message**，因此会需要大量的**Message**对象，使用消息池的方式可以减少频繁的创建销毁对象，大大的提高性能。我这里直接用最基本的方式获取，`Message.obtain();`,看源码吧：

```
	/**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
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
```
对着上面的字段介绍，很容易看出这个方法的作用就是将消息池的第一个空闲**Message**拿出来，然后消息池的数量减1，当然如果消息池已经没用空闲**Message**了，那就new一个返回了。

例子中只用到了**what**和**data**字段，简单给它们赋了值，然后就调用**handler.sendMessage(msg); **将消息发送给了我们在主线程创建的**handler**了。

```
	public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
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
前面两个方法的注释我没贴，因为跟第三个方法内容差不多，而且最终也都是调用第三个方法，所以看了第三个方法的注释和方法体应该很容易明白前两个方法的作用。
第一个方法是直接将消息发送handler处理，所以它使用0毫秒的延时作为参数调用第二个方法，第二个方法判断如果延时小于0则默认给0，因为小于0的延迟也就意味着比当前时间早，当然不可能回到过去处理这个消息了，然后它又将当前时间加上延时，调用了第三个方法，这样第三个方法拿到的时间就是一个绝对值了，然后我们看到了**mQueue**，这个创建**handler**的时候我们就看到了，指向主线程对应的**Looper**中的消息队列，最后return了一个方法调用，看名字终于要加入消息队列了。

```
 	private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
``` 
等等，原来我们刚才发送的**Message**还没对象呢，那啥，我不是说那个对象，正经点儿！所以加入消息队列前先给它一个对象吧！然后如果我们的**handler**是异步的将**msg**的**flag**标志位加上成异步,这下**msg**开开心心地领着对象去排队了~

```
	//MessageQueue
	boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

```
首先判断**Message**的对象**target**是否为空(所以说**Message**不能没有对象**target**),然后判断**Message**是否正在使用中,在经历的前面的入队步骤后，我们这个**Message**都满足了条件，然后进入一个同步块，首先判断是否正在退出中，因为主线程是不可退出的，所以我们也不用考虑，不过里面方法也是一目了然，回收**Message**,返回**false**通知上层入队失败。我们继续看满足条件的情况下，将**Message**标记为在使用中(`msg.markInUse()`),给**when**字段赋值，然后创建一个新对象指向消息队列中的第一个消息**mMessage**，定义了一个布尔类型的字段**needWake**,表示是否需要唤醒当前线程，因为没有消息需要处理的时候我们的Looper线程是阻塞的。当消息队列中的第一个消息为**null**或者我们本次需要入队的消息对象**msg**的**when**为0或者小于原来的队首消息的**when**值，则将**msg**插入到消息队列的队首，也就是**mMessage**之前，然后**mMessage**重新指向新的队首，也就是**msg**；当那些条件不满足的时候，则需要将**msg**插入到消息队列中而不是队首，插入的方式也很简单除暴，遍历原队列的消息，依次比对**when**的值，直到队尾或者找到下一个消息的**when**值比**msg**的**when**值大的时候跳出循环，将**msg**插到其中，这个入队的过程也就完成了，都是一些链表的基本操作。之前也有说到，因为每次有消息进入**mQueue**，我们都是以这种方式来插入的，所以有序的消息队列就是简单以消息的**when**的大小来排序的。消息插入到消息队列完成后，判断是否需要唤醒主线程，需要则调用native方法**nativeWake()**唤醒主线程来处理消息（` Message msg = queue.next();//Looper.loop()方法中被唤醒后取出消息并分发处理`）。最后返回**true**表明消息插入队列成功。这样**Message**入队的过程就算完成了，是不是挺简单的~


以上，就是一个完整的消息传递和分发过程。实际使用中，我们用到更多的可能是`handler.post(runnable);`,`handler.postDelayed(runnable,delayMillis);`,`view.postDelayed(runnable,delayMillis);`之类的方法，这些方法看上去好像没有**Message**什么事，但是点进去一看就发现，这个被post的**runnable**就是**Message**的**callback**，简单封装一下就又走上了刚刚讲完的消息传递路线，还没反应过来的回头看看**handler**的**dispatchMessage(Message msg)**方法，因此这些方式我们都不用重写**handleMessage(Message msg)**方法,使用起来非常方便。***友情提示：非常方便的同时也非常容易导致内存泄露、空指针和其他一些状态紊乱的错误（别问我怎么知道的，死过太多脑细胞😭），慎用！！！***

