## netty start
前面我们已经知道了netty的基本用法，那么本文将开始介绍netty启动一个服务的过程，从源代码的角度去看看netty是如何进行的

##　案例

还是一样我们首先要准备一个简单的netty示例去进行演示，通过debug的方式进行追踪会更加简单，因为充满了异步的操作。我们可能会被绕晕。

```java
public static void main(String[] args) throws InterruptedException {
        NioEventLoopGroup parentEventGroup = new NioEventLoopGroup();
        NioEventLoopGroup childEventGroup = new NioEventLoopGroup();

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        //链式的参数构建过程
        serverBootstrap.group(parentEventGroup,childEventGroup).channel(NioServerSocketChannel.class).childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline().addLast(new SimpleChannelInboundHandler<ByteBuf>() {
                    @Override
                    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf buf) throws Exception {
                        byte[] req = new byte[buf.readableBytes()];
                        buf.readBytes(req);
                        String body = new String(req,"UTF-8");
                        System.out.println("from client:"+body);
                    }
                });
            }
        });

        //真正的启动过程
        ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
        channelFuture.channel().closeFuture().sync();

    }
```
然后我们这里就不编写客户端了，我们直接使用windows的telnet命令或者linux的nc命令进行操作就可以了。

## 分析
主要的入口信息都在上述的代码中，比较重要的就是NioEventLoopGroup和ServerBootstrap初始化相关的信息，channel，handler等
针对不同的东西我们会慢慢随着代码的深入逐个去分析。

### NioEventLoopGroup
这是一个非常重要的类，在我们学习的时候我们大概的就是介绍了parentEventGroup和childEventGroup的作用了。
parentEventGroup负责接收我们socket连接，然后再交给真正的处理这些链接（逻辑的处理，我们常说的自定义协议就在这里进行解析和包装的）

但是从上述的代码我们可以看到两个都是NioEventLoopGroup的对象，那么如何进行区分处理肯定是在ServerBootstrap实际的初始化中进行的，这点我们稍后再说

首先我们看一下NioEventLoopGroup构造函数中是如何进行处理的
```java
/**
 *
 * @param nThreads 线程数  default => 0
 * @param executor 执行器  default => null
 * @param selectorProvider 选择器提供者 default => SelectorProvider.provider()
 * @param selectStrategyFactory 选择器策略工厂 default => DefaultSelectStrategyFactory.INSTANCE
 */
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}

```

这里对构造函数进行了简化，NioEventLoopGroup提供了多个构造函数，这里会逐步的通过默认参数方式调用上文中的这个最终提供的构造函数。

selectStrategyFactory使用的是单例模式的DefaultSelectStrategyFactory.INSTANCE进行提供的（netty中大量使用），因为这里是NIO的实现，所以selectorProvider都是使用Java并发包中的SelectorProvider.provider方法进行获取的。

最后这里会调用父类MultithreadEventLoopGroup的构造函数

```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

需要注意的是后面的几个参数都会被设置边长参数args，这个参数对应于MultithreadEventLoopGroup的`newChild`方法里的args。

这里需要注意默认情况下如果我们没有指定nThreads数值的时候使用的是0，为什么不使用1在这里就会体现。这里我们要看一下DEFAULT_EVENT_LOOP_THREADS对应的是什么样子的

```java
private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        //从properties获取配置[io.netty.eventLoopThreads]的线程数，如果没配置则为1，否则在对比配置的数量和Runtime获取处理器的数量*2
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
    }
```

很简单，这里优先使用我们在系统参数指定的io.netty.eventLoopThreads对应的数值，如果没有再尝试通过NettyRuntime.availableProcessors进行获取，而在NettyRuntime.availableProcessors方法中又会通过系统参数`io.netty.availableProcessors`获取，如果没有配置则使用`Runtime.getRuntime().availableProcessors()`方法

```java
synchronized int availableProcessors() {
    if (this.availableProcessors == 0) {
        final int availableProcessors =
            SystemPropertyUtil.getInt(
            "io.netty.availableProcessors",
            Runtime.getRuntime().availableProcessors());
        setAvailableProcessors(availableProcessors);
    }
    return this.availableProcessors;
}
```

上面我们已经知道了默认线程数构造过程后，我们继续往看父类的构造函数的调用

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```

这里又产生默认的参数EventExecutorChooserFactory，这里和我们之前介绍大的一样使用的是DefaultEventExecutorChooserFactory的单例对象，我们接着向下看

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }
    //如果没有制定executor，则创建默认的ThreadPerTaskExecutor
    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    //通过制定的数量创建executor
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            //构建一个executor数组
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }
    //通过children创建一个chooser
    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

可以从代码看到内容比较长，这里我们总结一下上面做的几件事

1.判断是否存在executor，如果不存在则构建默认的执行器ThreadPerTaskExecutor，这里的newDefaultThreadFactory方法其实就是包装了Java并发包里的ThreadFactory。主要设置了默认的名称（当前class的简称），优先级和是否为守护线程等相关信息。

2.回调newChild方法进行childEventExecutor创建过程，这里实现是在子类NioEventLoopGroup中进行创建的

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
                            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
```

所以这里的NioEventLoop参数对应的是NioEventLoopGroup中对应的参数。暂不深入的进行分析。

3.然后根据childEventExecutor和chooserFactory来创建对应的chooser。这里有多种策略，我们具体来看一下

```java
//io.netty.util.concurrent.DefaultEventExecutorChooserFactory#newChooser
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {//判断是否为2的次方
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

其实就是两种策略，两种策略的实现不太一样

PowerOfTwoEventExecutorChooser： 使用的类似HashMap的位操作来选择合适下标来获取EventExecutor

GenericEventExecutorChooser： 使用的是余除executors.size来获取对应的EventExecutor

4.为executor设置对应的终止future回调监听器

5.children转换成一个只读的set设置到readonlyChildren上。



综上，我们可以看到EventLoopGroup（NioEventLoopGroup）的初始化过程的主要目的就是为了创建内部执行任务的executor，而创建executor的方法是回调了子类的newChild方法，生成的executor对应的是EventLoop对象。



### ServerBootstrap

我们接下来继续看ServerBootstrap的初始化过程以及其启动过程的代码实现,由于我们的ServerBootstrap构造函数是空实现，所以暂且我们不看，这里我们直接去看他的链式初始化过程

```java
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    //在父类中记录父group
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    //子group设置到当前的类childGroup属性中
    this.childGroup = childGroup;
    return this;
}
```

这里会将我们两个group分别绑定到ServerBootStrap对应的变量上，parentGroup设置到AbstractBootstrap#group，childGroup设置到this.childGroup上。

然后我们继续看channel方法制定的channel

```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

这里我们看到其实构造了一个通过反射创建channel的工厂类ReflectiveChannelFactory，然后赋值到channelFactory中，这里提醒一下，如果我们使用的channel拥有无参的构造方法则推荐使用channel方法指定class方法进行反射创建。否则需要自己提供创建channel的工厂方法channelFactory。

然后我们看handler方法和childHandler方法

```java
public B handler(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
    return self();
}
public ServerBootstrap childHandler(ChannelHandler childHandler) {
    if (childHandler == null) {
        throw new NullPointerException("childHandler");
    }
    this.childHandler = childHandler;
    return this;
}
```

可以看到逻辑还是很简单的，就是将赋值到对应的属性上，具体肯定是后续在启动的时候才会调用。



接下来我们看看绑定和异步启动的过程逻辑。

```java
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}
```

bind方法其实有多种重载实现，但是最后调用的上述参数为SocketAddress的这个方法，然后通过validate方法校验group和channelFactory是否初始化，因为这里默认不是在构造函数中进行初始化的，所以需要检查，然后再检验localAddress是否为空。

最终通过doBind的方法进行真正的绑定操作（在netty中有大量的使用do开头的方法进行真实的调用）。

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    //初始化和注册一个channelFuture
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

其实上面的方法主要进行两件事情

1.进行channel的创建和注册过程

2.进行后面的绑定操作（因为有异步的情况所以有可能两个、）



