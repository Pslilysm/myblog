---
title: Handler、Looper、MessageQueue源码分析
date: 2020-10-27 09:00:00
tags:
    - Android
    - Handler
---
## 概述
安卓的Handler线程同步消息机制使用频率是相当高的，比如Activity的runOnUiThread()、View.post()等都是使用的Handler，因此掌握这个技术是作为一个安卓开发人员必不可少的技能。

### Looper的创建
先看下如何让一个线程进入消息循环，安卓有个HandlerThread类，他是对一个需要循环消息线程的封装。
使用Demo如下：
``` java
    void handlerThreadUsageDemo(){
        HandlerThread ht = new HandlerThread("work_queue");
        // 启动线程
        ht.start();
        Handler h = ht.getThreadHandler();
    }

    public class HandlerThread extends Thread {

        int mTid = -1;
        Looper mLooper;
        private @Nullable Handler mHandler;
    
        @Override
        public void run() {
            mTid = Process.myTid();
            // 准备Looper
            Looper.prepare();
            synchronized (this) {
                // 在当前线程的上下文环境获取当前线程的Looper对象
                mLooper = Looper.myLooper();
                notifyAll();
            }
            Process.setThreadPriority(mPriority);
            onLooperPrepared();
            // 循环消息
            Looper.loop();
            mTid = -1;
        }
    
        public Looper getLooper() {
            if (!isAlive()) {
                return null;
            }           
            // 如果线程启动成功，阻塞等待被唤醒然后返回mLooper
            synchronized (this) {
                while (isAlive() && mLooper == null) {
                    try {
                        wait();
                    } catch (InterruptedException e) {
                    }
                }
            }
            return mLooper;
        }

        public Handler getThreadHandler() {
            if (mHandler == null) {
                mHandler = new Handler(getLooper());
            }
            return mHandler;
        }

    }

```
HandlerThread的源码比较简单，主要点还是Looper类的相关方法。
``` java
    public final class Looper {
    
        public static void prepare() {
            prepare(true);
        }

        // ThreadLocal是一个泛型类，用于提供线程的局部变量。
        // 比如说你在A线程的工作环境中通过ThreadLocal保存了一个对象，在B线程通过这个ThreadLocal的get方法是得不到任何东西的，
        // 只有在A线程的工作环境中才能获取A线程存入的值。
        static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

        // 将当前线程初始化Looper对象
        private static void prepare(boolean quitAllowed) {
            if (sThreadLocal.get() != null) {
                // 当前线程已经创建过Looper工作环境，直接抛异常。
                throw new RuntimeException("Only one Looper may be created per thread");
            }
            // 对当前线程保存Looper
            sThreadLocal.set(new Looper(quitAllowed));
        }

        final MessageQueue mQueue;
        final Thread mThread;

        private Looper(boolean quitAllowed) {
            // 初始化MessageQueue
            mQueue = new MessageQueue(quitAllowed);
            mThread = Thread.currentThread();
        }

        public static @Nullable Looper myLooper() {
            return sThreadLocal.get();
        }

        public static void loop() {
            final Looper me = myLooper();
            if (me == null) {
                throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
            }
            final MessageQueue queue = me.mQueue;

            for (;;) {
                // 阻塞直到从MessageQueue读取到下一条消息为止
                Message msg = queue.next(); 
                if (msg == null) {
                    // 下一条消息返回null则退出消息循环
                    return;
                }

                /...
                try {
                    // 让此条消息的目标Handler处理消息
                    msg.target.dispatchMessage(msg);
                    /...
                } catch (Exception exception) {
                    /...
                    throw exception;
                } finally {
                    /...
                }
                // 将Message放入缓存池
                msg.recycleUnchecked();
            }
        }

    }
```
Looper.prepare()方法先为当前线程创建Looper，创建Looper的同时创建MessageQueue。
接着使用Looper.loop()方法无限从当前线程的MessageQueue里读取消息并处理掉。

### MessageQueue.next()
``` java
    public final class MessageQueue {
    
        private long mPtr; // 本地的指针
        Message mMessages; // 消息队列的消息头

        // 空闲时的Handler
        private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
        private IdleHandler[] mPendingIdleHandlers;

        // 标记线程是否正在退出
        private boolean mQuitting;
        // 标记线程是否被nativePollOnce()阻塞住了
        private boolean mBlocked;

        Message next() {
            final long ptr = mPtr;
            // 如果C层的MessageQueue指针为0， 退出
            if (ptr == 0) {
                return null;
            }

            int pendingIdleHandlerCount = -1; // 只有第一次循环为-1，之后的循环都不是-1也就都不会去跑IdleHandler;
            int nextPollTimeoutMillis = 0;
            for (;;) {
                if (nextPollTimeoutMillis != 0) {
                    Binder.flushPendingCommands();
                }

                // 阻塞nextPollTimeoutMillis毫秒
                // 如果传入-1，则无限阻塞直到被唤醒
                nativePollOnce(ptr, nextPollTimeoutMillis);

                synchronized (this) {
                    // Try to retrieve the next message.  Return if found.
                    final long now = SystemClock.uptimeMillis();
                    Message prevMsg = null;
                    Message msg = mMessages;
                    if (msg != null && msg.target == null) {
                        // 此时消息队列里出现了一个牌子，说正常的消息都暂停一下，先让插队的消息被处理。
                        // 于是去找下一个插队消息
                        do {
                            prevMsg = msg;
                            msg = msg.next;
                        } while (msg != null && !msg.isAsynchronous());
                    }
                    if (msg != null) {
                        // 找到一条需要处理的消息
                        if (now < msg.when) {
                            // 消息头需要处理的时间大于当前时间， 所以直接设置下个for循环的阻塞时长
                            nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                        } else {
                            // Got a message.
                            mBlocked = false;
                            if (prevMsg != null) {
                                // 消息头的上一个消息不为空说明处理的是插队消息，
                                // 因此将上一个消息链接到插队消息的下一个，因为插队消息马上要被消耗掉，应该将插队消息从单链表里踢出去。
                                prevMsg.next = msg.next;
                            } else {
                                // 将消息头指向下一条消息
                                mMessages = msg.next;
                            }
                            msg.next = null;
                            // 将消息标记正在使用中
                            msg.markInUse();
                            return msg;
                        }
                    } else {
                        // 未找到需要处理的消息，设置下次阻塞为永久。
                        nextPollTimeoutMillis = -1;
                    }

                    // 如果要退出MessageQueue, 则销毁本地的相应资源并返回null
                    if (mQuitting) {
                        dispose();
                        return null;
                    }

                    // 第一次for循环且没有找到要处理的消息时，进入下列分支
                    if (pendingIdleHandlerCount < 0
                            && (mMessages == null || now < mMessages.when)) {
                        // 获取需要在空闲时间干活的IdleHandler数量
                        pendingIdleHandlerCount = mIdleHandlers.size();
                    }
                    if (pendingIdleHandlerCount <= 0) {
                        // 没有需要在空闲时间干活的IdleHandler, 则标记为阻塞，然后进入下一次for循环并nativePollOnce
                        mBlocked = true;
                        continue;
                    }

                    if (mPendingIdleHandlers == null) {
                        mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                    }
                    mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
                }

                // 运行IdleHandlers
                for (int i = 0; i < pendingIdleHandlerCount; i++) {
                    final IdleHandler idler = mPendingIdleHandlers[i];
                    mPendingIdleHandlers[i] = null; 
                    boolean keep = false;
                    try {
                        keep = idler.queueIdle();
                    } catch (Throwable t) {
                        Log.wtf(TAG, "IdleHandler threw exception", t);
                    }

                    if (!keep) {
                        synchronized (this) {
                            mIdleHandlers.remove(idler);
                        }
                    }
                }

                // 设置为0，让下次for循环时如果又没有消息要处理也不会来运行IdleHandler
                pendingIdleHandlerCount = 0;

                // 刚跑完IdleHandler，可能有消息需要处理了，设置下次循环不阻塞
                nextPollTimeoutMillis = 0;
            }
        }
        
    }
```
Message类的next成员变量指向下一条Message，因此组成了一个单链表结构，为MessageQueue提供了一个按顺序读取消息的环境。
而MessageQueue的next()方法则一直从这个单链表里读取第一条或插队消息返回。
了解了MessageQueue的取出消息，下面看看发送消息时如何实现的。

### 向MessageQueue发送消息
发送消息想必大家都很熟悉，直接调用Handler的sendMessage()就可以了，下面看看Handler的相关方法
#### Handler.enqueueMessage()
``` java
    public class Handler {

        // 无参的构造方法会调用这个
        public Handler(@Nullable Callback callback, boolean async) {
            /...
            // 获取当前线程的Looper
            mLooper = Looper.myLooper();
            if (mLooper == null) {
                throw new RuntimeException(
                    "Can't create handler inside thread " + Thread.currentThread()
                            + " that has not called Looper.prepare()");
            }
            // 通过Looper获取MessageQueue
            mQueue = mLooper.mQueue;
            mCallback = callback;
            mAsynchronous = async;
        }
    
        // 将消息交给MessageQueue来插入队列
        private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
                long uptimeMillis) {
            // 将消息的targetHandler指向自己
            msg.target = this;
            msg.workSourceUid = ThreadLocalWorkSource.getUid();

            if (mAsynchronous) {
                // 插队消息
                msg.setAsynchronous(true);
            }
            return queue.enqueueMessage(msg, uptimeMillis);
        }

    }
```
Handler发送消息的方法最终还是交由他所绑定的MessageQueue来实现的，接下来看MessageQueue的enqueueMessage()方法。
#### MessageQueue.enqueueMessage()
``` java
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                // 如果MessageQueue要退出，则return false
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            // 将消息标记正在使用
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // 如果MessageQueue队列里没有消息 或者 msg要插在消息头的前面
                msg.next = p;
                mMessages = msg;
                // 是否需要唤醒等同于是否阻塞住了
                needWake = mBlocked;
            } else {
                // 将消息插入队列中间

                // 如果此时阻塞住了而且消息头为插队牌子而且需要发送的消息为发送消息 那么需要唤醒，
                // 为什么需要唤醒你们可以思考一下
                needWake = mBlocked && p.target == null && msg.isAsynchronous();

                
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        // 找到了需要插入的位置
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        // 如果在找插入位置时发现前面也有插队的消息，那么说明MessageQueue现在肯定在找p并想处理掉他，所以设置不唤醒
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            if (needWake) {
                // 打断nativePollOnce
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
MessageQueue对将要插入的消息进行排序，有必要时还会去唤醒正在阻塞的线程。
## 总结
最后再简单归纳下消息机制：
* Looper通过prepare()创建Looper并向TLS存入Looper，同时创建MessageQueue；
* 接着Looper通过loop()无限从MessageQueue读取消息，若没有消息或下一条消息还没到处理的时间MessageQueue则会阻塞，并在阻塞前会去运行一次IdleHandler；
* Handler通过sendMessage()将Message插入到MessageQueue队列，并且在需要时将MessageQueue唤醒；
安卓的消息机制java层源码基本上分析完了，想要深入的分析是如何唤醒MessageQueue就得去看C层的代码了。

