---
title: Hello World
date: 2020-10-26 18:27:00
tags:
    - Android
---
## 疑问
安卓的四大组件是受AMS所管理的，那么他们之间肯定有数据的通信，怎么建立起这个通信的呢？

### AMS启动进程
当AMS接受到请求创建我们应用的一个组件时，会先判断组件所需要的进程是否存在，如果不存在，则调用startProcessLocked方法向Zygote进程发送Socket,
请求他Fork出一个子进程出来，进程创建完成后，即调用ActivityThread.main()方法进入我们的运行态。

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
            /... 
        }

    }
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)
