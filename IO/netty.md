

## 一 Netty线程模型和模块



### 1. Reactor线程模型

Reactor模式是基于事件驱动开发的，核心组成部分包括Reactor和线程池，其中Reactor负责监听和分配事件，线程池负责处理事件，而根据Reactor的数量和线程池的数量，又将Reactor分为三种模型：单线程模型、多线程模型、主从多线程模型。



> #### 单线程模型

Reactor内部通过**selector** 监控连接事件，收到事件后通过**dispatch**进行分发，如果是连接建立的事件，则由Acceptor处理，**Acceptor**通过accept接受连接，并创建一个Handler来处理连接后续的各种事件,如果是读写事件，直接调用连接对应的**Handler**来处理。

![](..\images\IO\reactor-1.png)





> #### 多线程模型

主线程中，Reactor对象通过selector监控连接事件,收到事件后通过dispatch进行分发，如果是连接建立事件，则由Acceptor处理，Acceptor通过accept接收连接，并创建一个Handler来处理后续事件，而Handler只负责响应事件，不进行业务操作，也就是只进行read读取数据和write写出数据，业务处理交给一个线程池进行处理。

![](..\images\IO\reactor-2.png)





> #### 主从多线程模型

存在多个Reactor，**每个Reactor都有自己的selector选择器**，线程和dispatch。主线程中的 mainReactor 通过自己的 selector 监控连接建立事件，收到事件后通过 Accpetor 接收，将新的连接分配给某个子线程；子线程中的 subReactor 将 mainReactor 分配的连接加入连接队列中通过自己的 selector 进行监听，并创建一个 Handler 用于处理后续事件。

![](..\images\IO\reactor-3.png)



### 2. Netty的线程模型

![](..\images\IO\netty.png) 



**模型解释：**

- Netty中定义了两个线程组 BossGroup 和 WorkGroup，其中 BossGroup 负责处理客户端的连接请求，而 WorkGroup负责处理I/O事件。
- NioEventGroup 相当于是事件循环线程组，含有多个事件循环线程（NioEventLoop）。
- 每个 NioEventLoop 都有一个 selector，用于监听注册在 selector 上 socketChannel 的事件。
- 每个Boss NioEventLoop线程内部循环执行的步骤有 3 步：1）处理accept事件 , 与client 建立连接 , 生成 NioSocketChannel；2）将NioSocketChannel注册到某个worker NIOEventLoop上的selector；3）处理任务队列的任务 ， 即runAllTasks。
- 每个worker NIOEventLoop线程循环执行的步骤：1）轮询注册到自己selector上的所有NioSocketChannel 的read, write事件；2）处理 I/O 事件， 即read , write 事件， 在对应NioSocketChannel 处理业务；3）runAllTasks处理任务队列TaskQueue的任务 ，一些耗时的业务处理一般可以放入 TaskQueue中慢慢处理，这样不影响数据在 pipeline 中的流动处理。
- 每个worker NIOEventLoop处理NioSocketChannel业务时，会使用 pipeline (管道)，管道中维护 了很多 handler 处理器用来处理 channel 中的数据。



### 3. Netty模块组件

- **Bootstrap、ServerBootStrap**：客户端和服务端启动引导类，用于配置整个 Netty 程序，串联各个组件。
- **Future、ChannelFuture**：在 Netty 中所有的 IO 操作都是异步的，无法得知处理结果，可以通过 Future 或 ChannelFuture 注册监听，在处理结束后回调监听。
- **Channel**：与 NIO 类似。
- **Selector**：与 NIO 类似。
- **NioEventLoop**：维护了一个线程和任务队列，支持异步提交任务。
- **NioEventLoopGroup**：管理 NioEventLoop 的生命周期，维护了多个 NioEventLoop。
- **ChannelHandler**：处理 IO 事件或拦截 IO 操作，并转发到 ChannelPipeline 中的下一个处理程序。
- **ChannelPipline**：保存 ChannelHandler 的 list，用于处理或拦截 Channel 的出站和入站操作。

![](..\images\IO\channel.png)



> #### 出站和入站

以客户端为例，事件的方向从客户端到服务端（例如发送消息），那么该事件被称为出站，客户端发送的数据会通过管道中一系列 ChannelOutboundHandler（调用方向是从**tail 到 head**）,反之称为入站，入栈会经过 ChannelInboundHandler，方向为从 head 到 tail。



> #### ByteBuf

ByteBuf 由字节数组组成，在ByteBuf 中提供了两个索引，一个用于读取数据，一个用于写入数据，这两个索引通过在字节数组中移动来定位需要读或写的数据位置。

![](..\images\IO\bytebuf.png)



### 4. 粘包和拆包

TCP粘包拆包是指发送方发送的若干包数据到接收方接收时粘成一包或某个数据包被拆开接收。如下图所示，client发了两个数据包D1和 D2，但是server端可能会收到如下几种情况的数据。

![](..\images\IO\nc.png) 

**出现粘包和拆包的原因**

TCP 是面向连接的， 面向流的， 提供高可靠性服务。 收发两端（客户端和服务器端） 都要有成对的 socket，因此， 发送端为了将多个发给接收端的包， 更有效的发给对方，使用了优化方法（Nagle 算法），将多次间隔较小且数据量小的数据， 合并成一个大的数据块， 然后进行封包。 这样做虽然提高了效率， 但是接收端就难于分辨出完整的数据包了， 因为面向流的通信是无消息保护边界的。



**解决方案**

1. 设置固定的字符进行分割，这种方法需要保证消息内没有分隔符；
2. 发送数据时带上数据长度。



### 5. Netty的心跳机制

所谓心跳机制，就是在客户端和服务端之间定期发送特殊的数据包，通知对方自己还在线。

在 Netty 中，通过`IdleStateHandler`来实现心跳机制。

```java
/**
 * readerIdleTime：读超时；指定时间内没有从Channel读取到数据，会触发读超时时间
 * writerIdleTime：写超时
 * allIdleTime：读写超时
 * TimeUnit：时间单位
 */
public IdleStateHandler(
    long readerIdleTime, long writerIdleTime, long allIdleTime,
    TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}
```

以读取数据为例：

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
        reading = true;
        firstReaderIdleEvent = firstAllIdleEvent = true;
    }
    // 调用下一个hanndler处理channelRead事件
    ctx.fireChannelRead(msg);
}
```

`IdleStateHandler`机制的核心实现是在建立连接后，进行初始化实现的。

```java
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    // 初始化
    initialize(ctx);
    super.channelActive(ctx);
}

private void initialize(ChannelHandlerContext ctx) {
    // Avoid the case where destroy() is called before scheduling timeouts.
    // See: https://github.com/netty/netty/issues/143
    switch (state) {
        case 1:
        case 2:
            return;
    }

    state = 1;
    initOutputChanged(ctx);
    // 根据设置的读写超时事件，进行响应的处理
    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                                     readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                                     writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                                  allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}

// 开启一个线程处理超时事件
ScheduledFuture<?> schedule(ChannelHandlerContext ctx, Runnable task, long delay, TimeUnit unit) {
    return ctx.executor().schedule(task, delay, unit);
}

protected void run(ChannelHandlerContext ctx) {
    // 获取设置的超时时间
    long nextDelay = readerIdleTimeNanos;
    if (!reading) {
        // 计算剩余时间
        nextDelay -= ticksInNanos() - lastReadTime;
    }
	
    if (nextDelay <= 0) {
        // 没有出现超时，循环调用
        readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

        boolean first = firstReaderIdleEvent;
        firstReaderIdleEvent = false;

        try {
            IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
            // 调用handler的fireUserEventTriggered方法
            channelIdle(ctx, event);
        } catch (Throwable t) {
            ctx.fireExceptionCaught(t);
        }
    } else {
        // 循环调用
        readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}
```



## 二 源码

Netty中，客户端和服务端交互的大致流程如下：

![](..\images\IO\netty-1.png)



**Netty客户端与服务端核心流程：**

![](..\images\IO\netty-sc.png)







> #### NioEventLoopGroup UML图

![](..\images\IO\NioEventLoopGroup.png) 

`NioEventLoopGroup`实现了`Executor`接口，其实可以看做是一个线程池，最终会调用父类的构造方法。

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (executor == null) {
        // 初始化了一个ThreadPerTaskExecutor
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    // children是一个EventExecutor[]
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
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
}

// io.netty.channel.nio.NioEventLoopGroup#newChild
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    // 新建一个NioEventLoop，最后放入到NioEventLoop[]
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}

// io.netty.channel.nio.NioEventLoop#NioEventLoop
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
             EventLoopTaskQueueFactory queueFactory) {
    // 调用父类方法
    super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
          rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    // 根据SelectorProvider创建selector
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}

// io.netty.util.concurrent.SingleThreadEventExecutor#SingleThreadEventExecutor()
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                    RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    this.executor = ThreadExecutorMap.apply(executor, this);
    // 创建taskQueue，该队列会保存需要执行的任务
    this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
    this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```



###  1. 初始化流程

客户端和服务端Channel初始化和注册到selector上流程大致相同。

####  1.1 ServerBootstrap

`ServerBootstrap`是服务端的启动引导类，由它来配置 Netty，并串联各个组件。

```java
/**
 * 绑定线程组	
 */
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    // 把parentGroup赋值给父类的EventLoopGroup
    super.group(parentGroup);
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    // childGroup赋值
    this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
    return this;
}

/**
 * 设置channel类型
 */
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
        ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}

public ReflectiveChannelFactory(Class<? extends T> clazz) {
    ObjectUtil.checkNotNull(clazz, "clazz");
    try {
        this.constructor = clazz.getConstructor();
    } catch (NoSuchMethodException e) {
        throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                                           " does not have a public non-arg constructor", e);
    }
}

public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    ObjectUtil.checkNotNull(channelFactory, "channelFactory");
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }
    // 此时ChannelFactory只拿到了构造函数，还未获取到实例化的对象
    this.channelFactory = channelFactory;
    return self();
}

/**
 * 给childHandler赋值
 */
public ServerBootstrap childHandler(ChannelHandler childHandler) {
    this.childHandler = ObjectUtil.checkNotNull(childHandler, "childHandler");
    return this;
}
```

服务端启动是在`ServerBootstrap`的`bind()`方法中处理的。

```java
// io.netty.bootstrap.AbstractBootstrap#bind(int)
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}

// io.netty.bootstrap.AbstractBootstrap#doBind
private ChannelFuture doBind(final SocketAddress localAddress) {
	// 初始化channel，并注册到selector
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



#### 1.2 初始化通道

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 通过反射创建channel
        channel = channelFactory.newChannel();
        // 初始化channel
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

    // 将channel注册到selector
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



> #### channelFactory.newChannel()

通过反射调用到`NioSocketChannel`的构造函数

```java
public NioSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}

private static SocketChannel newSocket(SelectorProvider provider) {
    try {
        return provider.openSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a socket.", e);
    }
}

public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}

protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        // 省略
    }
}

protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}

protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}

// 创建pipeline中的链表
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```



> ####  init(channel)：初始化通道

```java
// io.netty.bootstrap.ServerBootstrap#init
void init(Channel channel) {
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

    // 在pipeline中添加了一个handler
    p.addLast(new ChannelInitializer<Channel>() {
        // 
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
			
            // 开启线程
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```



#### 1.3 将Channel注册到seletor上

```java
/**
 * config().group()：获取主线程组
 */
// io.netty.channel.MultithreadEventLoopGroup#register(io.netty.channel.Channel)
public ChannelFuture register(Channel channel) {
    // next()：从EventExecutor[]中选择一个EventExecutor，[idx.getAndIncrement() & executors.length - 1]
    return next().register(channel);
}

// io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

// io.netty.channel.AbstractChannel.AbstractUnsafe#register
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // 一些校验判断
    AbstractChannel.this.eventLoop = eventLoop;

    // 可以看到了两个分支最终都执行了register0()方法
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            // 最终会走到这个分支，开启一个线程来执行register0()
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
           
    }
}

private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        // doRegister()：将channel注册到selector
        doRegister();
        neverRegistered = false;
        registered = true;

        // 执行pipeline上的handler，此时在初始化的时候添加的handler会被执行，为其添加ServerBootstrapAcceptor
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
               
                beginRead();
            }
        }
    } catch (Throwable t) {
        
    }
}
```



> #### doRegister()：将channel注册到selector

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // javaChannel()拿到的是ServerSocketChannel
            // eventLoop().unwrappedSelector()获取到的是selector
            // 此处的逻辑和NIO一样，就是将ServerSocketChannel注册到selector上
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
           
        }
    }
}
```



> #### pipeline.invokeHandlerAddedIfNeeded()：执行handler

```java
// io.netty.channel.DefaultChannelPipeline#invokeHandlerAddedIfNeeded
final void invokeHandlerAddedIfNeeded() {
    assert channel.eventLoop().inEventLoop();
    if (firstRegistration) {
        firstRegistration = false;
        // 回调handler
        callHandlerAddedForAllHandlers();
    }
}
private void callHandlerAddedForAllHandlers() {
    final PendingHandlerCallback pendingHandlerCallbackHead;
    synchronized (this) {
        assert !registered;
        registered = true;
        // 在初始化init()方法中的addLast()中为pendingHandlerCallbackHead赋值
        pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
        // Null out so it can be GC'ed.
        this.pendingHandlerCallbackHead = null;
    }

    PendingHandlerCallback task = pendingHandlerCallbackHead;
    // 在这个循环中，初始化时加入pipeline中的handler会被执行
    while (task != null) {
        task.execute();
        task = task.next;
    }
}
```



> #### execute(Runnable task)

```java
// io.netty.bootstrap.ServerBootstrap#init
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                    ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});

// io.netty.util.concurrent.SingleThreadEventExecutor#execute(java.lang.Runnable)
public void execute(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}

private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    // 将task加入taskQueue中
    addTask(task);
    if (!inEventLoop) {
        // task与channel执行任务的不是同一个，开始执行	
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp mmediate) {
        wakeup(inEventLoop);
    }
}
```



> #### 开启线程死循环等待事件发生

```java
// io.netty.util.concurrent.SingleThreadEventExecutor#startThread
private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}

private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                // 执行
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                // 省略
            }
        } 
    });
}

// io.netty.channel.nio.NioEventLoop#run
protected void run() {
    int selectCnt = 0;
    // 死循环
    for (;;) {
        try {
            int strategy;
            try {
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                // 第一次进入是不会进入该分支
                switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:

                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                // 调用select()阻塞等待事件发生
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                    default:
                }
            } catch (IOException e) {
                
            }

            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // Ensure we always run tasks.
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else 
                // 第一次会走入这个分支，执行taskQueue中的task
                // 在这个方法会去执行
                ranTasks = runAllTasks(0); // This will run the minimum number of tasks
            }

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                 selectCnt - 1, selector);
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
           
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```



> #### 循环遍历selector上的keys：processSelectedKeys（）

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}

private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];

        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);

            selectAgain();
            i = -1;
        }
    }
}


private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            
            return;
        }
        
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    // 判断发生的事件，并进行相应的处理
    try {
        int readyOps = k.readyOps();
        
        // 连接事件
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // 写事件
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            ch.unsafe().forceFlush();
        }

        // 读取、接收连接事件
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // 在这里构造SocektChannel注册到selector上并将其交给workGroup线程
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

