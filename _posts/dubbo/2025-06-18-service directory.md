---
title: "3. Dubbo 3.x 服务目录源码分析"
date: 2025-06-18 12:00:00 +0800
categories: [从零开始的Dubbo源码阅读]
tags: [dubbo, 源码阅读, dubbo3]
---
# 服务目录

> 参考链接：[dubbo2.6源码分析](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/directory/)

## 1. 简介

本篇文章将开始分析 Dubbo 集群容错方面的源码。集群容错源码包含四个关键部分：
- 服务目录 Directory
- 服务路由 Router 
- 集群 Cluster
- 负载均衡 LoadBalance

这几个部分在功能上相互协作，但源码逻辑相对独立，因此我们将分四篇文章进行详细分析。

本篇作为集群容错的开篇，我们首先来理解服务目录的概念和实现。服务目录可以看作是服务提供者元数据的本地视图，它存储了服务提供者的关键信息，如IP、端口、服务协议等。通过这些信息，服务消费者可以建立远程连接并进行服务调用。

在分布式环境中，服务提供者是动态变化的：
- 新的服务提供者可能随时加入集群
- 现有的服务提供者可能下线或配置发生变更

服务目录负责动态响应这些变化，并实时更新本地视图。如果这样看，服务目录与注册中心似乎功能重叠？这种理解有一定道理，但还需进一步澄清：服务目录在获取注册中心的配置信息后，会为每条配置信息生成一个 `Invoker` 对象，这个 `Invoker` 才是服务目录最终持有的实体。

`Invoker` 是什么？顾名思义，它是一个具有远程调用能力的对象，是Dubbo中最核心的模型之一。因此，服务目录本质上是一个动态调整的 `Invoker` 集合，它会根据注册中心的变化实时更新内部的 `Invoker` 列表。

接下来，我们通过继承体系了解服务目录的设计与实现。

## 2. 继承体系

Dubbo 3.x 中的服务目录实现主要有两种：`StaticDirectory` 和 `RegistryDirectory`，它们都继承自 `AbstractDirectory`。在 Dubbo 3.x 中，还引入了 `DynamicDirectory` 作为动态目录的抽象基类，`RegistryDirectory` 继承自 `DynamicDirectory`。

`AbstractDirectory` 实现了 `Directory` 接口，其中包含一个核心方法 `list(Invocation invocation)`，用于根据调用信息列举可用的 `Invoker` 实例。下面是服务目录的继承体系图：

![服务目录继承体系图](/assets/dubbo/directory-inheritance.png)

从继承体系中我们可以看到:
- `Directory` 接口继承自 `Node` 接口，而 `Node` 接口被多种组件继承，如 Registry、Monitor、Invoker 等
- `Node` 接口提供了获取配置信息的 `getUrl()` 方法，使实现类能够对外提供统一的配置访问方式
- `RegistryDirectory` 实现了 `NotifyListener` 接口，使其能够监听注册中心的变更通知，并据此动态调整内部的 Invoker 列表

相比Dubbo 2.6，Dubbo 3.x引入了 `DynamicDirectory` 作为中间层，增强了设计的灵活性和可扩展性。

## 3. 源码分析

本章将分析 `AbstractDirectory` 及其子类的源码实现。我们先从 `AbstractDirectory` 开始，它封装了 Invoker 列举的框架流程，采用典型的模板方法模式，将具体的列举逻辑交由子类实现。

### 3.1 AbstractDirectory

首先看 `AbstractDirectory` 的 `list` 方法，这是服务目录的核心方法：

```java
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("/* ... */");
    }

    // 获取可用的Invoker列表
    BitList<Invoker<T>> availableInvokers;
    SingleRouterChain<T> singleChain = null;
    try {
        // 加锁避免并发修改
        try {
            if (routerChain != null) {
                routerChain.getLock().readLock().lock();
            }
            // 克隆Invoker列表避免在doList过程中被修改
            if (invokersInitialized) {
                // 使用有效的Invoker列表（已经过可用性检查）
                availableInvokers = validInvokers.clone();
            } else {
                // 使用所有Invoker列表
                availableInvokers = invokers.clone();
            }

            if (routerChain != null) {
                // 获取用于处理当前调用的路由链
                singleChain = routerChain.getSingleChain(getConsumerUrl(), availableInvokers, invocation);
                singleChain.getLock().readLock().lock();
            }
        } finally {
            if (routerChain != null) {
                routerChain.getLock().readLock().unlock();
            }
        }
        
        // 调用子类实现的doList方法进行路由
        List<Invoker<T>> routedResult = doList(singleChain, availableInvokers, invocation);
        
        // 处理路由结果为空的情况
        if (routedResult.isEmpty()) {
            // 2-2 - No provider available.
            logger.warn("/* ... */");
        }
        
        // 返回不可变列表，防止外部修改
        return Collections.unmodifiableList(routedResult);
    } finally {
        // 释放路由链的读锁
        if (singleChain != null) {
            singleChain.getLock().readLock().unlock();
        }
    }
}
```

`AbstractDirectory.list` 方法的核心流程如下：
1. 首先进行销毁状态检查，如果目录已销毁则抛出异常
2. 获取可用的 Invoker 列表，根据初始化状态决定使用 validInvokers 还是所有 invokers
3. 获取针对当前调用的路由链 SingleRouterChain
4. 调用子类实现的 doList 方法进行实际的路由过滤
5. 处理路由结果，并返回不可变的结果列表

通过模板方法模式，AbstractDirectory 定义了统一的框架流程，而具体的路由逻辑则由子类实现。这种设计使得代码结构清晰，易于维护和扩展。

在Dubbo 3.x中，AbstractDirectory引入了 BitList 数据结构和更完善的锁机制，提高了并发性能和安全性。接下来我们将分析其子类的实现。

### 3.2 StaticDirectory

`StaticDirectory` 是一种静态服务目录，它内部存放的 Invoker 列表在创建后通常不会发生变动（除非显式调用 `notify` 方法更新）。从名称可以看出，它是一个"静态"的目录实现，类似于一个不可变的列表。下面我们分析它的核心实现：

```java
public class StaticDirectory<T> extends AbstractDirectory<T> {

    private Class<T> interfaceClass;

    // 多种构造函数实现，支持不同参数组合
    public StaticDirectory(List<Invoker<T>> invokers) {
        this(null, invokers, null);
    }

    public StaticDirectory(List<Invoker<T>> invokers, RouterChain<T> routerChain) {
        this(null, invokers, routerChain);
    }

    public StaticDirectory(URL url, List<Invoker<T>> invokers) {
        this(url, invokers, null);
    }

    /**
     * 主构造函数，初始化StaticDirectory
     * 
     * @param url 服务URL，如果为null且invokers不为空，则使用第一个invoker的URL
     * @param invokers Invoker列表
     * @param routerChain 路由链
     */
    public StaticDirectory(URL url, List<Invoker<T>> invokers, RouterChain<T> routerChain) {
        // 调用父类构造函数，初始化URL和路由链
        super(url == null && CollectionUtils.isNotEmpty(invokers) ? invokers.get(0).getUrl() : url, routerChain, false);
        
        // 校验invokers非空
        if (CollectionUtils.isEmpty(invokers)) {
            throw new IllegalArgumentException("invokers == null");
        }
        
        // 设置invokers列表，使用BitList增强性能
        this.setInvokers(new BitList<>(invokers));
        
        // 获取接口类型
        this.interfaceClass = invokers.get(0).getInterface();
    }

    @Override
    public Class<T> getInterface() {
        return interfaceClass;
    }

    @Override
    public List<Invoker<T>> getAllInvokers() {
        return getInvokers();
    }

    /**
     * 检查目录是否可用
     * 只要有一个Invoker可用，整个目录就被视为可用
     */
    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        
        // 遍历有效的Invoker列表，检查可用性
        for (Invoker<T> invoker : getValidInvokers()) {
            if (invoker.isAvailable()) {
                return true;
            } else {
                // 将不可用的Invoker添加到无效列表
                addInvalidateInvoker(invoker);
            }
        }
        return false;
    }

    /**
     * 销毁目录及其中的所有Invoker
     */
    @Override
    public void destroy() {
        if (isDestroyed()) {
            return;
        }
        
        // 销毁所有Invoker
        for (Invoker<T> invoker : getInvokers()) {
            invoker.destroy();
        }
        
        // 调用父类销毁方法
        super.destroy();
    }

    /**
     * 构建路由链
     * Dubbo 3.x新增功能，用于支持更强大的路由能力
     */
    public void buildRouterChain() {
        RouterChain<T> routerChain = RouterChain.buildChain(getInterface(), getUrl());
        routerChain.setInvokers(getInvokers(), () -> {});
        this.setRouterChain(routerChain);
    }

    /**
     * 更新Invoker列表
     * 虽然是StaticDirectory，但仍支持通过此方法动态更新
     */
    public void notify(List<Invoker<T>> invokers) {
        BitList<Invoker<T>> bitList = new BitList<>(invokers);
        if (routerChain != null) {
            // 有路由链时，刷新路由器和Invoker列表
            refreshRouter(bitList.clone(), () -> this.setInvokers(bitList));
        } else {
            // 无路由链时，直接更新Invoker列表
            this.setInvokers(bitList);
        }
    }

    /**
     * 实现AbstractDirectory的抽象方法
     * 使用路由链对Invoker列表进行过滤
     */
    @Override
    protected List<Invoker<T>> doList(SingleRouterChain<T> singleRouterChain, 
                                      BitList<Invoker<T>> invokers, 
                                      Invocation invocation) throws RpcException {
        if (singleRouterChain != null) {
            try {
                // 通过路由链进行路由
                List<Invoker<T>> finalInvokers = singleRouterChain.route(getConsumerUrl(), invokers, invocation);
                return finalInvokers == null ? BitList.emptyList() : finalInvokers;
            } catch (Throwable t) {
                logger.error("/* ... */");
                return BitList.emptyList();
            }
        }
        // 没有路由链时直接返回所有Invoker
        return invokers;
    }

    /**
     * 获取目录的元数据信息
     * Dubbo 3.x新增功能，用于增强可观测性
     */
    @Override
    protected Map<String, String> getDirectoryMeta() {
        Map<String, String> metas = new HashMap<>();
        metas.put(REGISTRY_KEY, "static");
        metas.put(REGISTER_MODE_KEY, "static");
        return metas;
    }
}
```

从上述代码中，我们可以看出 `StaticDirectory` 的特点：

1. **简单性**：作为静态目录，其逻辑相对简单，主要是在创建时设置 Invoker 列表，之后不主动更新
2. **可用性检查**：通过检查内部任一 Invoker 的可用性来确定整个目录是否可用
3. **路由支持**：虽然是静态目录，但仍然支持路由功能，可以根据调用信息过滤 Invoker
4. **可观测性**：Dubbo 3.x 中增加了元数据支持，便于监控和排查问题

`StaticDirectory` 在以下场景中特别有用：
- 服务消费者直连服务提供者时（点对点调用）
- 服务合并场景中，每个服务分组内部使用静态目录
- 集群容错策略中，如 Failover、Failfast 等需要操作固定 Invoker 列表的场景

在 Dubbo 3.x 中，`StaticDirectory` 相比 2.6 版本的主要变化：
- 引入 `BitList` 替换普通 List，提升性能
- 增强了路由相关功能，引入 `RouterChain` 和 `SingleRouterChain`
- 将 `validInvokers` 变量移至父类 `AbstractDirectory`，优化了代码结构
- 添加了元数据支持，增强可观测性

接下来我们将分析更为复杂的 `RegistryDirectory` 实现。

### 3.3 RegistryDirectory

`RegistryDirectory` 是一种动态服务目录，它实现了 `NotifyListener` 接口，能够监听注册中心的变更通知。当注册中心的服务配置发生变化时，`RegistryDirectory` 可以收到相关变更并据此刷新本地的 Invoker 列表。

在 Dubbo 3.x 中，`RegistryDirectory` 继承自 `DynamicDirectory`，而非直接继承 `AbstractDirectory`，增加了一层抽象，使设计更加合理。`RegistryDirectory` 中有几个关键逻辑：

1. Invoker 的列举逻辑（路由过滤）
2. 接收服务配置变更的通知逻辑
3. Invoker 列表的刷新逻辑

接下来我们按顺序对这三块核心逻辑进行分析。

#### 3.3.1 列举 Invoker

Invoker 列举逻辑主要封装在父类 `DynamicDirectory` 的 `doList` 方法中，该方法是 `AbstractDirectory.list` 方法的回调实现：

```java
public List<Invoker<T>> doList(SingleRouterChain<T> singleRouterChain,
                              BitList<Invoker<T>> invokers, 
                              Invocation invocation) {
    // 检查是否禁止访问且需要快速失败
    if (forbidden && shouldFailFast) {
        // 服务被禁用（如提供者关闭或禁用了服务），抛出"No provider"异常
        throw new RpcException("/* ... */");
    }

    // 如果是多组服务，直接返回所有Invoker
    if (multiGroup) {
        return this.getInvokers();
    }

    try {
        // 使用SingleRouterChain进行路由过滤
        List<Invoker<T>> result = singleRouterChain.route(getConsumerUrl(), invokers, invocation);
        return result == null ? BitList.emptyList() : result;
    } catch (Throwable t) {
        // 路由过程发生异常，记录日志并返回空列表
        logger.error("/* ... */");
        return BitList.emptyList();
    }
}
```

可以看出，`DynamicDirectory.doList` 方法的逻辑相对简单：
1. 首先检查服务是否被禁止访问，如果禁止且配置了快速失败，则直接抛出异常
2. 对于多组服务场景，直接返回所有 Invoker（多组合并已在其他地方处理）
3. 使用路由链 `SingleRouterChain` 进行实际的路由过滤
4. 处理异常情况，确保即使路由过程出错也有合理的返回值

路由的核心逻辑位于 `SingleRouterChain.route` 方法：

```java
public List<Invoker<T>> route(URL url, BitList<Invoker<T>> availableInvokers, Invocation invocation) {
    // 校验invokers一致性，防止并发修改导致的数据不一致
    if (invokers.getOriginList() != availableInvokers.getOriginList()) {
        logger.error("/* ... */");
        throw new IllegalStateException("/* ... */");
    }
    
    // 根据运行环境决定是否需要打印路由快照（用于调试和问题排查）
    if (RpcContext.getServiceContext().isNeedPrintRouterSnapshot()) {
        return routeAndPrint(url, availableInvokers, invocation);
    } else {
        return simpleRoute(url, availableInvokers, invocation);
    }
}
```

这个方法首先验证传入的 Invoker 列表是否一致，然后根据上下文决定使用普通路由还是带快照记录的路由。接下来我们看一下这两种路由的实现。

##### 3.3.1.1 simpleRoute - 普通路由实现

```java
public List<Invoker<T>> simpleRoute(URL url, BitList<Invoker<T>> availableInvokers, Invocation invocation) {
    // 克隆一份Invoker列表，避免修改原始数据
    BitList<Invoker<T>> resultInvokers = availableInvokers.clone();

    // 1. 先通过 headStateRouter 进行第一轮路由过滤
    // headStateRouter 通常包含状态路由器，如地址状态路由、标签路由等
    resultInvokers = headStateRouter.route(resultInvokers, url, invocation, false, null);
    
    // 如果过滤后为空且需要快速失败或没有普通路由器，则打印路由快照并返回空列表
    if (resultInvokers.isEmpty() && (shouldFailFast || routers.isEmpty())) {
        printRouterSnapshot(url, availableInvokers, invocation);
        return BitList.emptyList();
    }
    
    // 如果没有普通路由器，直接返回当前结果
    if (routers.isEmpty()) {
        return resultInvokers;
    }
    
    // 将BitList转为ArrayList，用于后续普通路由器处理
    List<Invoker<T>> commonRouterResult = resultInvokers.cloneToArrayList();
    
    // 2. 依次通过所有普通路由器进行过滤
    // 普通路由器可能包括脚本路由、条件路由等
    for (Router router : routers) {
        RouterResult<Invoker<T>> routeResult = router.route(commonRouterResult, url, invocation, false);
        commonRouterResult = routeResult.getResult();
        
        // 检查是否需要快速失败
        if (CollectionUtils.isEmpty(commonRouterResult) && shouldFailFast) {
            printRouterSnapshot(url, availableInvokers, invocation);
            return BitList.emptyList();
        }

        // 检查是否需要继续路由
        if (!routeResult.isNeedContinueRoute()) {
            return commonRouterResult;
        }
    }
    
    // 最终检查，如果结果为空则打印快照
    if (commonRouterResult.isEmpty()) {
        printRouterSnapshot(url, availableInvokers, invocation);
        return BitList.emptyList();
    }

    return commonRouterResult;
}
```

`simpleRoute` 方法是路由链的核心实现，它分两个阶段进行路由：
1. **状态路由阶段**：首先通过 `headStateRouter`（通常是状态路由器）进行第一轮过滤
2. **普通路由阶段**：然后依次通过所有普通路由器进行进一步过滤

在每个阶段，都会检查路由结果是否为空，以及是否需要快速失败或继续路由。这种分层设计使得路由过程更加清晰和高效。

状态路由器主要是 `AbstractStateRouter` 的实现类，我们来看一下其路由方法：

```java
public final BitList<Invoker<T>> route(
    BitList<Invoker<T>> invokers,
    URL url,
    Invocation invocation,
    boolean needToPrintMessage,
    Holder<RouterSnapshotNode<T>> nodeHolder) throws RpcException {
    
    // 处理快照相关逻辑
    if (needToPrintMessage && (nodeHolder == null || nodeHolder.get() == null)) {
        needToPrintMessage = false;
    }

    RouterSnapshotNode<T> currentNode = null;
    RouterSnapshotNode<T> parentNode = null;
    Holder<String> messageHolder = null;

    // 如果需要打印路由快照，则构建当前节点的快照信息
    if (needToPrintMessage) {
        parentNode = nodeHolder.get();
        currentNode = new RouterSnapshotNode<>(this.getClass().getSimpleName(), invokers.clone());
        parentNode.appendNode(currentNode);

        if (parentNode.getNodeOutputSize() < invokers.size()) {
            parentNode.setNodeOutputInvokers(invokers.clone());
        }

        messageHolder = new Holder<>();
        nodeHolder.set(currentNode);
    }
    
    // 调用子类实现的doRoute方法进行实际路由
    BitList<Invoker<T>> routeResult = doRoute(invokers, url, invocation, 
                                             needToPrintMessage, nodeHolder, messageHolder);
    
    // 确保结果安全，取原始列表和路由结果的交集
    if (routeResult != invokers) {
        routeResult = invokers.and(routeResult);
    }
    
    // 如果当前路由器不支持自主决定是否继续路由，则根据配置进行判断
    if (!supportContinueRoute()) {
        if (!shouldFailFast || !routeResult.isEmpty()) {
            routeResult = continueRoute(routeResult, url, invocation, needToPrintMessage, nodeHolder);
        }
    }

    // 更新快照信息
    if (needToPrintMessage) {
        currentNode.setRouterMessage(messageHolder.get());
        if (currentNode.getNodeOutputSize() == 0) {
            // 没有子调用，设置当前节点的输出
            currentNode.setNodeOutputInvokers(routeResult.clone());
        }
        currentNode.setChainOutputInvokers(routeResult.clone());
        nodeHolder.set(parentNode);
    }
    
    return routeResult;
}
```

`AbstractStateRouter.route` 方法采用模板方法模式，通过调用子类的 `doRoute` 方法实现具体的路由逻辑。它还处理了路由快照的构建、结果安全检查和路由链继续路由的逻辑。

对于普通路由器，则使用 `Router` 接口的 `route` 方法：

```java
default <T> RouterResult<Invoker<T>> route(
    List<Invoker<T>> invokers, 
    URL url, 
    Invocation invocation, 
    boolean needToPrintMessage) throws RpcException {
    // 调用基本路由方法，返回RouterResult包装结果
    return new RouterResult<>(route(invokers, url, invocation));
}

// 基本路由方法，处理兼容性
default <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
    // 将新版Invoker转换为兼容2.6.x的Invoker
    List<com.alibaba.dubbo.rpc.Invoker<T>> invs = invokers.stream()
        .map(invoker -> new com.alibaba.dubbo.rpc.Invoker.CompatibleInvoker<T>(invoker))
        .collect(Collectors.toList());

    // 调用2.6.x兼容版本的route方法
    List<com.alibaba.dubbo.rpc.Invoker<T>> res = this.route(
        invs,
        new com.alibaba.dubbo.common.DelegateURL(url),
        new com.alibaba.dubbo.rpc.Invocation.CompatibleInvocation(invocation));

    // 将结果转回新版Invoker
    return res.stream()
        .map(inv -> inv.getOriginal())
        .filter(Objects::nonNull)
        .collect(Collectors.toList());
}
```

这段代码主要是处理与旧版本 Dubbo 的兼容性，将新版的 Invoker 转换为兼容旧版本的 Invoker，调用旧版本的路由方法，然后再将结果转换回来。

##### 3.3.1.2 routeAndPrint - 带快照的路由实现

对于需要调试或问题诊断的场景，Dubbo 3.x 提供了带快照的路由实现：

```java
public List<Invoker<T>> routeAndPrint(URL url, BitList<Invoker<T>> availableInvokers, Invocation invocation) {
    // 构建路由过程的快照
    RouterSnapshotNode<T> snapshot = buildRouterSnapshot(url, availableInvokers, invocation);
    // 将快照信息输出到日志
    logRouterSnapshot(url, invocation, snapshot);
    // 返回经过所有路由器过滤后的最终Invoker列表
    return snapshot.getChainOutputInvokers();
}
```

`routeAndPrint` 方法通过 `buildRouterSnapshot` 构建路由过程的详细快照，记录每个路由器的输入输出和过滤原因，然后输出到日志中，便于问题排查。核心方法 `buildRouterSnapshot` 逻辑与 `simpleRoute` 类似，但增加了快照记录功能，这里不再详细展开。

总结来说，Dubbo 3.x 的路由机制设计非常灵活且强大，它将路由过程分为状态路由和普通路由两个阶段，并引入了路由快照功能，使得路由过程更加可观测和可调试。接下来我们将分析 `RegistryDirectory` 如何接收服务变更通知。

#### 3.3.2 接收服务变更通知

`RegistryDirectory` 作为动态服务目录，其核心特性是能够响应注册中心的变更通知，动态调整内部的 Invoker 列表。这一特性通过实现 `NotifyListener` 接口的 `notify` 方法实现：

```java
public synchronized void notify(List<URL> urls) {
    // 销毁保护：如果目录已销毁，则不处理通知
    if (isDestroyed()) {
        return;
    }
    
    // 第一步：URL分类
    // 将收到的URL按类型（配置器、路由器、提供者）分组
    Map<String, List<URL>> categoryUrls = urls.stream()
        .filter(Objects::nonNull)                     // 过滤空URL
        .filter(this::isValidCategory)                // 过滤无效类别
        .filter(this::isNotCompatibleFor26x)          // 过滤2.6.x兼容URL
        .collect(Collectors.groupingBy(this::judgeCategory));  // 按类别分组
    
    // 第二步：处理配置和路由
    // 检查是否启用了2.6.x配置监听
    if (moduleModel
        .modelEnvironment()
        .getConfiguration()
        .convert(Boolean.class, ENABLE_26X_CONFIGURATION_LISTEN, true)) {
        
        // 处理配置器URL，更新配置
        List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
        this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

        // 处理路由器URL，更新路由规则
        List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
        toRouters(routerURLs).ifPresent(this::addRouters);
    }

    // 第三步：处理服务提供者URL
    List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());

    // 第四步：地址监听扩展处理
    // Dubbo 3.x新增的地址监听扩展点，允许对提供者列表进行二次处理
    ExtensionLoader<AddressListener> addressListenerExtensionLoader =
        getUrl().getOrDefaultModuleModel().getExtensionLoader(AddressListener.class);
    List<AddressListener> supportedListeners =
        addressListenerExtensionLoader.getActivateExtension(getUrl(), (String[]) null);
    
    if (supportedListeners != null && !supportedListeners.isEmpty()) {
        for (AddressListener addressListener : supportedListeners) {
            // 通过监听器处理提供者URL，可能会过滤或转换URL
            providerURLs = addressListener.notify(providerURLs, getConsumerUrl(), this);
        }
    }
    
    // 第五步：刷新覆盖参数和Invoker列表
    refreshOverrideAndInvoker(providerURLs);
}
```

`notify` 方法的处理流程清晰明了：

1. **URL分类**：将收到的URL按照类别（配置器、路由器、提供者）进行分组
2. **处理配置和路由**：更新配置器和路由器
3. **处理服务提供者**：获取提供者URL列表
4. **地址监听扩展处理**：通过扩展点机制，允许对提供者列表进行二次处理
5. **刷新Invoker**：根据最新的提供者列表刷新本地Invoker缓存

相比Dubbo 2.6，Dubbo 3.x的`notify`方法引入了几个重要改进：

- 使用Java 8 Stream API优化了URL分类逻辑，代码更简洁
- 增加了地址监听扩展点，提供了更灵活的地址处理机制
- 改进了兼容性处理，更好地支持从2.6.x升级到3.x
- 使用模块化模型(moduleModel)，支持多应用实例部署

`toRouters`和`toConfigurators`方法负责将URL转换为Router和Configurator对象，这两个方法主要是类型转换和扩展点加载，逻辑相对简单。下面我们重点看一下`refreshOverrideAndInvoker`方法，它是实现Invoker列表动态刷新的关键。

```java
protected synchronized void refreshOverrideAndInvoker(List<URL> urls) {
    // 第一步：应用配置覆盖
    // 将消费者URL与配置器应用，生成新的directoryUrl
    this.directoryUrl = overrideWithConfigurator(getOriginalConsumerUrl());
    // 第二步：刷新Invoker列表
    refreshInvoker(urls);
}

protected URL overrideWithConfigurator(URL providerUrl) {
    // 1. 处理dubbo 2.6及之前版本的全局配置器
    // 主要用于兼容旧版本的override://协议配置
    providerUrl = overrideWithConfigurators(this.configurators, providerUrl);
    // 2. 应用级别配置器处理
    // 从应用级配置监听器获取配置，应用到所有服务
    providerUrl = overrideWithConfigurators(consumerConfigurationListener.getConfigurators(), providerUrl);
    // 3. 服务级别配置器处理
    // 应用特定服务的配置，优先级最高
    if (referenceConfigurationListener != null) {
        providerUrl = overrideWithConfigurators(referenceConfigurationListener.getConfigurators(), providerUrl);
    }
    return providerUrl;
}
```

`refreshOverrideAndInvoker`方法分两步：
1. 先应用各级配置覆盖，更新`directoryUrl`
2. 然后刷新Invoker列表

`overrideWithConfigurator`方法体现了Dubbo 3.x配置的层次结构：
- 全局配置（2.6.x兼容）
- 应用级配置
- 服务级配置

这种层次化的配置设计使得Dubbo 3.x能够更灵活地管理服务配置，支持从全局到局部的精细化配置控制。接下来我们将深入分析Invoker列表的刷新逻辑。

#### 3.3.3 刷新 Invoker 列表

`refreshInvoker`方法是`RegistryDirectory`最核心的方法之一，它确保服务目录能够响应注册中心的变化，动态调整内部的Invoker列表。这个方法的逻辑相对复杂，我们将其逐步分解：

```java
private void refreshInvoker(List<URL> invokerUrls) {
    Assert.notNull(invokerUrls, "invokerUrls should not be null");
    // 场景一：服务下线或禁用（使用empty://协议）
    if (invokerUrls.size() == 1
        && invokerUrls.get(0) != null
        && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        // 设置路由规则拒绝所有请求，forbidden标记服务不可用
        refreshRouter(BitList.emptyList(), () -> this.forbidden = true);
        // 销毁所有现有的Invoker实例，释放资源
        destroyAllInvokers(); 
    } else {
        // 场景二：正常服务可用情况
        // 允许访问服务
        this.forbidden = false;
        // 处理空地址列表情况
        if (invokerUrls == Collections.<URL>emptyList()) {
            invokerUrls = new ArrayList<>();
        }
        // 使用本地引用避免并发问题
        Set<URL> localCachedInvokerUrls = this.cachedInvokerUrls;
        // 空地址保护：如果当前地址为空但有缓存地址，使用缓存地址
        if (invokerUrls.isEmpty()) {
            if (CollectionUtils.isNotEmpty(localCachedInvokerUrls)) {
                logger.warn("Empty address received, will use cached address: " + localCachedInvokerUrls);
                invokerUrls.addAll(localCachedInvokerUrls);
            }
        } else {
            // 更新缓存的地址列表
            localCachedInvokerUrls = new HashSet<>();
            localCachedInvokerUrls.addAll(invokerUrls);
            this.cachedInvokerUrls = localCachedInvokerUrls;
        }
        // 如果地址列表仍为空，直接返回
        if (invokerUrls.isEmpty()) {
            return;
        }
        // 使用本地引用避免NPE，因为urlInvokerMap可能在destroyAllInvokers()时被设为null
        Map<URL, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap;
        // 保存旧的Invoker映射，用于后续比较和清理
        Map<URL, Invoker<T>> oldUrlInvokerMap = null;
        if (localUrlInvokerMap != null) {
            // 创建合适大小的映射，避免扩容
            oldUrlInvokerMap = new LinkedHashMap<>(
                Math.round(1 + localUrlInvokerMap.size() / DEFAULT_HASHMAP_LOAD_FACTOR));
            // 复制旧的Invoker映射
            localUrlInvokerMap.forEach(oldUrlInvokerMap::put);
        }
        // 关键步骤：将URL列表转换为Invoker映射
        Map<URL, Invoker<T>> newUrlInvokerMap = toInvokers(oldUrlInvokerMap, invokerUrls);
        // 转换失败处理：可能是协议不匹配或注册中心数据异常
        if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
            logger.error("Failed to convert urls to invokers: " + invokerUrls);
            return;
        }
        // 获取所有新的Invoker列表
        List<Invoker<T>> newInvokers = Collections.unmodifiableList(
            new ArrayList<>(newUrlInvokerMap.values()));
        // 处理多组服务合并
        BitList<Invoker<T>> finalInvokers = multiGroup 
            ? new BitList<>(toMergeInvokerList(newInvokers)) 
            : new BitList<>(newInvokers);
        // 刷新路由器和更新最终的Invoker列表
        refreshRouter(finalInvokers.clone(), () -> this.setInvokers(finalInvokers));
        // 更新URL到Invoker的映射关系
        this.urlInvokerMap = newUrlInvokerMap;
        try {
            // 销毁不再使用的Invoker，释放资源
            destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap);
        } catch (Exception e) {
            logger.warn("/* ... */");
        }
        // 通知Invoker列表已变更
        this.invokersChanged();
    }
    // 记录地址变更事件
    logger.info("/* ... */");
}
```

`refreshInvoker`方法主要处理两种场景：
1. **服务下线或禁用**：收到empty://协议的URL，此时禁止服务访问并销毁所有Invoker
2. **正常服务可用**：构建新的Invoker映射，更新服务目录

这个方法的核心步骤包括：
- 空地址保护：防止因注册中心临时问题导致服务不可用
- URL到Invoker的转换：通过`toInvokers`方法实现
- 多组服务合并：支持跨组调用
- 销毁无用Invoker：及时释放资源，避免内存泄漏

接下来，我们分析几个关键的辅助方法。

##### 3.3.3.1 URL到Invoker的转换

`toInvokers`方法负责将URL列表转换为Invoker映射：

```java
/**
 * 将服务提供者URL列表转换为Invoker实例映射
 * 
 * Dubbo 3.x优化点：
 * - 使用ConcurrentHashMap提高并发性能
 * - 优化初始容量计算减少扩容
 * - 增强协议兼容性处理
 * 
 * @param oldUrlInvokerMap 旧的URL到Invoker映射，用于复用
 * @param urls 服务提供者URL列表
 * @return 新的URL到Invoker映射
 */
private Map<URL, Invoker<T>> toInvokers(Map<URL, Invoker<T>> oldUrlInvokerMap, List<URL> urls) {
    // 初始化新的Invoker映射，设置合理初始容量
    Map<URL, Invoker<T>> newUrlInvokerMap = new ConcurrentHashMap<>(
        urls == null ? 1 : (int) (urls.size() / 0.75f + 1));
    if (urls == null || urls.isEmpty()) {
        return newUrlInvokerMap;
    }
    // 获取消费者配置的协议列表
    String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
    // 遍历所有服务提供者URL
    for (URL providerUrl : urls) {
        // 过滤不支持的协议
        if (!checkProtocolValid(queryProtocols, providerUrl)) {
            continue;
        }
        // 合并URL参数（消费者配置覆盖提供者配置）
        URL url = mergeUrl(providerUrl);
        // 确定有效的协议
        String effectiveProtocol = getEffectiveProtocol(queryProtocols, url);
        if (!effectiveProtocol.equals(url.getProtocol())) {
            url = url.setProtocol(effectiveProtocol);
        }
        // 尝试复用已存在的Invoker
        Invoker<T> invoker = oldUrlInvokerMap == null ? null : oldUrlInvokerMap.remove(url);
        if (invoker == null) {
            // 创建新的Invoker
            try {
                // 检查服务是否启用
                boolean enabled = url.getParameter(ENABLED_KEY, true);
                if (enabled) {
                    // 通过协议扩展点创建Invoker实例
                    invoker = protocol.refer(serviceType, url);
                }
            } catch (Throwable t) {
                logger.error("/* ... */");
            }
            // 将新创建的Invoker添加到映射
            if (invoker != null) {
                newUrlInvokerMap.put(url, invoker);
            }
        } else {
            // 复用已存在的Invoker
            newUrlInvokerMap.put(url, invoker);
        }
    }
    return newUrlInvokerMap;
}
```

`toInvokers`方法主要完成以下工作：
1. 过滤不支持的协议和参数校验
2. 合并URL参数，应用消费者侧配置
3. 确定有效协议，处理协议兼容性
4. 优先复用已存在的Invoker，避免不必要的资源创建
5. 必要时创建新的Invoker实例

Dubbo 3.x在这一过程中引入了多项优化，如协议有效性检查、参数合并优化和Invoker复用机制的增强。

##### 3.3.3.2 多组服务合并

对于配置了多个服务分组的场景，Dubbo提供了合并调用的能力：

```java
/**
 * 合并不同分组的Invoker列表，支持多组服务调用
 * 
 * Dubbo 3.x优化：
 * - 使用computeIfAbsent简化代码
 * - 增强集群容错策略
 * - 优化静态目录的路由链构建
 * 
 * @param invokers 原始Invoker列表
 * @return 合并后的Invoker列表
 */
private List<Invoker<T>> toMergeInvokerList(List<Invoker<T>> invokers) {
    List<Invoker<T>> mergedInvokers = new ArrayList<>();
    
    // 按服务分组对Invoker进行分类
    Map<String, List<Invoker<T>>> groupMap = new HashMap<>();
    for (Invoker<T> invoker : invokers) {
        // 获取服务分组，默认为空字符串
        String group = invoker.getUrl().getGroup("");
        // 将Invoker添加到对应分组
        groupMap.computeIfAbsent(group, k -> new ArrayList<>()).add(invoker);
    }
    // 根据分组数量执行不同的合并策略
    if (groupMap.size() == 1) {
        // 只有一个分组，直接使用该组的Invoker列表
        mergedInvokers.addAll(groupMap.values().iterator().next());
    } else if (groupMap.size() > 1) {
        // 存在多个分组，为每个分组创建独立的集群
        for (List<Invoker<T>> groupList : groupMap.values()) {
            // 创建静态目录，包含该分组的所有Invoker
            StaticDirectory<T> staticDirectory = new StaticDirectory<>(groupList);
            
            // 构建路由链，支持组内路由
            staticDirectory.buildRouterChain();
            
            // 通过集群策略合并该分组的Invoker
            // 在Dubbo 3.x中，设置false表示不支持服务降级
            mergedInvokers.add(cluster.join(staticDirectory, false));
        }
    } else {
        // 没有分组信息，直接使用原始Invoker列表
        mergedInvokers = invokers;
    }
    return mergedInvokers;
}
```

多组服务合并的核心思路是：
1. 先按group对Invoker进行分组
2. 如果只有一个分组，直接使用该组的Invoker
3. 如果有多个分组，为每个分组创建一个StaticDirectory，并通过集群策略将其合并为一个Invoker
4. 合并后的多个组Invoker形成新的列表返回

Dubbo 3.x对此过程进行了优化，特别是在路由链构建和集群策略应用方面。

##### 3.3.3.3 销毁无用Invoker

当服务提供者下线或配置变更时，需要及时销毁不再使用的Invoker，释放资源：

```java
/**
 * 销毁不再使用的Invoker实例，释放系统资源
 * 
 * Dubbo 3.x优化：
 * - 使用CollectionUtils简化空检查
 * - 增强日志记录，便于问题诊断
 * - 优化异常处理
 * 
 * @param oldUrlInvokerMap 更新前的URL到Invoker映射
 * @param newUrlInvokerMap 更新后的URL到Invoker映射
 */
private void destroyUnusedInvokers(Map<URL, Invoker<T>> oldUrlInvokerMap, 
                                  Map<URL, Invoker<T>> newUrlInvokerMap) {
    // 新映射为空，销毁所有Invoker
    if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
        destroyAllInvokers();
        return;
    }
    // 旧映射为空，无需销毁
    if (CollectionUtils.isEmptyMap(oldUrlInvokerMap)) {
        return;
    }
    // 遍历旧映射，销毁不在新映射中的Invoker
    for (Map.Entry<URL, Invoker<T>> entry : oldUrlInvokerMap.entrySet()) {
        Invoker<T> invoker = entry.getValue();
        if (invoker != null) {
            try {
                // 调用destroy方法释放资源
                invoker.destroy();
                if (logger.isDebugEnabled()) {
                    logger.debug("/* ... */");
                }
            } catch (Exception e) {
                // 记录销毁失败信息
                logger.warn("/* ... */");
            }
        }
    }
    // 记录操作结果统计
    logger.info("/* ... */");
}
```

`destroyUnusedInvokers`方法主要做三件事：
1. 判断是否需要全量销毁或无需销毁
2. 遍历旧的Invoker映射，销毁不再使用的Invoker
3. 记录销毁统计信息，便于监控和问题排查

这一过程确保了资源的及时释放，避免了内存泄漏和连接泄漏等问题。

至此，我们完成了对`RegistryDirectory`核心逻辑的分析。它通过监听注册中心的变更通知，动态调整内部的Invoker列表，确保服务消费者能够准确地感知服务提供者的变化。

## 4. 总结

本篇文章对Dubbo 3.x的服务目录进行了全面分析，主要集中在以下几个方面：

1. **服务目录的概念和作用**：服务目录作为服务提供者元数据的本地视图，负责存储和管理Invoker对象，是服务调用的重要基础设施。

2. **继承体系**：分析了`Directory`接口及其实现类的继承关系，Dubbo 3.x中引入了`DynamicDirectory`作为中间层，优化了设计结构。

3. **StaticDirectory实现**：静态服务目录，提供了简单高效的Invoker管理，适用于点对点调用和多组服务合并等场景。

4. **RegistryDirectory实现**：动态服务目录，通过监听注册中心的变更通知，动态调整内部的Invoker列表，是Dubbo服务发现机制的核心实现。

5. **路由机制**：详细分析了Dubbo 3.x的路由链设计，包括状态路由和普通路由两个阶段，以及路由快照功能，使得路由过程更加可观测和可调试。

相比Dubbo 2.6，Dubbo 3.x在服务目录方面的主要改进包括：

- **性能优化**：引入`BitList`替代普通List，提升并发性能
- **设计优化**：新增`DynamicDirectory`抽象类，优化继承结构
- **路由增强**：重构路由链设计，增加路由快照功能
- **可观测性**：增强日志记录和元数据支持，便于问题诊断
- **扩展能力**：增加地址监听扩展点，提供更灵活的地址处理机制
- **兼容性**：提供与Dubbo 2.6.x的良好兼容性，降低升级成本

服务目录是Dubbo集群容错的重要基础，通过维护可用的服务提供者列表，为后续的路由筛选、负载均衡和实际调用提供支持。深入理解服务目录的工作原理，有助于我们更好地使用Dubbo进行分布式服务开发和运维。

下一篇文章，我们将继续分析Dubbo集群容错中的路由模块，探讨Dubbo如何根据各种规则和策略选择合适的服务提供者。

