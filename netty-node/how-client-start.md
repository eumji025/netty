## 前言

前面已经详细的介绍了netty的服务端是如何进行启动的以及如何监听客户端的OP_ACCEPT和OP_READ事件的，那么我们会到最初的地方，来看看作为一个客户端来说我们应该如何连接服务端并注册和发送数据呢？

而得力于netty统一的编程模型，让我们的编码显得格外的轻松和惬意，具体如何进行我们将以一个例子的方式来开始演示以及分析

## DEMO

前面也说过了netty使用统一的编程模型，其目的就是为了减少我们之前使用NIO，BIO不同IO模型的切换导致的复杂的编码以及处理机制，下面演示一个netty基于NIO的客户端示例

```java
public static void main(String[] args) throws InterruptedException {

    NioEventLoopGroup clientLoopGroup = new NioEventLoopGroup();

    Bootstrap bootstrap = new Bootstrap();
    bootstrap.group(clientLoopGroup).channel(NioSocketChannel.class).handler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ch.pipeline().addLast(new EchoClientHandler());
        }
    });
    ChannelFuture channelFuture = bootstrap.connect("localhost",8080).sync();
    channelFuture.channel().closeFuture().sync();
}

private static class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        System.out.println("receive server info :"+msg.toString(Charset.defaultCharset()));
    }
}
```

从上述的代码看到，整体的编程模型和使用方式没有太大的变化，客户端只使用一个EventLoopGroup来进行处理，其实也很正常因为毕竟他是有客户端主动发起的，所以不需要服务端那种分发的操作。然后也是用Bootstrap这样我们就不会注册服务端的ACCEPT事件也不会注册`ServerBootstrapAcceptor`到我们pipeline。最后启动的时候也会有少许的不同，采用connect方法表示主动的向别人发起连接。

所以毫无疑问我们将会从connect方法开始进行分析的逻辑

```java
public ChannelFuture connect(SocketAddress remoteAddress) {
    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    }

    validate();
    return doResolveAndConnect(remoteAddress, config.localAddress());
}
```

这里会校验remoteAddress是否为空以及validate方法校验handler是否为空，不满足则无法启动，这些都是必要条件。

然后我们继续看`doResolveAndConnect`方法具体链接的过程

```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();//构建channel并初始化以及注册
    final Channel channel = regFuture.channel();

    if (regFuture.isDone()) {//判断是否已经完成
        if (!regFuture.isSuccess()) {//判断是否成功，isDone并不代表一定成功
            return regFuture;
        }
        //调用doResolveAndConnect0处理
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else {
        // 预备注册的包装
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {//添加完成的监听器
                //异常检查，存在异常则设置启动失败原因
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    //设置已注册
                    promise.registered();
                    //并调用调用doResolveAndConnect0处理
                    doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

是否看起来和我们的server启动非常的相似。都是调用io.netty.bootstrap.AbstractBootstrap#initAndRegister方法注册，但是不同的是对于BootStrap的init方法实现是有区别的

```java
void init(Channel channel) throws Exception {
    //注册handler到pipeline
    ChannelPipeline p = channel.pipeline();
    p.addLast(config.handler());
	//option设置
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }
	//attrs设置
    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }
    }
}
```

可以看到这里和我们ServerBootStrap的init方法还是有区别的，这里仅仅对我们前面对bootStrap的赋值转移到channel中，并不会产生新的内置的channelInitializer的特殊处理，也不会产生`ServerBootstrapAcceptor`。

然后我们继续回到我们doResolveAndConnect方法中，我们继续看我们获取channel的Future对象后的处理，其实也和我们server中的处理比较相似，只是调用方法名不同，那么其中的逻辑我们接下来重点看一下

```java
private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                           final SocketAddress localAddress, final ChannelPromise promise) {
    try {
        final EventLoop eventLoop = channel.eventLoop();
        //获取eventLoop的笛子解析器
        final AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);
        //判断远程地址是否支持，或者地址已经被解析
        if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
            // 进行连接的操作
            doConnect(remoteAddress, localAddress, promise);
            return promise;
        }
        //用resolver解析地址
        final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);

        if (resolveFuture.isDone()) {//判断地址解析是否完成
            final Throwable resolveFailureCause = resolveFuture.cause();
            if (resolveFailureCause != null) {//存在异常，则关闭并设置异常原因到promise
                // Failed to resolve immediately
                channel.close();
                promise.setFailure(resolveFailureCause);
            } else {
                //否则进行连接操作
                doConnect(resolveFuture.getNow(), localAddress, promise);
            }
            return promise;
        }

        // 异步的没有完成则添加监听
        resolveFuture.addListener(new FutureListener<SocketAddress>() {
            @Override
            public void operationComplete(Future<SocketAddress> future) throws Exception {
                if (future.cause() != null) {
                    channel.close();
                    promise.setFailure(future.cause());
                } else {
                    doConnect(future.getNow(), localAddress, promise);
                }
            }
        });
    } catch (Throwable cause) {
        promise.tryFailure(cause);
    }
    return promise;
}
```

其实从上面的代码中可以看出目的就是一件事，那就调用`doConnect`方法进行远程地址的连接操作，但是为了达到这个目的需要先对地址解析，如果已经解析过则我们就直接连接，不需要每次解析，

如果我们没有解析过，则首次进行异步的解析，如果异步在我们向下执行之前已经完成，则直接调用`doConnect`方法处理，否则就是通过监听器的方式进行回调。

所以最终我们还是逃不过分析这里重要的doConnect方法具体实现

```java
private static void doConnect(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {

        // 获取channel，提交任务到制定的eventLoop中执行
        final Channel channel = connectPromise.channel();
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (localAddress == null) {
                    channel.connect(remoteAddress, connectPromise);
                } else {
                    channel.connect(remoteAddress, localAddress, connectPromise);
                }
                connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            }
        });
    }
```

还是一样，netty管用模式将任务提交和channel制定的eventLoop进行操作，所以我们重点内容还是channel的connect方法，我们前文中设置的channel为NioSocketChannel,所以我们继续看，毫不意外的因为这里是包装所以还是调用我们pipeline的链路去处理，我们直接看一下调用链路图

![](images\client-channel-pipeline-handler.png)

所以最终还是和服务端有点相似的调用了NioSocketChannelUnsafe的父类connect方法

```java
//io.netty.channel.nio.AbstractNioChannel.AbstractNioUnsafe#connect
public final void connect(
    final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {//不可用则结束
        return;
    }

    try {
        if (connectPromise != null) {//判断连接异步对象是否已经存在
            // Already a connect in process.
            throw new ConnectionPendingException();
        }

        boolean wasActive = isActive();
        if (doConnect(remoteAddress, localAddress)) {//进行连接处理
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // Schedule connect timeout.连接超时的时长
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {//构造一个延时任务
                    @Override
                    public void run() {//异常处理
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                            new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {//如果超时的future存在，则取消
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```

其实上述这一长段的代码就只有两件事：

1.进行连接的操作

2.操作成功和操作失败的处理，失败的时候根据超时时长，注册延时任务来检测是否是失败，失败了则进行异常处理。另外还会添加一个取消的监听器如果执行到这里被取消了则进行取消的逻辑

这里我们先看看doConnect方法具体的处理逻辑

```java
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {//如果是本地地址，则进行绑定
        doBind0(localAddress);
    }

    boolean success = false;
    try {//否则借住SocketUtils进行连接
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if (!connected) {//连接失败则注册等待被connect的事件
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}
```

其实这里也很简单，如果是本地的地址那么则直接使用NIO的bind方法进行处理，否则我们就借助SocketUtils方法进行连接操作，如果连接不成功则注册一个等待OP_CONNECT事件等待别人连接，所以这里我们继续看看SocketUtils工具类的实现

```java
protected SocketChannel javaChannel() {
    return (SocketChannel) super.javaChannel();
}
public static boolean connect(final SocketChannel socketChannel, final SocketAddress remoteAddress)
    throws IOException {
    try {
        return AccessController.doPrivileged(new PrivilegedExceptionAction<Boolean>() {
            @Override
            public Boolean run() throws IOException {
                return socketChannel.connect(remoteAddress);
            }
        });
    } catch (PrivilegedActionException e) {
        throw (IOException) e.getCause();
    }
}

```

最后还是调用JAVA中nio的channel.connect方法注册逻辑。



### 消息

至于发送消息和消息的接受其实和我们在server中的EventLoop循环监听没有区别，最后都离不开unsafe<NioSocketChannelUnsafe>的逻辑执行。

其中比较麻烦的就是pipeline的扭转过程。

