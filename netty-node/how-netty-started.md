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
            if (!success) {//如果没有成功，在优雅关闭
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {//等待关闭
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

PowerOfTwoEventExecutorChooser： 使用的类似HashMap的位操作来选择合适下标来获取EventExecutor（2^n-1）全为1便于位操作

GenericEventExecutorChooser： 使用的是余除executors.size来获取对应的EventExecutor

其实就是为了极致的性能体验，类似于hashMap的容量调整，自动调整为为2^n

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

2.进行后面的绑定操作（因为有异步的情况所以有可能两个、但是都要调用doBind0方法进行绑定）



### channel注册

首先我们看看initAndRegister方法是如何初始化和注册channel的

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        //通过指定的factory创建channel channelFactory对应我们channel(NioServerSocketChannel.class)方法啊
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    //把channel注册到parentGroup config = ServerBootstrapConfig group = bossGroup
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

1.通过我们的channelFactory创建一个channel，由于我们服务端通过使用的是channel(NioServerSocketChannel.class)方法注册的channelFactory，对应的就是ReflectiveChannelFactory，并通过init方法初始化channel

2.然后我们再想group注册channel，这里的group指向的就是bossEventLoopGroup。

第一步的实例化其实就是调用反射进行创建的，当然不同的channelFactory有不同的实现需要区别对待，这里主要看一下init方法

```java
void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        //每个参数设置到channel中
        setChannelOptions(channel, options, logger);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            //设置每个属性对应的值
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    //子group和handler处理
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        //子参数记录
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
    }
    synchronized (childAttrs) {
        //子属性记录
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    }

    //记录一个channelInitializer
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {//注册bossGroup的handler
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {//向group注册一个任务
                    //设置一个ChannelHandler -等同于reactor中第一个acceptor
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

这个方法主要展示了channel初始化的相关过程，下面简单总结一下

1.根据我们初始化的时候进行的option(ChannelOption<T> option, T value)和attr(AttributeKey<T> key, T value)方法设置的参数，注册到channel中

2.为channel的pipeline注册ChannelInitializer，而在ChannelInitializer内部实现中，会通过eventLoop再执行的方法中指定pipeLine的ChannelHandler，这个handler为Acceptor，我们大概也知道了他的作用是什么了把。



然后我们继续看第二步中group的注册过程。

```java
//io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```

这里是一个包装方法，将我们的channel参数包装成DefaultChannelPromise，然后再调用register方法

```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

这里最终是调用channel的unsafe的register方法进行注册的，由于我们前面没介绍NioServerSocketChannel的实例化的介绍，所以这里看看unsafe是怎么被构建的，首先通过NioServerSocketChannel的构造函数调用父类AbstractChannel的构造方法

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```

可以看到这里构建了channel的ID,unsafe对象以及非常重要的pipeline对象

```java
//io.netty.channel.nio.AbstractNioMessageChannel#newUnsafe
protected AbstractNioUnsafe newUnsafe() {
    return new NioMessageUnsafe();
}
//io.netty.channel.AbstractChannel#newChannelPipeline
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```



接下来我们已经知道了unsafe对象后，我们继续看看unsafe的register方法的注册逻辑

```java
//io.netty.channel.AbstractChannel.AbstractUnsafe#register
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

上面其实只是一个辅助方法，主要是用来判断条件是否满足，然后再根据`eventLoop.inEventLoop`方法判断是采用异步提交任务的方式还是同步方法调用`register0`来进行真正的注册，这里需要介绍一下这个判断到底怎么进行的，Netty中有大量使用到这个方法来进行判断是否进行异步的提交

```java
//io.netty.util.concurrent.AbstractEventExecutor#inEventLoop
public boolean inEventLoop() {
    return inEventLoop(Thread.currentThread());
}
//io.netty.util.concurrent.SingleThreadEventExecutor#inEventLoop
public boolean inEventLoop(Thread thread) {
    return thread == this.thread;
}
```

其实目的就是判断是否是和eventLoop在同一个线程中，如果是则同步执行，否则异步的进行任务提交执行（知道被指定的线程执行），接下来我们继续看看`register0`方法的逻辑。

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        //@see io.netty.channel.nio.AbstractNioChannel#doRegister
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        //触发pipeline的fireChannelRegistered方法
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (isActive()) {//判断是否可用
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {//判断是否自动可读的
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

可以看到上面我们离真相越来越近了，这里归纳总结一下

1.通过条件判断channel是否被取消了，是否能够被打开

2.然后调用`doRegister`方法进行注册。

3.然后调用pipeline相应周期事件来进行回调。

这里我们重点看看doRegister方法如何进行注册的，因为我们还没看到Nio相关真正绑定的逻辑哈

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            //将channel注册到selector中
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
```

上面刚刚好久满足了我们的条件，调用Nio中的channel注册到selector中的方法，但是需要注意的是ops=0并不是监听Accept事件。所以肯定很纳闷这怎么玩，原来netty其实会在后面做修改的，具体我们后面再说

#### 小结

上面的一大段终于弄明白了channel的实例化，初始化以及注册过程，并知道了Netty是如何一步步的将这些东西桥接到Nio的实现上的，Netty主要就是做了一圈的包装和各种条件的判断





### doBind0方法

上面我们已经介绍了channel的注册过程，那么现在我们只差进行地址的绑定就完事了，就可以开启监听了。

```java
private static void doBind0(
    final ChannelFuture regFuture, final Channel channel,
    final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

其实可以看懂关键的调用就是调用channel.bind方法进行真正的绑定。

```java
//io.netty.channel.AbstractChannel#bind(java.net.SocketAddress, io.netty.channel.ChannelPromise)
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
//io.netty.channel.DefaultChannelPipeline#bind(java.net.SocketAddress, io.netty.channel.ChannelPromise)
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
...
//io.netty.channel.AbstractChannel.AbstractUnsafe#bind  
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    boolean wasActive = isActive();
    try {
        doBind(localAddress);//绑定
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();//触发pipelinechannelActive事件
            }
        });
    }
	safeSetSuccess(promise);
}
```

上面关于channel的整个调用调用链并没有完整的介绍，这里会根据DefaultChannelPipeline双向链表的关系层层的进行调用，最终HeadContext会触发unsafe的bind方法。

在unsafe的bind方法会和我们之前注册channel的方法相似，主要以下几个步骤

1.进行条件判断各种条件是否满足如channel是否取消，channel是否处于打开状态等

2.调用真正的`doBind`方法进行绑定

3.触发pipeline对应事件的回调方法。

我们还是看`doBind`方法具体是如何实现的

```java
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {//判断java版本
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

最后可以看到终于来到了Nio里的channel进行绑定的实现。JDK1.7之后开启了新的bind API接口，不用再获取socket后再绑定。



这里我们还要讲下`fireChannelActive`方法，因为前面我们还留有一点疑问就是OPS注册的时候为0，答案就在`fireChannelActive`方法。至于双向链表的调用链关系这里不做过多的说明，这里主要展示核心链路

```java
//io.netty.channel.DefaultChannelPipeline.HeadContext#channelActive
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();

    readIfIsAutoRead();
}
private void readIfIsAutoRead() {
    if (channel.config().isAutoRead()) {
        channel.read();
    }
}
...
    public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {//满足
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

简单的来说就是pipeline的回调方法会触发headContext对应的回调方法，而在HeadContext中会判断channel的config中是否默认为autoRead模式，如果是则进行read方法调用。一轮调用之后回调到HeadContext的read方法，而他又调用了unsafe的beginRead方法。是不是和我们之前的实现有点类似

最后在doBeginRead方法进行正式的操作interestOps = 0，但是我们的readInterestOp = 16，这是正构建NioServerSocketChannel的时候指定的。最后重新设置interestOps为16也就是Accept事件。



## 总结

至此，我们已经完全的分析了NettyServer端启动整体的逻辑，主要包含的内容如下

1.ServerBootStrap是如何进行链式参数设置的并且是如何绑定到Nio对应的channel的。

2.了解bind方法整体的启动过程（包括EventLoopGroup内部executor，EventLoop任务，Pipeline，channel…）

3.channel的实例化，初始化过程（尤其重要且复杂）

4.了解channel是如何进行绑定的，相对简单绑定到Nio对应channel制定的端口就可以了。



## 结语

还有很多细节还没有去整理，待续



与君共勉

