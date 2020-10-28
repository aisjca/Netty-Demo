# Netty-Http Demo

这是一个基于Netty的简单Http服务器，主要目的用于熟悉Netty网络编程和Netty的一些组件。

该服务器主要分为以下几个部分。

1. 服务启动类
2. 服务业务逻辑处理类
3. 客户端启动类
4. 客户端业务逻辑处理类

## 服务启动类

首先创建引导器，然后配置线程模型，通过引导器绑定业务逻辑处理类，再配置一些参数，最后绑定端口就可以启动了。

```java
public class HttpServer {
    public void start(int port) {
        ServerBootstrap s = new ServerBootstrap();
        EventLoopGroup boss = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();
        try{
            s.group(boss, worker)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(port)
                    .childHandler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception{
                            ch.pipeline()
                                    .addLast(new HttpServerHandler());
                        }
                    });
						s.childOption(ChannelOption.SO_KEEPALIVE, true);
            ChannelFuture future = s.bind(port).sync();
            System.out.println("Server is started,port is " + port);
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        new HttpServer().start(8001);
    }
}
```

该服务启动类主要可以分为三个部分

* 配置线程池

Netty 是采用 Reactor 模型进行开发，主要有三种 Reactor 模式：单线程模式、多线程模式、主从多线程模式，本来采用**主从多线程模式**。

boss是主Reactor，worker是从NioEventLoopGroup，它们使用不用的NioEventLoopGroup，主Reactor主要负责接收请求，然后把Channel注册到从Reactor。从Reactor主要负责Channel生命周期内IO事件

```java
 ServerBootstrap s = new ServerBootstrap();
 EventLoopGroup boss = new NioEventLoopGroup();
 EventLoopGroup worker = new NioEventLoopGroup();
 s.group(boss, worker)
```

* Channel初始化

  * 设置Channel类型

    ```java
    b.channel(NioServerSocketChannel.class)
    ```

    这是初始化一个非阻塞的SocketChannel，除了这个以外，还可以切换为OioServerSocketChannel、EpollServerSocketChannel 等。

  * 注册ChannelHandler

    在 Netty 中可以通过 ChannelPipeline 去注册多个 ChannelHandler，每个 ChannelHandler 各司其职，这样就可以实现最大化的代码复用。

    ```java
    s.childHandler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(SocketChannel ch) throws Exception{
                                ch.pipeline()
                                        .addLast(new HttpServerHandler());
                            }
                        });
    ```

    ServerBootstrap 的 childHandler() 方法需要注册一个 ChannelHandler。ChannelInitializer是实现了 ChannelHandler接口的匿名类，通过实例化 ChannelInitializer 作为 ServerBootstrap 的参数.

    Pipeline 底层是一个双链表，管理了多个 ChannelHandler，可以利用addLast()方法添加多个 ChannelHandler，I/O事件会在多个 ChannelHandler依次执行。

  * 设置Channel参数

    ```java
    s.childOption(ChannelOption.SO_KEEPALIVE, true);
    ```

    设置 Channel 属性有option和childOption两个方法,option 主要负责设置 Boss 线程组，而 childOption 对应的是 Worker 线程组。

    常用的参数含义

    |          参数          |                             含义                             |
    | :--------------------: | :----------------------------------------------------------: |
    |      SO_KEEPALIVE      | 设置为 true 代表启用了 TCP SO_KEEPALIVE 属性，TCP 会主动探测连接状态，即连接保活 |
    |       SO_BACKLOG       | 已完成三次握手的请求队列最大长度，同一时刻服务端可能会处理多个连接，在高并发海量连接的场景下，该参数应适当调大 |
    |      TCP_NODELAY       | 设置为 true 标识 TCP 会将网络数据包累积到一定量才会发送，会造成一定的数据延迟。如果对数据传输延迟敏感，那么应该禁用该参数 |
    |       SO_SNDBUF        |                    TCP 数据发送缓冲区大小                    |
    |       SO_RCVBUF        |                    TCP数据接收缓冲区大小                     |
    |       SO_LINGER        |        设置延迟关闭的时间，等待缓冲区中的数据发送完成        |
    | CONNECT_TIMEOUT_MILLIS |                      建立连接的超时时间                      |

    

* 端口绑定

直接绑定port端口启动即可

```java
ChannelFuture future = s.bind(port).sync();
```

## 服务业务逻辑处理类

HttpServerHandler是ChannelHandler的逻辑处理类，负责接收数据，和发送数据给客户端。

```java
public class HttpServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("Server receive msg is " + buf.toString(CharsetUtil.UTF_8));
        ctx.write(Unpooled.copiedBuffer("Hello Client",CharsetUtil.UTF_8));
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
                .addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

### channelRead

对于每个传入的信息都要调用

```java
@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("Server receive msg is " + buf.toString(CharsetUtil.UTF_8));
        ctx.write(Unpooled.copiedBuffer("Hello Client",CharsetUtil.UTF_8));
    }
```

将接收到的信息发送给客户，而不冲刷出站信息。

### channelReadComplete

通知ChannelInboundHandler最后一次对channelRead()的调用是当前批量读取中的最后一条消息

```java 
@Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
                .addListener(ChannelFutureListener.CLOSE);
    }
```

将未处理的信息冲刷到远程节点，并关闭Channel

### exceptionCaught异常捕获

```java 
@Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
```



## 客户端启动类

```java
public class HttpClient {
    public void connect(String host, int port) throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap b = new Bootstrap();
        try{
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .remoteAddress(new InetSocketAddress(host, port))
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline()
                                    .addLast(new HttpClientHandler());
                        }
                    });
            ChannelFuture future = b.connect().sync();
            future.channel().closeFuture().sync();
        }finally {
            group.shutdownGracefully();
        }

    }
    public static void main(String[] args) throws Exception {
        new HttpClient().connect("127.0.0.1", 8001);
    }
}
```

该服务启动类主要可以分为三个部分

* 配置线程池

* 初始化Channel

  * 设置Channel类型

    ```java
     b.channel(NioSocketChannel.class)
    ```

    这是初始化一个非阻塞的SocketChannel。

  * 注册ChannelHandler

    ```java 
    b.handler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(SocketChannel ch) throws Exception {
                                ch.pipeline()
                                        .addLast(new HttpClientHandler());
                            }
                        });
    ```

    在 Netty 中可以通过 ChannelPipeline 去注册多个 ChannelHandler,I/O事件会在多个 ChannelHandler依次执行。

  * 设置Channel参数

    客户端只能使用option()方法，不能使用childOption。

* 连接服务器

  ```
   ChannelFuture future = b.connect().sync();
  ```

  连接到远程节点，阻塞等待直到完成连接为止。

## 客户端业务逻辑处理类

HttpClientHandler是ChannelHandler的逻辑处理类，负责接收数据，和发送数据给服务端。

```java
public class HttpClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("Hello Server", CharsetUtil.UTF_8));
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        System.out.println("Client receive msg is " + msg.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

### channelActive

当被通知 Channel是活跃的时候，发送一条消息

### channelRead0

接收服务端发送过来的信息。

### exceptionCaught

在发生异常时，记录错误并关闭Channel

# 相关问题

* 为什么客户端逻辑处理类使用的是 SimpleChannelInboundHandler，而服务端逻辑处理类中是使用的 ChannelInboundHandlerAdapter 。

  客户端中，当ChannelRead0()方法完成后， SimpleChannelInboundHandler会释放负责指向保存信息的ByteBuf。

  服务端中，由于仍然需要将传入信息回发给客户端，而write是异步操作，直到channelRead执行完后，write可能还没执行完，因此使用ChannelInboundHandlerAdapter，因为ChannelInboundHandlerAdapter在其时间节点不会释放信息。消息在channelReadComplete()执行完后，调用writeAndFlush()把消息释放。

