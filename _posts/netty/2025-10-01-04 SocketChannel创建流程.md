---
title: 4. SocketChannel创建流程
date: 2024-10-04 12:00:00 +0800
categories: [从零开始的Netty源码阅读]
tags: [blog]
---
# NioSocketChannel初始化流程

 在之前的流程中，ServerBootStrap在创建时会创建NioSocketChannel，现在开始深入具体的创建流程

```java
public NioServerSocketChannel() {
    this(DEFAULT_SELECTOR_PROVIDER);
}

public NioServerSocketChannel(SelectorProvider provider) {
    this(provider, null);
}

public NioServerSocketChannel(SelectorProvider provider, InternetProtocolFamily family) {
    // 
    this(newChannel(provider, family));
}
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

前文中并没有提及config的创建，此处做补充。

`javaChannel().socket()`是拿到了NioServerSocketChannel.ch的socket

```java
private NioServerSocketChannelConfig(NioServerSocketChannel channel, ServerSocket javaSocket) {
    super(channel, javaSocket);
}

public DefaultServerSocketChannelConfig(ServerSocketChannel channel, ServerSocket javaSocket) {
    super(channel, new ServerChannelRecvByteBufAllocator());
    this.javaSocket = ObjectUtil.checkNotNull(javaSocket, "javaSocket");
}
protected DefaultChannelConfig(Channel channel, RecvByteBufAllocator allocator) {
    setRecvByteBufAllocator(allocator, channel.metadata());
    this.channel = channel;
}
```

创建NioServerSocketChannelConfig前首先调用父类DefaultServerSocketChannelConfig的构造器，此处创建了一个ServerChannelRecvByteBufAllocator

```java
// 初始化一个接收字节缓冲区分配器，设置每次读取的最大消息数为 1，并且可能会忽略已读取的字节数。
public ServerChannelRecvByteBufAllocator() {
    super(1, true);
}
// 创建一个接收字节缓冲区分配器，允许用户自定义每次读取的最大消息数和是否忽略已读取的字节数。
DefaultMaxMessagesRecvByteBufAllocator(int maxMessagesPerRead, boolean ignoreBytesRead) {
    this.ignoreBytesRead = ignoreBytesRead;
    maxMessagesPerRead(maxMessagesPerRead);
}
// 设置每次读取操作中允许读取的最大消息数。
public MaxMessagesRecvByteBufAllocator maxMessagesPerRead(int maxMessagesPerRead) {
    checkPositive(maxMessagesPerRead, "maxMessagesPerRead");
    this.maxMessagesPerRead = maxMessagesPerRead;
    return this;
}
```

回到DefaultChannelConfig

```java
protected DefaultChannelConfig(Channel channel, RecvByteBufAllocator allocator) {
    setRecvByteBufAllocator(allocator, channel.metadata());
    this.channel = channel;
}
```

初始化了channel, 在channel初始化之前, 调用了setRecvByteBufAllocator(allocator, channel.metadata())方法, 这是用于设置缓冲区分配器的方法, 第一个参数是刚刚分析过的新建的AdaptiveRecvByteBufAllocator对象, 第二个传入的是与channel绑定的ChannelMetadata对象

```java
public ChannelMetadata metadata() {
    return METADATA;
}
// NioServerSocketChannel.java
private static final ChannelMetadata METADATA = new ChannelMetadata(false, 16);

public ChannelMetadata(boolean hasDisconnect, int defaultMaxMessagesPerRead) {
    checkPositive(defaultMaxMessagesPerRead, "defaultMaxMessagesPerRead");
    this.hasDisconnect = hasDisconnect;
    this.defaultMaxMessagesPerRead = defaultMaxMessagesPerRead;
}
```

只初始化了两个属性:

```java
hasDisconnect=false
defaultMaxMessagesPerRead=16
```

defaultMaxMessagesPerRead=16代表在读取对方的链接或者channel的字节流时(无论server还是client), 最多只循环16次

再回到DefaultChannelConfig

```java
protected DefaultChannelConfig(Channel channel, RecvByteBufAllocator allocator) {
    setRecvByteBufAllocator(allocator, channel.metadata());
    this.channel = channel;
}

private void setRecvByteBufAllocator(RecvByteBufAllocator allocator, ChannelMetadata metadata) {
    checkNotNull(allocator, "allocator");
    checkNotNull(metadata, "metadata");
    if (allocator instanceof MaxMessagesRecvByteBufAllocator) {
        ((MaxMessagesRecvByteBufAllocator) allocator).maxMessagesPerRead(metadata.defaultMaxMessagesPerRead());
    }
    // TODO: 其他类型Allocator
    setRecvByteBufAllocator(allocator);
}
```

首先会判断传入的缓冲区分配器是不是MaxMessagesRecvByteBufAllocator类型，之后将metadata中的maxMessagesPerRead赋值给allocator，最后初始化成员变量rcvBufAllocator。

至此NioSocketChannelConfig的初始化完成

# 事件处理

首先回顾NioEventLoop的processSelectedKey ()方法:

```java
// 处理选中的键
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    // 获取通道的安全操作接口
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();

    // 检查选择键是否有效
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            // 尝试获取与通道关联的事件循环
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // 如果通道实现抛出异常，表示没有事件循环，我们忽略这个异常
            // 因为我们只是想确定通道是否注册到这个事件循环，并因此有权关闭通道。
            return;
        }
        // 仅在通道仍然注册到此事件循环时关闭通道
        // 通道可能已经从事件循环中注销，因此选择键可能在注销过程中被取消，但通道仍然健康，不应关闭。
        // 参见：https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // 如果选择键不再有效，则关闭通道
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        // key 合法
        // 获取当前选择键的准备操作
        int readyOps = k.readyOps();
        // 在尝试触发 read(...) 或 write(...) 之前，我们首先需要调用 finishConnect()
        // 否则 NIO JDK 通道实现可能会抛出 NotYetConnectedException。
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // 移除 OP_CONNECT，否则 Selector.select(..) 将始终返回而不阻塞
            // 参见：https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT; // 清除 OP_CONNECT
            k.interestOps(ops); // 更新选择键的兴趣操作

            // 完成连接操作
            unsafe.finishConnect();
        }

        // 首先处理 OP_WRITE，因为我们可能能够写入一些排队的缓冲区，从而释放内存。
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // 调用 forceFlush，这也会在没有剩余可写内容时清除 OP_WRITE
            unsafe.forceFlush();
        }

        // 还要检查 readOps 是否为 0，以避免可能导致无限循环的 JDK 错误
        // 如果当前的NioEventLoop是工作线程，那么这里处理的是op_read事件；
        // 如果是主线程，那么这里处理的是op_accept事件。
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // 触发读取操作
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        // 如果选择键被取消，关闭通道
        unsafe.close(unsafe.voidPromise());
    }
}
```

进入AbstractNioMessageChannel.NioMessageUnsafe#read方法

```java
// 读取方法，处理接收到的消息
public void read() {
    // 确保当前线程在事件循环中
    assert eventLoop().inEventLoop();
    // 获取通道配置
    final ChannelConfig config = config();
    // 获取通道管道
    final ChannelPipeline pipeline = pipeline();
    // 获取接收字节缓冲分配器的句柄，以便管理接收的字节缓冲区和消息读取过程
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    // 设置配置
    allocHandle.reset(config);

    boolean closed = false; // 标记通道是否关闭
    Throwable exception = null; // 捕获异常
    try {
        try {
            // 循环读取消息
            do {
                // 从通道读取消息到缓冲区
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break; // 如果没有读取到消息，退出循环
                }
                if (localRead < 0) {
                    closed = true; // 如果读取到负值，标记通道关闭
                    break;
                }

                // 增加读取的消息计数
                allocHandle.incMessagesRead(localRead);
            } while (continueReading(allocHandle)); // 检查是否继续读取
        } catch (Throwable t) {
            exception = t; // 捕获异常
        }

        // 获取读取缓冲区的大小
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false; // 标记读取操作已完成
            // 触发通道读取事件
            pipeline.fireChannelRead(readBuf.get(i));
        }
        readBuf.clear(); // 清空读取缓冲区
        allocHandle.readComplete(); // 标记读取完成
        pipeline.fireChannelReadComplete(); // 触发读取完成事件

        if (exception != null) {
            closed = closeOnReadError(exception); // 处理读取错误
            pipeline.fireExceptionCaught(exception); // 触发异常事件
        }

        if (closed) {
            inputShutdown = true; // 标记输入关闭
            if (isOpen()) {
                close(voidPromise()); // 关闭通道
            }
        }
    } finally {
        // 检查是否有未处理的读取请求
        // 这可能是由于两种原因：
        // * 用户在 channelRead(...) 方法中调用了 Channel.read() 或 ChannelHandlerContext.read()
        // * 用户在 channelReadComplete(...) 方法中调用了 Channel.read() 或 ChannelHandlerContext.read()
        // 参见：https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp(); // 移除读取操作
        }
    }
}
```

首先获取与NioServerSocketChannel绑定config和pipeline，先看

```java
// 获取接收字节缓冲分配器的句柄，以便管理接收的字节缓冲区和消息读取过程
final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
```

这里通过RecvByteBufAllocator接口调用了其内部接口Handler

查看接口的具体内容

```java
public interface RecvByteBufAllocator {
    Handle newHandle();
    interface Handle {
        int guess();
        void reset(ChannelConfig config);
        void incMessagesRead(int numMessages);
        void lastBytesRead(int bytes);
        int lastBytesRead();
        void attemptedBytesRead(int bytes);
        int attemptedBytesRead();
        boolean continueReading();
        void readComplete();    
    }
}
```

我们看到RecvByteBufAllocator接口只有一个方法newHandle(), 顾名思义就是用于创建Handle对象的方法, 而Handle中的方法, 才是实际用于操作的方法

在RecvByteBufAllocator实现类中包含Handle的子类:

```java
final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();

```

unsafe()返回当前channel绑定的unsafe对象, recvBufAllocHandle()最终会调用AbstractChannel内部类AbstractUnsafe的recvBufAllocHandle()方法

往下到AbstractChannel.AbstractUnsafe#recvBufAllocHandle

```java
public RecvByteBufAllocator.Handle recvBufAllocHandle() {
    if (recvHandle == null) {
        recvHandle = config().getRecvByteBufAllocator().newHandle();
    }
    return recvHandle;
}
```

如果如果是第一次执行到这里, 自身属性recvHandle为空, 会创建一个recvHandle实例, config()返回NioServerSocketChannel绑定的ChannelConfig, getRecvByteBufAllocator()获取其RecvByteBufAllocator对象, 这两部分上一小节剖析过了, 这里通过newHandle()创建一个Handle, 这里会走到AdaptiveRecvByteBufAllocator类中的newHandle()方法中

进入newHandle方法发现是ServerChannelRecvByteBufAllocator#newHandle

```java
public Handle newHandle() {
    return new MaxMessageHandle() {
        @Override
        public int guess() {
            return 128;
        }
    };
}
```

创建了一个MaxMessageHandle的匿名子类，先看看继承关系

```java
public abstract class MaxMessageHandle implements ExtendedHandle {
    // ChannelConfig用于配置通道的参数
    private ChannelConfig config;
    // 每次读取的最大消息数 16
    private int maxMessagePerRead;
    // 已读取的总消息数
    private int totalMessages;
    // 已读取的总字节数
    private int totalBytesRead;
    // 尝试读取的字节数
    private int attemptedBytesRead;
    // 最后读取的字节数
    private int lastBytesRead;
    // 是否尊重可能还有更多数据的标志
    private final boolean respectMaybeMoreData = DefaultMaxMessagesRecvByteBufAllocator.this.respectMaybeMoreData;
    private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {
        @Override
        public boolean get() {
            return attemptedBytesRead == lastBytesRead;
        }
    };

    ...
}

interface ExtendedHandle extends Handle {

    boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier);
}
```

// TODO: 参数的作用

继续回到read()方法:

```java
allocHandle.reset(config);
```

这段代码是重新设置配置, 也就是将之前的配置信息进行初始化

查看具体实现

```java
public void reset(ChannelConfig config) {
    this.config = config;
    maxMessagePerRead = maxMessagesPerRead();
    totalMessages = totalBytesRead = 0;
}
```

仅对几个属性进行赋值

config:当前channelConfig对象

maxMessagePerRead:表示读取消息的时候可以读取几次(循环次数), maxMessagesPerRead()返回的是RecvByteBufAllocator的maxMessagesPerRead属性

totalMessages:代表目前读循环已经读取的消息个数, 在NIO传输模式下也就是已经执行的循环次数, 这里初始化为0

totalBytesRead:代表目前已经读取到的消息字节总数, 这里同样也初始化为

继续看read()，首先是一个do-while循环

```java
// 循环读取消息
do {
    // 从通道读取消息到缓冲区
    int localRead = doReadMessages(readBuf);
    if (localRead == 0) {
        break; // 如果没有读取到消息，退出循环
    }
    if (localRead < 0) {
        closed = true; // 如果读取到负值，标记通道关闭
        break;
    }

    // 增加读取的消息计数
    allocHandle.incMessagesRead(localRead);
} while (continueReading(allocHandle)); // 检查是否继续读取
```

doReadMessages(readBuf)先跳过，先看`allocHandle.incMessagesRead(localRead);`。进入到

```java
public final void incMessagesRead(int amt) {
    totalMessages += amt;
}
```

这里totalMessage, 刚才已经剖析过, 在NIO传输模式下也就是已经执行的循环次数, 这里每次执行一次循环都会加1

再去看循环终止条件`allocHandle.continueReading()`

```java
protected boolean continueReading(RecvByteBufAllocator.Handle allocHandle) {
    return allocHandle.continueReading();
}
public boolean continueReading() {
    return continueReading(defaultMaybeMoreSupplier);
}
// 继续读取的判断方法，决定是否可以继续读取数据
// 参数 maybeMoreDataSupplier 是一个无检查的布尔值供应者，用于判断是否还有更多数据可读
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    // 检查通道配置是否允许自动读取
    // 如果 config.isAutoRead() 返回 true，表示允许自动读取
    // 如果 respectMaybeMoreData 为 false，或者 maybeMoreDataSupplier.get() 返回 true，表示可以继续读取
    // totalMessages < maxMessagePerRead 确保已读取的消息数量小于每次读取的最大消息数量
    // (ignoreBytesRead || totalBytesRead > 0) 确保在忽略字节读取的情况下，已读取的字节数大于 0
    return config.isAutoRead() &&
        (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
        totalMessages < maxMessagePerRead && (ignoreBytesRead || totalBytesRead > 0);
}

private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {
    @Override
    public boolean get() {
        return attemptedBytesRead == lastBytesRead;
    }
};

```

config.isAutoRead(): 这里默认为true

 maybeMoreDataSupplier.get() 也就是 totalMessages < maxMessagePerRead: 表示当前读取的次数是否小于最大读取次数, 我们知道totalMessages每次循环都会自增, 而maxMessagePerRead默认值为16, 所以这里会限制循环不能超过16次, 也就是最多一次只能读取16条连接

这里就剖析完了Handle的创建和初始化过程, 并且剖析了循环终止条件等相关的逻辑

接着看`int localRead = doReadMessages(readBuf);`的逻辑。

首先看readBuf

```java
private final List<Object> readBuf = new ArrayList<Object>();
```

定义了一个ArrayList, doReadMessages(readBuf)用于将读到的链接放在这个list中, 因为这里是NioServerSocketChannel所以这走到了NioServerSocketChannel的doReadMessage()方法

```java
// 该方法用于读取消息并将其添加到给定的缓冲区中
protected int doReadMessages(List<Object> buf) throws Exception {
    // 尝试接受一个新的SocketChannel
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        // 如果成功接受到SocketChannel
        if (ch != null) {
            // 将新的NioSocketChannel添加到缓冲区
            buf.add(new NioSocketChannel(this, ch));
            return 1; // 返回成功读取的消息数量
        }
    } catch (Throwable t) {
        // 如果创建新通道失败，记录警告信息
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            // 尝试关闭SocketChannel
            ch.close();
        } catch (Throwable t2) {
            // 如果关闭SocketChannel失败，记录警告信息
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0; // 返回0表示没有读取到消息
}
```

首先拿到ServerSocketChannel中的jdk的Channel，尝试从channel来获取一个新的连接，并将其封装成NioSocketChannel后添加至buf中。

创建NioSocketChannel这部分之前已经提到过，最终会使用AbstractNioChannel来初始化channel，并监听SelectionKey.OP_READ事件。

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

接着看父类

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    // 创建唯一ID
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```

初始化unsafe, 跟到newUnsafe()方法中

```java
// 创建一个新的 unsafe 的通道实例
protected AbstractNioUnsafe newUnsafe() {
    return new NioSocketChannelUnsafe();
}

private final class NioSocketChannelUnsafe extends NioByteUnsafe {
    @Override
    // 准备关闭通道的方法
    protected Executor prepareToClose() {
        try {
            // 检查通道是否打开且SO_LINGER大于0
            if (javaChannel().isOpen() && config().getSoLinger() > 0) {
                // 取消通道的键，以避免在事件循环中出现死循环
                // 因为我们尝试在实际关闭之前进行读取或写入，这可能会由于SO_LINGER处理而延迟
                // 参考：https://github.com/netty/netty/issues/4449
                doDeregister();
                return GlobalEventExecutor.INSTANCE; // 返回全局事件执行器
            }
        } catch (Throwable ignore) {
            // 忽略错误，因为底层通道可能已经关闭，因此
            // getSoLinger()可能会产生异常。在这种情况下，我们只需返回null。
            // 参考：https://github.com/netty/netty/issues/4449
        }
        return null; // 返回null表示没有准备好的执行器
    }
}
```

最后初始化pipeline，这里先跳过。

回到NioSocketChannel

```java
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}

private NioSocketChannelConfig(NioSocketChannel channel, Socket javaSocket) {
    super(channel, javaSocket);
    // 计算每次聚集写入的最大字节数
    // Multiply by 2 to give some extra space in case the OS can process write data faster than we can provide.
    calculateMaxBytesPerGatheringWrite();
}

public DefaultSocketChannelConfig(SocketChannel channel, Socket javaSocket) {
    super(channel);
    // 检查并设置Java Socket对象
    this.javaSocket = ObjectUtil.checkNotNull(javaSocket, "javaSocket");

    // 如果可能，默认启用TCP_NODELAY以减少延迟
    if (PlatformDependent.canEnableTcpNoDelayByDefault()) {
        try {
            setTcpNoDelay(true);
        } catch (Exception e) {
            // 忽略异常
        }
    }
}

public DefaultChannelConfig(Channel channel) {
    this(channel, new AdaptiveRecvByteBufAllocator());
}

protected DefaultChannelConfig(Channel channel, RecvByteBufAllocator allocator) {
    setRecvByteBufAllocator(allocator, channel.metadata());
    this.channel = channel;
}
```

无论NioServerSocketChannel和NioSocketChannel, 最后都会初始化DefaultChannelConfig, 并创建可变ByteBuf分配器

在结束do-while循环后会通过for循环遍历readBuf，并传入pipeline.fireChannelRead进行读事件。

```java
int size = readBuf.size();
for (int i = 0; i < size; i ++) {
    readPending = false; // 标记读取操作已完成
    // 触发通道读取事件
    pipeline.fireChannelRead(readBuf.get(i));
}
```

最终使用ServerBootstrap.ServerBootstrapAcceptor#channelRead方法

```java
// 处理接收到的消息
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 将接收到的消息转换为子通道
    final Channel child = (Channel) msg;

    // 将子处理器添加到子通道的管道中
    child.pipeline().addLast(childHandler);

    // 设置子通道的选项
    setChannelOptions(child, childOptions, logger);
    // 设置子通道的属性
    setAttributes(child, childAttrs);

    // 如果有扩展，执行后初始化操作
    if (!extensions.isEmpty()) {
        for (ChannelInitializerExtension extension : extensions) {
            try {
                // 调用扩展的后初始化方法
                extension.postInitializeServerChildChannel(child);
            } catch (Exception e) {
                // 记录扩展初始化过程中抛出的异常
                logger.warn("Exception thrown from postInitializeServerChildChannel", e);
            }
        }
    }

    // 尝试注册子通道到子事件循环组
    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                // 如果注册失败，强制关闭子通道
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        // 如果注册过程中发生异常，强制关闭子通道
        forceClose(child, t);
    }
}
```

其中的msg即readBuf中的NioSocketChannel。在对child初始化参数后尝试注册到childGroup，这里的childGroup是启动时指定的workerGroup。

继续进入到SingleThreadEventLoop#register(io.netty.channel.ChannelPromise)方法

```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

这里的unsage也就是NioSocketChannelUnsafe，register最终会调用AbstractUnsafe的register()

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

    // 检查当前线程是否在 eventLoop 中运行，如果是，则直接调用 register0 方法进行注册;
    // 显然，当前方法在 main() 线程中与逆行
    // 如果当前线程不在 eventLoop 中运行，则通过 eventLoop.execute 提交一个任务到 eventLoop 中执行 register0 方法：
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

首先判断是不是当前NioEventLoop线程, 如果是, 则直接进行注册操作, 如果不是, 则封装成task在当前NioEventLoop中执行。

这里并不是当前NioEventLoop线程, 这是boss线程执行的, 所以这里会走到else, 如果是第一次的连接操作, work线程的NioEventLoop并没有启动, 所以这里也会启动NioEventLoop, 并开始轮询操作

接着看register0做实际的注册

```java
private void register0(ChannelPromise promise) {
    try {
        // 检查通道是否仍然开放，因为在事件循环外调用注册时，通道可能已经关闭
        // 参数检查，确保 promise 是不可取消的，并且通道是开放的
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return; // 如果不满足条件，直接返回
        }

        // 记录是否是第一次注册
        boolean firstRegistration = neverRegistered;

        // 执行实际的注册操作
        doRegister();

        // 更新状态，标记通道已经注册
        neverRegistered = false;
        registered = true;

        // 确保在通知 promise 之前调用 handlerAdded(...) 方法
        // 这是为了防止用户在 ChannelFutureListener 中触发事件时出现问题
        pipeline.invokeHandlerAddedIfNeeded();

        // 将 promise 设置为成功状态，并触发 pipeline.fireChannelRegistered 事件，通知管道已经注册
        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();

        // 仅在通道从未注册的情况下触发 channelActive 事件
        // 这可以防止在通道被注销并重新注册时多次触发 channelActive 事件
        if (isActive()) {
            if (firstRegistration) {
                // 如果这是第一次注册，触发 channelActive 事件
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // 如果通道之前已经注册，并且 autoRead() 被设置为 true，
                // 则需要重新开始读取以处理传入数据
                beginRead();
            }
        }
    } catch (Throwable t) {
        // 如果发生异常，直接关闭通道以避免文件描述符泄漏
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t); // 设置 promise 为失败状态
    }
}
```



```java
protected void doRegister() throws Exception {
    boolean selected = false; // 标记是否已经强制选择过
    for (;;) {
        try {
            // 将当前通道注册到事件循环的选择器中，初始兴趣操作设置为0
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return; // 注册成功，退出方法
        } catch (CancelledKeyException e) {
            // 如果选择键被取消，检查是否已经强制选择过
            if (!selected) {
                // 强制选择器立即选择，因为被取消的选择键可能仍然被缓存
                // 并且未被移除，因为尚未调用 Select.select(..) 操作
                eventLoop().selectNow();
                selected = true; // 标记为已强制选择
            } else {
                // 如果之前已经强制选择过，但选择键仍然被缓存，抛出异常
                // 这可能是JDK的一个bug
                throw e;
            }
        }
    }
}
```

这部分也是之前剖析过的jdk底层的注册, 只是不同的是, 这里的javaChannel()是SocketChanel而不是ServerSocketChannel

同样, 这里也是表示不关心任何事件, 只是在当前NioEventLoop绑定的selector上注册

做完实际的注册后后进入一段判断

```java
if (isActive()) {
    if (firstRegistration) {
        // 如果这是第一次注册，触发 channelActive 事件
        pipeline.fireChannelActive();
    } else if (config().isAutoRead()) {
        // 如果通道之前已经注册，并且 autoRead() 被设置为 true，
        // 则需要重新开始读取以处理传入数据
        beginRead();
    }
}
```

向下执行到doBeginRead

```java
public final void beginRead() {
    assertEventLoop(); // 断言当前线程在事件循环中

    try {
        // 调用 doBeginRead 方法执行实际的读取操作
        doBeginRead();
    } catch (final Exception e) {
        // 如果发生异常，使用 invokeLater 将异常处理推迟到事件循环中执行
        invokeLater(new Runnable() {
            @Override
            public void run() {
                // 在管道中触发异常捕获事件，通知处理异常
                pipeline.fireExceptionCaught(e);
            }
        });
        // 关闭通道并返回一个空的承诺
        close(voidPromise());
    }
}
```

服务端channel注册完之后也走到了这里。创建NioSocketChannel的时候初始化的是read事件, selectionKey是channel在注册时候返回的key, 所以selectionKey.interestOps(interestOps | readInterestOp)这一步, 会将当前channel的读事件注册到selector中去

注册完成之后, NioEventLoop就可以轮询当前channel的读事件了