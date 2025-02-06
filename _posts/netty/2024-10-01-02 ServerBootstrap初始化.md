---
title: 2. 启动服务前ServerBootstrap类型如何初始化
date: 2024-10-02 12:00:00 +0800
categories: [从零开始的Netty源码阅读]
tags: [blog]
---
# 启动服务前ServerBootstrap类型如何初始化

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
```

```java
Bootstrap bootstrap = new Bootstrap();
```

## 了解Bootstrap的建模

以下为简单的继承关系图，服务端与客户端的配置类均继承于AbstractBootstrap。

![BootStrap](/assets/netty/BootStrap.png)

图2.1 Bootstrap类继承关系

| 类型              | 说明                                                  |
| ----------------- | ----------------------------------------------------- |
| AbstractBootstrap | **抽象配置类**，用于简化Netty中`Channel`的启动过程。  |
| ServerBootstrap   | **服务端配置类**，用于简化服务端`Channel`的启动流程。 |
| Bootstrap         | **客户端配置类**，用于简化客户端`Channel`的启动流程。 |

## ServerBootstrap构造器的初始化调用链

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
```

根据Java基础的构造器知识,在每个构造器的第一行都会有个super()方法来调用父类的构造器，当前这个super方法我们可以省略，但是Java编译器底层还是会为我们默认加上这么一行super()代码来调用父类构造器。
## 父类AbstractBootstrap构造器的初始化
```java
AbstractBootstrap() {
    // Disallow extending from a different package.
}
```
父类构造器这里并没有执行逻辑。随后开始类变量的初始化。

在ServerBootstrap和Bootstrap均存在类变量config，用于暴露bootstrap的配置。

```java
// ServerBootstrap
private final ServerBootstrapConfig ServerBootstrapConfig config = new ServerBootstrapConfig(this);
// Bootstrap
private final BootstrapConfig config = new BootstrapConfig(this);
```

ServerBootstrapConfig和BootstrapConfig的构造器均会调用父类构造器AbstractBootstrapConfig(B bootstrap)，简单地判断非空之后赋值给bootstrap。

```java
protected AbstractBootstrapConfig(B bootstrap) {
    this.bootstrap = ObjectUtil.checkNotNull(bootstrap, "bootstrap");
}
```

此处先跳过NioEventLoopGroup的初始化，需要明白的是这里创建了两个线程组。

## Bootstrap的参数设置

### EventLoopGroup

```java
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
    return this;
}
```

ServerBootStrap这里会调用父类的group(EventLoopGroup group)方法。

```java
public B group(EventLoopGroup group) {
    ObjectUtil.checkNotNull(group, "group");
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return self();
}

@SuppressWarnings("unchecked")
private B self() {
    return (B) this;
}
```

父类简单地检查非空后将`parentGroup`赋值给`group`，`childGroup`赋值给`ServerBootstrap`的`childGroup`。而客户端`Bootstrap.group`方法直接调用父类的`group`方法。最后返回this不断进行build添加配置。

### Channel

服务端与客户端的channel方法均来源于AbstractBootstrap

```java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
        ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}

@Deprecated
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    ObjectUtil.checkNotNull(channelFactory, "channelFactory");
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return self();
}

@SuppressWarnings({ "unchecked", "deprecation" })
public B channelFactory(io.netty.channel.ChannelFactory<? extends C> channelFactory) {
    return channelFactory((ChannelFactory<C>) channelFactory);
}
```

该方法的主要逻辑是使用反射创建`Channel`的实例。

首先检查传入的`channelClass`是否为空，通过`ObjectUtil.checkNotNull`进行非空验证。接着，它使用`ReflectiveChannelFactory`将`channelClass`包装为工厂类，并将其传递给`channelFactory()`方法。

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Constructor<? extends T> constructor;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }

    @Override
    public T newChannel() {
        try {
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(ReflectiveChannelFactory.class) +
                '(' + StringUtil.simpleClassName(constructor.getDeclaringClass()) + ".class)";
    }
}
```

### Option

在设置完`Channel`后服务端调用了`option`和`childOption`两个方法来分别设置`bossGroup`和`workerGroup`的参数。

### Handler

同样简单地初始化了`Handler`属性。

## Bind

绑定到指定端口。

```java
ChannelFuture channelFuture = serverBootstrap.bind(10009).sync();
```

往下走看AbstractBootstrap.bind的逻辑。

```java
public ChannelFuture bind(int inetPort) {
    // 创建一个绑定到指定端口的SocketAddress，并调用bind(SocketAddress)方法
    return bind(new InetSocketAddress(inetPort));
}
```

创建Socket对象

```java
public ChannelFuture bind(SocketAddress localAddress) {
    // 验证必要参数是否已经设置，如EventLoopGroup和ChannelFactory
    validate();
    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
}
```

validate()方法中简单地进行非空检查。两种Bootstrap具有不同的实现：

```java
// ServerBootstrap.java
@Override
public ServerBootstrap validate() {
    super.validate();
    if (childHandler == null) {
        throw new IllegalStateException("childHandler not set");
    }
    if (childGroup == null) {
        logger.warn("childGroup is not set. Using parentGroup instead.");
        childGroup = config.group();
    }
    return this;
}
```

```java
// Bootstrap.java
@Override
public Bootstrap validate() {
    super.validate();
    if (config.handler() == null) {
        throw new IllegalStateException("handler not set");
    }
    return this;
}
```

在doBind方法中执行绑定前的工作。

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    // 初始化并注册Channel，返回一个注册的ChannelFuture
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    // 如果注册时发生错误，直接返回错误的结果
    if (regFuture.cause() != null) {
        return regFuture;
    }

    // 如果注册已经完成且成功，创建一个ChannelPromise
    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        // 进行实际的绑定操作
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        // 如果注册还未完成，创建PendingRegistrationPromise并添加监听器
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                // 如果注册失败，将错误设置到Promise中
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    // 注册成功后，执行绑定操作
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

完成创建等操作后在doBind0中交给eventloop执行实际的绑定操作。

```java
private static void doBind0(
    final ChannelFuture regFuture, final Channel channel,
    final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    // 将实际的绑定操作放入到EventLoop的执行队列中，避免阻塞主线程
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

至此就完成了端口绑定，接下来深入理解其中调用的方法。

### initAndRegister

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 创建Channel
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
		// 省略报错
    }
    
    // 注册一个新的Channel到EventLoopGroup中，并返回一个ChannelFuture对象
    ChannelFuture regFuture = config().group().register(channel);
	// 省略报错

    return regFuture;
}
```

此处的channelFactory来源于BootStrap所设置的ReflectiveChannelFactory。

```java
@Override
public T newChannel() {
    try { 
        return clazz.newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + clazz, t);
    }
}
```

通过反射来创建NioServerSocketChannel对象。

```java
public NioServerSocketChannel() {
    this(DEFAULT_SELECTOR_PROVIDER);
}
```

该无参构造器调用了另一个有参构造器，先来看看其中的常量DEFAULT_SELECTOR_PROVIDER。

```java
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
```

此处初始化了一个SelectorProvider对象，[定义了创建 Selector、ServerSocketChannel、SocketChannel 等方法，并在不同平台下提供不同的实现](https://www.cnblogs.com/binarylei/p/11147083.html)。

```java
public NioServerSocketChannel(SelectorProvider provider) {
    this(provider, null);
}

public NioServerSocketChannel(SelectorProvider provider, InternetProtocolFamily family) {
    this(newChannel(provider, family));
}
```

继续看newChannel方法

```java
private static ServerSocketChannel newChannel(SelectorProvider provider, InternetProtocolFamily family) {
    try {
        ServerSocketChannel channel =
            SelectorProviderUtil.newChannel(OPEN_SERVER_SOCKET_CHANNEL_WITH_FAMILY, provider, family);
        return channel == null ? provider.openServerSocketChannel() : channel;
    } catch (IOException e) {
        throw new ChannelException("Failed to open a socket.", e);
    }
}
```

使用SelectorProviderUtil来创建Channel，失败的话使用provider创建ServerSocketChannel。

// TODO：SelectorProviderUtil.newChannel

最后进入到

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    // 调用一堆super来得到channel，实际就是参数中的channel
    // 使用channel初始化config
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

```java
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent, ch, readInterestOp);
}
```

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    // 保存channel
    this.ch = ch;
    // 对应的事件
    this.readInterestOp = readInterestOp;
    try {
        // 设置为非阻塞
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            logger.warn(
                "Failed to close a partially initialized socket.", e2);
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

`this.ch = ch;`绑定jdk底层的ServerSocketChannel, 至此，jdk的channel和netty定义的channel是组合关系，netty的channel中有个jdk的channel的成员变量，而**这个成员变量就定义在AbstractNioChannel这个类当中**。


```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    // 创建唯一ID
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```

在顶层父类中初始化了两个属性unsafe，和pipeline，目前只需要知道这两个属性是在这里初始化。

## init

接着看initAndRegister方法中的调用的init方法。服务端与客户端均重写了该方法，先从服务端开始。

```java
void init(Channel channel) {
    // 获取的用户定义的选项和属性
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, newAttributesArray());
    // 获取channel的pipeline，这是一个处理入站和出站数据的处理器链
    ChannelPipeline p = channel.pipeline();

    // 获取用于配置的变量
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);
    final Collection<ChannelInitializerExtension> extensions = getInitializerExtensions();

    p.addLast(new ChannelInitializer<Channel>() {
        // 初始化channel
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
            // 添加 ServerBootstrapAcceptor
            // ServerBootstrapAcceptor负责处理已接受的通道，设置它们的选项、属性，并将它们注册到子事件循环组。
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs,
                        extensions));
                }
            });
        }
    });
    if (!extensions.isEmpty() && channel instanceof ServerChannel) {
        ServerChannel serverChannel = (ServerChannel) channel;
        for (ChannelInitializerExtension extension : extensions) {
            try {
                extension.postInitializeServerListenerChannel(serverChannel);
            } catch (Exception e) {
                logger.warn("Exception thrown from postInitializeServerListenerChannel", e);
            }
        }
    }
}
```

### 注册channel

接着看initAndRegister方法中的调用的`config().group().register(channel)`方法。服务端与客户端均重写了该方法，先从服务端开始。

```java
@Override
public final ServerBootstrapConfig config() {
    return config;
}
```

这段代码意思是返回在之前提到过的对应于每个BootstrapConfig。

再进到AbstractBootstrapConfig#group方法：

```java
public final EventLoopGroup group() {
    return bootstrap.group();
}
```

调用了AbstractBootstrap#group()，也就是返回了ServerBootstrap的bossGroup。由于创建EventLoopGroup是使用NioEventLoopGroup，最后的注册也是父类MultithreadEventLoopGroup的register方法。

最后进到MultithreadEventLoopGroup#register(io.netty.channel.Channel)方法：

```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

此处的next()的作用是从EventLoopGroup中选出一个EventLoop实例，也就是从线程组中选出一个线程。

继续进入到父类SingleThreadEventLoop#register(io.netty.channel.Channel)方法：

```java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```

暂时先跳过DefaultChannelPromise的部分，继续看SingleThreadEventLoop#register(io.netty.channel.ChannelPromise)：

```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

这行代码调用 `promise` 对象的 `channel` 方法获取关联的 `Channel` 对象，然后通过 `unsafe` 方法获取 `Channel` 的内部 `Unsafe` 实现，最后调用 `register` 方法将当前的 `SingleThreadEventLoop` 实例和 `promise` 注册到 `Channel` 中。此处的unsafe就是AbstractChannel#AbstractChannel(io.netty.channel.Channel)中所创建的unsafe。

// TODO：unsafe创建流程。

继续深入AbstractChannel.AbstractUnsafe#register(EventLoop eventLoop, final ChannelPromise promise)：

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // 参数检查
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
            new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    // 明确地指定了要设置的是 AbstractChannel 实例的 eventLoop 属性，而不是 AbstractUnsafe 实例的属性。
    AbstractChannel.this.eventLoop = eventLoop;

    // 检查当前线程是否在 eventLoop 中运行，如果是，则直接调用 register0 方法进行注册：
    // 如果当前线程不在 eventLoop 中运行，则通过 eventLoop.execute 提交一个任务到 eventLoop 中执行 register0 方法：
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            // 线程启动
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

注意：

在 AbstractChannel 类中，register 方法是 AbstractUnsafe 的一个方法，而 AbstractUnsafe 是 AbstractChannel 的一个内部类。因此，this 关键字在 AbstractUnsafe 中指的是 AbstractUnsafe 的实例，而不是 AbstractChannel 的实例。

为了明确地引用外部类 AbstractChannel 的实例，使用 AbstractChannel.this。这确保了 eventLoop 属性被正确地设置为 AbstractChannel 实例的 eventLoop 属性，而不是 AbstractUnsafe 实例的属性。

继续进到AbstractChannel.AbstractUnsafe#register0(ChannelPromise promise)：

```java
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        // 参数检查
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        // 做实际的注册
        doRegister();
        // 更新状态
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        // 调用 pipeline.invokeHandlerAddedIfNeeded 确保在通知 promise 之前调用 handlerAdded 方法
        // 这是为了防止用户在 ChannelFutureListener 中触发事件时出现问题
        pipeline.invokeHandlerAddedIfNeeded();

        // 将 promise 设置为成功状态，并触发 pipeline.fireChannelRegistered 事件，通知管道已经注册：
        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        // 如果 Channel 是活跃的，并且这是第一次注册，则触发 pipeline.fireChannelActive 事件。
        // 如果 Channel 之前已经注册过，并且 config().isAutoRead() 返回 true，则调用 beginRead 方法开始读取数据：
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
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

重点关注doRegister方法。进入到io.netty.channel.nio.AbstractNioChannel#doRegister()：

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
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

方法中使用无限循环尝试组测Channel。首先看javaChannel方法。

```java
// io.netty.channel.socket.nio.NioServerSocketChannel#javaChannel
protected ServerSocketChannel javaChannel() {
    return (ServerSocketChannel) super.javaChannel();
}
// io.netty.channel.nio.AbstractNioChannel#javaChannel
protected SelectableChannel javaChannel() {
    return ch;
}
```

实际上会返回的是AbstractNioChannel.ch，也就是jdk的ServerSocketChannel。eventLoop().unwrappedSelector()，是获得每一个eventLoop绑定的唯一的selector，0代表这次只是注册, 并不监听任何事件，this是代表将自身(NioEventLoopChannel)作为属性绑定在返回的selectionKey当中, 这个selectionKey就是与每个channel绑定的jdk底层的SelectionKey对象。

接着看jdk底层的java.nio.channels.spi.AbstractSelectableChannel#register：

```java
public final SelectionKey register(Selector sel, int ops,
                                   Object att)
    throws ClosedChannelException
{
    synchronized (regLock) {
        if (!isOpen())
            throw new ClosedChannelException();
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (blocking)
            throw new IllegalBlockingModeException();
        // 方法通过调用 findKey 方法检查当前通道是否已经注册到指定的 Selector 中
        SelectionKey k = findKey(sel);
        // 如果已经注册，则更新其兴趣操作集（interestOps）并附加新的附件（att）
        if (k != null) {
            k.interestOps(ops);
            k.attach(att);
        }
        // 如果当前通道尚未注册到指定的 Selector 中，通过调用 AbstractSelector 的 register 方法将通道注册到 Selector 中，并将返回的 SelectionKey 添加到当前通道的键集合中
        if (k == null) {
            // New registration
            synchronized (keyLock) {
                if (!isOpen())
                    throw new ClosedChannelException();
                k = ((AbstractSelector)sel).register(this, ops, att);
                addKey(k);
            }
        }
        return k;
    }
}
```

### doBind0

在运行完initAndRegister()方法后，完成注册Channel。接着深入doBind0(regFuture, channel, localAddress, promise)方法。

```java
private static void doBind0(
    final ChannelFuture regFuture, final Channel channel,
    final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    // 将实际的绑定操作放入到EventLoop的执行队列中，避免阻塞主线程
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                // 绑定端口
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```

实际是使用channel来进行绑定。

```java
// io.netty.channel.AbstractChannel#bind(java.net.SocketAddress, io.netty.channel.ChannelPromise)
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
// io.netty.channel.DefaultChannelPipeline#bind(java.net.SocketAddress, io.netty.channel.ChannelPromise)
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
// io.netty.channel.AbstractChannelHandlerContext#bind(java.net.SocketAddress, io.netty.channel.ChannelPromise)
public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(localAddress, "localAddress");
    if (isNotValidPromise(promise, false)) {
        // cancelled
        return promise;
    }

    // 查找下一个出站处理器上下文，该上下文具有 MASK_BIND 掩码
    final AbstractChannelHandlerContext next = findContextOutbound(MASK_BIND);
    // 获取该上下文的执行器（EventExecutor），并检查当前线程是否在执行器的事件循环中。
    // 如果是，则直接调用 next.invokeBind 方法执行绑定操作
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeBind(localAddress, promise);
        // 如果当前线程不在执行器的事件循环中，则通过 safeExecute 方法将绑定操作提交给执行器执行
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                next.invokeBind(localAddress, promise);
            }
        }, promise, null, false);
    }
    return promise;
}
```

invokeBind的调用链在此省略，会进入到AbstractChannel.AbstractUnsafe#bind(final SocketAddress localAddress, final ChannelPromise promise)：

```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();

    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    // See: https://github.com/netty/netty/issues/576
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        // Warn a user about the fact that a non-root user can't receive a
        // broadcast packet on *nix if the socket is bound on non-wildcard address.
        logger.warn(
            "A non-root user can't receive a broadcast packet if the socket " +
            "is not bound to a wildcard address; binding to a non-wildcard " +
            "address (" + localAddress + ") anyway as requested.");
    }

    boolean wasActive = isActive();
    try {
        // 执行实际的绑定操作
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }

    // 如果 Channel 在绑定后变为活跃状态，并且之前不是活跃的，则通过 invokeLater 方法触发 pipeline.fireChannelActive 事件，通知管道 Channel 已经活跃
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }

    safeSetSuccess(promise);
}
```

最后进入NioServerSocketChannel#doBind(SocketAddress localAddress)：

```java
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

此处javaChannel()返回的是jdk的ServerSocketChannel，所调用的bind方法也就是jdk底层的端口绑定逻辑。

最后回到.NettyServer#main

```java
ChannelFuture channelFuture = serverBootstrap.bind(10009).sync();
```

最后调用了sync方法等待ChannelFuture返回结果。