---

title: 【网络编程】Netty 实现WebSocket 服务
tag: 网络编程
categories: 网络编程
date: 2021/4/13 20:46:25
index_img:   <https://s2.loli.net/2024/01/23/DOQteIg2SAsfxwm.png>
---

# Netty 实现WebSocket 服务

在使用 Netty 实现 WebSocket 服务时，我们需要几个关键步骤来确保服务的正确运行和优化性能。首先，我们需要设置 Netty 服务器，并配置相应的通道初始化器来处理 WebSocket 请求。其次，我们需要实现 WebSocket 处理器来管理连接、消息和关闭事件。

1. **设置 Netty 服务器**：启动一个 Netty 服务器，监听特定端口并等待 WebSocket 客户端的连接。
2. **配置通道初始化器**：在通道初始化器中添加必要的处理器，例如 `HttpServerCodec`、`HttpObjectAggregator` 和 `WebSocketServerProtocolHandler`。
3. **实现 WebSocket 处理器**：编写自定义处理器来处理 WebSocket 的连接、消息和断开事件。确保处理器能够处理文本消息、二进制消息以及心跳检测。
4. **实现WebSocket握手鉴权**：实际情况服务端需要对客户端的握手请求进行相关判断策略验证，通过才可以连接成功，不通过就认为是非法链接拒绝链接。

通过以上步骤，我们可以使用 Netty 搭建一个高效的 WebSocket 服务，支持实时通信需求。

在工作中通常是使用的是将Netty服务当作一个bean交给了spring管理。下面我们开始数据搭建过程

# 一、设置Netty服务器

**Menservants** 类是spring的一个bean 在初始化的时候，和销毁时自动对WebSocket服务进行启动和关闭

```java

@Component
public class CAgentServer {

	private static Logger log = LoggerFactory.getLogger(CAgentServer.class);

	@Value("${server.cagent.listener.port}")
	private int port = 8888;
	@Autowired
	private CAgentServerInitializer serverInitializer;

	private EventLoopGroup bossGroup;
	private EventLoopGroup workerGroup;
	private ChannelFuture channelFuture;

	@PostConstruct
	public void run() throws Exception {
		log.info("Before run server");
		bossGroup = new NioEventLoopGroup();
		workerGroup = new NioEventLoopGroup(512);
		
		ServerBootstrap b = new ServerBootstrap();
		b.group(bossGroup, workerGroup);
		b.channel(NioServerSocketChannel.class);
		// b.channel(OioServerSocketChannel.class);
		b.childHandler(serverInitializer);
		b.option(ChannelOption.SO_BACKLOG, 128);
		b.childOption(ChannelOption.SO_KEEPALIVE, true);
		channelFuture = b.bind(port).sync();
	}

	@PreDestroy
	public void destory() throws Exception {
		try {
			channelFuture.channel().close().sync();
			log.info("after stop server");
		} finally {
			workerGroup.shutdownGracefully();
			bossGroup.shutdownGracefully();
		}
	}
}

```

这一部分代码基本上没什么变化 ，`b.childHandler(serverInitializer);`这里设置了一个通道初始化器，这里包含了我们对通道的所有处理逻辑。

# 二、配置通道初始化器

**ChannelInitializer** 类用于设置 Netty 处理管道，包括处理 HTTP 请求和 WebSocket 协议的处理器。以下是一个示例实现：

```java
@Component
public class CAgentServerInitializer extends ChannelInitializer<SocketChannel> {

    private static final Logger log = LoggerFactory.getLogger(CAgentServerInitializer.class);

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        // 添加 HTTP 服务端编解码器
        pipeline.addLast(new HttpServerCodec());
        // 添加 HTTP 对象聚合器
        pipeline.addLast(new HttpObjectAggregator(65536));
        // 添加 WebSocket 协议处理器
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
        // 添加自定义的 WebSocket 处理器
        pipeline.addLast(new CAgentWebSocketHandler());

        log.info("WebSocket channel initialized");
    }
}

```

通过以上步骤，我们可以确保 Netty 服务器能够正确处理 WebSocket 请求，并且可以在必要的情况下进行握手鉴权。

# 三、自定义 WebSocket 处理器

自定义的 WebSocket 处理器 `CAgentWebSocketHandler` 用于处理连接成功事件、消息事件和断开事件。以下是一个示例实现：

```java
public class CAgentWebSocketHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    private static final Logger log = LoggerFactory.getLogger(CAgentWebSocketHandler.class);

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        log.info("Client connected: {}", ctx.channel().id().asLongText());
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        log.info("Client disconnected: {}", ctx.channel().id().asLongText());
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        String request = msg.text();
        log.info("Received message: {}", request);

        // 处理消息，响应给客户端
        ctx.channel().writeAndFlush(new TextWebSocketFrame("Server received your message: " + request));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        log.error("Error occurred: ", cause);
        ctx.close();
    }
}

```

在这个处理器中，我们实现了以下方法：

1. `handlerAdded(ChannelHandlerContext ctx)`：当客户端成功连接时调用。
2. `handlerRemoved(ChannelHandlerContext ctx)`：当客户端断开连接时调用。
3. `channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg)`：当服务器接收到客户端发送的消息时调用。
4. `exceptionCaught(ChannelHandlerContext ctx, Throwable cause)`：当发生异常时调用，记录错误并关闭连接。

## 3.1 **ChannelHandler的生命周期与事件处理机制**

![Untitled](Netty%20%E5%AE%9E%E7%8E%B0WebSocket%20%E6%9C%8D%E5%8A%A1%20657c76b554e648f6aadfc6c919ca1493/Untitled.png)

### **概述**

Netty的`ChannelHandler`是处理网络事件（如数据读取、数据写入、连接建立、连接关闭等）的核心组件。

在Netty中，`ChannelHandler`的生命周期与`Channel`的状态紧密相关，主要涉及到以下几个阶段：

1. **初始化（Initialization）**:
    - `handlerAdded` 方法被调用，这通常发生在`ChannelPipeline`初始化时，表示一个新的`ChannelHandler`被加入到`ChannelPipeline`中。
2. **注册（Registration）**:
    - `channelRegistered` 方法被调用，这表示`Channel`已经成功注册到它的`EventLoop`上。
3. **激活（Activation）**:
    - `channelActive` 方法被调用，表示`Channel`已经成功激活，可以开始接收和发送数据。
4. **读取数据（Read）**:
    - `channelRead` 方法被调用，这表示从`Channel`中读取到了数据。
5. **读完成（Read Complete）**:
    - `channelReadComplete` 方法被调用，这表示一次读取操作完成。
6. **关闭（Deactivation）**:
    - `channelInactive` 方法被调用，表示`Channel`与远端主机失去了连接，变成了非激活状态。
7. **注销（Deregistration）**:
    - `channelUnregistered` 方法被调用，表示`Channel`从它的`EventLoop`上注销。
8. **移除（Removal）**:
    - `handlerRemoved` 方法被调用，表示`ChannelHandler`被从`ChannelPipeline`中移除。

这些方法的调用顺序与`Channel`的状态转换顺序相对应，形成了一个完整的生命周期。在实际应用中，根据不同的需求，开发者可以重写这些方法来实现自定义的逻辑处理，比如处理超时、心跳保活、数据编解码等。

<aside>
💡 常见用法

handlerAdded() 与 handlerRemoved()可以用在一些资源的申请和释放
 channelActive() 与 channelInActive()可以统计单机的连接数，channelActive() 被调用，连接数加一，channelInActive() 被调用，连接数减一。channelActive() 还可以实现ip黑白名单的过滤

channelRead()用来拆包读取信息

channelReadComplete()实现批量刷新的机制，这样channelRead()中只使用write() 方法而不用writeAndFlush()每次都刷新写入到缓存，从而提高性能。

</aside>

**生命周期Handler Dem**

```java
package com.artisan.reconnect;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

/**
 *  handler的生命周期回调接口调用顺序:
 *  handlerAdded -> channelRegistered -> channelActive -> channelRead -> channelReadComplete
 *  -> channelInactive -> channelUnRegistered -> handlerRemoved
 *
 * handlerAdded: 新建立的连接会按照初始化策略，把handler添加到该channel的pipeline里面，也就是channel.pipeline.addLast(new LifeCycleInBoundHandler)执行完成后的回调；
 * channelRegistered: 当该连接分配到具体的worker线程后，该回调会被调用。
 * channelActive：channel的准备工作已经完成，所有的pipeline添加完成，并分配到具体的线上上，说明该channel准备就绪，可以使用了。
 * channelRead：客户端向服务端发来数据，每次都会回调此方法，表示有数据可读；
 * channelReadComplete：服务端每次读完一次完整的数据之后，回调该方法，表示数据读取完毕；
 * channelInactive：当连接断开时，该回调会被调用，说明这时候底层的TCP连接已经被断开了。
 * channelUnRegistered: 对应channelRegistered，当连接关闭后，释放绑定的workder线程；
 * handlerRemoved： 对应handlerAdded，将handler从该channel的pipeline移除后的回调方法。
 */
public class LifeCycleInBoundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRegistered(ChannelHandlerContext ctx)
            throws Exception {
        System.out.println("channelRegistered: channel注册到NioEventLoop");
        super.channelRegistered(ctx);
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) 
            throws Exception {
        System.out.println("channelUnregistered: channel取消和NioEventLoop的绑定");
        super.channelUnregistered(ctx);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) 
            throws Exception {
        System.out.println("channelActive: channel准备就绪");
        super.channelActive(ctx);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) 
            throws Exception {
        System.out.println("channelInactive: channel被关闭");
        super.channelInactive(ctx);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) 
            throws Exception {
        System.out.println("channelRead: channel中有可读的数据" );
        super.channelRead(ctx, msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) 
            throws Exception {
        System.out.println("channelReadComplete: channel读数据完成");
        super.channelReadComplete(ctx);
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) 
            throws Exception {
        System.out.println("handlerAdded: handler被添加到channel的pipeline");
        super.handlerAdded(ctx);
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) 
            throws Exception {
        System.out.println("handlerRemoved: handler从channel的pipeline中移除");
        super.handlerRemoved(ctx);
    }
}
```

# 四、心跳超时剔除

为了确保 WebSocket 连接的稳定性和及时释放资源，我们可以在服务器中实现心跳检测机制，并在连接超时时剔除不活跃的连接。以下是一个实现心跳检测的示例：

```java
@Component
public class CAgentServerInitializer extends ChannelInitializer<SocketChannel> {

    private static final Logger log = LoggerFactory.getLogger(CAgentServerInitializer.class);

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        // 添加 HTTP 服务端编解码器
        pipeline.addLast(new HttpServerCodec());
        // 添加 HTTP 对象聚合器
        pipeline.addLast(new HttpObjectAggregator(65536));
        // 添加 WebSocket 协议处理器
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
        // 添加心跳检测处理器（如果超过60秒没有接收到客户端的心跳包，则关闭连接）
        pipeline.addLast(new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS));
        pipeline.addLast(new HeartbeatHandler());
        // 添加自定义的 WebSocket 处理器
        pipeline.addLast(new CAgentWebSocketHandler());

        log.info("WebSocket channel initialized");
    }
}

public class HeartbeatHandler extends ChannelInboundHandlerAdapter {

    private static final Logger log = LoggerFactory.getLogger(HeartbeatHandler.class);

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.READER_IDLE) {
                log.info("No heartbeat received from client, closing connection: {}", ctx.channel().id().asLongText());
                ctx.close();
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
}

```

在此实现中，我们在 `CAgentServerInitializer` 中添加了 `IdleStateHandler` 和 `HeartbeatHandler` 两个处理器：

1. `IdleStateHandler`：用于检测连接的空闲状态。如果在指定时间内（如60秒）没有接收到客户端的任何数据，该处理器会触发 `IdleStateEvent` 事件。
2. `HeartbeatHandler`：继承自 `ChannelInboundHandlerAdapter`，用于处理 `IdleStateEvent` 事件。如果检测到读取空闲状态（即超过指定时间没有接收到客户端的心跳包），则关闭该连接。

通过这种方式，我们可以确保 WebSocket 服务能够及时剔除不活跃的连接，保持连接的健康状态。

---

## 另外一种操作,阅读下源码

`IdleStateHandler` 内有三个内部类, `ReaderIdleTimeoutTask`,`WriterIdleTimeoutTask ,AllIdleTimeoutTask` 里面都会调用  channelIdle 方法  也就是出现超时事件时 都会执行这个方法

![Untitled](Netty%20%E5%AE%9E%E7%8E%B0WebSocket%20%E6%9C%8D%E5%8A%A1%20657c76b554e648f6aadfc6c919ca1493/Untitled%201.png)

```java
    /**
     * Is called when an {@link IdleStateEvent} should be fired. This implementation calls
     * {@link ChannelHandlerContext#fireUserEventTriggered(Object)}.
     */
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
        ctx.fireUserEventTriggered(evt);
    }
```

如果出现超时直接关闭channel 其实也可以写一个类继承 `IdleStateHandler` 直接重写 `channelIdle` 然后加入判断pipeline内,,逻辑就是对应的事件处理,这样也可以做到,而且方法的参数就是IdleStateEvent  不用判断类型,也算是一个骚操作吧 不过这样不是官方设计的用法。官方推荐第一种用法

```java
@Override
  protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    if (evt.state() == IdleState.READER_IDLE) {
      log.warn("no data received after 60s, channel=" + ctx.channel().remoteAddress().toString()
          + " will close");
      ctx.close();
    }
  }
```

```java
 * <pre>
 * // An example that sends a ping message when there is no outbound traffic
 * // for 30 seconds.  The connection is closed when there is no inbound traffic
 * // for 60 seconds.
 *
 * public class MyChannelInitializer extends {@link ChannelInitializer}&lt;{@link Channel}&gt; {
 *     {@code @Override}
 *     public void initChannel({@link Channel} channel) {
 *         channel.pipeline().addLast("idleStateHandler", new {@link IdleStateHandler}(60, 30, 0));
 *         channel.pipeline().addLast("myHandler", new MyHandler());
 *     }
 * }
 *
 * // Handler should handle the {@link IdleStateEvent} triggered by {@link IdleStateHandler}.
 * public class MyHandler extends {@link ChannelDuplexHandler} {
 *     {@code @Override}
 *     public void userEventTriggered({@link ChannelHandlerContext} ctx, {@link Object} evt) throws {@link Exception} {
 *         if (evt instanceof {@link IdleStateEvent}) {
 *             {@link IdleStateEvent} e = ({@link IdleStateEvent}) evt;
 *             if (e.state() == {@link IdleState}.READER_IDLE) {
 *                 ctx.close();
 *             } else if (e.state() == {@link IdleState}.WRITER_IDLE) {
 *                 ctx.writeAndFlush(new PingMessage());
 *             }
 *         }
 *     }
 * }
```

**相关链接**

https://blog.csdn.net/weixin_43935927/article/details/112001309

https://blog.csdn.net/m0_60259116/article/details/137680824

https://blog.csdn.net/RisenMyth/article/details/104441155