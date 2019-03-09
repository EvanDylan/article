## Scrapy架构

### 数据流及Scrapy的组件

流程图展现了Scrapy的组件以及系统内部处理数据的流向

![scrapy_architecture_01](./scrapy_architecture_01.png)

### 组件

#### Scrapy 引擎

Scrapy引擎控制数据在各个组件键的流向，并在对应的行为发生的时候触发相应的事件。

#### Scheduler

调度器接收来自引擎的请求并将它们入队等待引擎的调度请求将对应的请求传递给引擎。

#### Downloader

下载器接收引擎的请求并抓取对应的web页面返回给引擎，引擎又将对应的抓取内容传给spider。

#### Spiders

蜘蛛是Scrapyy用户开发的内容，用来处理转换抓取内容、提取**items**。

#### Item Pipeline

一旦数据被**Spiders**处理之后，就由Item Pipeline接收继续处理其他任务。比如：

数据清洗、验证、落地（保存到数据库）。

#### Downloader middlewares

引擎和爬虫之间的钩子，可以在Request被**Downloader**执行前以及**Response**被引擎接收到之前执行特殊的处理逻辑工作。

假如你有以下的需求可以考虑使用**Downloader middlewares**:

1. 在请求被发送到**Downloader** 之前处理请求内容。
2. 在**spider**接收到之前更改返回内容。
3. 丢弃本次的抓取内容，重新发送一个新的请求。
4. 不执行抓取步骤，直接返回Response。
5. 直接丢弃某些抓取内容。

#### Spider middlewares

引擎和爬虫之间的钩子，可以在引擎将Response发给爬虫时、爬虫将处理好的Item返回给引擎时执行特殊的处理逻辑工作。

假如你有以下的需求可以考虑使用**Spider middlewares**

1. 处理爬虫异常。
2. 前置处理爬虫的输出内容，改变、添加、删除request或者items。
3. 当某些返回内容发生时主动抛出异常。

### 事件驱动网络

Scrapy使用了Twisted(广受欢迎的事件驱动网络框架)，因此Scrapy实现了并发条件下的异步非阻塞。

### 数据流程

1. **Engine**从**Spider**获取初始化的Requests。
2. **Engine**调度Requests到**Scheduler**并要求**Scheduler**出队下一个Requests。
3. **Scheduler**出队一个请求给**Engine**。
4. **Engine**发送请求到**Downloader**，穿梭通过**Downloader middlewares**。
5. 一旦**Downloader**完成下载工作就会生成Response，并发送Response到**Engine**，再次穿梭通过**Downloader middlewares**。
6. **Engine**收到来自**Downloader**的Response将它发送到**Spider**,穿梭通过**Spider middlewares**。
7. **Spider**处理Response并返回抓取到的Items和一个新的请求到**Engine**，再次穿梭通过**Spider middlewares**。
8. **Engine**发送处理过的Items到**Item Pipeline**，然后发送处理过的Requets到**Scheduler**并获取下个需要处理的Request。
9. 重复步骤，直到没有新的请求进入。