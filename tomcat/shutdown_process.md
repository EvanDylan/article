Table of Contents
=================

* [Tomcat的关闭流程](#tomcat%E7%9A%84%E5%85%B3%E9%97%AD%E6%B5%81%E7%A8%8B)
  * [前言背景](#%E5%89%8D%E8%A8%80%E8%83%8C%E6%99%AF)
  * [两种关闭方式](#%E4%B8%A4%E7%A7%8D%E5%85%B3%E9%97%AD%E6%96%B9%E5%BC%8F)
    * [通过端口方式](#%E9%80%9A%E8%BF%87%E7%AB%AF%E5%8F%A3%E6%96%B9%E5%BC%8F)
      * [await()](#await)
      * [stop()](#stop)
    * [通过脚本方式](#%E9%80%9A%E8%BF%87%E8%84%9A%E6%9C%AC%E6%96%B9%E5%BC%8F)
  * [彩蛋](#%E5%BD%A9%E8%9B%8B)


# Tomcat的关闭流程

## 前言背景

和同启动流程一样，关闭流程也是通过脚本`shutdown.sh`进行，调用的也是`catalina.sh`脚本，最终执行`Bootstrap`类的主函数，并传入了`stop`命令行参数。还有一种通过监听端口8005，收到指定的**SHUTDOWN**关闭命令，然后关闭。

## 两种关闭方式

### 通过端口方式

从启动流程中我们可以得知，启动的工作都是由`Bootstrap`通过调用`Catalina`一系列方法来完成的。我们这里仅仅关注启动时对`daemon.setAwait(true)`的调用，追踪源码会发现最终是设置了`Catalina`的`await`为**true**。那么`Catalina`中使用到`await`代码端如下：

```java
/**
 * Start a new server instance.
 */
public void start() {
	// 省略无关代码
    if (await) {
        await();
        stop();
    }
}
```

#### await()

进一步追踪await()到`StandardServer`,内容如下：

```java
public void await() {
    // Set up a server socket to wait on
    try {
        awaitSocket = new ServerSocket(port, 1,
                InetAddress.getByName(address));
    } catch (IOException e) {
        log.error("StandardServer.await: create[" + address
                           + ":" + port
                           + "]: ", e);
        return;
    }

    try {
        awaitThread = Thread.currentThread();

        // Loop waiting for a connection and a valid command
        while (!stopAwait) {
            ServerSocket serverSocket = awaitSocket;
            if (serverSocket == null) {
                break;
            }

            // Wait for the next connection
            Socket socket = null;
            StringBuilder command = new StringBuilder();
            try {
                InputStream stream;
                long acceptStartTime = System.currentTimeMillis();
                try {
                    socket = serverSocket.accept();
                    socket.setSoTimeout(10 * 1000);  // Ten seconds
                    stream = socket.getInputStream();
                } catch (SocketTimeoutException ste) {
                    // This should never happen but bug 56684 suggests that
                    // it does.
                    log.warn(sm.getString("standardServer.accept.timeout",
                            Long.valueOf(System.currentTimeMillis() - acceptStartTime)), ste);
                    continue;
                } catch (AccessControlException ace) {
                    log.warn("StandardServer.accept security exception: "
                            + ace.getMessage(), ace);
                    continue;
                } catch (IOException e) {
                    if (stopAwait) {
                        // Wait was aborted with socket.close()
                        break;
                    }
                    log.error("StandardServer.await: accept: ", e);
                    break;
                }

                // Read a set of characters from the socket
                int expected = 1024; // Cut off to avoid DoS attack
                while (expected < shutdown.length()) {
                    if (random == null)
                        random = new Random();
                    expected += (random.nextInt() % 1024);
                }
                while (expected > 0) {
                    int ch = -1;
                    try {
                        ch = stream.read();
                    } catch (IOException e) {
                        log.warn("StandardServer.await: read: ", e);
                        ch = -1;
                    }
                    // Control character or EOF (-1) terminates loop
                    if (ch < 32 || ch == 127) {
                        break;
                    }
                    command.append((char) ch);
                    expected--;
                }
            } finally {
                // Close the socket now that we are done with it
                try {
                    if (socket != null) {
                        socket.close();
                    }
                } catch (IOException e) {
                    // Ignore
                }
            }

            // Match against our command string
            boolean match = command.toString().equals(shutdown);
            if (match) {
                log.info(sm.getString("standardServer.shutdownViaPort"));
                break;
            } else
                log.warn("StandardServer.await: Invalid command '"
                        + command.toString() + "' received");
        }
    } finally {
        ServerSocket serverSocket = awaitSocket;
        awaitThread = null;
        awaitSocket = null;

        // Close the server socket and return
        if (serverSocket != null) {
            try {
                serverSocket.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }
}
```

内容虽多，但是主要工作就以下几点：

1. 启动`ServerSocket`并监听在8005端口
2. 读取输入流里的内容解析成字符串，并判断是否和预留的**SHUTDOWN**字符串相同。
3. 如果相同则关闭监听，如果不同则一直循环。

#### stop()

当8005的监听退出时，后续继续调用`Catalina`中的`stop()`方法。

### 通过脚本方式
在前言中有提到通过脚本方式实际也是通过调用`Bootstrap`中的`main()`方法，并传入`stop`的命令行参数。那么处理`stop`命令部分的代码简化之后如下：

```java
public static void main(String args[]) {
    // 省略代码
    else if (command.equals("stop")) {
        daemon.stopServer(args);
    }
    // 省略无关代码
}
```
进一步追踪`stopServer()`方法：

```java
public void stopServer(String[] arguments)
    throws Exception {

    Object param[];
    Class<?> paramTypes[];
    if (arguments==null || arguments.length==0) {
        paramTypes = null;
        param = null;
    } else {
        paramTypes = new Class[1];
        paramTypes[0] = arguments.getClass();
        param = new Object[1];
        param[0] = arguments;
    }
    Method method =
        catalinaDaemon.getClass().getMethod("stopServer", paramTypes);
    method.invoke(catalinaDaemon, param);

}
```

这里的`catalinaDaemon`是在启动流程中初始化的`Catalina`的实例，和通过端口方式调用的是同一个`stop()`方法。方法内容如下：

```java
public void stop() {

    try {
        // Remove the ShutdownHook first so that server.stop()
        // doesn't get invoked twice
        if (useShutdownHook) {
            Runtime.getRuntime().removeShutdownHook(shutdownHook);

            // If JULI is being used, re-enable JULI's shutdown to ensure
            // log messages are not lost
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                        true);
            }
        }
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        // This will fail on JDK 1.2. Ignoring, as Tomcat can run
        // fine without the shutdown hook.
    }

    // Shut down the server
    try {
        Server s = getServer();
        LifecycleState state = s.getState();
        if (LifecycleState.STOPPING_PREP.compareTo(state) <= 0
                && LifecycleState.DESTROYED.compareTo(state) >= 0) {
            // Nothing to do. stop() was already called
        } else {
            s.stop();
            s.destroy();
        }
    } catch (LifecycleException e) {
        log.error("Catalina.stop", e);
    }

}
```

这里主流程的代码其实就三行：

```java
public void stop() {
    Server s = getServer();
    s.stop();
    s.destroy();
}
```

这样看起来是不是简洁很多^-^，`getServer()`获取`StandardServer`实例调用其对应的`stop()`、`destroy()`方法。然后`StandardServer`在调用其内部管理的各个`Service`组件，`Service`组件在调用其管理的内部组件...。一次类推，俄罗斯套娃般。等到最后一个组件被关闭，tomcat的生命也就走向了终点。。。

## 彩蛋
如何才能禁用8005端口这种方式关闭后门呢？

```java
public void await() {
    // Negative values - don't wait on port - tomcat is embedded or we just don't like ports
    if( port == -2 ) {
        // undocumented yet - for embedding apps that are around, alive.
        return;
    }
    if( port==-1 ) {
        try {
            awaitThread = Thread.currentThread();
            while(!stopAwait) {
                try {
                    Thread.sleep( 10000 );
                } catch( InterruptedException ex ) {
                    // continue and check the flag
                }
            }
        } finally {
            awaitThread = null;
        }
        return;
    }
    // 略~~~
}
```

答案：在`server.xml`文件中将8005端口改为-2即可。