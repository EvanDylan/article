# 服务端启动流程

服务端示例代码如下：

```java
public class Server {

    public static void main(String[] args) {

        ServerBootstrap s = new ServerBootstrap();

        NioEventLoopGroup boss = new NioEventLoopGroup(1);
        NioEventLoopGroup work = new NioEventLoopGroup();

        try {
            ChannelFuture channelFuture = s.group(boss, work)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler())
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringEncoder());
                            pipeline.addLast(new StringDecoder());
                            pipeline.addLast(new RegisteredHandler());
                            pipeline.addLast(new BroadcastMessageHandler());
                        }
                    })
                    .bind(7000).sync();

            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            boss.shutdownGracefully();
            work.shutdownGracefully();
        }
    }
}
```

