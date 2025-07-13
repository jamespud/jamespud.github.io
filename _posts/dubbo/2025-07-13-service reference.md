---
title: "8. Dubbo3 服务引用源码分析"
date: 2025-07-13 18:50:00 +0800
categories: [从零开始的Dubbo源码阅读]
tags: [dubbo, 源码阅读]
---
# 服务引用

> 参考链接：[dubbo2.6源码分析](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/refer-service/)

## 1. 简介

上一篇文章详细分析了服务导出的过程，本篇文章我们趁热打铁，继续分析服务引用过程。

在 Dubbo 中，我们可以通过两种方式引用远程服务。第一种是使用服务直连的方式引用服务，第二种方式是基于注册中心进行引用。服务直连的方式仅适合在调试或测试服务的场景下使用，不适合在线上环境使用。因此，本文我将重点分析通过注册中心引用服务的过程。从注册中心中获取服务配置只是服务引用过程中的一环，除此之外，服务消费者还需要经历 Invoker 创建、代理类创建等步骤。这些步骤，将在后续章节中一一进行分析。

## 2.服务引用原理

Dubbo 服务引用的时机有两个，第一个是在 Spring 容器调用 `ReferenceAnnotationBeanPostProcessor#postProcessProperties` 方法时引用服务，第二个是在 ReferenceBean 对应的服务被注入到其他类中时引用。这两个引用服务的时机区别在于，第一个是饿汉式的，第二个是懒汉式的。默认情况下，Dubbo 使用懒汉式引用服务。如果需要使用饿汉式，可通过配置 <dubbo:reference> 的 init 属性开启。下面我们按照 Dubbo 默认配置进行分析，整个分析过程从 ReferenceBean 的 getObject 方法开始。

当我们的服务被注入到其他类中时，Spring 会第一时间调用 getObject 方法，并由该方法执行服务引用逻辑。按照惯例，在进行具体工作之前，需先进行配置检查与收集工作。接着根据收集到的信息决定服务用的方式，有三种，第一种是引用本地 (JVM) 服务，第二是通过直连方式引用远程服务，第三是通过注册中心引用远程服务。不管是哪种引用方式，最后都会得到一个 Invoker 实例。如果有多个注册中心，多个服务提供者，这个时候会得到一组 Invoker 实例，此时需要通过集群管理类 Cluster 将多个 Invoker 合并成一个实例。合并后的 Invoker 实例已经具备调用本地或远程服务的能力了，但并不能将此实例暴露给用户使用，这会对用户业务代码造成侵入。此时框架还需要通过代理工厂类 (ProxyFactory) 为服务接口生成代理类，并让代理类去调用 Invoker 逻辑。避免了 Dubbo 框架代码对业务代码的侵入，同时也让框架更容易使用。

以上就是服务引用的大致原理，下面我们深入到代码中，详细分析服务引用细节。

## 3.源码分析

服务引用的入口方法为 `ReferenceBean#getObject` 方法，该方法定义在 Spring 的 `FactoryBean` 接口中，`ReferenceBean` 实现了这个方法。实现代码如下：

```java
public T getObject() {
    if (lazyProxy == null) {
        createLazyProxy();
    }
    return (T) lazyProxy;
}
```

以上两个方法的代码比较简短，并不难理解。

### 3.1 处理配置

这里使用了懒加载来创建代理对象，避免启动时同时创建大量代理类影响启动性能。

```java
/**
 * 创建服务接口的懒加载代理对象
 * 支持多种代理生成方式，包括 JDK 动态代理和 Javassist 字节码增强
 */
private void createLazyProxy() {
    // 构建代理对象需要实现的接口列表
    // 参考：org.apache.dubbo.rpc.proxy.AbstractProxyFactory.getProxy 方法
    List<Class<?>> interfaces = new ArrayList<>();
    // 添加服务接口本身
    interfaces.add(interfaceClass);
    // 添加 Dubbo 内部通用接口
    Class<?>[] internalInterfaces = AbstractProxyFactory.getInternalInterfaces();
    Collections.addAll(interfaces, internalInterfaces);
    // 如果接口名与接口类名不一致，尝试加载接口名对应的类并添加到接口列表
    if (!StringUtils.isEquals(interfaceClass.getName(), interfaceName)) {
        try {
            // 动态加载服务接口类
            Class<?> serviceInterface = ClassUtils.forName(interfaceName, beanClassLoader);
            interfaces.add(serviceInterface);
        } catch (ClassNotFoundException e) {
            // 通用调用场景下可能本地没有服务接口类，忽略异常
        }
    }
    // 当运行在 GraalVM Native Image 环境时，强制使用 JDK 代理
    if (NativeDetector.inNativeImage()) {
        generateFromJdk(interfaces);
    }
    // 优先使用 Javassist 生成代理（性能更好），条件：
    // 1. 懒代理尚未生成
    // 2. 未指定代理类型或代理类型为默认值 "javassist"
    if (this.lazyProxy == null
            && (StringUtils.isEmpty(this.proxy) || CommonConstants.DEFAULT_PROXY.equalsIgnoreCase(this.proxy))) {
        generateFromJavassistFirst(interfaces);
    }
    // 如果上述方式都失败，回退到使用 JDK 动态代理
    if (this.lazyProxy == null) {
        generateFromJdk(interfaces);
    }
}
```

上面的代码很长，做的事情比较多。这里根据代码逻辑，对代码进行了分块，下面我们一起来看一下。

### 3.2 引用服务

本节我们要从 `createLazyProxy` 开始看起。从字面意思上来看，`createLazyProxy` 似乎只是用于创建代理对象的。但实际上并非如此，该方法还会调用其他方法构建以及合并 Invoker 实例。具体细节如下。

```java
/**
 * 优先使用Javassist字节码增强技术生成代理对象，失败时回退到JDK动态代理
 * @param interfaces 代理需要实现的接口列表
 */
private void generateFromJavassistFirst(List<Class<?>> interfaces) {
    try {
        // 使用Javassist生成代理对象
        // 优点：支持方法体增强，性能通常优于JDK代理
        this.lazyProxy = Proxy.getProxy(interfaces.toArray(new Class[0]))
                .newInstance(new LazyTargetInvocationHandler(new DubboReferenceLazyInitTargetSource()));
    } catch (Throwable fromJavassist) {
        // Javassist生成失败，尝试回退到JDK动态代理
        try {
            // JDK动态代理基于接口实现
            // 缺点：只能代理实现了接口的类
            this.lazyProxy = java.lang.reflect.Proxy.newProxyInstance(
                    beanClassLoader,
                    interfaces.toArray(new Class[0]),
                    new LazyTargetInvocationHandler(new DubboReferenceLazyInitTargetSource()));
            logger.error("");
        } catch (Throwable fromJdk) {
            // 两种代理方式均失败，记录详细错误信息
            logger.error("");
            // 抛出原始的Javassist异常，保留初始错误上下文
            throw fromJavassist;
        }
    }
}
```

首先通过`Proxy.getProxy`方法创建代理，如果创建失败则回退到jdk动态代理。

```java
public static Proxy getProxy(Class<?>... ics) {
    // 检查接口数量是否超过限制
    if (ics.length > MAX_PROXY_COUNT) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // 使用第一个接口的类加载器和保护域
    // 确保能正确加载接口及相关类
    ClassLoader cl = ics[0].getClassLoader();
    ProtectionDomain domain = ics[0].getProtectionDomain();
    // 构建接口组合的唯一标识键
    // 格式：类加载器哈希码 + 接口全限定名列表
    String key = buildInterfacesKey(cl, ics);
    // 获取类加载器对应的缓存Map
    // 不同类加载器的缓存相互隔离
    final Map<String, Proxy> cache;
    synchronized (PROXY_CACHE_MAP) {
        cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new ConcurrentHashMap<>());
    }

    // 从缓存中获取代理类
    Proxy proxy = cache.get(key);
    if (proxy == null) {
        // 双重检查锁机制，确保多线程环境下只生成一次代理类
        synchronized (ics[0]) {
            proxy = cache.get(key);
            if (proxy == null) {
                // 动态生成代理类
                proxy = new Proxy(buildProxyClass(cl, ics, domain));
                // 将生成的代理类放入缓存
                cache.put(key, proxy);
            }
        }
    }
    return proxy;
}
```

实际的代理生成逻辑在`buildProxyClass`方法里。

```java
private static Class<?> buildProxyClass(ClassLoader cl, Class<?>[] ics, ProtectionDomain domain) {
    ClassGenerator ccp = null;
    try {
        // 创建Javassist类生成器实例
        ccp = ClassGenerator.newInstance(cl);
        // 用于去重方法签名和存储所有需要代理的方法
        Set<String> worked = new HashSet<>();
        List<Method> methods = new ArrayList<>();
        // 默认使用第一个接口的包名和类作为参考点
        String pkg = ics[0].getPackage().getName();
        Class<?> neighbor = ics[0];
        // 遍历所有需要实现的接口
        for (Class<?> ic : ics) {
            String npkg = ic.getPackage().getName();
            // 检查非公共接口是否来自同一包（Java要求）
            if (!Modifier.isPublic(ic.getModifiers())) {
                if (!pkg.equals(npkg)) {
                    throw new IllegalArgumentException("non-public interfaces from different packages");
                }
            }
            // 添加接口到代理类
            ccp.addInterface(ic);
            // 处理接口中的每个方法
            for (Method method : ic.getMethods()) {
                String desc = ReflectUtils.getDesc(method);
                // 跳过已处理的方法和静态方法
                if (worked.contains(desc) || Modifier.isStatic(method.getModifiers())) {
                    continue;
                }
                worked.add(desc);
                // 记录方法索引，用于后续调用
                int ix = methods.size();
                Class<?> rt = method.getReturnType();
                Class<?>[] pts = method.getParameterTypes();
                // 动态生成方法体代码，将方法调用转发给InvocationHandler
                StringBuilder code = new StringBuilder("Object[] args = new Object[")
                        .append(pts.length)
                        .append("];");
                // 收集方法参数
                for (int j = 0; j < pts.length; j++) {
                    code.append(" args[")
                            .append(j)
                            .append("] = ($w)$")
                            .append(j + 1)
                            .append(';');
                }
                // 调用InvocationHandler的invoke方法
                code.append(" Object ret = handler.invoke(this, methods[")
                        .append(ix)
                        .append("], args);");
                // 处理返回值
                if (!Void.TYPE.equals(rt)) {
                    code.append(" return ").append(asArgument(rt, "ret")).append(';');
                }
                // 保存方法信息并添加生成的方法到代理类
                methods.add(method);
                ccp.addMethod(
                        method.getName(),
                        method.getModifiers(),
                        rt,
                        pts,
                        method.getExceptionTypes(),
                        code.toString());
            }
        }
        // 创建代理类
        String pcn = neighbor.getName() + "DubboProxy" + PROXY_CLASS_COUNTER.getAndIncrement();
        ccp.setClassName(pcn);
        // 添加静态方法数组字段和InvocationHandler字段
        ccp.addField("public static java.lang.reflect.Method[] methods;");
        ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
        // 添加构造函数
        ccp.addConstructor(
                Modifier.PUBLIC, new Class<?>[] {InvocationHandler.class}, new Class<?>[0], "handler=$1;");
        ccp.addDefaultConstructor();
        // 生成字节码并加载类
        Class<?> clazz = ccp.toClass(neighbor, cl, domain);
        // 初始化静态methods字段
        clazz.getField("methods").set(null, methods.toArray(new Method[0]));
        return clazz;
    } catch (RuntimeException e) {
        throw e;
    } catch (Exception e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        // 释放资源
        if (ccp != null) {
            ccp.release();
        }
    }
}
```

在创建完代理类后，通过`Proxy#newInstance`创建出代理类对象。

以上即为Javassist代理的创建逻辑，jdk 代理的创建在此跳过。

#### 3.2.1 创建 Invoker

Invoker 是 Dubbo 的核心模型，代表一个可执行体。在服务提供方，Invoker 用于调用服务提供类。在服务消费方，Invoker 用于执行远程调用。Invoker 是由 Protocol 实现类构建而来。Protocol 实现类有很多，本节会分析最常用的两个，分别是 RegistryProtocol 和 DubboProtocol，其他的大家自行分析。下面先来分析 DubboProtocol 的 refer 方法源码。如下：

```java
/**
 * 创建服务接口的远程引用，返回一个可调用的Invoker对象
 * 该方法是服务引用的入口点，负责基本检查和协议绑定
 */
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    // 检查当前Protocol实例是否已销毁
    checkDestroyed();
    // 委派给具体的协议绑定实现
    return protocolBindingRefer(type, url);
}

/**
 * 协议绑定的服务引用实现，创建并返回DubboInvoker
 * 该方法负责创建客户端连接、优化序列化方式等核心逻辑
 */
@Override
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
    // 检查当前Protocol实例是否已销毁
    checkDestroyed();
    // 根据URL参数优化序列化方式，提升性能
    optimizeSerialization(url);
    // 创建RPC调用器，负责处理远程方法调用
    // 参数说明：
    // - serviceType: 服务接口类型
    // - url: 服务提供者URL
    // - getClients(url): 获取客户端连接列表
    // - invokers: 所有活跃的调用器集合，用于资源管理
    DubboInvoker<T> invoker = new DubboInvoker<>(serviceType, url, getClients(url), invokers);
    // 将新创建的调用器添加到活跃调用器列表，便于统一管理和资源释放
    invokers.add(invoker);
    return invoker;
}
```

上面方法看起来比较简单，不过这里有一个调用需要我们注意一下，即 `getClients`。这个方法用于获取客户端实例，实例类型为 `ClientsProvider`。`ClientsProvider`实际上是`ExchangeClient`的包装类，内部通过一个list保存对应的`ExchangeClient`。ExchangeClient 实际上并不具备通信能力，它需要基于更底层的客户端实例进行通信。比如 NettyClient、MinaClient 等，默认情况下，Dubbo 使用 NettyClient 进行通信。接下来，我们简单看一下 getClients 方法的逻辑。

```java
private ClientsProvider getClients(URL url) {
    // 从URL参数中获取连接数配置
    int connections = url.getParameter(CONNECTIONS_KEY, 0);
    // 判断连接共享策略：
    // 若connections=0，表示使用共享连接模式（默认）
    // 否则使用独占连接模式（每个服务独立维护connections个连接）
    if (connections == 0) {
        /*
         * XML配置优先级高于属性配置
         * 优先从URL参数获取共享连接数，若未配置则从全局配置获取
         */
        String shareConnectionsStr = StringUtils.isBlank(url.getParameter(SHARE_CONNECTIONS_KEY, (String) null))
                ? ConfigurationUtils.getProperty(
                        url.getOrDefaultApplicationModel(), SHARE_CONNECTIONS_KEY, DEFAULT_SHARE_CONNECTIONS)
                : url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
        // 解析共享连接数配置
        connections = Integer.parseInt(shareConnectionsStr);
        // 获取共享客户端连接，适用于轻量级服务调用场景
        return getSharedClient(url, connections);
    }
    // 独占连接模式：为当前服务创建指定数量的独立连接
    List<ExchangeClient> clients =
            IntStream.range(0, connections)
                    .mapToObj((i) -> initClient(url))  // 初始化每个连接
                    .collect(Collectors.toList());
    // 返回独占客户端提供者，适用于高并发或资源隔离需求的场景
    return new ExclusiveClientsProvider(clients);
}
```

这里根据 connections 数量决定是获取共享客户端还是创建新的客户端实例，默认情况下，使用共享客户端实例。getSharedClient 方法中也会调用 initClient 方法，因此下面我们一起看一下这两个方法。

```java
/**
 * 获取或创建共享连接提供者，实现连接池的复用和引用计数管理
 */
private SharedClientsProvider getSharedClient(URL url, int connectNum) {
    // 使用服务地址作为共享连接池的键，相同地址的服务共享同一连接池
    String key = url.getAddress();
    // 确保连接数至少为1
    int expectedConnectNum = Math.max(connectNum, 1);
    // 使用原子操作获取或创建共享连接池
    // compute方法确保在并发环境下每个地址只创建一个连接池实例
    return referenceClientMap.compute(key, (originKey, originValue) -> {
        // 如果连接池已存在且可以成功增加引用计数，则复用现有连接池
        if (originValue != null && originValue.increaseCount()) {
            return originValue;
        } else {
            // 否则创建新的共享连接池
            // 参数说明：
            // - this: 当前Protocol实例，用于资源管理
            // - originKey: 服务地址键
            // - buildReferenceCountExchangeClientList(): 创建带引用计数的客户端连接列表
            return new SharedClientsProvider(
                    this, originKey, buildReferenceCountExchangeClientList(url, expectedConnectNum));
        }
    });
}

/**
 * 构建指定数量的带引用计数的客户端连接列表
 */
private List<ReferenceCountExchangeClient> buildReferenceCountExchangeClientList(URL url, int connectNum) {
    List<ReferenceCountExchangeClient> clients = new ArrayList<>();
    // 创建指定数量的连接，并包装为引用计数客户端
    for (int i = 0; i < connectNum; i++) {
        clients.add(buildReferenceCountExchangeClient(url));
    }
    return clients;
}

/**
 * 构建单个带引用计数的客户端连接
 */
private ReferenceCountExchangeClient buildReferenceCountExchangeClient(URL url) {
    // 初始化底层交换客户端
    ExchangeClient exchangeClient = initClient(url);
    // 包装为引用计数客户端，支持连接复用和优雅关闭
    ReferenceCountExchangeClient client = new ReferenceCountExchangeClient(exchangeClient, DubboCodec.NAME);
    // 读取配置，设置优雅关闭等待时间
    int shutdownTimeout = ConfigurationUtils.getServerShutdownTimeout(url.getScopeModel());
    client.setShutdownWaitTime(shutdownTimeout);
    return client;
}
```

上面方法先访问缓存，若缓存未命中，则通过 initClient 方法创建新的 ExchangeClient 实例，并将该实例传给 ReferenceCountExchangeClient 构造方法创建一个带有引用计数功能的 ExchangeClient 实例。ReferenceCountExchangeClient 内部实现比较简单，就不分析了。下面我们再来看一下 initClient 方法的代码。

```java
/**
 * 初始化底层交换客户端，支持多种连接模式和协议配置
 */
private ExchangeClient initClient(URL url) {
    /*
     * 注意：URL参数会被共享给同一服务实例的所有服务
     * 由于当前客户端是共享的，这通常不会有问题
     */
    // 获取客户端类型配置，默认使用netty4
    String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
    // 检查客户端类型是否支持，防止使用性能不佳的BIO
    if (StringUtils.isNotEmpty(str)
            && !url.getOrDefaultFrameworkModel()
                    .getExtensionLoader(Transporter.class)
                    .hasExtension(str)) {
        throw new RpcException("/*...*/"));
    }
    try {
        ScopeModel scopeModel = url.getScopeModel();
        int heartbeat = UrlUtils.getHeartbeat(url);
        // 替换为ServiceConfigURL，确保参数正确传递
        url = new ServiceConfigURL(
                DubboCodec.NAME,
                url.getUsername(),
                url.getPassword(),
                url.getHost(),
                url.getPort(),
                url.getPath(),
                url.getAllParameters());
        // 设置编解码和心跳参数
        url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
        // 默认启用心跳机制
        url = url.addParameterIfAbsent(HEARTBEAT_KEY, Integer.toString(heartbeat));
        url = url.setScopeModel(scopeModel);
        // 根据配置决定是否使用懒连接模式
        // 懒连接模式：仅在首次调用时建立实际连接
        return url.getParameter(LAZY_CONNECT_KEY, false)
                ? new LazyConnectExchangeClient(url, requestHandler)
                : Exchangers.connect(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("/*...*/");
    }
}
```

initClient 方法首先获取用户配置的客户端类型，默认为 netty (DEFAULT_REMOTING_CLIENT)。然后检测用户配置的客户端类型是否存在，不存在则抛出异常。最后根据 lazy 配置决定创建什么类型的客户端。这里的 LazyConnectExchangeClient 代码并不是很复杂，该类会在 request 方法被调用时通过 Exchangers 的 connect 方法创建 ExchangeClient 客户端，该类的代码本节就不分析了。下面我们分析一下 Exchangers 的 connect 方法。

```java
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // 获取 Exchanger 实例，默认为 HeaderExchangeClient
    return getExchanger(url).connect(url, handler);
}
```

如上，getExchanger 会通过 SPI 加载 HeaderExchanger 实例，这个方法比较简单，大家自己看一下吧。接下来分析 HeaderExchanger.connect 的实现。

```java
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    // 这里包含了多个调用，分别如下：
    // 1. 创建 HeaderExchangeHandler 对象
    // 2. 创建 DecodeHandler 对象
    // 3. 通过 Transporters 构建 Client 实例
    // 4. 创建 HeaderExchangeClient 对象
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
}
```

这里的调用比较多，我们这里重点看一下 Transporters 的 connect 方法。如下：

```java
public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    ChannelHandler handler;
    if (handlers == null || handlers.length == 0) {
        handler = new ChannelHandlerAdapter();
    } else if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        // 如果 handler 数量大于1，则创建一个 ChannelHandler 分发器
        handler = new ChannelHandlerDispatcher(handlers);
    }
    // 获取 Transporter 自适应拓展类，并调用 connect 方法生成 Client 实例
    return getTransporter(url).connect(url, handler);
}
```

如上，getTransporter 方法返回的是自适应拓展类，该类会在运行时根据客户端类型加载指定的 Transporter 实现类。若用户未配置客户端类型，则默认加载 NettyTransporter，并调用该类的 connect 方法。如下：

```java
public Client connect(URL url, ChannelHandler listener) throws RemotingException {
    // 创建 NettyClient 对象
    return new NettyClient(url, listener);
}
```

到这里就不继续跟下去了，在往下就是通过 Netty 提供的 API 构建 Netty 客户端了，大家有兴趣自己看看。到这里，关于 DubboProtocol 的 refer 方法就分析完了。

接下来，继续分析 RegistryProtocol 的 refer 方法逻辑。

```java
/**
 * 引用远程服务，返回服务调用器（Invoker）
 * 核心逻辑：与注册中心交互，根据服务分组配置选择集群策略，生成可执行远程调用的Invoker
 */
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    // 从原始URL中提取注册中心的URL（可能包含注册中心地址、协议等信息）
    url = getRegistryUrl(url);
    // 根据注册中心URL获取注册中心实例（如ZookeeperRegistry、NacosRegistry等）
    Registry registry = getRegistry(url);
    // 特殊处理：如果引用的是注册中心服务本身（RegistryService接口），直接为注册中心创建代理 Invoker
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }
    // 从URL属性中获取服务引用相关的查询参数（如group、cluster等配置）
    Map<String, String> qs = (Map<String, String>) url.getAttribute(REFER_KEY);
    // 提取服务分组配置（group用于区分同一服务的不同分组，如"group-a,group-b"或"*"）
    String group = qs.get(GROUP_KEY);
    // 处理多分组或全量分组（"*"）的场景
    if (StringUtils.isNotEmpty(group)) {
        // 如果group包含多个值（用逗号分隔）或为"*"（表示匹配所有分组）
        if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
            // 使用"可合并集群"策略（MERGEABLE_CLUSTER_NAME），用于合并多个分组的服务结果
            return doRefer(
                    Cluster.getCluster(url.getScopeModel(), MERGEABLE_CLUSTER_NAME), 
                    registry, 
                    type, 
                    url, 
                    qs);
        }
    }
    // 单分组场景：根据配置获取对应的集群策略（默认是failover，也可配置failfast、failsafe等）
    Cluster cluster = Cluster.getCluster(url.getScopeModel(), qs.get(CLUSTER_KEY));
    // 执行实际的服务引用逻辑，返回对应集群策略的Invoker
    return doRefer(cluster, registry, type, url, qs);
}
```

上面代码首先为 url 设置协议头，然后根据 url 参数加载注册中心实例。然后获取 group 配置，根据 group 配置决定 doRefer 第一个参数的类型。这里的重点是 doRefer 方法，如下：

```java
/**
 * 执行实际的服务引用逻辑，创建集群调用器并应用拦截器
 */
protected <T> Invoker<T> doRefer(
        Cluster cluster, Registry registry, Class<T> type, URL url, Map<String, String> parameters) {
    // 复制并处理服务消费者属性，移除引用相关的临时属性
    Map<String, Object> consumerAttribute = new HashMap<>(url.getAttributes());
    consumerAttribute.remove(REFER_KEY);
    // 确定消费者协议，默认为"consumer"
    String p = isEmpty(parameters.get(PROTOCOL_KEY)) ? CONSUMER : parameters.get(PROTOCOL_KEY);
    // 构建消费者URL，包含服务路径、参数和属性
    URL consumerUrl = new ServiceConfigURL(
            p,                          // 协议
            null,                       // 用户名
            null,                       // 密码
            parameters.get(REGISTER_IP_KEY),  // 注册IP
            0,                          // 端口（消费者不使用端口）
            getPath(parameters, type),  // 服务路径
            parameters,                 // 参数
            consumerAttribute);         // 属性
    // 将消费者URL存入原始URL的属性中，便于后续使用
    url = url.putAttribute(CONSUMER_URL_KEY, consumerUrl);
    // 创建支持服务迁移的集群调用器
    // 该调用器可在服务提供者变更时动态调整调用策略
    ClusterInvoker<T> migrationInvoker = getMigrationInvoker(
            this, cluster, registry, type, url, consumerUrl);
    // 应用拦截器链，增强调用器功能（如日志、权限校验等）
    return interceptInvoker(migrationInvoker, url, consumerUrl);
}
```

interceptInvoker的调用链为：

```java
/**
 * 应用注册协议监听器对调用器进行拦截增强
 */
protected <T> Invoker<T> interceptInvoker(ClusterInvoker<T> invoker, URL url, URL consumerUrl) {
    // 查找所有注册协议监听器
    List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
    if (CollectionUtils.isEmpty(listeners)) {
        return invoker;
    }
    // 调用所有监听器的onRefer方法进行增强
    for (RegistryProtocolListener listener : listeners) {
        listener.onRefer(this, invoker, consumerUrl, url);
    }
    return invoker;
}

/**
 * 处理服务引用事件，初始化迁移规则处理器并执行迁移
 */
public void onRefer(
        RegistryProtocol registryProtocol, ClusterInvoker<?> invoker, URL consumerUrl, URL registryURL) {
    // 为每个迁移调用器创建对应的迁移规则处理器
    MigrationRuleHandler<?> migrationRuleHandler =
            ConcurrentHashMapUtils.computeIfAbsent(handlers, (MigrationInvoker<?>) invoker, _key -> {
                // 设置迁移规则监听器
                ((MigrationInvoker<?>) invoker).setMigrationRuleListener(this);
                return new MigrationRuleHandler<>((MigrationInvoker<?>) invoker, consumerUrl);
            });
    // 执行迁移逻辑
    migrationRuleHandler.doMigrate(rule);
}

/**
 * 根据迁移规则执行服务发现迁移
 */
public void doMigrate(MigrationRule rule) {
    lock.lock();
    try {
        // 如果是服务发现迁移调用器，直接强制迁移到应用级模式
        if (migrationInvoker instanceof ServiceDiscoveryMigrationInvoker) {
            refreshInvoker(MigrationStep.FORCE_APPLICATION, 1.0f, rule);
            return;
        }
        // 初始迁移步骤为APPLICATION_FIRST
        MigrationStep step = MigrationStep.APPLICATION_FIRST;
        float threshold = -1f;
        // 从规则中获取迁移步骤和阈值
        try {
            step = rule.getStep(consumerURL);
            threshold = rule.getThreshold(consumerURL);
        } catch (Exception e) {
            logger.error("/*...*/");
        }
        // 根据步骤和阈值刷新调用器
        if (refreshInvoker(step, threshold, rule)) {
            // 刷新成功后更新当前迁移规则
            setMigrationRule(rule);
        }
    } finally {
        lock.unlock();
    }
}

/**
 * 根据指定的迁移步骤和阈值刷新调用器
 */
private boolean refreshInvoker(MigrationStep step, Float threshold, MigrationRule newRule) {
    if (step == null || threshold == null) {
        throw new IllegalStateException("/*...*/");
    }
    MigrationStep originStep = currentStep;
    // 当步骤或阈值发生变化时执行迁移
    if ((currentStep == null || currentStep != step) || !currentThreshold.equals(threshold)) {
        boolean success = true;
        switch (step) {
            // 应用优先模式：同时使用接口级和应用级注册
            case APPLICATION_FIRST:
                migrationInvoker.migrateToApplicationFirstInvoker(newRule);
                break;
            // 强制应用级模式：仅使用应用级注册
            case FORCE_APPLICATION:
                success = migrationInvoker.migrateToForceApplicationInvoker(newRule);
                break;
            // 强制接口级模式：仅使用接口级注册
            case FORCE_INTERFACE:
            default:
                success = migrationInvoker.migrateToForceInterfaceInvoker(newRule);
        }
        if (success) {
            // 更新当前步骤和阈值
            setCurrentStepAndThreshold(step, threshold);
            report(step, originStep, "true");
        } else {
            // 迁移失败，记录警告并上报
            report(step, originStep, "false");
        }
        return success;
    }
    // 步骤未变化时，直接返回成功
    return true;
}

/**
 * 迁移到应用优先模式，同时维护接口级和应用级服务发现
 */
public void migrateToApplicationFirstInvoker(MigrationRule newRule) {
    CountDownLatch latch = new CountDownLatch(0);
    // 刷新接口级服务发现调用器
    refreshInterfaceInvoker(latch);
    // 刷新应用级服务发现调用器
    refreshServiceDiscoveryInvoker(latch);
    // 直接计算首选调用器，地址通知时会重新计算
    calcPreferredInvoker(newRule);
}

/**
 * 刷新接口级服务发现调用器
 */
protected void refreshInterfaceInvoker(CountDownLatch latch) {
    // 清除旧调用器的监听器
    clearListener(invoker);
    
    // 如果需要刷新，则销毁旧调用器并创建新的
    if (needRefresh(invoker)) {
        if (logger.isDebugEnabled()) {
            logger.debug("/*...*/");
        }
        if (invoker != null) {
            invoker.destroy();
        }
        invoker = registryProtocol.getInvoker(cluster, registry, type, url);
    }
    // 设置监听器，地址变更时触发相应操作
    setListener(invoker, () -> {
        latch.countDown();
        if (reportService.hasReporter()) {
            // 报告接口级服务消费状态
 reportService.reportConsumptionStatus(reportService.createConsumptionReport(
                    consumerUrl.getServiceInterface(),
                    consumerUrl.getVersion(),
                    consumerUrl.getGroup(),
                    "interface"));
        }
        // 当前处于应用优先模式时，重新计算首选调用器
        if (step == APPLICATION_FIRST) {
            calcPreferredInvoker(rule);
        }
    });
}

/**
 * 创建集群调用器
 */
public <T> ClusterInvoker<T> getInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
    // 创建注册目录，用于服务发现
    DynamicDirectory<T> directory = new RegistryDirectory<>(type, url);
    // 创建并返回集群调用器
    return doCreateInvoker(directory, cluster, registry, type);
}
```

如上，getInvoker方法创建一个 RegistryDirectory 实例，然后生成服务者消费者链接，并向注册中心进行注册。注册完毕后，紧接着订阅 providers、configurators、routers 等节点下的数据。完成订阅后，RegistryDirectory 会收到这几个节点下的子节点信息。由于一个服务可能部署在多台服务器上，这样就会在 providers 产生多个节点，这个时候就需要 Cluster 将多个服务节点合并为一个，并生成一个 Invoker。

```java
/**
 * 创建集群调用器，用于服务消费
 */
protected <T> ClusterInvoker<T> doCreateInvoker(
        DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {
    // 设置服务目录的注册中心和协议
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // 从消费者URL中提取所有参数
    Map<String, String> parameters =
            new HashMap<>(directory.getConsumerUrl().getParameters());
    // 构建用于注册到注册中心的URL
    URL urlToRegistry = new ServiceConfigURL(
            // 使用参数中的协议或默认使用"consumer"协议
            parameters.get(PROTOCOL_KEY) == null ? CONSUMER : parameters.get(PROTOCOL_KEY),
            // 注册IP地址
            parameters.remove(REGISTER_IP_KEY),
            0, // 端口号设为0，表示不需要端口
            // 获取服务路径
            getPath(parameters, type),
            parameters);
    // 设置URL的作用域模型和服务模型，保持与消费者URL一致
    urlToRegistry = urlToRegistry.setScopeModel(directory.getConsumerUrl().getScopeModel());
    urlToRegistry = urlToRegistry.setServiceModel(directory.getConsumerUrl().getServiceModel());
    // 如果需要注册消费者信息，则向注册中心注册
    if (directory.isShouldRegister()) {
        directory.setRegisteredConsumerUrl(urlToRegistry);
        registry.register(directory.getRegisteredConsumerUrl());
    }
    // 构建路由链，用于服务调用时的服务实例过滤和选择
    directory.buildRouterChain(urlToRegistry);
    // 订阅服务提供者地址变更事件
    directory.subscribe(toSubscribeUrl(urlToRegistry));
    // 创建集群调用器，将服务目录作为参数传入
    // 集群调用器负责负载均衡、容错等策略
    return (ClusterInvoker<T>) cluster.join(directory, true);
}
```

#### 3.2.2 创建代理

Invoker 创建完毕后，接下来要做的事情是为服务接口生成代理对象。有了代理对象，即可进行远程调用。代理对象生成的入口方法为 ProxyFactory 的 getProxy，接下来进行分析。

```java
public <T> T getProxy(Invoker<T> invoker) throws RpcException {
    // 调用重载方法
    return getProxy(invoker, false);
}

/**
 * 创建服务代理对象，支持本地存根和泛化调用
 */
public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    // 使用代理工厂创建基础代理对象
    T proxy = proxyFactory.getProxy(invoker, generic);
    // 如果不是泛型服务接口，则处理本地存根逻辑
    if (GenericService.class != invoker.getInterface()) {
        URL url = invoker.getUrl();
        // 获取本地存根配置
        String stub = url.getParameter(STUB_KEY, url.getParameter(LOCAL_KEY));
        if (ConfigUtils.isNotEmpty(stub)) {
            Class<?> serviceType = invoker.getInterface();
            
            // 如果配置为默认值，则使用接口名+Stub/Local作为存根类名
            if (ConfigUtils.isDefault(stub)) {
                if (url.hasParameter(STUB_KEY)) {
                    stub = serviceType.getName() + "Stub";
                } else {
                    stub = serviceType.getName() + "Local";
                }
            }
            
            try {
                // 加载存根类
                Class<?> stubClass = ReflectUtils.forName(stub);
                
                // 验证存根类是否实现了服务接口
                if (!serviceType.isAssignableFrom(stubClass)) {
                    throw new IllegalStateException("/*...*/");
                }
                
                try {
                    // 获取存根类的构造函数(需要接受服务接口类型的参数)
                    Constructor<?> constructor = ReflectUtils.findConstructor(stubClass, serviceType);
                    
                    // 使用原始代理对象创建存根实例
                    proxy = (T) constructor.newInstance(new Object[] {proxy});
                    
                    // 导出存根服务，用于事件通知
                    URLBuilder urlBuilder = URLBuilder.from(url);
                    if (url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT)) {
                        urlBuilder.addParameter(
                                STUB_EVENT_METHODS_KEY,
                                StringUtils.join(
                                        Wrapper.getWrapper(proxy.getClass()).getDeclaredMethodNames(), ","));
                        urlBuilder.addParameter(IS_SERVER_KEY, Boolean.FALSE.toString());
                        
                        try {
                            // 导出存根服务，使其能够处理事件
                            export(proxy, invoker.getInterface(), urlBuilder.build());
                        } catch (Exception e) {
                            LOGGER.error("/*...*/");
                        }
                    }
                } catch (NoSuchMethodException e) {
                    throw new IllegalStateException("/*...*/");
                }
            } catch (Throwable t) {
                // 忽略异常，继续使用原始代理
            }
        }
    }
    return proxy;
}

/**
 * 创建服务代理对象，收集并处理代理需要实现的所有接口
 */
public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    // 保持接口顺序不变，用于原生镜像编译
    LinkedHashSet<Class<?>> interfaces = new LinkedHashSet<>();
    ClassLoader classLoader = getClassLoader(invoker);

    // 从URL参数中获取配置的接口
    String config = invoker.getUrl().getParameter(INTERFACES);
    if (StringUtils.isNotEmpty(config)) {
        String[] types = COMMA_SPLIT_PATTERN.split(config);
        for (String type : types) {
            try {
                interfaces.add(ReflectUtils.forName(classLoader, type));
            } catch (Throwable e) {
                // 忽略无法加载的接口
            }
        }
    }

    Class<?> realInterfaceClass = null;
    if (generic) {
        try {
            // 从URL中获取真实接口类型
            String realInterface = invoker.getUrl().getParameter(Constants.INTERFACE);
            realInterfaceClass = ReflectUtils.forName(classLoader, realInterface);
            interfaces.add(realInterfaceClass);
        } catch (Throwable e) {
            // 忽略异常
        }

        // 添加泛型服务接口支持
        if (GenericService.class.isAssignableFrom(invoker.getInterface())
                && Dubbo2CompactUtils.isEnabled()
                && Dubbo2CompactUtils.isGenericServiceClassLoaded()) {
            interfaces.add(Dubbo2CompactUtils.getGenericServiceClass());
        }
        if (!GenericService.class.isAssignableFrom(invoker.getInterface())) {
            if (Dubbo2CompactUtils.isEnabled() && Dubbo2CompactUtils.isGenericServiceClassLoaded()) {
                interfaces.add(Dubbo2CompactUtils.getGenericServiceClass());
            } else {
                interfaces.add(org.apache.dubbo.rpc.service.GenericService.class);
            }
        }
    }

    // 添加服务接口和内部接口
    interfaces.add(invoker.getInterface());
    interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES));

    try {
        // 使用收集的接口创建代理
        return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
    } catch (Throwable t) {
        if (generic) {
            // 泛化调用失败处理：尝试不使用真实接口创建代理
            if (realInterfaceClass != null) {
                interfaces.remove(realInterfaceClass);
            }
            interfaces.remove(invoker.getInterface());
            return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
        } else {
            throw t;
        }
    }
}

/**
 * 使用指定接口创建服务代理对象
 */
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    try {
        // 优先使用Javassist动态生成代理类
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    } catch (Throwable fromJavassist) {
        // Javassist失败时，回退到JDK动态代理
        try {
            T proxy = jdkProxyFactory.getProxy(invoker, interfaces);
            return proxy;
        } catch (Throwable fromJdk) {
            throw fromJavassist;
        }
    }
}
```

如上，上面大段代码都是用来获取 interfaces 数组的，我们继续往下看。getProxy(Invoker, Class[]) 这个方法是一个抽象方法，下面我们到 JavassistProxyFactory 类中看一下该方法的实现代码。

```java
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        try {
            return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
        } catch (Throwable fromJavassist) {
            // try fall back to JDK proxy factory
            try {
                T proxy = jdkProxyFactory.getProxy(invoker, interfaces);
                return proxy;
            } catch (Throwable fromJdk) {
                throw fromJavassist;
            }
        }
    }
```

上面代码并不多，首先是通过 Proxy 的 getProxy 方法获取 Proxy 子类，然后创建 InvokerInvocationHandler 对象，并将该对象传给 newInstance 生成 Proxy 实例。InvokerInvocationHandler 实现 JDK 的 InvocationHandler 接口，具体的用途是拦截接口类调用。该类逻辑比较简单，这里就不分析了。下面我们重点关注一下 Proxy 的 getProxy 方法，如下。

后续的调用前面已经说明，在此略过。大家在阅读这段代码时，要搞清楚 ccp 和 ccm 的用途，不然会被搞晕。ccp 用于为服务接口生成代理类，比如我们有一个 DemoService 接口，这个接口代理类就是由 ccp 生成的。ccm 则是用于为 org.apache.dubbo.common.bytecode.Proxy 抽象类生成子类，主要是实现 Proxy 类的抽象方法。下面以 org.apache.dubbo.demo.DemoService 这个接口为例，来看一下该接口代理类代码大致是怎样的（忽略 EchoService 接口）。

```java
package org.apache.dubbo.common.bytecode;

public class proxy0 implements org.apache.dubbo.demo.DemoService {

    public static java.lang.reflect.Method[] methods;

    private java.lang.reflect.InvocationHandler handler;

    public proxy0() {
    }

    public proxy0(java.lang.reflect.InvocationHandler arg0) {
        handler = $1;
    }

    public java.lang.String sayHello(java.lang.String arg0) {
        Object[] args = new Object[1];
        args[0] = ($w) $1;
        Object ret = handler.invoke(this, methods[0], args);
        return (java.lang.String) ret;
    }
}
```

好了，到这里代理类生成逻辑就分析完了。整个过程比较复杂，大家需要耐心看一下。

## 4.总结

本篇文章对服务引用的过程进行了较为详尽的分析，还有一些逻辑暂时没有分析到，比如 Directory、Cluster。这些接口及实现类功能比较独立，后续会单独成文进行分析。暂时我们可以先把这些类看成黑盒，只要知道这些类的用途即可。关于服务引用过程就分析到这里。