# NioEventLoop启动过程

本篇我们开始分析`NioEventLoop`的启动过程，代码入口在`AbstractBootstrap#doBind()` ->`AbstractBootstrap#doBind0()` -> `SingleThreadEventExecutor#execute`.

`AbstractBootstrap#doBind0()`方法内容如下：

```java
private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

通过`AbstractChannel`获取已经创建完成的（上篇中分析创建）`EventLoop`来执行它提供的`execute()`方法，方法内的`channel.bind()`在`Channel`相关博文中再进行剖析。

上篇中提到netty对`Executor`的装饰，此时`execute`方法被调用，对应的以下的逻辑也会被执行：

- 将创建完毕之后的线程启动
- 将线程放入到`FastThreadLocal`中

接着我们进一步追踪到`SingleThreadEventExecutor#execute()`中，代码如下：

```java
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                // The task queue does not support removal so the best thing we can do is to just move on and
                // hope we will be able to pick-up the task before its completely terminated.
                // In worst case we will log on termination.
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```

`boolean inEventLoop = inEventLoop();` 判断当前线程是否是EventLoop中的线程，第一次进入此方法的肯定是**main**线程，所以这里是false。

`addTask(task);`将要执行的任务添加到队列中。

接着执行`startThread();` 方法：

```java
private void startThread() {
    if (state == ST_NOT_STARTED) {
        // 记录线程启动状态，如果没有启动则执行if内的逻辑内容
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}

private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                // 调用NioEventLoop的run方法，开始进入事件循环
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                // 忽略部分代码
            }
        }
    });
}
```

至此本篇对`NioEventLoop`启动过程已经分析完毕，过程还是非常简单明了的，除了装饰器模式隐式被执行的方法外（这里如果没注意，确实很难发现）。那么我们下篇将继续分析`NioEventLoop`，来剖析它的轮询周期内到底处理了哪些工作内容。