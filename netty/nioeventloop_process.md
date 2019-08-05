# NioEventLoop事件轮询过程

上接[NioEventLoop启动过程](./start_nioeventloop.md)，在上篇我们分析`NioEventLoop`的启动过程，那么在启动之后紧接着就开始了事件和任务的处理过程。在进入源码剖析之前，先将循环中主要执行任务列举如下：

- 执行`selector()`操作，获取可以执行的事件集合
- 解决jdk空轮询bug（会导致cpu跑满）
- 执行任务队列中的任务

## io事件及任务调度

```java
protected void run() {
        for (;;) {
            try {
                try {
                    /**
                     * 表达式拆分以下几步看：
                     * 1. hasTasks() 判断当前队列中是否有待执行的任务
                     * 2. 执行DefaultSelectStrategy#calculateStrategy方法'hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;'
                     *  2.1 如果当前队列中没有任务,则执行SelectStrategy.SELECT策略的select - 阻塞式
                     *  2.2 如果当前队列中有任务,则执行selectSupplier.get()调用,相当于调用了selector.selectNow(),如果有新的事件进来则返回大于0的数字,
                     *      否则返回0。此时这种情况switch中永远都是大于0的情况(讨论仅限于NIO)。所以case里的分支全部都走不到。
                     */
                    switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                    // the selector and retry. https://github.com/netty/netty/issues/8566
                    rebuildSelector0();
                    handleLoopException(e);
                    continue;
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                // 默认用来处理io事件占用总执行时间的百分比。(事件处理+执行任务队列)
                final int ioRatio = this.ioRatio;
                // 如果设置为100,则先把io时间处理完,最终在处理任务调度的工作
                if (ioRatio == 100) {
                    try {
                        // 处理IO事件
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        // 执行任务调度
                        runAllTasks();
                    }
                } else {
                    // 如果不是100,则先计算执行io事件所用时间,然后按照设定百分比分配给执行任务调度时间
                    // 举例：默认情况该值设定为50,如果执行io事件用时1s钟,则用来执行任务调度时间最大可以为1s
                    final long ioStartTime = System.nanoTime();
                    try {
                        // 处理IO事件
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        // 执行任务调度
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```

概括上面代码所做的工作如下：执行一个死循环，先执行`select()`将新的事件选出来，根据**ioRatio**配比，分配不同的时间来分别执行io事件以及处理任务队列中的任务。

## 执行select()及解决空轮询bug

执行阻塞式`select(wakenUp.getAndSet(false));`调用的内容如下：

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        // 当前任务调度队列中最近需要执行任务的最晚时间
        // 比如当前时间为XXX年X月X分X秒,离需要最先被调度的任务时间为还有1s钟,那么最终得到的时间为XXX年X月X分X+1秒,
        // 如果超过这个时间点任务还没有得到执行那么就会发生超时。如果存在空闲检测的任务被超时了,后果很严重。
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        for (;;) {
            // 还是上面的例子,这里结果为1s + 0.5ms
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            // 如果值为小于等于0的数字,则说明任务调度已经晚了0.5毫秒了
            if (timeoutMillis <= 0) {
                // 如果一次selectNow()的都没有调用过,则调用一次之后立刻跳出循环,去执行io事件、任务调度的工作
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
            // Selector#wakeup. So we need to check task queue again before executing select operation.
            // If we don't, the task might be pended until select operation was timed out.
            // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
            // 执行到这里说明里最近需要执行的任务还有一会儿
            // 如果任务队列里有任务并且可以执行非阻塞select
            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                // 执行非阻塞的selectNow()后立刻跳出循环出去干活
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            // 执行阻塞式的select操作
            int selectedKeys = selector.select(timeoutMillis);
            /**
                 * 能执行到这里肯定是发生了以下几种情况之一
                 * 1. 有新的事件
                 * 2. 一直等超时了,但是没有新的事件产生
                 * 3. 发生了jdk空轮询bug,即便以上情况没有发生线程也继续执行了,而不是继续等待
                 */
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                // - Selected something, 有新的事件
                // - waken up by user, or 被唤醒
                // - the task queue has a pending task. 任务队列有任务
                // - a scheduled task is ready for processing 调度队列准备就绪
                break;
            }
            if (Thread.interrupted()) {
                // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                // As this is most likely a bug in the handler of the user or it's client library we will
                // also log it.
                //
                // See https://github.com/netty/netty/issues/2426
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                                 "Thread.currentThread().interrupt() was called. Use " +
                                 "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                // 已经超时了但是没有任何新的事件
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                       selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                // The code exists in an extra method to ensure the method is not too big to inline as this
                // branch is not very likely to get hit very frequently.
                // 默认判定空轮询bug循环次数为512次,到了这个阈值就对整个selector上的SelectionKey进行重建
                // 重建一个新的selector将老的selector上的SelectionKey重新注册,并取消掉老的selector的注册。
                selector = selectRebuildSelector(selectCnt);
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                             selectCnt - 1, selector);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                         selector, e);
        }
        // Harmless exception - log anyway
    }
}
```

概括上面代码所做的工作如下：在一个死循环内，限定时间执行阻塞式`select()`调用，如果时间分片用光、或者有新的事件、任务需要处理则跳出循环，回到上面方法内继续干活。其中比较有意思的就是解决jdk空轮询bug的思路。

如果在整个时间分片内，该循环执行了512次，则证明发生了空轮询则进行`Selector`重建的工作。（至于为什么执行了那么多次就能证明发生了空轮询的疑惑，请看下面代码中的注释中的解释）

```java
 // 执行阻塞式的select操作
int selectedKeys = selector.select(timeoutMillis);
/**
  * 能执行到这里肯定是发生了以下几种情况之一
  * 1. 有新的事件,下次循环时就会跳出循环了
  * 2. 一直等超时了,但是没有新的事件产生，如果超时下次循环时就直接返回了
  * 3. 发生了jdk空轮询bug,即便以上情况没有发生线程也继续执行了,而不是继续等待。
  */
selectCnt ++;
```

至于解决方法嘛，关键代码内容如下：

```java
 private void rebuildSelector0() {
        final Selector oldSelector = selector;
        final SelectorTuple newSelectorTuple;

        if (oldSelector == null) {
            return;
        }

        try {
            // 创建一个新的Selector
            newSelectorTuple = openSelector();
        } catch (Exception e) {
            logger.warn("Failed to create a new Selector.", e);
            return;
        }

        // Register all channels to the new Selector.
        int nChannels = 0;
        // 遍历旧的Selector上的SelectionKey
        for (SelectionKey key: oldSelector.keys()) {
            Object a = key.attachment();
            try {
                if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                    continue;
                }

                int interestOps = key.interestOps();
                // 将旧的Selector上的SelectionKey的绑定关系取消
                key.cancel();
                // 将SelectionKey重新注册到新的Selector上
                SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
                if (a instanceof AbstractNioChannel) {
                    // Update SelectionKey
                    ((AbstractNioChannel) a).selectionKey = newKey;
                }
                nChannels ++;
            } catch (Exception e) {
                logger.warn("Failed to re-register a Channel to the new Selector.", e);
                if (a instanceof AbstractNioChannel) {
                    AbstractNioChannel ch = (AbstractNioChannel) a;
                    ch.unsafe().close(ch.unsafe().voidPromise());
                } else {
                    @SuppressWarnings("unchecked")
                    NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                    invokeChannelUnregistered(task, key, e);
                }
            }
        }

        selector = newSelectorTuple.selector;
        unwrappedSelector = newSelectorTuple.unwrappedSelector;

        try {
            // time to close the old selector as everything else is registered to the new one
            // 关闭旧的Selector
            oldSelector.close();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }

        if (logger.isInfoEnabled()) {
            logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
        }
    }
```

创建一个新的`Selector`并将原来`Selector`上的`SelectKey`重新绑定关系到新的上来，将老的替换掉。



