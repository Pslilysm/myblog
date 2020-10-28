---
title: Application与AMS通信的搭建
date: 2020-10-26 18:27:00
tags:
    - Android
    - AMS
    - ActivityThread
---
## 概述

熟悉安卓开发的朋友都知道四大组件，同时也熟悉他们的生命周期，我们只需要在相应的生命周期里做相应的操作就可以了，比如Activity的onCreate()中初始化资源，onDestroy()中释放资源。但这些方法是何时被回调又如何被回调的呢？这里就不得不提到四大组件的管理者ActivityManagerService(AMS)，这个服务为一个系统服务，运行在system_server进程中，因此AMS和我们APP的组件是运行在不同的进程中的，那么其中肯定存在进程的通信，怎么通信的呢？下面我们看看。

### AMS启动进程

当AMS接受到请求创建我们应用的一个组件时，会先判断组件所需要的进程是否存在，如果不存在，则调用startProcessLocked()方法向Zygote进程发送Socket,请求他Fork出一个子进程出来，进程创建完成后，即调用ActivityThread.main()方法进入我们APP的运行期了。

### ActivityThread.main()
``` java
    public final class ActivityThread extends ClientTransactionHandler {
    
        public static void main(String[] args) {
            /...

            // 准备开启主线程消息循环
            Looper.prepareMainLooper();

            /...
            ActivityThread thread = new ActivityThread();
            // 初始化操作
            thread.attach(false, startSeq);

            if (sMainThreadHandler == null) {
                sMainThreadHandler = thread.getHandler();
            }

            if (false) {
                Looper.myLooper().setMessageLogging(new
                        LogPrinter(Log.DEBUG, "ActivityThread"));
            }

            // 循环读取并处理主线程的消息
            Looper.loop();

            throw new RuntimeException("Main thread loop unexpectedly exited");
        }

        final ApplicationThread mAppThread = new ApplicationThread();

        @UnsupportedAppUsage
        private void attach(boolean system, long startSeq) {
            // 静态变量保持ActivityThread对象
            sCurrentActivityThread = this;
            mSystemThread = system;
            if (!system) {
                // 不是系统UID进程 进入该分支
                /...
                // 通过ServiceManager获取AMS的BpBinder
                final IActivityManager mgr = ActivityManager.getService();
                try {
                    // 将自己进程的一个BnBinder通过Binder通信传给AMS，
                    // 这样AMS所在的进程就能通过这个Binder来回调我们
                    mgr.attachApplication(mAppThread, startSeq);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
                // Watch for getting close to heap limit.
                /...
            } 
            /...
        }

        // 该类是IApplicationThread.Stub的实现类， 为BnBinder
        private class ApplicationThread extends IApplicationThread.Stub {
        }
    }
```

## 总结

APP进程创建后便运行ActivityThread的main()方法，首先准备消息循环，然后进行attach操作，最后循环消息。
在attach操作里会通过ActivityManager.getService()获取AMS的BpBinder，实际上是通过ServiceManager来完成的。
得到AMS的BpBinder后便将我们APP的一个ApplicationThread对象传递给AMS，这样AMS就能通过我们传给他的Binder来回调我们。

