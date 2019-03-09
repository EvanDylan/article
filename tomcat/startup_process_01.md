## Tomcat启动流程（上）

### 前言
承接Tomcat开门篇，如果还未有看过的老铁点击[传送门](./overview.md)。上篇简单介绍了各个主要组件的功能作用，本篇通过分析Tomcat Server的启动流程，了解下各个组件之间是如何相互的协作完成启动的工作。为了减少篇幅，会针对非关键性代码做一定的省略。

### 从startup.sh说起

正常启动tomcat的流程，命令行运行`./startup.sh start` 完事。透过表象看本质，脚本里到底哪些工作

```shell
os400=false
case "`uname`" in
OS400*) os400=true;;
esac

# resolve links - $0 may be a softlink
# 取当前脚本的路径地址
PRG="$0"

while [ -h "$PRG" ] ; do
  ls=`ls -ld "$PRG"`
  link=`expr "$ls" : '.*-> \(.*\)$'`
  if expr "$link" : '/.*' > /dev/null; then
    PRG="$link"
  else
    PRG=`dirname "$PRG"`/"$link"
  fi
done

PRGDIR=`dirname "$PRG"`
EXECUTABLE=catalina.sh

# Check that target executable exists
if $os400; then
  # -x will Only work on the os400 if the files are:
  # 1. owned by the user
  # 2. owned by the PRIMARY group of the user
  # this will not work if the user belongs in secondary groups
  eval
else
  if [ ! -x "$PRGDIR"/"$EXECUTABLE" ]; then
    echo "Cannot find $PRGDIR/$EXECUTABLE"
    echo "The file is absent or does not have execute permission"
    echo "This file is needed to run this program"
    exit 1
  fi
fi

exec "$PRGDIR"/"$EXECUTABLE" start "$@"
```

简单解释下最后一行`exec "$PRGDIR"/"$EXECUTABLE" start "$@"`意思。`$PRGDIR`代表当前脚本的路径，`$EXECUTABLE`代表了`catalina.sh`脚本文件，`start`是传入的命令行参数， `$@`是将剩余命令行参数传递给`catalina.sh`脚本。

`catalina.sh`脚本内容太多就不贴出来了，总结下它的工作内容，大概分为以下几个部分：

1. 初始化环境变量`CATALINA_HOME`、`CATALINA_BASE`、`CATALINA_OUT`、`CATALINA_TMPDIR`、`LOGGING_CONFIG`等

2. 清空用户自己定义的`CLASSPATH`，并加载`setenv.sh`脚本的环境变量。

   `setenv.sh`这个脚本是需要自己在`catalina.sh`的同级目录下创建，用来指定自定义的环境变量内容，我们可以借此指定Java虚拟机的参数，并且这种方式是官方推荐的，不推荐直接修改`catalina.sh`的内容。

3. 将`bootstrap.jar`、`tomcat-juli.jar`加入到`CLASSPATH`环境变量中。
4. 处理命令行参数`debug`、`run`、`start`等。

贴下启动的关键命令行内容：

```shell
$_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
  -D$ENDORSED_PROP="\"$JAVA_ENDORSED_DIRS\"" \
  -classpath "\"$CLASSPATH\"" \
  -Dcatalina.base="\"$CATALINA_BASE\"" \
  -Dcatalina.home="\"$CATALINA_HOME\"" \
  -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
  org.apache.catalina.startup.Bootstrap "$@" start 
```

终于看到了Tomcat的启动类`org.apache.catalina.startup.Bootstrap`并在这里传入了`start`的启动命令。

### 一切起源于main函数

终于来到了`Bootstrap`类中的`main()`方法，内容简写（省略非关键性代码）如下：

```java
Bootstrap bootstrap = new Bootstrap();
bootstrap.init();

String command = "start";
if (args.length > 0) {
    command = args[args.length - 1];
}
if (command.equals("start")) {
    daemon.setAwait(true);
    daemon.load(args);
    daemon.start();
    if (null == daemon.getServer()) {
        System.exit(1);
    }
}

```

这里通过`init()`初始化了`Bootstrap`,然后再调用`setAwait()`、`load()`、`start()`方法。

#### init()方法

```java
public void init() throws Exception {

    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();

    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);

    catalinaDaemon = startupInstance;

}
```

总结大致做了以下工作：

1. `initClassLoaders()`部分分别初始化了`commonLoader`、`serverLoader`、`sharedLoader`自定义类加载器。
2. 通过反射调用`Catalina`的`setParentClassLoader()`方法，并传入`sharedLoader`作为`Catalina`的父加载器。
3. 将`Catalina`实例`startupInstance`赋值给`Bootstrap`的`catalinaDaemon`变量。

#### setAwait()方法

将`await`变量设置为`true`，具体作用可以查看在关闭流程中的分析，[传送门](./shutdown_process.md)。

#### load()方法

```java
/**
 * Start a new server instance.
 */
public void load() {

    if (loaded) {
        return;
    }
    loaded = true;

    long t1 = System.nanoTime();

    initDirs();

    // Before digester - it may be needed
    initNaming();

    // Create and execute our Digester
    Digester digester = createStartDigester();

    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        try {
            file = configFile();
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail", file), e);
            }
        }
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource(getConfigFile()).toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            getConfigFile()), e);
                }
            }
        }

        // This should be included in catalina.jar
        // Alternative: don't bother with xml, just create it manually.
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                        .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                (getClass().getClassLoader()
                        .getResource("server-embed.xml").toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            "server-embed.xml"), e);
                }
            }
        }


        if (inputStream == null || inputSource == null) {
            if  (file == null) {
                log.warn(sm.getString("catalina.configFail",
                        getConfigFile() + "] or [server-embed.xml]"));
            } else {
                log.warn(sm.getString("catalina.configFail",
                        file.getAbsolutePath()));
                if (file.exists() && !file.canRead()) {
                    log.warn("Permissions incorrect, read permission is not allowed on the file.");
                }
            }
            return;
        }

        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        } catch (SAXParseException spe) {
            log.warn("Catalina.start using " + getConfigFile() + ": " +
                    spe.getMessage());
            return;
        } catch (Exception e) {
            log.warn("Catalina.start using " + getConfigFile() + ": " , e);
            return;
        }
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }

    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    // Stream redirection
    initStreams();

    // Start the new server
    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error("Catalina.start", e);
        }
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {
        log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
    }
}
```

`load`方法内几个重要点：

1. 通过调用`Digester``createStartDigester()`、`digester.push(this)`、`digester.parse(inputSource)`等方法解析`server.xml`配置文件，显示的实例化各个主要组件，如`StandardContext`、`StandardServer`、`StandardService`、`Engine`、`Connector`等。此时各个主要的组件已经被创建。

2. 调用`getServer().init()`方法追踪后发现最终调用了`StandardServer.init()`方法，进一步追踪会调用`initInternal()`方法。

   ```java
   protected void initInternal() throws LifecycleException {
   	// 略~~~
       
       // Initialize our defined Services
       for (int i = 0; i < services.length; i++) {
           services[i].init();
       }
   }
   ```

   这里明显的可以看出对`StandardService`进行初始化，而`StandardService`初始化代码内容如下：

   ```java
   @Override
   protected void initInternal() throws LifecycleException {
   
       super.initInternal();
   
       if (engine != null) {
           engine.init();
       }
   
       // Initialize any Executors
       for (Executor executor : findExecutors()) {
           if (executor instanceof JmxEnabled) {
               ((JmxEnabled) executor).setDomain(getDomain());
           }
           executor.init();
       }
   
       // Initialize mapper listener
       mapperListener.init();
   
       // Initialize our defined Connectors
       synchronized (connectorsLock) {
           for (Connector connector : connectors) {
               try {
                   connector.init();
               } catch (Exception e) {
                   String message = sm.getString(
                           "standardService.connector.initFailed", connector);
                   log.error(message, e);
   
                   if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"))
                       throw new LifecycleException(message);
               }
           }
       }
   }
   ```

   以上代码又调用了`Engine`、`Connector`等组件的`init()`方法进行初始化。`Engine`初始化内容较为简单这里暂且略过，我们重点关注`Connector`的`init`过程。配置文件`server.xml`中关于`Connector`默认配置内容如下：

   ```xml
   <Connector port="8080" protocol="HTTP/1.1"
                  connectionTimeout="20000"
                  redirectPort="8443" />
   <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
   ```

   在上面`server.xml`解析的步骤中会根据协议的不同适配对应的处理类，适配关键代码如下：

   ```java
   public void setProtocol(String protocol) {
   
       boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
               AprLifecycleListener.getUseAprConnector();
   
       if ("HTTP/1.1".equals(protocol) || protocol == null) {
           if (aprConnector) {
               setProtocolHandlerClassName("org.apache.coyote.http11.Http11AprProtocol");
           } else {
               setProtocolHandlerClassName("org.apache.coyote.http11.Http11NioProtocol");
           }
       } else if ("AJP/1.3".equals(protocol)) {
           if (aprConnector) {
               setProtocolHandlerClassName("org.apache.coyote.ajp.AjpAprProtocol");
           } else {
               setProtocolHandlerClassName("org.apache.coyote.ajp.AjpNioProtocol");
           }
       } else {
           setProtocolHandlerClassName(protocol);
       }
   }
   ```

   这里输入的参数有`HTTP/1.1`、`AJP/1.3`，由于没有配置APR的环境依赖库，所以最终适配的类分别为：`Http11NioProtocol`、`AjpNioProtocol`，然后会继续调用它们各自的`init()`方法。关于协议处理类的具体初始过程涉及内容较多，准备单独开一篇进行详细分析。

   到此主函数中的`load`过程已经执行完毕。

   #### start()方法

   追踪`start`方法后发现，最终调用的是`Catalina`的`start`方法，内容如下：

   ```java
   public void start() {
   
       if (getServer() == null) {
           load();
       }
   
       if (getServer() == null) {
           log.fatal("Cannot start server. Server instance is not configured.");
           return;
       }
   
       long t1 = System.nanoTime();
   
       // Start the new server
       try {
           getServer().start();
       } catch (LifecycleException e) {
           log.fatal(sm.getString("catalina.serverStartFail"), e);
           try {
               getServer().destroy();
           } catch (LifecycleException e1) {
               log.debug("destroy() failed for failed Server ", e1);
           }
           return;
       }
   
       long t2 = System.nanoTime();
       if(log.isInfoEnabled()) {
           log.info("Server startup in " + ((t2 - t1) / 1000000) + " ms");
       }
   
       // Register shutdown hook
       if (useShutdownHook) {
           if (shutdownHook == null) {
               shutdownHook = new CatalinaShutdownHook();
           }
           Runtime.getRuntime().addShutdownHook(shutdownHook);
   
           // If JULI is being used, disable JULI's shutdown hook since
           // shutdown hooks run in parallel and log messages may be lost
           // if JULI's hook completes before the CatalinaShutdownHook()
           LogManager logManager = LogManager.getLogManager();
           if (logManager instanceof ClassLoaderLogManager) {
               ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                       false);
           }
       }
   
       if (await) {
           await();
           stop();
       }
   }
   ```

   提炼重要过程步骤：

   1. 通过`getServer()`获取`StandardServer`然后调用其`start()`方法。
   2. `start()`完成之后，后续就是监听8005端口执行关闭的流程了，详细内容见[传送门](./shutdown_process.md)。

   追踪`StandardServer.start()`方法后内容如下

   ```java
   protected void startInternal() throws LifecycleException {
   
       fireLifecycleEvent(CONFIGURE_START_EVENT, null);
       setState(LifecycleState.STARTING);
   
       globalNamingResources.start();
   
       // Start our defined Services
       synchronized (servicesLock) {
           for (int i = 0; i < services.length; i++) {
               services[i].start();
           }
       }
   }
   ```

接着调用`StandardService`的`start()`方法，内容如下：

```java
protected void startInternal() throws LifecycleException {

    if(log.isInfoEnabled())
        log.info(sm.getString("standardService.start.name", this.name));
    setState(LifecycleState.STARTING);

    // Start our defined Container first
    if (engine != null) {
        synchronized (engine) {
            engine.start();
        }
    }

    synchronized (executors) {
        for (Executor executor: executors) {
            executor.start();
        }
    }

    mapperListener.start();

    // Start our defined Connectors second
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            try {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            } catch (Exception e) {
                log.error(sm.getString(
                        "standardService.connector.startFailed",
                        connector), e);
            }
        }
    }
}
```

这里完成了`Engine`、`Connector`等重要组件的启动过程。

### 写在最后

本篇完成了Tomcat的启动过程分析，等等。。。是不是漏掉了什么？`Server`、`Service`、`Engine`、`Connector`等组件都有讲到，但是Host`、`Context`是不是漏掉了，Tomcat在哪里处理`web.xml`以及如何部署应用的呢？由于篇幅有限，那么我们下篇见，[传送门](./startup_process_02.md)。

