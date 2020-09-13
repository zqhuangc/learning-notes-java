## web 服务器

### 服务端

```java
EventLoopGroup masterGroup = new NioEventLoopGroup();
EventLoopGroup workGroup = new NioEventLoopGroup();

try {
    ServerBootstrap s = new ServerBootstrap();
    s.group(masterGroup, workGroup)
        .channel(NioServerSocketChannel.class)
        .childHandler(new ChannelInitializer<SocketChannel>() {

            @Override
            protected void initChannel(SocketChannel socketChannel) throws Exception {
                ChannelPipeline pipeline = socketChannel.pipeline();

                // 解析自定义协议
                pipeline.addLast(new IMDecoder());
                pipeline.addLast(new IMEncoder());
                pipeline.addLast(new SocketHandler());

                /** 解析 HTTP 请求 */
                pipeline.addLast(new HttpServerCodec());
                //主要是将同一个http请求或响应的多个消息对象变成一个 fullHttpRequest完整的消息对象
                pipeline.addLast(new HttpObjectAggregator(64 * 1024));
                //主要用于处理大数据流,比如一个1G大小的文件如果你直接传输肯定会撑暴jvm内存的 ,使用该 handler我们就不用考虑这个问题了
                pipeline.addLast(new ChunkedWriteHandler());
                // 自定义处理
                pipeline.addLast(new HttpHandler());

                /** 解析 webSocket 请求 */
                pipeline.addLast(new WebSocketServerProtocolHandler("/im"));
                // 自定义处理
                pipeline.addLast(new WebSocketHandler());
            }
        })
        .option(ChannelOption.SO_BACKLOG,1024)
        .childOption(ChannelOption.SO_KEEPALIVE, true);

    ChannelFuture f = s.bind(this.port).sync();
    LOG.info("服务已启动,监听端口" + this.port);
    f.channel().closeFuture().sync();

} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    masterGroup.shutdownGracefully();
    workGroup.shutdownGracefully();
}
```



### 客户端

```java
EventLoopGroup workerGroup = new NioEventLoopGroup();

try {
    Bootstrap b = new Bootstrap();
    b.group(workerGroup)
        .channel(NioSocketChannel.class)
        .option(ChannelOption.SO_KEEPALIVE, true)
        .handler(new ChannelInitializer<SocketChannel>() {

            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline pipeline = ch.pipeline();
                pipeline.addLast(new IMDecoder());
                pipeline.addLast(new IMEncoder());
                pipeline.addLast(clientHandler);
            }
        });

    ChannelFuture f = b.connect("localhost", 80).sync();
    f.channel().closeFuture().sync();
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    workerGroup.shutdownGracefully();
}
```







### chat