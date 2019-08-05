# NioEventLoop事件处理

上篇中我们做`NioEventLoop`事件循环主要步骤进行了整体的介绍，本篇我们将深入**事件处理**细节，深挖一下netty事件处理过程。

调用链：`NioEventLoop#run()` -> `NioEventLoop#processSelectedKeys()` -> `NioEventLoop#processSelectedKeysOptimized()` -> `NioEventLoop#processSelectedKey()`

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}

private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();
	    // 如果是Nio事件
        if (a instanceof AbstractNioChannel) {
            // 事件处理
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            // 如果是自定义的Nio任务
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }
		
        // 如果有SelectionKey调用cancel()取消注册Selector
        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            // 将当前位置之后所有的SelectedKeys在数组上置为null,help gc
            selectedKeys.reset(i + 1);
		   // 重新进行select()
            selectAgain();
            i = -1;
        }
    }
}
```

`processSelectedKeysOptimized`方法中主要做了以下工作：

1. 处理新的IO事件
2. 处理用户对于新的IO事件定义的NioTask
3. 如果存在还未来及处理就被cancel的事件，则重新进行select。

我们本篇重点关注事件处理相关内容，方法`processSelectedKey()`内容如下：

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    // 如果k无效，取消当前k与selector注册关系。
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        // 处理写事件
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        // 处理读事件或者接受新连接
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

通过**SelectionKey.readyOps()**调用，返回已就绪的操作集。

## 预定义操作符

先来看下预先定义的操作类型有哪些：

```java
// 可读
public static final int OP_READ = 1 << 0; // 二进制00000001 
// 可写
public static final int OP_WRITE = 1 << 2; // 二进制00000010
// 可连接
public static final int OP_CONNECT = 1 << 3; // 二进制00000100
// 可接受新连接
public static final int OP_ACCEPT = 1 << 4; // 二进制00001000
```

假设已连接的Channel，进入可读可写的状态，那么在调用`readyOps()`时就会返回11，带入上面源码中的是否可读的判读表达式`if ((readyOps & SelectionKey.OP_WRITE) != 0)`，可以转换为`if((00000011 &  00000010) !=0)`  结果显然是满足的。

## 读事件处理

进一步追踪读取源码`AbstractNioByteChannel#read()`

## 写事件处理

