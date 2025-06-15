---
title: 3. EventLoopGroup初始化
date: 2024-10-03 12:00:00 +0800
categories: [从零开始的Netty源码阅读]
tags: [blog]
---
# 事件循环组的初始化

在服务端的代码中，我们创建了两个EventLoopGroup来分别处理连接请求和业务操作。

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

首先看看该类型的继承关系

![EventLoopGroup](/assets/netty/EventLoopGroup.png)

图 3.1 EventLoopGroup继承关系图

## NioEventLoopGroup创建流程

以bossGroup为例，一步步深入源码

```java
// bossGroup
public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}

//workerGroup
public NioEventLoopGroup() {
    this(0);
}

public NioEventLoopGroup(int nThreads, Executor executor) {
    this(nThreads, executor, SelectorProvider.provider());
}

public NioEventLoopGroup(
    int nThreads, Executor executor, final SelectorProvider selectorProvider) {
    this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}

public NioEventLoopGroup(int nThreads, Executor executor, 
                         final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, selectorProvider,selectStrategyFactory,RejectedExecutionHandlers.reject());
}
```

最终会调用父类的构造器。

// TODO SelectorProvider.provider(), RejectedExecutionHandlers.reject()

```java
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

在这里判断了线程数是否为0，为0则赋值为CPU核心数的两倍。

> ```java
> DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
>     "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
> ```

同样，构造器也经过了几次调用后才开始真正创建实例

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}

protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    // 参数判断
    checkPositive(nThreads, "nThreads");

    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    
    // 创建 EventExecutor 数组
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
			// 省略部分代码
        }
    }

    // 绑定线程选择器
    chooser = chooserFactory.newChooser(children);

    // 为每个EventExecutor添加终止监听器
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

### 创建默认的线程工厂

> ```java
> executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
> ```

首先看newDefaultThreadFactory()方法的内容，跳过繁琐的构造器直接进入DefaultThreadFactory#DefaultThreadFactory(java.lang.String, boolean, int, java.lang.ThreadGroup)：

```java
public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
    ObjectUtil.checkNotNull(poolName, "poolName");

    if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
        throw new IllegalArgumentException(
                "priority: " + priority + " (expected: Thread.MIN_PRIORITY <= priority <= Thread.MAX_PRIORITY)");
    }

    prefix = poolName + '-' + poolId.incrementAndGet() + '-';
    this.daemon = daemon;
    this.priority = priority;
    this.threadGroup = threadGroup;
}
```

> ```java
> public class DefaultThreadFactory implements ThreadFactory {...}
> ```

DefaultThreadFactory继承于ThreadFactory用于创建线程

此处的`poolName`为传递过来的`simpleClassName`类名，也就是`"nioEventLoopGroup"`。方法中设置线程名前缀和其他参数。

```java
public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
    this.threadFactory = ObjectUtil.checkNotNull(threadFactory, "threadFactory");
}
```

此处也只是初始化变量。

### 创建 EventExecutor 数组

使用数组来保存EventExecutor，继续看该数组的初始化：

> ```java
> children[i] = newChild(executor, args);
> ```

实际调用的是NioEventLoopGroup#newChild

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    SelectorProvider selectorProvider = (SelectorProvider) args[0];
    SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory) args[1];
    RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler) args[2];
    EventLoopTaskQueueFactory taskQueueFactory = null;
    EventLoopTaskQueueFactory tailTaskQueueFactory = null;

    int argsLength = args.length;
    if (argsLength > 3) {
        taskQueueFactory = (EventLoopTaskQueueFactory) args[3];
    }
    if (argsLength > 4) {
        tailTaskQueueFactory = (EventLoopTaskQueueFactory) args[4];
    }
    return new NioEventLoop(this, executor, selectorProvider,
            selectStrategyFactory.newSelectStrategy(),
            rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
}
```

最后创建了一个NioEventLoop对象。继续深入NioEventLoop#NioEventLoop的调用链

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
             EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) {
    super(parent, executor, false, newTaskQueue(taskQueueFactory), newTaskQueue(tailTaskQueueFactory),
          rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    // 调用openSelector()方法来打开一个新的Selector实例。
    // Selector是Java NIO中用于多路复用I/O操作的核心组件。
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}
```

SingleThreadEventLoop#SingleThreadEventLoop

```java
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                boolean addTaskWakesUp, Queue<Runnable> taskQueue, Queue<Runnable> tailTaskQueue,
                                RejectedExecutionHandler rejectedExecutionHandler) {
    super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
    tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
}
```

SingleThreadEventExecutor#SingleThreadEventExecutor

```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                    RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    // 将传入的executor包装成一个新的Executor，并将当前的SingleThreadEventExecutor实例与之关联
    this.executor = ThreadExecutorMap.apply(executor, this);
    this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
    this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

关联的目的是使SingleThreadEventExecutor可以使用传入的 executor 来执行任务，同时保持对任务执行环境的控制。

```java
protected AbstractScheduledEventExecutor(EventExecutorGroup parent) {
    super(parent);
}
```

```java
protected AbstractEventExecutor(EventExecutorGroup parent) {
    this.parent = parent;
}
```

这里初始化了parent，也就是从NioEventLoop(this, executor,  ...);传过来的bossGroup。

### 绑定线程线程选择器

```java
chooser = chooserFactory.newChooser(children);
```

进入EventExecutorChooserFactory.EventExecutorChooser来根据

children的长度，也就是线程数是否是2的倍数来选择EventExecutorChooser：

```java
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

看看这两个的区别：

```java
private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    private final AtomicInteger idx = new AtomicInteger();
    private final EventExecutor[] executors;

    PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
}
```

```java
private static final class GenericEventExecutorChooser implements EventExecutorChooser {
    // Use a 'long' counter to avoid non-round-robin behaviour at the 32-bit overflow boundary.
    // The 64-bit long solves this by placing the overflow so far into the future, that no system
    // will encounter this in practice.
    private final AtomicLong idx = new AtomicLong();
    private final EventExecutor[] executors;

    GenericEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[(int) Math.abs(idx.getAndIncrement() % executors.length)];
    }
}
```

区别仅在于next()方法所实现的索引在executors内循环的逻辑，前者使用位运算达到更高的性能。

最后给所有children添加终止监听器后完成创建。

### 优化selector

在创建NioEventLoop时，有一段openSelector()方法的调用还没仔细探讨：

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
             EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) {
    super(parent, executor, false, newTaskQueue(taskQueueFactory), newTaskQueue(tailTaskQueueFactory),
          rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    // 调用openSelector()方法来打开一个新的Selector实例。
    // Selector是Java NIO中用于多路复用I/O操作的核心组件。
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}
```

下面看看这个方法具体做了什么：

`openSelector()` 方法的主要目的是打开一个新的 `Selector` 实例，这是 Java NIO 中用于多路复用 I/O 操作的核心组件。该方法还尝试对选择器进行性能优化，以提高事件循环的效率。

1. **打开选择器**：

   - 方法首先通过调用 `provider.openSelector()` 来创建一个新的 `Selector` 实例。如果打开选择器失败，方法会捕获 `IOException` 并抛出一个 `ChannelException`，确保调用者能够得到明确的错误信息。

   ```java
   try {
       unwrappedSelector = provider.openSelector(); // 打开新的选择器
   } catch (IOException e) {
       throw new ChannelException("failed to open a new selector", e); // 处理异常
   }
   ```

2. **选择器优化的条件判断**：

   - 方法检查静态常量 `DISABLE_KEY_SET_OPTIMIZATION`，默认值为`false`。如果该值为 `true`，则表示不进行选择器的优化，直接返回未优化的选择器。这种设计允许在某些情况下（例如调试或特定环境）禁用优化。

   ```java
   if (DISABLE_KEY_SET_OPTIMIZATION) {
       return new SelectorTuple(unwrappedSelector); // 返回未优化的选择器
   }
   ```

3. **选择器实现类的检查**：

   - 方法使用 `AccessController.doPrivileged` 来尝试获取 `sun.nio.ch.SelectorImpl` 类的引用。这是为了确保当前的选择器实现是可以被优化的。如果获取失败，方法会记录调试信息。

   ```java
   Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
       @Override
       public Object run() {
           try {
               return Class.forName("sun.nio.ch.SelectorImpl", false, PlatformDependent.getSystemClassLoader());
           } catch (Throwable cause) {
               return cause;
           }
       }
   });
   ```

4. **选择器字段的反射访问**：

   - 如果选择器实现类是可优化的，方法会尝试通过反射访问选择器的 `selectedKeys` 和 `publicSelectedKeys` 字段。这是为了替换这些字段为自定义的 `SelectedSelectionKeySet`，以提高性能。

   ```java
   Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
   Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");
   ```

5. **使用 `Unsafe` 进行优化**：

   - 如果 Java 版本支持并且可以使用 `sun.misc.Unsafe`，方法会尝试通过 `Unsafe` 来替换选择器的 `selectedKeys`。这允许在 Java 9 及以上版本中进行优化，而无需额外的标志。

   ```java
   if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
       // 使用 Unsafe 替换 SelectionKeySet
       // ... 省略代码 ...
   }
   ```

6. **返回选择器元组**：

   - 最后，方法返回一个 `SelectorTuple` 对象，其中包含新创建的选择器和可能的优化选择器。这使得调用者可以使用这个元组进行后续的 I/O 操作。

   ```java
   return new SelectorTuple(unwrappedSelector, optimizedSelector); // 返回优化后的选择器
   ```

## NioEventLoop启动

在使用`children[i] = newChild(executor, args);`创建完NioEventLoop之后没有直接调用启动函数，线程
是否还未启动？

> ```java
> // 注册一个新的Channel到EventLoopGroup中，并返回一个ChannelFuture对象
> ChannelFuture regFuture = config().group().register(channel);
> ```

在SingleThreadEventExecutor#execute(java.lang.Runnable, boolean)打上断点后debug发现调用栈为：

![NioEventLoopCallStack](/assets/netty/NioEventLoopCallStack.png)

可以发现，该方法最初是由第2节中提到的AbstractBootstrap#initAndRegister方法所调用的register来进行调用。在[第2节](/posts/02-ServerBootstrap初始化)中已经提到过register的流程，现在直接看execute的执行过程。

```java
// SingleThreadEventExecutor#execute(java.lang.Runnable)
public void execute(Runnable task) {
    execute0(task);
}
// SingleThreadEventExecutor#execute0
private void execute0(@Schedule Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    execute(task, wakesUpForTask(task));
}
// SingleThreadEventExecutor#execute(java.lang.Runnable, boolean)
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                // The task queue does not support removal so the best thing we can do is to just move on and
                // hope we will be able to pick-up the task before its completely terminated.
                // In worst case we will log on termination.
            }
            if (reject) {
                reject();
            }
        }
    }

    // 是否需要唤醒执行线程
    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}
```

首先看看inEventLoop的作用

```java
public boolean inEventLoop(Thread thread) {
    return thread == this.thread;
}
```

// TODO：

这段保证了同一个channel的注册、读、写等都在eventLoop完成，避免多线程的锁竞争。

SingleThreadEventExecutor#execute有这么一段代码

```java
if (!addTaskWakesUp && immediate) {
    wakeup(inEventLoop);
}
```

addTaskWakesUp代表添加task之后，是否需要唤醒执行线程, 这个属性是在初始化NioEventLoop的时候传入，默认值为false。

> ```java
> NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
>              SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
>              EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) {
>     super(parent, executor, false, newTaskQueue(taskQueueFactory), newTaskQueue(tailTaskQueueFactory),
>             rejectedExecutionHandler);
> 	// 省略代码
> }
> ```

接着看核心的startThread()方法

```java
// SingleThreadEventExecutor#startThread
private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                // 启动线程
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
// SingleThreadEventExecutor#doStartThread
private void doStartThread() {
    // 确保线程还未启动
    assert thread == null;
    // 通过 executor 来启动线程
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
                // 执行任务
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                // 省略代码
            }
        }
    });
}
```

首先看executor.execute()：

```java
// ThreadExecutorMap#apply
public static Executor apply(final Executor executor, final EventExecutor eventExecutor) {
    ObjectUtil.checkNotNull(executor, "executor");
    ObjectUtil.checkNotNull(eventExecutor, "eventExecutor");
    return new Executor() {
        @Override
        public void execute(final Runnable command) {
            // 实际调用这个execute方法
            executor.execute(apply(command, eventExecutor));
        }
    };
}
// ThreadPerTaskExecutor#execute
// 真正调用的方法
public void execute(Runnable command) {
    threadFactory.newThread(command).start();
}
```

这里的apply()也就是之前提到的将当前的SingleThreadEventExecutor实例与之关联的方法。

```java
public static Runnable apply(final Runnable command, final EventExecutor eventExecutor) {
    ObjectUtil.checkNotNull(command, "command");
    ObjectUtil.checkNotNull(eventExecutor, "eventExecutor");
    return new Runnable() {
        @Override
        public void run() {
            // 设置当前线程的 EventExecutor
            setCurrentEventExecutor(eventExecutor);
            try {
                command.run();
            } finally {
                // 移除当前线程的 EventExecutor
                setCurrentEventExecutor(null);
            }
        }
    };
}
```

查看真正的execute方法后发现，其实是使用线程工厂创建线程后直接执行。

### NioEventLoop#run

`run()` 方法是 `NioEventLoop` 类中启动的核心方法，负责执行事件循环的主要逻辑。以下是对该方法的详细说明，包括其主要功能和各个部分的解释：

```java
// 
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
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
            } else {
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
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                             selector, e);
            }
        } catch (Error e) {
            throw e;
        } catch (Throwable t) {
            handleLoopException(t);
        } finally {
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Error e) {
                throw e;
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
}

```

`run()` 方法是一个无限循环，负责处理 I/O 事件、调度任务和管理选择器的状态。它通过选择器来监控多个通道的 I/O 操作，并根据需要执行相应的任务。

1. **选择策略计算**：
   
- 方法首先初始化 `selectCnt` 计数器，用于跟踪选择器的调用次数。然后进入一个无限循环，尝试计算选择策略。
   
   ```java
   int selectCnt = 0;
   for (;;) {
       try {
           int strategy;
           try {
               strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
```
   
2. **选择策略的处理**：
   
   - 根据计算出的策略，方法会决定如何处理 I/O 操作：
     - **继续**：如果策略是 `SelectStrategy.CONTINUE`，则跳过当前循环，继续下一次迭代。
     - **忙等待**：如果策略是 `SelectStrategy.BUSY_WAIT`，则不支持忙等待，直接跳到选择操作。
  - **选择**：如果策略是 `SelectStrategy.SELECT`，则调用选择器进行选择操作，获取准备好的通道。
   
   ```java
   switch (strategy) {
   case SelectStrategy.CONTINUE:
    continue;
   
   case SelectStrategy.BUSY_WAIT:
    // fall-through to SELECT since the busy-wait is not supported with NIO
   
   case SelectStrategy.SELECT:
       long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
       if (curDeadlineNanos == -1L) {
           curDeadlineNanos = NONE; // nothing on the calendar
       }
       nextWakeupNanos.set(curDeadlineNanos);
       try {
           if (!hasTasks()) {
               strategy = select(curDeadlineNanos);
           }
       } finally {
           nextWakeupNanos.lazySet(AWAKE);
       }
```
   
3. **异常处理**：
   - 如果在选择过程中发生 `IOException`，则调用 `rebuildSelector0()` 方法重建选择器，并重置 `selectCnt` 计数器。然后调用 `handleLoopException(e)` 处理异常。

   ```java
   } catch (IOException e) {
       rebuildSelector0();
       selectCnt = 0;
       handleLoopException(e);
       continue;
   }
   ```

4. **处理已选择的键**：
   
   - 方法增加 `selectCnt` 计数器，并重置 `cancelledKeys` 和 `needsToSelectAgain` 标志。根据 `ioRatio` 的值，决定如何处理已选择的键和运行任务：
     - 如果 `ioRatio` 为 100，优先处理已选择的键，然后运行所有任务。
  - 如果 `ioRatio` 小于 100，先处理已选择的键，再根据 I/O 操作的时间比例运行任务。
   
   ```java
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
           ranTasks = runAllTasks();
       }
   } else if (strategy > 0) {
       final long ioStartTime = System.nanoTime();
       try {
           processSelectedKeys();
       } finally {
           final long ioTime = System.nanoTime() - ioStartTime;
           ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
       }
   } else {
       ranTasks = runAllTasks(0); // This will run the minimum number of tasks
   }
```
   
5. **选择器调用次数的监控**：
   - 如果在处理任务时有任务被执行，或者选择策略大于 0，方法会检查 `selectCnt` 是否超过 `MIN_PREMATURE_SELECTOR_RETURNS`，并记录调试信息。

   ```java
   if (ranTasks || strategy > 0) {
       if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
           logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                   selectCnt - 1, selector);
       }
       selectCnt = 0;
   } else if (unexpectedSelectorWakeup(selectCnt)) {
       selectCnt = 0;
   }
   ```

6. **异常捕获与处理**：
   - 方法捕获 `CancelledKeyException`，并记录调试信息。对于其他异常，调用 `handleLoopException(t)` 进行处理。

   ```java
   } catch (CancelledKeyException e) {
       if (logger.isDebugEnabled()) {
           logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                   selector, e);
       }
   } catch (Error e) {
       throw e;
   } catch (Throwable t) {
       handleLoopException(t);
   }
   ```

7. **清理与关闭**：
   - 在 `finally` 块中，方法始终检查是否正在关闭事件循环。如果是，则调用 `closeAll()` 关闭所有通道，并确认关闭。

   ```java
   finally {
       try {
           if (isShuttingDown()) {
               closeAll();
               if (confirmShutdown()) {
                   return;
               }
           }
       } catch (Error e) {
           throw e;
       } catch (Throwable t) {
           handleLoopException(t);
       }
   }
   ```

`run()` 方法是 `NioEventLoop` 的核心，负责管理事件循环的执行。它通过选择器监控 I/O 操作，调度任务，并处理各种异常情况。方法的设计确保了高效的 I/O 处理和任务调度，同时具备良好的异常处理机制，以应对可能出现的错误和异常情况。

### 选择通道

`select(curDeadlineNanos)` 方法是 Java NIO 中的一个关键方法，用于监控多个通道的 I/O 状态。它会阻塞当前线程，直到有一个或多个通道准备好进行 I/O 操作，或者直到指定的超时时间到达。

#### 方法调用的上下文

在 `run()` 方法中，`select(curDeadlineNanos)` 被调用的上下文如下：

```java
long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
if (curDeadlineNanos == -1L) {
    curDeadlineNanos = NONE; // nothing on the calendar
}
nextWakeupNanos.set(curDeadlineNanos);
try {
    if (!hasTasks()) {
        strategy = select(curDeadlineNanos); // 调用 select 方法进行选择操作
    }
} finally {
    nextWakeupNanos.lazySet(AWAKE); // 更新下次唤醒时间
}
```

1. **获取截止时间**：
   - `long curDeadlineNanos = nextScheduledTaskDeadlineNanos();`
     - 这行代码调用 `nextScheduledTaskDeadlineNanos()` 方法，获取下一个计划任务的截止时间（以纳秒为单位）。如果没有计划任务，返回值将是 `-1L`。

2. **设置超时值**：
   - `if (curDeadlineNanos == -1L) { curDeadlineNanos = NONE; }`
     - 如果 `curDeadlineNanos` 的值为 `-1L`，则将其设置为 `NONE`，表示选择器将无限期等待，直到有通道准备好进行 I/O 操作。

3. **设置下次唤醒时间**：
   - `nextWakeupNanos.set(curDeadlineNanos);`
     - 这行代码更新下次唤醒的时间，以便在后续的逻辑中使用。这个时间用于决定何时需要再次唤醒选择器。

4. **调用 `select` 方法**：
   - `strategy = select(curDeadlineNanos);`
     - 这行代码调用 `select` 方法，传入截止时间，开始监控通道的 I/O 状态。

#### `select(long deadlineNanos)` 方法实现

下面是 `select(long deadlineNanos)` 方法的简化实现，结合其内部逻辑进行详细解释：

```java
private int select(long deadlineNanos) throws IOException {
    if (deadlineNanos == NONE) {
        return selector.select(); // 无限期等待
    }
    // Timeout will only be 0 if deadline is within 5 microsecs
    long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
    return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis); // 根据超时值选择
}
```

1. **检查截止时间**：
   - `if (deadlineNanos == NONE) { return selector.select(); }`
     - 如果 `deadlineNanos` 的值为 `NONE`，表示没有超时限制，调用 `selector.select()` 方法进行无限期等待，直到有通道准备好进行 I/O 操作。
     - 这个方法会阻塞当前线程，直到至少有一个通道准备好进行 I/O 操作，或者选择器被唤醒。

2. **计算超时值**：
   - `long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;`
     - 这行代码将截止时间转换为毫秒。`deadlineToDelayNanos` 是一个辅助方法，用于将纳秒转换为适合选择器的超时格式。
     - `995000L` 是一个微调值，用于确保在计算超时时间时，避免因时间精度问题导致的错误。

3. **选择操作**：
   - `return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);`
     - 如果计算出的超时值小于等于 0，调用 `selector.selectNow()` 方法立即返回，表示没有通道准备好。
     - 否则，调用 `selector.select(timeoutMillis)` 方法，阻塞当前线程，直到有通道准备好进行 I/O 操作，或者超时到达。
     - `selector.select(timeoutMillis)` 方法会返回准备好的通道数量，供后续逻辑使用。

#### `selector.select()` 和 `selector.selectNow()` 方法

- **`selector.select()`**：
  - 该方法会阻塞当前线程，直到至少有一个通道准备好进行 I/O 操作，或者选择器被唤醒。它会返回准备好的通道数量。

- **`selector.selectNow()`**：
  - 该方法不会阻塞，它会立即返回当前准备好的通道数量。如果没有通道准备好，则返回 0。

`strategy = select(curDeadlineNanos);` 的主要作用是调用选择器的 `select` 方法，监控多个通道的 I/O 状态。通过传入截止时间，选择器能够决定等待的时间长度，从而有效地管理 I/O 操作。这个过程是事件循环中处理 I/O 事件的关键步骤，确保系统能够及时响应来自客户端或其他通道的请求。

通过这种方式，事件循环能够高效地管理多个通道的 I/O 操作，确保系统的性能和响应能力。

### 处理IO

`run()`方法内部调用`processSelectedKeys()`来处理IO。

`processSelectedKeys()` 方法的主要目的是处理在选择器中准备好的通道的 I/O 操作。它会遍历所有已选择的键（即准备好进行 I/O 操作的通道），并根据每个键的状态执行相应的操作。

#### 方法内部实现

以下是 `processSelectedKeys()` 方法的简化示例，结合其内部逻辑进行详细解释：

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(); // 优化处理已选择的键
    } else {
        processSelectedKeysPlain(selector.selectedKeys()); // 普通处理已选择的键
    }
}
```

##### 1. 检查 `selectedKeys`

- `selectedKeys` 是一个用于存储已选择的键的集合。如果 `selectedKeys` 不为 `null`，则调用 `processSelectedKeysOptimized()` 方法进行优化处理。
- 如果 `selectedKeys` 为 `null`，则调用 `processSelectedKeysPlain(selector.selectedKeys())` 方法，使用选择器的默认已选择键集合进行处理。

##### 2. 优化处理

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i]; // 获取当前选择的键
        selectedKeys.keys[i] = null; // 清空当前键以便垃圾回收

        final Object a = k.attachment(); // 获取与键关联的通道或任务

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a); // 处理 NIO 通道
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task); // 处理 NioTask
        }

        if (needsToSelectAgain) {
            selectedKeys.reset(i + 1); // 重置已选择的键
            selectAgain(); // 重新选择
            i = -1; // 重置索引
        }
    }
}
```

- **遍历已选择的键**：使用 `for` 循环遍历 `selectedKeys` 中的每个键。
- **获取键和清空**：通过 `selectedKeys.keys[i]` 获取当前的 `SelectionKey`，并将其清空以便于垃圾回收。
- **获取附件**：`k.attachment()` 方法返回与该键关联的对象，通常是一个通道或任务。
- **处理通道或任务**：
  - 如果附件是 `AbstractNioChannel` 的实例，调用 `processSelectedKey(k, (AbstractNioChannel) a)` 方法处理该通道。
  - 如果附件是 `NioTask` 的实例，调用 `processSelectedKey(k, task)` 方法处理该任务。

- **重新选择**：如果 `needsToSelectAgain` 为 `true`，则调用 `selectedKeys.reset(i + 1)` 重置已选择的键，并调用 `selectAgain()` 重新选择。

##### 3. 普通处理

```java
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
    if (selectedKeys.isEmpty()) {
        return; // 如果没有已选择的键，直接返回
    }

    Iterator<SelectionKey> i = selectedKeys.iterator(); // 获取迭代器
    for (;;) {
        final SelectionKey k = i.next(); // 获取当前选择的键
        final Object a = k.attachment(); // 获取与键关联的对象
        i.remove(); // 从集合中移除当前键

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a); // 处理 NIO 通道
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task); // 处理 NioTask
        }

        if (!i.hasNext()) {
            break; // 如果没有更多的键，退出循环
        }

        if (needsToSelectAgain) {
            selectAgain(); // 重新选择
            selectedKeys = selector.selectedKeys(); // 更新已选择的键集合

            if (selectedKeys.isEmpty()) {
                break; // 如果已选择的键集合为空，退出循环
            } else {
                i = selectedKeys.iterator(); // 重新获取迭代器
            }
        }
    }
}
```

- **检查是否为空**：首先检查 `selectedKeys` 是否为空，如果为空，直接返回。
- **获取迭代器**：使用 `selectedKeys.iterator()` 获取一个迭代器，以便遍历已选择的键。
- **遍历键**：使用 `for` 循环遍历每个 `SelectionKey`。
- **处理通道或任务**：与优化处理相同，根据附件的类型调用相应的处理方法。
- **移除键**：在处理完当前键后，从集合中移除该键。
- **重新选择**：如果 `needsToSelectAgain` 为 `true`，则调用 `selectAgain()` 重新选择，并更新已选择的键集合。

`processSelectedKeys()` 方法的主要作用是处理在选择器中准备好的通道的 I/O 操作。它通过遍历已选择的键并根据每个键的状态执行相应的操作，确保事件循环能够有效地响应 I/O 事件。该方法的实现分为优化处理和普通处理两种方式，具体取决于 `selectedKeys` 的状态。通过这些处理，事件循环能够高效地管理多个通道的 I/O 操作，确保系统的性能和响应能力。

### 执行任务

#### 代码上下文

这段代码位于 `NioEventLoop` 类的 `run()` 方法中，主要负责根据 I/O 操作的比例（`ioRatio`）来处理已选择的键和运行任务。以下是代码的完整上下文：

```java
if (ioRatio == 100) {
    try {
        if (strategy > 0) {
            processSelectedKeys(); // 处理已选择的键
        }
    } finally {
        // 确保始终运行任务
        ranTasks = runAllTasks(); // 运行所有任务
    }
} else if (strategy > 0) {
    final long ioStartTime = System.nanoTime(); // 记录 I/O 开始时间
    try {
        processSelectedKeys(); // 处理已选择的键
    } finally {
        // 确保始终运行任务
        final long ioTime = System.nanoTime() - ioStartTime; // 计算 I/O 操作耗时
        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio); // 根据 I/O 耗时运行任务
    }
} else {
    ranTasks = runAllTasks(0); // 运行最少数量的任务
}
```

1. **判断 I/O 比例**：
   - `if (ioRatio == 100)`：这行代码检查 `ioRatio` 的值是否为 100。`ioRatio` 是一个表示 I/O 操作与非 I/O 操作时间比例的变量。如果 `ioRatio` 为 100，表示事件循环将专注于 I/O 操作，不会尝试平衡 I/O 和非 I/O 任务。

2. **处理 I/O 操作**：
   - `if (strategy > 0)`：这行代码检查 `strategy` 的值是否大于 0。`strategy` 是选择器返回的值，表示有多少通道准备好进行 I/O 操作。如果 `strategy` 大于 0，表示有通道准备好进行 I/O 操作。
   - `processSelectedKeys();`：调用 `processSelectedKeys()` 方法来处理这些准备好的通道。这个方法会遍历所有已选择的键，并根据每个键的状态执行相应的 I/O 操作。

3. **确保运行所有任务**：
   - `finally` 块中的 `ranTasks = runAllTasks();` 确保无论是否有 I/O 操作，都会运行所有待处理的任务。`runAllTasks()` 方法会执行事件循环中所有排队的任务，确保系统能够处理所有的任务。

4. **处理 I/O 比例小于 100 的情况**：
   - `else if (strategy > 0)`：如果 `ioRatio` 小于 100，但 `strategy` 大于 0，表示有通道准备好进行 I/O 操作。
   - `final long ioStartTime = System.nanoTime();`：记录当前时间，以便后续计算 I/O 操作的耗时。`System.nanoTime()` 返回当前时间的纳秒值。

5. **处理已选择的键**：
   - `processSelectedKeys();`：调用该方法处理已选择的键，执行 I/O 操作。这个方法会遍历所有准备好的通道，并根据通道的状态执行相应的 I/O 操作。

6. **计算 I/O 耗时**：
   - `finally` 块中的代码计算 I/O 操作的耗时，并根据 I/O 耗时和 `ioRatio` 运行相应数量的任务。
   - `final long ioTime = System.nanoTime() - ioStartTime;`：计算 I/O 操作的耗时，单位为纳秒。

7. **运行任务**：
   - `ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);`：根据 I/O 耗时和 `ioRatio` 运行相应数量的任务。
     - 例如，如果 `ioRatio` 是 50，表示 I/O 和非 I/O 任务的时间应该平衡，那么将 I/O 耗时的一半用于运行非 I/O 任务。

8. **处理没有 I/O 操作的情况**：
   - `else` 块处理没有通道准备好的情况，调用 `runAllTasks(0)` 方法运行最少数量的任务。
   - `ranTasks = runAllTasks(0);`：这行代码会运行最少数量的任务，通常是为了确保事件循环不会被阻塞。

好的，让我们结合你提供的 `processSelectedKeys()`、`runAllTasks()` 和 `runAllTasks(long timeoutNanos)` 方法的实现，重新输出上一个回答中的方法实现部分，并进行详细解释。

#### `processSelectedKeys()` 方法

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(); // 如果 selectedKeys 不为 null，调用优化处理方法
    } else {
        processSelectedKeysPlain(selector.selectedKeys()); // 否则，调用普通处理方法
    }
}
```

##### 1. `processSelectedKeysOptimized()`

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        selectedKeys.keys[i] = null; // 允许垃圾回收

        final Object a = k.attachment(); // 获取与键关联的对象

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a); // 处理通道
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task); // 处理任务
        }

        if (needsToSelectAgain) {
            selectedKeys.reset(i + 1); // 重置已选择的键
            selectAgain(); // 重新选择
            i = -1; // 重置索引
        }
    }
}
```

- **遍历已选择的键**：
  - `for (int i = 0; i < selectedKeys.size; ++i)`：遍历 `selectedKeys` 中的每个 `SelectionKey`。
  
- **内存管理**：
  - `selectedKeys.keys[i] = null;`：将当前键设置为 `null`，以便在通道关闭后允许垃圾回收。

- **获取附件**：
  - `final Object a = k.attachment();`：获取与当前键关联的对象，可能是一个通道或任务。

- **处理逻辑**：
  - `if (a instanceof AbstractNioChannel)`：检查附件是否为 `AbstractNioChannel` 的实例。
    - 如果是，调用 `processSelectedKey(k, (AbstractNioChannel) a)` 处理该通道。
  - 否则，假设附件是 `NioTask<SelectableChannel>` 的实例，调用 `processSelectedKey(k, task)` 处理该任务。

- **重新选择**：
  - `if (needsToSelectAgain)`：如果需要重新选择通道，重置已选择的键并调用 `selectAgain()` 方法。

##### 2. `processSelectedKeysPlain(Set<SelectionKey> selectedKeys)`

```java
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
    if (selectedKeys.isEmpty()) {
        return; // 如果集合为空，直接返回
    }

    Iterator<SelectionKey> i = selectedKeys.iterator();
    for (;;) {
        final SelectionKey k = i.next(); // 获取当前选择的键
        final Object a = k.attachment(); // 获取与键关联的对象
        i.remove(); // 从集合中移除当前键

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a); // 处理通道
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task); // 处理任务
        }

        if (!i.hasNext()) {
            break; // 如果没有更多的键，退出循环
        }

        if (needsToSelectAgain) {
            selectAgain(); // 重新选择
            selectedKeys = selector.selectedKeys(); // 更新已选择的键集合

            if (selectedKeys.isEmpty()) {
                break; // 如果已选择的键集合为空，退出循环
            } else {
                i = selectedKeys.iterator(); // 重新获取迭代器
            }
        }
    }
}
```

- **检查是否为空**：
  - `if (selectedKeys.isEmpty())`：检查传入的 `selectedKeys` 是否为空。如果为空，直接返回，避免创建新的迭代器。

- **遍历已选择的键**：
  - `Iterator<SelectionKey> i = selectedKeys.iterator();`：获取迭代器以遍历 `selectedKeys`。
  - `for (;;) { ... }`：使用无限循环遍历每个 `SelectionKey`。

- **处理逻辑**：
  - `final SelectionKey k = i.next();`：获取当前选择的键。
  - `final Object a = k.attachment();`：获取与当前键关联的对象。
  - `i.remove();`：从集合中移除当前键。

- **调用处理方法**：
  - `if (a instanceof AbstractNioChannel)`：检查附件是否为 `AbstractNioChannel` 的实例。
    - 如果是，调用 `processSelectedKey(k, (AbstractNioChannel) a)` 处理该通道。
  - 否则，假设附件是 `NioTask<SelectableChannel>` 的实例，调用 `processSelectedKey(k, task)` 处理该任务。

- **重新选择**：
  - `if (needsToSelectAgain)`：如果需要重新选择通道，调用 `selectAgain()` 方法，并更新 `selectedKeys` 集合。

##### 3. `processSelectedKey(SelectionKey k, AbstractNioChannel ch)`

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe(); // 获取通道的 NioUnsafe 实例
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop(); // 获取通道的事件循环
        } catch (Throwable ignored) {
            return; // 如果获取事件循环失败，直接返回
        }
        if (eventLoop == this) {
            unsafe.close(unsafe.voidPromise()); // 如果通道仍然注册在当前事件循环中，关闭通道
        }
        return;
    }

    try {
        int readyOps = k.readyOps(); // 获取准备好的操作
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps(); // 获取当前兴趣操作
            ops &= ~SelectionKey.OP_CONNECT; // 移除连接操作
            k.interestOps(ops); // 更新兴趣操作
            unsafe.finishConnect(); // 完成连接
        }

        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            unsafe.forceFlush(); // 处理写操作
        }

        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read(); // 处理读操作
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise()); // 如果键被取消，关闭通道
    }
}
```

- **获取 NioUnsafe 实例**：
  - `final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();`：获取通道的 `NioUnsafe` 实例，用于执行底层的 I/O 操作。

- **检查键的有效性**：
  - `if (!k.isValid())`：检查 `SelectionKey` 是否有效。如果无效，尝试获取通道的事件循环并判断通道是否仍然注册在当前事件循环中。

- **处理 I/O 操作**：
  - `int readyOps = k.readyOps();`：获取当前键的准备操作。
  - **连接操作**：
    - `if ((readyOps & SelectionKey.OP_CONNECT) != 0)`：如果准备操作中包含连接操作，调用 `finishConnect()` 完成连接。
  - **写操作**：
    - `if ((readyOps & SelectionKey.OP_WRITE) != 0)`：如果准备操作中包含写操作，调用 `forceFlush()` 进行写入。
  - **读操作**：
    - `if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0)`：如果准备操作中包含读操作或接受操作，调用 `read()` 进行读取。

##### 4. `processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task)`

```java
private void processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task) {
    // 处理与任务相关的逻辑
}
```

- **处理逻辑**：与处理通道的逻辑类似，检查键的有效性，并根据任务的状态执行相应的操作。具体实现可能与处理通道的逻辑相似。

通过结合源码分析，我们可以看到 `processSelectedKeys()` 方法通过调用 `processSelectedKeysOptimized()` 和 `processSelectedKeysPlain()` 方法来处理已选择的键。前者用于优化处理，后者用于普通处理。`processSelectedKey()` 方法用于处理每个准备好的通道或任务的 I/O 操作。这些方法共同确保了事件循环能够有效地响应 I/O 事件，处理来自客户端或其他通道的数据。

####  `runAllTasks()`

```java
protected boolean runAllTasks() {
    assert inEventLoop();
    boolean fetchedAll;
    boolean ranAtLeastOne = false;

    do {
        fetchedAll = fetchFromScheduledTaskQueue(); // 从计划任务队列中获取任务
        if (runAllTasksFrom(taskQueue)) { // 从任务队列中运行所有任务
            ranAtLeastOne = true; // 标记至少运行了一个任务
        }
    } while (!fetchedAll); // 继续处理，直到所有计划任务都被获取

    if (ranAtLeastOne) {
        lastExecutionTime = getCurrentTimeNanos(); // 更新最后执行时间
    }
    afterRunningAllTasks(); // 执行所有任务后的处理
    return ranAtLeastOne; // 返回是否运行了至少一个任务
}
```

1. **`assert inEventLoop();`**：
   - 确保当前线程在事件循环中执行，防止在错误的线程中调用该方法。

2. **`fetchedAll = fetchFromScheduledTaskQueue();`**：
   - 调用 `fetchFromScheduledTaskQueue()` 方法，从计划任务队列中获取任务。
   - 如果计划任务队列为空，返回 `true`，表示所有任务都已获取。

3. **`if (runAllTasksFrom(taskQueue)) { ranAtLeastOne = true; }`**：
   - 调用 `runAllTasksFrom(taskQueue)` 方法，从任务队列中运行所有任务。
   - 如果成功运行了任务，将 `ranAtLeastOne` 设置为 `true`。

4. **`while (!fetchedAll);`**：
   - 使用 `do-while` 循环，继续处理，直到所有计划任务都被获取。

5. **`if (ranAtLeastOne) { lastExecutionTime = getCurrentTimeNanos(); }`**：
   - 如果至少运行了一个任务，更新最后执行时间。

6. **`afterRunningAllTasks();`**：
   - 执行所有任务后的处理，可能用于清理或状态更新。

7. **`return ranAtLeastOne;`**：
   - 返回是否运行了至少一个任务。

#### `fetchFromScheduledTaskQueue()`

```java
private boolean fetchFromScheduledTaskQueue() {
    if (scheduledTaskQueue == null || scheduledTaskQueue.isEmpty()) {
        return true; // 如果计划任务队列为空，返回 true
    }
    long nanoTime = getCurrentTimeNanos(); // 获取当前时间
    for (;;) {
        Runnable scheduledTask = pollScheduledTask(nanoTime); // 获取下一个计划任务
        if (scheduledTask == null) {
            return true; // 如果没有更多计划任务，返回 true
        }
        if (!taskQueue.offer(scheduledTask)) {
            // 如果任务队列没有空间，将任务添加回计划任务队列
            scheduledTaskQueue.add((ScheduledFutureTask<?>) scheduledTask);
            return false; // 返回 false，表示没有获取到所有任务
        }
    }
}
```

1. **检查计划任务队列**：
   - `if (scheduledTaskQueue == null || scheduledTaskQueue.isEmpty())`：检查计划任务队列是否为空。如果为空，返回 `true`。

2. **获取当前时间**：
   - `long nanoTime = getCurrentTimeNanos();`：获取当前时间的纳秒值。

3. **循环获取计划任务**：
   - `for (;;) { ... }`：使用无限循环获取计划任务。
   - `Runnable scheduledTask = pollScheduledTask(nanoTime);`：调用 `pollScheduledTask(nanoTime)` 方法获取下一个计划任务。

4. **检查任务是否为空**：
   - `if (scheduledTask == null)`：如果没有更多计划任务，返回 `true`。

5. **添加任务到队列**：
   - `if (!taskQueue.offer(scheduledTask))`：如果任务队列没有空间，将任务添加回计划任务队列，并返回 `false`。

#### `pollScheduledTask(long nanoTime)`

```java
protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop(); // 确保在事件循环中

    ScheduledFutureTask<?> scheduledTask = peekScheduledTask(); // 获取下一个计划任务
    if (scheduledTask == null || scheduledTask.deadlineNanos() - nanoTime > 0) {
        return null; // 如果没有计划任务或任务未到期，返回 null
    }
    scheduledTaskQueue.remove(); // 从队列中移除任务
    scheduledTask.setConsumed(); // 标记任务为已消费
    return scheduledTask; // 返回计划任务
}
```

1. **检查事件循环**：
   - `assert inEventLoop();`：确保当前线程在事件循环中执行。

2. **获取下一个计划任务**：
   - `ScheduledFutureTask<?> scheduledTask = peekScheduledTask();`：调用 `peekScheduledTask()` 方法获取下一个计划任务。

3. **检查任务有效性**：
   - `if (scheduledTask == null || scheduledTask.deadlineNanos() - nanoTime > 0)`：如果没有计划任务或任务未到期，返回 `null`。

4. **移除任务**：
   - `scheduledTaskQueue.remove();`：从队列中移除任务。

5. **标记任务**：
   - `scheduledTask.setConsumed();`：标记任务为已消费。

6. **返回任务**：
   - 返回获取到的计划任务。

#### `peekScheduledTask()`

```java
final ScheduledFutureTask<?> peekScheduledTask() {
    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    return scheduledTaskQueue != null ? scheduledTaskQueue.peek() : null; // 返回队列的下一个任务
}
```

1. **获取计划任务队列**：
   - `Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;`：获取当前的计划任务队列。

2. **返回下一个任务**：
   - `return scheduledTaskQueue != null ? scheduledTaskQueue.peek() : null;`：返回队列的下一个任务，如果队列为空，返回 `null`。

#### `runAllTasksFrom(Queue<Runnable> taskQueue)`

```java
protected final boolean runAllTasksFrom(Queue<Runnable> taskQueue) {
    Runnable task = pollTaskFrom(taskQueue); // 从任务队列中获取任务
    if (task == null) {
        return false; // 如果没有任务，返回 false
    }
    for (;;) {
        safeExecute(task); // 安全地执行任务
        task = pollTaskFrom(taskQueue); // 获取下一个任务
        if (task == null) {
            return true; // 如果没有更多任务，返回 true
        }
    }
}
```

1. **获取任务**：
   - `Runnable task = pollTaskFrom(taskQueue);`：从任务队列中获取任务。

2. **检查任务是否为空**：
   - `if (task == null)`：如果没有任务，返回 `false`。

3. **循环执行任务**：
   - `for (;;) { ... }`：使用无限循环执行任务。
   - `safeExecute(task);`：安全地执行当前任务。
   - `task = pollTaskFrom(taskQueue);`：获取下一个任务。

4. **返回值**：
   - `if (task == null) { return true; }`：如果没有更多任务，返回 `true`。

####  `runAllTasks(long timeoutNanos)`

```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue(); // 获取计划任务
    Runnable task = pollTask(); // 从任务队列中获取任务
    if (task == null) {
        afterRunningAllTasks(); // 如果没有任务，执行后处理
        return false; // 返回 false
    }

    final long deadline = timeoutNanos > 0 ? getCurrentTimeNanos() + timeoutNanos : 0; // 计算截止时间
    long runTasks = 0; // 记录运行的任务数量
    long lastExecutionTime;
    for (;;) {
        safeExecute(task); // 执行任务

        runTasks++; // 增加运行任务的计数

        // 每 64 个任务检查一次超时
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = getCurrentTimeNanos(); // 获取当前时间
            if (lastExecutionTime >= deadline) {
                break; // 如果超时，退出循环
            }
        }

        task = pollTask(); // 获取下一个任务
        if (task == null) {
            lastExecutionTime = getCurrentTimeNanos(); // 如果没有更多任务，更新最后执行时间
            break; // 退出循环
        }
    }

    afterRunningAllTasks(); // 执行后处理
    this.lastExecutionTime = lastExecutionTime; // 更新最后执行时间
    return true; // 返回 true，表示至少运行了一个任务
}
```

1. **获取计划任务**：
   - `fetchFromScheduledTaskQueue();`：调用该方法获取计划任务。

2. **获取任务**：
   - `Runnable task = pollTask();`：从任务队列中获取任务。

3. **检查任务是否为空**：
   - `if (task == null)`：如果没有任务，调用 `afterRunningAllTasks()` 执行后处理，并返回 `false`。

4. **计算截止时间**：
   - `final long deadline = timeoutNanos > 0 ? getCurrentTimeNanos() + timeoutNanos : 0;`：计算任务的截止时间。

5. **循环执行任务**：
   - `for (;;) { ... }`：使用无限循环执行任务。
   - `safeExecute(task);`：安全地执行当前任务。
   - `runTasks++;`：增加运行任务的计数。

6. **检查超时**：
   - `if ((runTasks & 0x3F) == 0) { ... }`：每 64 个任务检查一次超时。
   - `if (lastExecutionTime >= deadline) { break; }`：如果当前时间超过截止时间，退出循环。

7. **获取下一个任务**：
   - `task = pollTask();`：获取下一个任务。
   - `if (task == null) { ... }`：如果没有更多任务，更新最后执行时间并退出循环。

8. **后处理**：
   - `afterRunningAllTasks();`：执行所有任务后的处理。

9. **返回值**：
   - `return true;`：返回 `true`，表示至少运行了一个任务。

通过逐步分析 `runAllTasks()` 和 `runAllTasks(long timeoutNanos)` 方法中的调用流程，我们可以看到它们如何处理 I/O 操作。`runAllTasks()` 方法从计划任务队列和任务队列中获取任务并执行，而 `runAllTasks(long timeoutNanos)` 方法则在执行任务时考虑了超时限制。这些方法共同确保了事件循环能够高效地管理任务的执行，及时响应 I/O 事件。