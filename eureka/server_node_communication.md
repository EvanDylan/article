# Eureka集群节点同步

##  自动装配

借助Spring Cloud使用`Eureka`作为注册中心时，我们只需要使用`@EnableEurekaServer`注解，再加上些许配置即可使用。注解类内容如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```

不难发现`EnableEurekaServer`又通过`Import`注解引入了`EurekaServerMarkerConfiguration`，那么`EurekaServerMarkerConfiguration`中配置的Bean对象就会被创建。

```java
@Configuration
public class EurekaServerMarkerConfiguration {

   @Bean
   public Marker eurekaServerMarkerBean() {
      return new Marker();
   }

   class Marker {

   }
}
```

但是这个Bean对象又是一空对象，所以创建一个空对象有什么用呢？

回到`EurekaServerAutoConfiguration`自动装配注解类内容如下，不能发现该类通过`@ConditionalOnBean`注解，来判断如果`EurekaServerMarkerConfiguration.Marker`对象被载入则自动装配会起作用。

```java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
// 通过判断EurekaServerMarkerConfiguration Marker对象有没有被载入判断是否启用自动配置
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
    
}
```

## 启动引导

上面我们简单介绍了Eureka服务自动装配的原理。那么在`EurekaServerAutoConfiguration`中到底吃做了哪些工作呢？

- 自动了装配了以下几个关键bean对象：

```java
@Bean
public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
      ServerCodecs serverCodecs) {
   this.eurekaClient.getApplications(); // force initialization
   return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
         serverCodecs, this.eurekaClient,
         this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
         this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
}

@Bean
@ConditionalOnMissingBean
public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
      ServerCodecs serverCodecs) {
   return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
         this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
}

@Bean
public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
      PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
   return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
         registry, peerEurekaNodes, this.applicationInfoManager);
}

@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
      EurekaServerContext serverContext) {
   return new EurekaServerBootstrap(this.applicationInfoManager,
         this.eurekaClientConfig, this.eurekaServerConfig, registry,
         serverContext);
}
```

- 通过`Import`引入`EurekaServerInitializerConfiguration`来引导Eureka Server的启动过程。

```java
@Configuration
public class EurekaServerInitializerConfiguration
      implements ServletContextAware, SmartLifecycle, Ordered {
    // 省略
   @Override
   public void start() {
      new Thread(new Runnable() {
         @Override
         public void run() {
            try {
               // TODO: is this class even needed now?
               eurekaServerBootstrap.contextInitialized(
                     EurekaServerInitializerConfiguration.this.servletContext);
               log.info("Started Eureka Server");

               publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
               EurekaServerInitializerConfiguration.this.running = true;
               publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
            }
            catch (Exception ex) {
               // Help!
               log.error("Could not initialize Eureka servlet context", ex);
            }
         }
      }).start();
   }
    // 省略
}
```

由于`EurekaServerInitializerConfiguration`实现了`SmartLifecycle`接口中的`start`方法。该接口方法在Spring容器初始化完毕之后会被调用。剩下的就委托给真正的启动引导类`EurekaServerBootstrap`的`contextInitialized`了。

```java
public void contextInitialized(ServletContext context) {
   try {
       // 初始化环境变量
      initEurekaEnvironment();
       // 初始化Server端上下文
      initEurekaServerContext();

      context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
   }
   catch (Throwable e) {
      log.error("Cannot bootstrap eureka server :", e);
      throw new RuntimeException("Cannot bootstrap eureka server :", e);
   }
}
```

## 启动过程分析

在上面我们可得知，在自动装配过程中创建了如下的对象。

- `InstanceRegistry`
- `RefreshablePeerEurekaNodes`
- `DefaultEurekaServerContext`
- `EurekaServerBootstrap`

等Spring 容器将所有的对象创建完毕后，再调用`EurekaServerInitializerConfiguration`的`contextInitialized`方法。

不得不说大佬写的代码还真的难懂，等我点开`DefaultEurekaServerContext`时，看到了如下的代码眼泪就流了下来

```java
@PostConstruct
@Override
public void initialize() {
    logger.info("Initializing ...");
    peerEurekaNodes.start();
    try {
        registry.init(peerEurekaNodes);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    logger.info("Initialized");
}
```

隐藏了一个`@PostConstruct`注解（该注解功能是，在创建对象之前，先调用被该注解注释的方法内容），所以这里也是一个隐藏的比较深的引导启动点。

那么总结下来服务端的启动流程就有两条主线路，由于启动过程代码过于繁杂。下面给出整体的启动时序图，以便从整体上把握脉络，体会其设计的原理。

![](./images/03_01.png)

![](./images/03_02.png)

以上两个时序图可以看出，Eureka Server端仅仅提供了简单的定时任务去更新缓存信息，来保护其时效性。极端场景下有可能服务已经离线两分钟了才被剔除掉。（默认剔除时间是60s运行一次，默认认为服务离线规则为90s内没有续约上报状态。那么极端场景下就需要两个剔除检测周期才能有效的剔除离线的服务）

## Peer to Peer

从这里开始才是正题（手动滑稽）。在多机部署的同时，虽然保证服务的高可用，但是同样也会带来数据的不一致的副作用。那么Eureka是如何通过何种协议互通数据的呢，节点间是如何协同的呢。

- 使用HTTP协议，
- 被动接受由**Provider** 服务提供方发送的`registry`、`cancle`、`renew`请求。接受到请求后，遍历其他的**Peer**发送相同的请求完成节点间的数据同步操作。

概括完了原理，下面进行源码分析过程。服务端负责执行执行同步的核心关键类为`PeerAwareInstanceRegistryImpl`。用来处理数据同步操作的几个关键方法内容如下：

```java
@Override
public void register(final InstanceInfo info, final boolean isReplication) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    super.register(info, leaseDuration, isReplication);
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}

@Override
public boolean renew(final String appName, final String id, final boolean isReplication) {
    if (super.renew(appName, id, isReplication)) {
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}

public boolean cancel(final String appName, final String id,
                          final boolean isReplication) {
    if (super.cancel(appName, id, isReplication)) {
        replicateToPeers(Action.Cancel, appName, id, null, null, isReplication);
        synchronized (lock) {
            if (this.expectedNumberOfClientsSendingRenews > 0) {
                // Since the client wants to cancel it, reduce the number of clients to send renews
                this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews - 1;
                updateRenewsPerMinThreshold();
            }
        }
        return true;
    }
    return false;
}
```

所有的方法最终都是通过调用方法`replicateToPeers`来完成同步工作。

```java
private void replicateToPeers(Action action, String appName, String id,
                              InstanceInfo info /* optional */,
                              InstanceStatus newStatus /* optional */, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }

        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}

private void replicateInstanceActionsToPeers(Action action, String appName,
                                                 String id, InstanceInfo info, InstanceStatus newStatus,
                                                 PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register:
                node.register(info);
                break;
            case StatusUpdate:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
    }
}
```

而最终负责执行真正同步工作的是`PeerEurekaNode`的方法。

```java
public void register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    batchingDispatcher.process(
            taskId("register", info),
            new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                public EurekaHttpResponse<Void> execute() {
                    return replicationClient.register(info);
                }
            },
            expiryTime
    );
}
```

从上面代码中可以大致推测出，最终通过线程池异步的执行HTTP调用来完成最终的数据同步过程。（关于异步执行设计原理后续单独开篇剖析）。

##  总结概述

本篇通过分析Eureka的启动过程，剖析了集群模式下节点之间的信息同步过程。同时介绍了每个节点启动的后台线程的作用，有的用来剔除无效的注册节点信息，有的用来更新集群的节点信息，有的则用来更新来自客户端的续约阀值信息（该值和Eureka Server的自我保护机制息息相关，后面再[高可用篇章](./high_availability.md)会进行介绍）。整体的设计还是非常简单有效的，但是在代码实现上稍显复杂。