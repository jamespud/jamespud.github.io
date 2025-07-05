---
title: "7. Dubbo3 服务导出源码分析"
date: 2025-07-03 20:50:00 +0800
categories: [从零开始的Dubbo源码阅读]
tags: [dubbo, 源码阅读]
---
# 服务导出

> 参考链接：[dubbo2.6源码分析](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/export-service/)

## 简介

当我们在应用中使用`@DubboService`或XML配置声明一个服务时，Dubbo框架究竟做了什么？服务是如何被暴露给其他应用调用的？今天，我们将从源码角度一探究竟，理解Dubbo 3服务导出的完整流程。

与Dubbo 2.6相比，Dubbo 3已将事件传播机制从Spring容器依赖转向框架自身实现，提升了灵活性与扩展性。这一架构优化使Dubbo能更好地适应多种场景，而不再强依赖于Spring生态。

Dubbo3服务导出过程可分为三个核心阶段：

1. 前置准备 - 参数检查与URL组装

1. 服务暴露 - 本地导出(JVM内部)与远程导出(网络)

1. 注册中心交互 - 服务注册与配置订阅

下面，我们将按照代码执行顺序，逐一解析每个环节的实现细节。

## 服务启动

服务的启动入口在`DubboBootstrap#start()`方法

```java
public DubboBootstrap start() {
    this.start(true);
    return this;
}

public DubboBootstrap start(boolean wait) {
    Future future = applicationDeployer.start();
    if (wait) {
        try {
            future.get();
        } catch (Exception e) {
            throw new IllegalStateException("await dubbo application start finish failure", e);
        }
    }
    return this;
}
```

核心逻辑委托给了`DefaultApplicationDeployer#start()`方法，这里通过Future模式支持同步/异步两种启动方式，非常灵活。

```java
public Future start() {
    // 使用startLock确保启动过程的原子性，防止多线程竞争导致的重复启动
    synchronized (startLock) {
        // 校验应用生命周期状态：禁止已停止/失败的应用重新启动
        if (isStopping() || isStopped() || isFailed()) {
            throw new IllegalStateException("/*...*/");
        }

        try {
            // 检查是否有新注册的模块需要启动（例如动态添加的远程服务模块）
            boolean hasPendingModule = hasPendingModule();

            // 场景1：应用正在启动中（可能由多个线程并发触发）
            if (isStarting()) {
                // 若有新模块，在已有启动流程中加入模块启动逻辑
                if (hasPendingModule) {
                    startModules(); // 启动所有待处理模块（如新注册的ServiceBean）
                }
                // 复用现有启动Future，避免创建多个启动任务
                return startFuture;
            }
            // 场景2：应用已启动且无新模块，直接返回成功状态
            if (isStarted() && !hasPendingModule) {
                return CompletableFuture.completedFuture(false);
            }
            // 场景3：首次启动或重启（有新模块）
            // 更新状态为STARTING并触发状态变更事件（监听者可在此阶段注入配置）
            onStarting();
            // 初始化应用模型、配置解析和注册中心连接等基础组件
            initialize();
            // 执行核心启动逻辑：
            // 1. 初始化所有模块的服务提供者
            // 2. 向注册中心注册服务
            // 3. 解析并连接所有服务消费者
            // 4. 启动Netty等通信组件
            doStart();
        } catch (Throwable e) {
            // 启动失败处理：
            // 1. 更新状态为FAILED
            // 2. 触发失败事件通知监听者
            // 3. 释放已分配的资源
            onFailed(getIdentifier() + " start failure", e);
            throw e;
        }
        // 返回启动Future，可用于异步监听启动完成事件
        return startFuture;
    }
}
```

这个方法主要分为三点：

- `onStarting()`这个是启动之前的状态切换
- `initialize()`应用的初始化逻辑 
- `doStart()` 启动模块

我们把重心放在启动的核心方法`doStart()`上：

```java
private void doStart() {
    startModules();
}
```

`doStart()`方法进一步委托到`startModules()`，实现模块级别的启动：

```java
private void startModules() {
    // 确保内部模块先运行
    prepareInternalModule();

    for (ModuleModel moduleModel : applicationModel.getModuleModels()) {
        if (moduleModel.getDeployer().isPending()) {
            moduleModel.getDeployer().start();
        }
    }
}

public void prepareInternalModule() {
    // 快速检查：如果内部模块已准备好，直接返回
    if (hasPreparedInternalModule) {
        return;
    }
    synchronized (internalModuleLock) {
        if (hasPreparedInternalModule) {
            return;
        }
        // 获取内部模块的部署器
        // 内部模块包含配置解析、注册中心连接、元数据管理等核心功能
        ModuleDeployer internalModuleDeployer =
                applicationModel.getInternalModule().getDeployer();
        // 启动内部模块（如果尚未启动）
        if (!internalModuleDeployer.isStarted()) {
            // 异步启动内部模块
            Future future = internalModuleDeployer.start();
            // 等待内部模块启动完成，超时时间为5秒
            try {
                future.get(5, TimeUnit.SECONDS);
                // 标记内部模块已准备好（仅在成功启动后设置）
                hasPreparedInternalModule = true;
            } catch (Exception e) {
                // 记录警告日志，包含启动失败的原因
                // 注意：此处使用了参数化日志，避免字符串拼接带来的性能开销
                logger.warn("/*...*/");
            }
        }
    }
}
```

这里采用了"先内后外"的策略 - 先启动内部模块（如配置中心、注册中心连接），再启动业务模块。可以看出内部模块和外部模块的启动均为`ModuleDeployer#start`来执行。dubbo 中使用默认实现`DefaultModuleDeployer#start`：

```java
public Future start() throws IllegalStateException {
    // initialize，maybe deadlock applicationDeployer lock & moduleDeployer lock
    applicationDeployer.initialize();

    return startSync();
}

private synchronized Future startSync() throws IllegalStateException {
    // 校验模块状态：禁止已停止或失败的模块重启
    if (isStopping() || isStopped() || isFailed()) {
        throw new IllegalStateException("/*...*/");
    }
    try {
        // 复用已有启动任务：若模块正在启动或已启动，直接返回现有Future
        if (isStarting() || isStarted()) {
            return startFuture;
        }
        // 触发模块启动事件（状态变更为STARTING）
        onModuleStarting();
        // 1. 初始化模块配置和资源
        initialize();
        // 2. 导出本地服务为远程服务（创建Invoker和Exporter）
        exportServices();
        // 准备应用内部模块（如配置中心、注册中心等基础服务）
        // 排除内部模块自身，避免循环依赖
        if (moduleModel != moduleModel.getApplicationModel().getInternalModule()) {
            applicationDeployer.prepareInternalModule();
        }
        // 3. 引用远程服务（创建Proxy代理对象）
        referServices();
        // 同步启动模式（无异步导出/引用任务）
        if (asyncExportingFutures.isEmpty() && asyncReferringFutures.isEmpty()) {
            // 触发模块已启动事件（状态变更为STARTED）
            onModuleStarted();
            // 4.1 将服务注册到注册中心（如ZooKeeper、Nacos）
            registerServices();
            // 检查服务引用配置的有效性
            checkReferences();
            // 标记启动任务完成（带成功状态）
            completeStartFuture(true);
        } 
        // 异步启动模式（存在异步导出/引用任务）
        else {
            // 使用框架共享线程池执行异步等待逻辑
            frameworkExecutorRepository.getSharedExecutor().submit(() -> {
                try {
                    // 等待所有异步导出任务完成
                    waitExportFinish();
                    // 等待所有异步引用任务完成
                    waitReferFinish();
                    // 触发模块已启动事件
                    onModuleStarted();
                    // 4.1 注册服务到注册中心
                    registerServices();
                    // 检查服务引用配置
                    checkReferences();
                } catch (Throwable e) {
                    // 记录异步启动失败日志
                    logger.warn("/*...*/");
                    // 标记模块启动失败
                    onModuleFailed(getIdentifier() + " start failed: " + e, e);
                } finally {
                    // 无论成功或失败，都标记启动任务完成
                    completeStartFuture(true);
                }
            });
        }
    } catch (Throwable e) {
        // 处理同步启动异常
        onModuleFailed("/*...*/");
        throw e;
    }
    // 返回启动Future供外部监听
    return startFuture;
}
```

模块启动有四个关键步骤：

1. 初始化(initialize) - 加载配置，准备环境
2. 服务导出(exportServices) - 将本地实现对外暴露
3. 服务引用(referServices) - 创建远程服务代理
4. 注册服务(registerServices) - 将服务信息注册到注册中心

这里同时支持同步和异步两种启动模式

### 初始化

先来看初始化部分做了什么：

```java
public void initialize() throws IllegalStateException {
    // 快速检查：如果已初始化，直接返回
    if (initialized) {
        return;
    }
    synchronized (this) {
        if (initialized) {
            return;
        }
        // 触发初始化事件，通知监听器模块开始初始化
        onInitialize();
        // 加载模块级配置
        // 包括服务提供者、消费者的通用配置，以及模块特定参数
        loadConfigs();
        // 获取模块配置对象
        // 若默认模块配置未初始化，抛出异常
        ModuleConfig moduleConfig = moduleModel
                .getConfigManager()
                .getModule()
                .orElseThrow(() -> new IllegalStateException("/*...*/"));
        // 决定服务导出（暴露为远程服务）是否采用异步模式
        exportAsync = Boolean.TRUE.equals(moduleConfig.getExportAsync());
        // 决定服务引用（创建远程服务代理）是否采用异步模式
        referAsync = Boolean.TRUE.equals(moduleConfig.getReferAsync());
        // 解析后台运行模式
        // 后台模式下模块启动不会阻塞应用主线程
        background = moduleConfig.getBackground();
        if (background == null) {
            // 兼容旧版本用法：若未显式配置，则根据导出/引用的后台模式推断
            background = isExportBackground() || isReferBackground();
        }
        // 标记初始化完成
        initialized = true;
        // 记录初始化完成日志
        if (logger.isInfoEnabled()) {
            logger.info("/*...*/");
        }
    }
}
```

加载了服务相关的参数，并判断了服务的导出方式。

### 导出服务

```java
private void exportServices() {
    // 遍历配置管理器中所有服务配置
    for (ServiceConfigBase sc : configManager.getServices()) {
        // 执行单个服务的内部导出逻辑
        exportServiceInternal(sc);
    }
}

private void exportServiceInternal(ServiceConfigBase sc) {
    // 类型转换：将服务配置基类转换为具体服务配置
    ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
    // 刷新配置：确保服务配置是最新的
    if (!serviceConfig.isRefreshed()) {
        serviceConfig.refresh();
    }
    // 状态检查：若服务已导出则直接返回
    if (sc.isExported()) {
        return;
    }
    // 异步导出判断：根据模块配置或服务自身配置决定是否异步导出
    if (exportAsync || sc.shouldExportAsync()) {
        // 获取服务导出专用线程池
        ExecutorService executor = executorRepository.getServiceExportExecutor();
        // 异步导出：使用线程池执行导出任务
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            try {
                if (!sc.isExported()) {
                    // 执行服务导出（注册类型为自动注册）
                    sc.export(RegisterTypeEnum.AUTO_REGISTER_BY_DEPLOYER);
                    // 记录已导出的服务
                    exportedServices.add(sc);
                }
            } catch (Throwable t) {
                // 异步导出异常处理
                logger.error("/*...*/");
            }
        }, executor);
        
        // 收集异步导出任务，用于后续等待完成
        asyncExportingFutures.add(future);
    } else {
        // 同步导出：直接在当前线程执行导出
        if (!sc.isExported()) {
            // 执行服务导出（注册类型为自动注册）
            sc.export(RegisterTypeEnum.AUTO_REGISTER_BY_DEPLOYER);
            // 记录已导出的服务
            exportedServices.add(sc);
        }
    }
}

public void export(RegisterTypeEnum registerType) {
    // 快速检查：如果服务已导出，直接返回
    if (this.exported) {
        return;
    }
    // 生命周期管理：根据ScopeModel的管理模式准备或启动模块
    if (getScopeModel().isLifeCycleManagedExternally()) {
        // 外部管理模式：仅准备模块（如加载配置）
        getScopeModel().getDeployer().prepare();
    } else {
        // 内部管理模式：启动模块（兼容旧API用法）
        getScopeModel().getDeployer().start();
    }
    synchronized (this) {
        if (this.exported) {
            return;
        }
        // 刷新配置：确保配置是最新的
        if (!this.isRefreshed()) {
            this.refresh();
        }
        // 判断是否应该导出服务
        if (this.shouldExport()) {
            // 初始化服务配置
            this.init();
            // 根据延迟配置决定导出方式
            if (shouldDelay()) {
                // 延迟导出模式：注册服务但延迟暴露
                doDelayExport();
            } 
            // 手动注册模式：由用户显式控制注册时机
            else if (Integer.valueOf(-1).equals(getDelay())
                       && Boolean.parseBoolean(ConfigurationUtils.getProperty(
                           getScopeModel(), CommonConstants.DUBBO_MANUAL_REGISTER_KEY, "false"))) {
                doExport(RegisterTypeEnum.MANUAL_REGISTER);
            } 
            // 立即导出模式
            else {
                doExport(registerType);
            }
        }
    }
}
```

在`exportServices`中通过遍历`configManager`中所有服务配置，实现模块内服务的批量导出。从批量操作到单个服务的精细化控制，`exportServiceInternal`方法是连接两者的桥梁。该方法最核心的能力是支持同步 / 异步两种导出模式。

Dubbo 支持三种导出策略，满足不同场景下的服务暴露需求：

- **延迟导出（doDelayExport）**：适用于需要预热的服务，注册到注册中心但延迟实际暴露，避免流量突增
- **手动注册（MANUAL_REGISTER）**：由开发者显式控制注册时机，适合复杂依赖场景
- **立即导出**：默认策略，适用于大多数常规服务

确定导出策略后，执行`doExport`进行服务导出。

```java
protected synchronized void doExport(RegisterTypeEnum registerType) {
    if (unexported) {
        throw new IllegalStateException("/*...*/");
    }
    if (exported) {
        return;
    }
    if (StringUtils.isEmpty(path)) {
        path = interfaceName;
    }
    doExportUrls(registerType);
    exported();
}
```

导出服务主要分成了两个部分，导出url和注册到注册中心

```java
private void doExportUrls(RegisterTypeEnum registerType) {
    // 获取作用域模型中的服务仓库（存储服务描述符和提供者模型）
    ModuleServiceRepository repository = getScopeModel().getServiceRepository();
    ServiceDescriptor serviceDescriptor;
    // 判断是否为服务器端服务（提供者）
    final boolean serverService = ref instanceof ServerService;
    if (serverService) {
        // 从服务器服务中获取服务描述符（包含接口名称、方法等元数据）
        serviceDescriptor = ((ServerService) ref).getServiceDescriptor();
        // 配置是否使用Java包路径作为服务路径
        if (!this.provider.getUseJavaPackageAsPath()) {
            // 对于Stub服务，路径始终使用接口名或IDL包名
            this.path = serviceDescriptor.getInterfaceName();
        }
        // 注册服务描述符到服务仓库
        repository.registerService(serviceDescriptor);
    } else {
        // 客户端服务（消费者）：根据接口类注册服务描述符
        serviceDescriptor = repository.registerService(getInterfaceClass());
    }
    // 创建提供者模型（封装服务关键信息）
    providerModel = new ProviderModel(
            serviceMetadata.getServiceKey(),       // 服务唯一标识
            ref,                                  // 服务实现引用
            serviceDescriptor,                    // 服务描述符
            getScopeModel(),                      // 作用域模型
            serviceMetadata,                      // 服务元数据
            interfaceClassLoader);                 // 接口类加载器
    // 兼容旧版本对ServiceModel#getServiceConfig()的依赖（未来版本将移除）
    providerModel.setConfig(this);
    // 设置服务销毁时的执行器（用于资源释放）
    providerModel.setDestroyRunner(getDestroyRunner());
    // 注册提供者模型到服务仓库
    repository.registerProvider(providerModel);
    // 加载所有注册中心URL（支持多个注册中心）
    List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);
    // 遍历所有协议配置（如dubbo、http、zookeeper等）
    for (ProtocolConfig protocolConfig : protocols) {
        // 构建服务路径键（用于服务路由和发现）
        String pathKey = URL.buildKey(
                getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
        // 非服务器端服务需要额外注册路径键到接口类的映射
        if (!serverService) {
            repository.registerService(pathKey, interfaceClass);
        }
        // 为单个协议执行URL导出逻辑
        doExportUrlsFor1Protocol(protocolConfig, registryURLs, registerType);
    }
    // 将所有导出的URL设置到提供者模型中
    providerModel.setServiceUrls(urls);
}

private void doExportUrlsFor1Protocol(
        ProtocolConfig protocolConfig, List<URL> registryURLs, RegisterTypeEnum registerType) {
    // 构建服务属性（包含协议参数、服务配置等元数据）
    Map<String, String> map = buildAttributes(protocolConfig);
    // 清理空键和空值（避免无效参数影响URL生成）
    map.keySet().removeIf(key -> StringUtils.isEmpty(key) || StringUtils.isEmpty(map.get(key)));
    // 将属性添加到服务元数据的附件中（供后续流程使用）
    serviceMetadata.getAttachments().putAll(map);
    // 根据协议配置和属性构建服务URL
    URL url = buildUrl(protocolConfig, map);
    // 处理服务执行器（如设置线程池、超时时间等执行参数）
    processServiceExecutor(url);
    // 导出URL到注册中心（核心注册逻辑）
    exportUrl(url, registryURLs, registerType);
    // 初始化服务方法的指标收集（如QPS、响应时间等监控数据）
    initServiceMethodMetrics(url);
}
```

`doExportUrlsFor1Protocol`宏观上的服务导出逻辑的流程为：

1. 构建参数映射（配置、注解、环境变量等）
2. 生成服务URL（核心标识）
3. 配置执行环境（线程池、超时等）
4. 执行导出操作（本地+远程）
5. 初始化监控指标（便于服务治理）

#### 参数构建

```java
private Map<String, String> buildAttributes(ProtocolConfig protocolConfig) {
    // 初始化参数映射表
    Map<String, String> map = new HashMap<>();
    // 标记服务角色为提供者（与消费者区分）
    map.put(SIDE_KEY, PROVIDER_SIDE);
    // 1. 添加基础配置参数
    // 追加运行时参数（如Dubbo版本、框架标识等）
    ServiceConfig.appendRuntimeParameters(map);
    // 追加应用级配置参数（如应用名称、日志配置等）
    AbstractConfig.appendParameters(map, getApplication());
    // 追加模块级配置参数（如模块名称、全局超时等）
    AbstractConfig.appendParameters(map, getModule());
    // 追加服务提供者全局配置参数（如默认线程池、重试次数等）
    AbstractConfig.appendParameters(map, provider);
    // 追加协议配置参数（如协议类型、端口、序列化方式等）
    AbstractConfig.appendParameters(map, protocolConfig);
    // 追加服务自身配置参数（如接口名、版本、分组等）
    AbstractConfig.appendParameters(map, this);
    // 注：参数优先级规则：服务自身配置 > 协议配置 > 提供者配置 > 模块配置 > 应用配置
    // （后添加的参数会覆盖前面的同名参数）
    // 2. 追加方法级配置参数（如方法级超时、重试次数等）
    if (CollectionUtils.isNotEmpty(getMethods())) {
        getMethods().forEach(method -> appendParametersWithMethod(method, map));
    }
    // 3. 处理泛化调用配置
    if (isGeneric(generic)) {
        // 泛化调用模式：允许调用方无需接口类直接调用
        map.put(GENERIC_KEY, generic); // 标记为泛化调用（如"true"、"native"等类型）
        map.put(METHODS_KEY, ANY_VALUE); // 泛化调用支持任意方法，用"*"表示
    } else {
        // 非泛化调用：需明确接口方法信息
        // 获取接口的版本修订号（用于服务版本管理，如接口变更时更新）
        String revision = Version.getVersion(interfaceClass, version);
        if (StringUtils.isNotEmpty(revision)) {
            map.put(REVISION_KEY, revision); // 存储修订号，用于服务治理（如灰度发布）
        }
        // 提取接口中定义的所有方法名
        String[] methods = methods(interfaceClass);
        if (methods.length == 0) {
            // 接口无方法时的警告日志
            logger.warn("/*...*/"); // 无方法时默认支持任意方法（实际不会触发）
        } else {
            // 存储方法列表（去重并排序），用于服务调用校验
            map.put(METHODS_KEY, StringUtils.join(new TreeSet<>(Arrays.asList(methods)), COMMA_SEPARATOR));
        }
    }
    // 4. 处理服务调用的token验证配置
    // 如果当前服务未配置token，尝试从全局提供者配置中获取
    if (ConfigUtils.isEmpty(token) && provider != null) {
        token = provider.getToken();
    }
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            // 若token为默认值，生成UUID作为随机token（用于简单身份验证）
            map.put(TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            // 使用配置的固定token（用于自定义身份验证逻辑）
            map.put(TOKEN_KEY, token);
        }
    }
    // 5. 处理原生服务存根配置（若服务实现为ServerService）
    if (ref instanceof ServerService) {
        // 设置代理类型为原生存根（可能用于与非Java客户端的交互优化）
        map.put(PROXY_KEY, CommonConstants.NATIVE_STUB);
    }
    return map;
}
```

该方法通过多轮参数合并，构建服务导出的完整属性集，优先级从低到高为：

```plaintext
运行时参数 → 应用配置 → 模块配置 → 提供者全局配置 → 协议配置 → 服务自身配置 → 方法配置
```

**后添加的参数会覆盖同名的前置参数**，确保更具体的配置（如服务级）能覆盖全局配置（如应用级）。

#### 生成URL

```java
private URL buildUrl(ProtocolConfig protocolConfig, Map<String, String> params) {
    // 1. 确定协议名称（默认使用dubbo协议）
    String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO; // 若未配置协议名称，默认使用dubbo协议
    }
    // 2. 确定服务暴露的主机地址（支持IPv6）
    // 从协议配置、提供者配置或参数中查找配置的主机地址
    String host = findConfiguredHosts(protocolConfig, provider, params);
    // 处理IPv6地址格式：标准URL中IPv6地址需用方括号包裹（如[fe80::1]）
    if (NetUtils.isIPV6URLStdFormat(host)) {
        if (!host.contains("[")) {
            host = "[" + host + "]"; // 补全IPv6地址的方括号
        }
    } 
    // 若存在本地IPv6地址，将其添加到参数中（供后续网络处理使用）
    else if (NetUtils.getLocalHostV6() != null) {
        String ipv6Host = NetUtils.getLocalHostV6();
        params.put(CommonConstants.IPV6_KEY, ipv6Host);
    }
    // 3. 确定服务暴露的端口
    // 从协议配置、提供者配置或协议扩展中查找端口（支持默认端口，如dubbo默认20880）
    Integer port = findConfiguredPort(
        protocolConfig, 
        provider, 
        this.getExtensionLoader(Protocol.class), // 协议扩展加载器（用于获取默认端口）
        name, // 协议名称
        params
    );
    // 4. 构建服务路径（结合上下文路径和服务基础路径）
    // 上下文路径由协议配置指定（如/webapp），服务基础路径通常为接口名
    String servicePath = getContextPath(protocolConfig)
        .map(contextPath -> contextPath + "/" + path) // 拼接上下文路径和基础路径
        .orElse(path); // 无上下文路径时直接使用基础路径
    // 5. 创建基础服务URL对象
    // ServiceConfigURL是Dubbo自定义的URL实现，包含服务元数据
    URL url = new ServiceConfigURL(
        name,           // 协议名称（如dubbo）
        null,           // 用户名（预留，暂未使用）
        null,           // 密码（预留，暂未使用）
        host,           // 主机地址（支持IPv6）
        port,           // 端口号
        servicePath,    // 服务路径（用于服务路由）
        params          // 服务参数（由buildAttributes构建的键值对）
    );
    // 6. 通过扩展点自定义URL配置（支持动态调整URL参数）
    // 检查是否存在当前协议对应的ConfiguratorFactory扩展（用于自定义URL配置）
    if (this.getExtensionLoader(ConfiguratorFactory.class).hasExtension(url.getProtocol())) {
        // 获取配置器并应用自定义配置（如动态添加参数、修改地址等）
        url = this.getExtensionLoader(ConfiguratorFactory.class)
            .getExtension(url.getProtocol()) // 获取协议对应的配置器工厂
            .getConfigurator(url) // 创建配置器
            .configure(url); // 应用配置到URL
    }
    // 7. 关联URL与作用域模型、服务模型
    // 作用域模型（ScopeModel）用于环境隔离（如应用、模块级别）
    url = url.setScopeModel(getScopeModel());
    // 服务模型（ProviderModel）关联服务实现和元数据（供服务治理使用）
    url = url.setServiceModel(providerModel);

    return url;
}
```

`buildUrl`方法的核心是将各种配置（协议、主机、端口、参数等）组装成 Dubbo 服务的唯一标识 URL。这个 URL 会被注册到注册中心，供消费者发现和调用。

#### 导出URL

```java
private void exportUrl(URL url, List<URL> registryURLs, RegisterTypeEnum registerType) {
    // 获取scope参数（控制服务导出范围：local/remote/none）
    String scope = url.getParameter(SCOPE_KEY);
    // 1. 若scope为none，不执行任何导出操作
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {
        // 2. 导出到本地（仅当scope不是remote时）
        // SCOPE_REMOTE表示仅导出到远程，此时不导出本地；其他情况（local/none以外）需导出本地
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            exportLocal(url); // 本地导出（如Injvm协议，用于进程内调用优化）
        }
        // 3. 导出到远程（仅当scope不是local时）
        // SCOPE_LOCAL表示仅导出到本地，此时不导出远程；其他情况需导出远程
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            // 获取额外协议配置（支持同时导出多种协议，如dubbo+http）
            String extProtocol = url.getParameter(EXT_PROTOCOL, "");
            List<String> protocols = new ArrayList<>();

            // 处理原始URL（主协议）的参数调整（若有额外协议）
            if (StringUtils.isNotBlank(extProtocol)) {
                // 为原始URL添加PU服务器标记（可能用于协议转换场景，如Protobuf-Unknown协议）
                url = URLBuilder.from(url)
                        .addParameter(IS_PU_SERVER_KEY, Boolean.TRUE.toString())
                        .build();
            }
            // 导出主协议URL到远程注册中心
            url = exportRemote(url, registryURLs, registerType);
            // 发布服务元数据到元数据中心（非泛化调用且非内部模块时需要）
            // 元数据包含接口定义、方法签名等，用于服务发现和治理
            if (!isGeneric(generic) && !getScopeModel().isInternal()) {
                MetadataUtils.publishServiceDefinition(
                        url, 
                        providerModel.getServiceModel(), 
                        getApplicationModel()
                );
            }
            // 处理额外协议（如配置了extProtocol="http,grpc"）
            if (StringUtils.isNotBlank(extProtocol)) {
                // 拆分多个额外协议（支持逗号分隔）
                String[] extProtocols = extProtocol.split(",", -1);
                protocols.addAll(Arrays.asList(extProtocols));
            }
            // 逐个导出额外协议的URL
            for (String protocol : protocols) {
                if (StringUtils.isNotBlank(protocol)) {
                    // 构建额外协议的URL（基于主URL调整协议类型和参数）
                    URL localUrl = URLBuilder.from(url)
                            .setProtocol(protocol) // 设置为当前额外协议（如http）
                            .addParameter(IS_EXTRA, Boolean.TRUE.toString()) // 标记为额外协议
                            .removeParameter(EXT_PROTOCOL) // 移除额外协议参数，避免递归处理
                            .build();
                    
                    // 导出额外协议URL到远程注册中心
                    localUrl = exportRemote(localUrl, registryURLs, registerType);
                    
                    // 发布额外协议的服务元数据
                    if (!isGeneric(generic) && !getScopeModel().isInternal()) {
                        MetadataUtils.publishServiceDefinition(
                                localUrl, 
                                providerModel.getServiceModel(), 
                                getApplicationModel()
                        );
                    }
                    // 记录额外协议导出的URL
                    this.urls.add(localUrl);
                }
            }
        }
    }
    // 记录主协议导出的URL（无论是否导出，均保留记录）
    this.urls.add(url);
}
```

上面代码根据 url 中的 scope 参数决定服务导出的目标范围：

- **`SCOPE_NONE`**：不导出（既不本地也不远程），用于临时禁用服务。
- **`SCOPE_LOCAL`**：仅导出到本地（通过 Injvm 协议），供进程内调用，不注册到远程注册中心。
- **`SCOPE_REMOTE`**：仅导出到远程注册中心，供跨进程调用，不支持本地调用。
- **默认（无 scope 或其他值）**：同时导出到本地和远程，兼顾本地优化和跨进程调用。

在之前的服务目录中，我们指导Invoker即为每个服务提供者的信息，通过注册中心使其全局可见。那么接下来就接着看导出到远程注册中的过程。

##### 导出URL到本地

先来看相对简单的导出到本地。本地导出针对`Injvm`协议，优化进程内调用性能：

```java
private void exportLocal(URL url) {
    // 1. 构建本地服务URL（基于原URL调整协议、主机和端口）
    URL localUrl = URLBuilder.from(url)
            .setProtocol(LOCAL_PROTOCOL) // 协议设置为本地协议（injvm）
            .setHost(LOCALHOST_VALUE)    // 主机固定为本地回环地址（127.0.0.1）
            .setPort(0)                  // 端口设为0（本地协议无需端口）
            .build();
    // 2. 关联作用域模型和服务模型（用于环境隔离和元数据管理）
    localUrl = localUrl.setScopeModel(getScopeModel()).setServiceModel(providerModel);
    // 3. 添加本地协议的导出监听器参数（标识为本地导出）
    localUrl = localUrl.addParameter(EXPORTER_LISTENER_KEY, LOCAL_PROTOCOL);
    // 4. 执行本地导出逻辑（不携带元数据，使用自动注册类型）
    doExportUrl(localUrl, false, RegisterTypeEnum.AUTO_REGISTER);
    // 5. 记录本地导出日志
    logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + localUrl);
}
```

```java
private void doExportUrl(URL url, boolean withMetaData, RegisterTypeEnum registerType) {
    // 1. 根据URL中的注册开关调整注册类型
    // 若URL中register参数为false（显式禁止注册），强制使用手动注册类型
    if (!url.getParameter(REGISTER_KEY, true)) {
        registerType = RegisterTypeEnum.MANUAL_REGISTER;
    }
    // 2. 对特定注册类型强制关闭自动注册
    // （NEVER_REGISTER：从不注册；MANUAL_REGISTER：手动注册；AUTO_REGISTER_BY_DEPLOYER：部署器自动注册）
    if (registerType == RegisterTypeEnum.NEVER_REGISTER
            || registerType == RegisterTypeEnum.MANUAL_REGISTER
            || registerType == RegisterTypeEnum.AUTO_REGISTER_BY_DEPLOYER) {
        url = url.addParameter(REGISTER_KEY, false);
    }
    // 3. 创建服务调用器Invoker（核心步骤：将服务实现转换为可调用的抽象）
    // proxyFactory通过Javassist或JDK动态代理生成Invoker，封装服务调用逻辑
    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
    // 4. 若需要元数据，包装Invoker以携带服务元数据（如接口定义、方法签名）
    if (withMetaData) {
        invoker = new DelegateProviderMetaDataInvoker(invoker, this);
    }
    // 5. 通过协议扩展点导出服务（生成Exporter管理服务生命周期）
    // protocolSPI根据URL协议（如injvm、dubbo）选择对应的协议实现
    Exporter<?> exporter = protocolSPI.export(invoker);
    // 6. 按注册类型缓存Exporter（便于后续服务销毁或管理）
    // exporters是线程安全的Map，键为注册类型，值为该类型对应的Exporter列表
    exporters
            .computeIfAbsent(registerType, k -> new CopyOnWriteArrayList<>())
            .add(exporter);
}
```

本地导出相对简单，主要是将协议转为`Injvm`，确保调用不经过网络层。若需导出，则创建一个新的 URL 并将协议头、主机名以及端口设置成新的值。然后创建 Invoker，并调用 `InjvmProtocol#export`方法导出服务。下面我们来看一下 `InjvmProtocol#export` 方法都做了哪些事情。

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 创建 InjvmExporter
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```

这里只是创建了一个`InjvmExporter`，在此不做深入了解。

##### 导出URL到注册中心

导出到注册中心更符合RPC的使用场景，现在来深入了解。

```java
private URL exportRemote(URL url, List<URL> registryURLs, RegisterTypeEnum registerType) {
    // 1. 处理注册中心场景（存在注册中心且注册类型不为NEVER_REGISTER）
    if (CollectionUtils.isNotEmpty(registryURLs) && registerType != RegisterTypeEnum.NEVER_REGISTER) {
        for (URL registryURL : registryURLs) {
            // 1.1 服务名称映射支持（针对服务注册协议，如service-registry）
            if (SERVICE_REGISTRY_PROTOCOL.equals(registryURL.getProtocol())) {
                url = url.addParameterIfAbsent(SERVICE_NAME_MAPPING_KEY, "true");
            }
            // 1.2 跳过本地协议（Injvm）的注册（本地协议仅用于进程内调用，无需注册）
            if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                continue;
            }
            // 1.3 添加动态注册参数（从注册中心URL继承dynamic参数，控制服务是否动态注册）
            url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
            // 1.4 加载监控中心配置（若配置了监控中心，将监控URL存入服务URL的属性）
            URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
            if (monitorUrl != null) {
                url = url.putAttribute(MONITOR_KEY, monitorUrl);
            }
            // 1.5 传递代理配置（若服务配置了自定义代理，传递给注册中心URL）
            String proxy = url.getParameter(PROXY_KEY);
            if (StringUtils.isNotEmpty(proxy)) {
                registryURL = registryURL.addParameter(PROXY_KEY, proxy);
            }
            // 1.6 日志记录（区分注册和导出操作）
            if (logger.isInfoEnabled()) {
                if (url.getParameter(REGISTER_KEY, true)) {
                    logger.info("/*...*/");
                } else {
                    logger.info("/*...*/");
                }
            }
            // 1.7 执行实际的导出和注册逻辑
            // 将服务URL存入注册中心URL的属性，并指定是否注册和注册类型
            doExportUrl(registryURL.putAttribute(EXPORT_KEY, url), true, registerType);
        }

    // 2. 处理无注册中心场景（直接导出服务，可能仅用于本地测试或元数据注册）
    } else {
        if (logger.isInfoEnabled()) {
            logger.info("/*...*/");
        }
        // 无注册中心时，直接导出服务URL（不进行注册中心注册）
        doExportUrl(url, true, registerType);
    }
    return url;
}
```

最终通过协议扩展点（如`DubboProtocol`）将服务导出为可远程调用的`Exporter`，并根据注册类型决定是否将服务 URL 注册到注册中心。无论本地还是远程导出，最终都会调用`doExportUrl`方法：

```java
private void doExportUrl(URL url, boolean withMetaData, RegisterTypeEnum registerType) {
    // 1. 处理注册参数（根据注册类型调整URL参数）
    // 若URL中register参数为false（显式禁止注册），强制设置注册类型为手动注册
    if (!url.getParameter(REGISTER_KEY, true)) {
        registerType = RegisterTypeEnum.MANUAL_REGISTER;
    }
    // 对于不自动注册的类型（如NEVER_REGISTER、MANUAL_REGISTER），
    // 设置register参数为false，确保不会自动注册到注册中心
    if (registerType == RegisterTypeEnum.NEVER_REGISTER
            || registerType == RegisterTypeEnum.MANUAL_REGISTER
            || registerType == RegisterTypeEnum.AUTO_REGISTER_BY_DEPLOYER) {
        url = url.addParameter(REGISTER_KEY, false);
    }
    // 2. 创建服务调用器Invoker（将服务实现转换为可调用的抽象）
    // proxyFactory是SPI扩展点，默认实现为JavassistProxyFactory
    // ref是服务实现实例（如UserServiceImpl），interfaceClass是服务接口（如UserService）
    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
    // 3. 包装元数据Invoker（若需要携带服务元数据）
    // DelegateProviderMetaDataInvoker用于在服务调用时传递提供者元数据（如接口方法信息）
    if (withMetaData) {
        invoker = new DelegateProviderMetaDataInvoker(invoker, this);
    }
    // 4. 通过协议扩展点导出服务（将Invoker转换为可远程调用的Exporter）
    // protocolSPI是SPI扩展点，默认实现为DubboProtocol
    // export方法会启动服务端（如NettyServer）并注册服务到注册中心
    Exporter<?> exporter = protocolSPI.export(invoker);
    // 5. 记录导出的Exporter（按注册类型分类存储，便于后续管理）
    // exporters是ConcurrentHashMap<RegisterTypeEnum, List<Exporter<?>>>
    // CopyOnWriteArrayList保证线程安全，适用于读多写少的场景
    ConcurrentHashMapUtils.computeIfAbsent(exporters, registerType, k -> new CopyOnWriteArrayList<>())
            .add(exporter);
}
```

在这里，有一行代码很重要：

```java
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
```

通过`ProxyFactory `获取`Invoker`，并且由该接口的SPI注解可知，dubbo的默认实现为`JavassistProxyFactory`，再来看获取`Invoker`的逻辑：

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    try {
        // 1. 获取服务类的包装器（优先使用目标类，若为代理类则使用接口类）
        // Wrapper是Dubbo内部生成的工具类，用于高效调用对象方法
        // 处理类名包含'$'的情况（如匿名内部类、CGLIB代理类），此时使用接口类
        final Wrapper wrapper =
                Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        // 2. 创建基于包装器的Invoker实现
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments)
                    throws Throwable {
                // 通过包装器调用实际方法（避免反射调用，提升性能）
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    } catch (Throwable fromJavassist) {
        // 3. Javassist生成失败时，回退到JDK动态代理
        try {
            Invoker<T> invoker = jdkProxyFactory.getInvoker(proxy, type, url);
            logger.error("/*...*/");
            return invoker;
        } catch (Throwable fromJdk) {
            // 4. JDK动态代理也失败时，记录双重错误并抛出原始异常
            logger.error("/*...*/");
            logger.error("/*...*/");
            throw fromJavassist;
        }
    }
}
```

`JavassistProxyFactory` 创建了一个继承自 `AbstractProxyInvoker` 类的匿名对象，并覆写了抽象方法 `doInvoke`。覆写后的 `doInvoke` 逻辑比较简单，仅是将调用请求转发给了 `Wrapper` 类的 `invokeMethod `方法。`Wrapper` 用于“包裹”目标类，`Wrapper` 是一个抽象类，仅可通过 `getWrapper(Class)` 方法创建子类。在创建 `Wrapper` 子类的过程中，子类代码生成逻辑会对 `getWrapper` 方法传入的 Class 对象进行解析，拿到诸如类方法，类成员变量等信息。以及生成` invokeMethod `方法代码和其他一些方法代码。代码生成完毕后，通过 Javassist 生成 Class 对象，最后再通过反射创建 `Wrapper` 实例。相关的代码如下：

```java
public static Wrapper getWrapper(Class<?> c) {
    // 处理动态生成的类（如CGLIB代理类），获取其原始父类
    while (ClassGenerator.isDynamicClass(c)) {
        c = c.getSuperclass();
    }
    // 若目标类是Object，返回预设的Object包装器
    if (c == Object.class) {
        return OBJECT_WRAPPER;
    }
    // 从缓存获取Wrapper，若不存在则调用makeWrapper生成并缓存
    return ConcurrentHashMapUtils.computeIfAbsent(WRAPPER_MAP, c, Wrapper::makeWrapper);
}
```

`makeWrapper`方法是Dubbo性能优化的核心，它通过动态代码生成，实现了比反射更高效的方法调用：

````java
private static Wrapper makeWrapper(Class<?> c) {
    // 基本类型不支持生成Wrapper
    if (c.isPrimitive()) {
        throw new IllegalArgumentException("Can not create wrapper for primitive type: " + c);
    }
    String className = c.getName();
    ClassLoader classLoader = ClassUtils.getClassLoader(c);
    // 构建Wrapper类的三个核心方法源码：
    // 1. setPropertyValue：设置属性值（支持字段和setter方法）
    // 2. getPropertyValue：获取属性值（支持字段和getter方法）
    // 3. invokeMethod：调用目标类的方法
    StringBuilder setPropCode = new StringBuilder("public void setPropertyValue(Object o, String n, Object v){ ");
    StringBuilder getPropCode = new StringBuilder("public Object getPropertyValue(Object o, String n){ ");
    StringBuilder invokeMethodCode = new StringBuilder(
        "public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws "
        + InvocationTargetException.class.getName() + "{ "
    );
    // 类型转换：将输入的Object强制转换为目标类实例（避免反射，提升性能）
    setPropCode.append(className)
        .append(" w; try{ w = ((")
        .append(className)
        .append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    getPropCode.append(className)
        .append(" w; try{ w = ((")
        .append(className)
        .append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    invokeMethodCode.append(className)
        .append(" w; try{ w = ((")
        .append(className)
        .append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    // 存储属性信息（名称→类型）
    Map<String, Class<?>> propertyTypes = new HashMap<>();
    // 存储方法信息（方法签名→Method对象）
    Map<String, Method> methodMap = new LinkedHashMap<>();
    // 存储所有方法名和目标类声明的方法名
    List<String> allMethodNames = new ArrayList<>();
    List<String> declaredMethodNames = new ArrayList<>();
    // 1. 处理目标类的字段（生成字段访问的代码）
    for (Field field : c.getFields()) {
        String fieldName = field.getName();
        Class<?> fieldType = field.getType();
        // 跳过静态、 transient、final字段
        if (Modifier.isStatic(field.getModifiers())
            || Modifier.isTransient(field.getModifiers())
            || Modifier.isFinal(field.getModifiers())) {
            continue;
        }
        // 生成setPropertyValue代码：通过字段直接赋值
        setPropCode.append(" if( $2.equals(\"")
            .append(fieldName)
            .append("\") ){ ((")
            .append(field.getDeclaringClass().getName())
            .append(")w).")
            .append(fieldName)
            .append('=')
            .append(arg(fieldType, "$3")) // 处理参数类型转换（如将Object转为int）
            .append("; return; }");
        // 生成getPropertyValue代码：通过字段直接取值
        getPropCode.append(" if( $2.equals(\"")
            .append(fieldName)
            .append("\") ){ return ($w)((")
            .append(field.getDeclaringClass().getName())
            .append(")w).")
            .append(fieldName)
            .append("; }");
        propertyTypes.put(fieldName, fieldType);
    }
    // 获取类池（用于操作字节码）
    final ClassPool classPool = ClassGenerator.getClassPool(classLoader);
    // 收集目标类的所有方法（排除动态生成的方法）
    List<String> allMethodSignatures = new ArrayList<>();
    try {
        // 通过CtMethod获取所有方法签名（包含继承的方法）
        final CtMethod[] ctMethods = classPool.get(className).getMethods();
        for (CtMethod method : ctMethods) {
            allMethodSignatures.add(ReflectUtils.getDesc(method));
        }
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    // 过滤出目标类实际声明或继承的方法
    Method[] methods = Arrays.stream(c.getMethods())
        .filter(method -> allMethodSignatures.contains(ReflectUtils.getDesc(method)))
        .collect(Collectors.toList())
        .toArray(new Method[0]);
    boolean hasMethods = ClassUtils.hasMethods(methods);
    if (hasMethods) {
        // 统计同名方法数量（用于处理方法重载）
        Map<String, Integer> sameNameMethodCount = new HashMap<>((int) (methods.length / 0.75f) + 1);
        for (Method method : methods) {
            sameNameMethodCount.compute(method.getName(), (key, count) -> count == null ? 1 : count + 1);
        }
        invokeMethodCode.append(" try{");
        // 为每个方法生成调用代码
        for (Method method : methods) {
            // 跳过Object类的方法（如toString、hashCode）
            if (method.getDeclaringClass() == Object.class) {
                continue;
            }
            String methodName = method.getName();
            // 方法名匹配判断
            invokeMethodCode.append(" if( \"").append(methodName).append("\".equals( $2 ) ");
            int paramCount = method.getParameterTypes().length;
            // 参数数量匹配判断
            invokeMethodCode.append(" && ").append(" $3.length == ").append(paramCount);

            // 处理方法重载：参数类型需完全匹配
            boolean isOverload = sameNameMethodCount.get(methodName) > 1;
            if (isOverload) {
                if (paramCount > 0) {
                    for (int i = 0; i < paramCount; i++) {
                        invokeMethodCode.append(" && ")
                            .append(" $3[")
                            .append(i)
                            .append("].getName().equals(\"")
                            .append(method.getParameterTypes()[i].getName())
                            .append("\")");
                    }
                }
            }
            invokeMethodCode.append(" ) { ");
            // 生成方法调用代码（区分返回值是否为void）
            if (method.getReturnType() == Void.TYPE) {
                invokeMethodCode.append(" w.")
                    .append(methodName)
                    .append('(')
                    .append(args(method.getParameterTypes(), "$4")) // 生成参数列表（带类型转换）
                    .append(");")
                    .append(" return null;");
            } else {
                invokeMethodCode.append(" return ($w)w.")
                    .append(methodName)
                    .append('(')
                    .append(args(method.getParameterTypes(), "$4"))
                    .append(");");
            }
            invokeMethodCode.append(" }");
            // 记录方法名
            allMethodNames.add(methodName);
            if (method.getDeclaringClass() == c) {
                declaredMethodNames.add(methodName);
            }
            methodMap.put(ReflectUtils.getDesc(method), method);
        }
        // 异常处理：包装为InvocationTargetException
        invokeMethodCode.append(" } catch(Throwable e) { ");
        invokeMethodCode.append("     throw new java.lang.reflect.InvocationTargetException(e); ");
        invokeMethodCode.append(" }");
    }
    // 方法不存在时抛出异常
    invokeMethodCode.append(" throw new ")
        .append(NoSuchMethodException.class.getName())
        .append("(\"Not found method \\\"\"+$2+\"\\\" in class ")
        .append(className)
        .append(".\"); }");

    // 2. 处理getter/setter方法（补充属性访问逻辑）
    Matcher matcher;
    for (Map.Entry<String, Method> entry : methodMap.entrySet()) {
        String methodSignature = entry.getKey();
        Method method = entry.getValue();

        // 处理getter方法（如getName() → 属性name）
        if ((matcher = ReflectUtils.GETTER_METHOD_DESC_PATTERN.matcher(methodSignature)).matches()) {
            String propName = propertyName(matcher.group(1));
            getPropCode.append(" if( $2.equals(\"")
                .append(propName)
                .append("\") ){ return ($w)w.")
                .append(method.getName())
                .append("(); }");
            propertyTypes.put(propName, method.getReturnType());
        }
        // 处理is/has/can开头的getter（如isEnabled() → 属性enabled）
        else if ((matcher = ReflectUtils.IS_HAS_CAN_METHOD_DESC_PATTERN.matcher(methodSignature)).matches()) {
            String propName = propertyName(matcher.group(1));
            getPropCode.append(" if( $2.equals(\"")
                .append(propName)
                .append("\") ){ return ($w)w.")
                .append(method.getName())
                .append("(); }");
            propertyTypes.put(propName, method.getReturnType());
        }
        // 处理setter方法（如setName(String) → 属性name）
        else if ((matcher = ReflectUtils.SETTER_METHOD_DESC_PATTERN.matcher(methodSignature)).matches()) {
            Class<?> paramType = method.getParameterTypes()[0];
            String propName = propertyName(matcher.group(1));
            setPropCode.append(" if( $2.equals(\"")
                .append(propName)
                .append("\") ){ w.")
                .append(method.getName())
                .append('(')
                .append(arg(paramType, "$3")) // 参数类型转换
                .append("); return; }");
            propertyTypes.put(propName, paramType);
        }
    }

    // 属性不存在时抛出异常
    setPropCode.append(" throw new ")
        .append(NoSuchPropertyException.class.getName())
        .append("(\"Not found property \\\"\"+$2+\"\\\" field or setter method in class ")
        .append(className)
        .append(".\"); }");
    getPropCode.append(" throw new ")
        .append(NoSuchPropertyException.class.getName())
        .append("(\"Not found property \\\"\"+$2+\"\\\" field or getter method in class ")
        .append(className)
        .append(".\"); }");

    // 3. 动态生成Wrapper类字节码
    long wrapperId = WRAPPER_CLASS_COUNTER.getAndIncrement();
    ClassGenerator classGenerator = ClassGenerator.newInstance(classLoader);
    // 生成的Wrapper类名为：目标类名 + "DubboWrap" + 自增ID（避免类名冲突）
    classGenerator.setClassName(className + "DubboWrap" + wrapperId);
    classGenerator.setSuperClass(Wrapper.class); // 继承Wrapper抽象类
    // 添加默认构造函数
    classGenerator.addDefaultConstructor();
    // 添加静态字段存储元数据：
    // - pns：属性名数组
    // - pts：属性类型映射
    // - mns：所有方法名数组
    // - dmns：目标类声明的方法名数组
    // - mtsN：第N个方法的参数类型数组
    classGenerator.addField("public static String[] pns;");
    classGenerator.addField("public static " + Map.class.getName() + " pts;");
    classGenerator.addField("public static String[] mns;");
    classGenerator.addField("public static String[] dmns;");
    for (int i = 0, len = methodMap.size(); i < len; i++) {
        classGenerator.addField("public static Class[] mts" + i + ";");
    }
    // 添加Wrapper接口的抽象方法实现
    classGenerator.addMethod("public String[] getPropertyNames(){ return pns; }");
    classGenerator.addMethod("public boolean hasProperty(String n){ return pts.containsKey($1); }");
    classGenerator.addMethod("public Class getPropertyType(String n){ return (Class)pts.get($1); }");
    classGenerator.addMethod("public String[] getMethodNames(){ return mns; }");
    classGenerator.addMethod("public String[] getDeclaredMethodNames(){ return dmns; }");
    // 添加前面构建的三个核心方法
    classGenerator.addMethod(setPropCode.toString());
    classGenerator.addMethod(getPropCode.toString());
    classGenerator.addMethod(invokeMethodCode.toString());
    try {
        // 生成字节码并加载类
        Class<?> wrapperClass = classGenerator.toClass(c);
        // 初始化静态字段（元数据）
        wrapperClass.getField("pts").set(null, propertyTypes);
        wrapperClass.getField("pns").set(null, propertyTypes.keySet().toArray(new String[0]));
        wrapperClass.getField("mns").set(null, allMethodNames.toArray(new String[0]));
        wrapperClass.getField("dmns").set(null, declaredMethodNames.toArray(new String[0]));
        int index = 0;
        for (Method method : methodMap.values()) {
            wrapperClass.getField("mts" + index++).set(null, method.getParameterTypes());
        }
        // 实例化Wrapper并返回
        return (Wrapper) wrapperClass.getDeclaredConstructor().newInstance();
    } catch (RuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        // 释放资源
        classGenerator.release();
        propertyTypes.clear();
        methodMap.clear();
        allMethodNames.clear();
        declaredMethodNames.clear();
    }
}
````

这段代码是Dubbo性能优化的精髓，通过动态代码生成实现了比Java反射更高效的方法调用。对于一个服务接口方法，比如`User getUserById(Long id)`，生成的`invokeMethod`代码类似于：

```java
public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws InvocationTargetException {
    com.example.UserService w = (com.example.UserService) o;
    try {
        if ("getUserById".equals(n) && p.length == 1 && p[0].getName().equals("java.lang.Long")) {
            return (Object) w.getUserById((Long) v[0]);
        }
        // 其他方法...
    } catch (Throwable e) {
        throw new InvocationTargetException(e);
    }
    throw new NoSuchMethodException(...);
}
```

这种直接调用的方式性能接近原生方法调用，远优于反射。

与导出服务到本地相比，导出服务到远程的过程更加复杂。接下来，我们深入分析服务导出过程中的关键环节 - `RegistryProtocol`的`export`方法：

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    // 1. 解析注册中心URL和服务提供者URL
    // 注册中心URL（如zookeeper://127.0.0.1:2181）
    URL registryUrl = getRegistryUrl(originInvoker);
    // 服务提供者URL（如dubbo://192.168.1.100:20880/com.example.UserService）
    URL providerUrl = getProviderUrl(originInvoker);
    // 2. 构建配置覆盖的订阅URL（用于订阅注册中心的配置覆盖规则）
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
    // 创建配置覆盖监听器（用于接收注册中心推送的配置更新）
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    // 3. 注册配置覆盖监听器（关联订阅URL和监听器，用于后续通知）
    Map<URL, Set<NotifyListener>> overrideListeners =
            getProviderConfigurationListener(overrideSubscribeUrl).getOverrideListeners();
    // 按订阅URL分组存储监听器，确保线程安全（使用ConcurrentHashSet）
    overrideListeners
            .computeIfAbsent(overrideSubscribeUrl, k -> new ConcurrentHashSet<>())
            .add(overrideSubscribeListener);
    // 4. 应用配置覆盖规则（用注册中心的配置覆盖服务本地配置）
    providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
    // 5. 执行本地服务导出（通过具体协议如DubboProtocol导出服务，返回可变更的Exporter包装器）
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);
    // 6. 获取注册中心实例并定制注册URL（添加注册中心相关参数）
    final Registry registry = getRegistry(registryUrl);
    final URL registeredProviderUrl = customizeURL(providerUrl, registryUrl);
    // 7. 决定是否需要注册服务（服务和注册中心均允许注册时才执行）
    boolean register = providerUrl.getParameter(REGISTER_KEY, true) 
            && registryUrl.getParameter(REGISTER_KEY, true);
    if (register) {
        // 将服务URL注册到注册中心（注册中心会记录服务地址，供消费者发现）
        register(registry, registeredProviderUrl);
    }
    // 8. 在服务模型中注册状态URL（用于监控和治理）
    registerStatedUrl(registryUrl, registeredProviderUrl, register);
    // 9. 初始化Exporter的注册和订阅信息
    exporter.setRegisterUrl(registeredProviderUrl);       // 记录注册到注册中心的URL
    exporter.setSubscribeUrl(overrideSubscribeUrl);      // 记录订阅配置覆盖的URL
    exporter.setNotifyListener(overrideSubscribeListener);// 关联配置覆盖监听器
    exporter.setRegistered(register);                     // 标记是否已注册
    // 10. 兼容旧版本配置订阅（2.6.x及之前版本，非服务发现模式下订阅配置覆盖规则）
    ApplicationModel applicationModel = getApplicationModel(providerUrl.getScopeModel());
    if (applicationModel
            .modelEnvironment()
            .getConfiguration()
            .convert(Boolean.class, ENABLE_26X_CONFIGURATION_LISTEN, true)) {
        if (!registry.isServiceDiscovery()) {
            // 向注册中心订阅配置覆盖规则，当配置变更时会触发overrideSubscribeListener
            registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        }
    }
    // 11. 通知服务导出完成（触发相关监听器）
    notifyExport(exporter);
    // 返回可销毁的Exporter包装器，确保每次导出都返回新实例
    return new DestroyableExporter<>(exporter);
}
```

以上代码主要做了：

1. 获取向注册中心注册的url
2. 向注册中心进行订阅 override 数据
3. 调用 doLocalExport 本地导出服务，也就是本地开启服务监听
4. 向注册中心注册服务
5. 创建并返回 DestroyableExporter

###### 本地导出服务

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
    // 1. 生成服务唯一标识（用于缓存和引用计数）
    // 格式通常为：接口名:版本:分组（如com.example.UserService:1.0.0:default）
    String providerUrlKey = getProviderUrlKey(originInvoker);
    // 注册中心URL的简化标识（用于区分不同注册中心配置）
    String registryUrlKey = getRegistryUrlKey(originInvoker);
    // 2. 创建Invoker代理（封装原始Invoker和最终生效的providerUrl）
    // 用于隔离原始Invoker和导出过程中的URL修改，保持原始Invoker不变
    Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
    // 3. 创建引用计数Exporter（支持多注册中心共享同一本地服务实例）
    // exporterFactory根据providerUrlKey缓存Exporter，避免重复导出相同服务
    // 当多个注册中心引用同一服务时，引用计数+1；全部取消引用时才真正销毁服务
    ReferenceCountExporter<?> exporter = exporterFactory.createExporter(
        providerUrlKey, 
        () -> protocol.export(invokerDelegate) // 延迟执行实际导出逻辑
    );
    // 4. 生成并缓存可变更的Exporter包装器（支持动态替换底层Exporter）
    // bounds结构：Map<providerUrlKey, Map<registryUrlKey, ExporterChangeableWrapper>>
    // 确保同一服务在不同注册中心配置下有独立的包装器，但共享同一本地Exporter
    return (ExporterChangeableWrapper<T>) bounds
        .computeIfAbsent(providerUrlKey, k -> new ConcurrentHashMap<>())
        .computeIfAbsent(
            registryUrlKey,
            s -> new ExporterChangeableWrapper<>((ReferenceCountExporter<T>) exporter, originInvoker)
        );
}
```

把重点放在 `Protocol#export`方法上。它根据URL协议类型选择相应的`Protocol`实现（如DubboProtocol）执行服务导出。

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    // 检查当前协议实例是否已销毁，若已销毁则抛出异常
    checkDestroyed();
    // 获取服务URL（包含协议、地址、参数等信息）
    URL url = invoker.getUrl();
    // 1. 生成服务唯一标识（格式：接口名:版本:分组:端口）
    String key = serviceKey(url);
    // 创建DubboExporter，关联invoker、服务标识和Exporter缓存
    DubboExporter<T> exporter = new DubboExporter<>(invoker, key, exporterMap);
    // 2. 处理存根服务的事件分发（用于消费者本地存根接收提供者事件）
    // 判断是否支持存根事件（默认false，需通过stub.event=true开启）
    boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
    // 判断是否为回调服务（用于处理消费者向提供者注册的回调接口）
    boolean isCallbackService = url.getParameter(IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackService) {
        // 获取存根服务中需要触发事件的方法列表（通过stub.event.methods指定）
        String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            // 若开启了存根事件但未指定方法，打印警告日志
            if (logger.isWarnEnabled()) {
                logger.warn("/*...*/");
            }
        }
    }
    // 3. 启动服务端（如NettyServer）监听指定端口，接收远程请求
    openServer(url);
    // 4. 优化序列化方式（根据URL参数调整序列化策略，提升性能）
    optimizeSerialization(url);
    // 返回封装后的Exporter，用于后续服务撤销（unexport）操作
    return exporter;
}
```

接着看`openServer`方法，`openServer`方法负责启动网络服务器，监听客户端连接：

```java
private void openServer(URL url) {
    // 检查协议实例是否已销毁，确保操作安全
    checkDestroyed();
    // 1. 生成服务端唯一标识（格式：host:port，如127.0.0.1:20880）
    String key = url.getAddress();
    // 2. 判断是否作为服务端（默认true，支持通过参数关闭）
    // 某些场景下，客户端也可暴露服务供其他服务调用（如回调服务）
    boolean isServer = url.getParameter(IS_SERVER_KEY, true);
    if (isServer) {
        // 3. 检查是否已存在该地址的服务端实例
        ProtocolServer server = serverMap.get(key);
        if (server == null) {
            synchronized (this) {
                server = serverMap.get(key);
                if (server == null) {
                    // 创建并启动新的服务端实例（如NettyServer）
                    serverMap.put(key, createServer(url));
                    return;
                }
            }
        }
        // 4. 服务端已存在时，重置配置（支持运行时动态修改服务端参数）
        // 例如：通过override机制动态调整线程池大小、心跳间隔等
        server.reset(url);
    }
}
```

`createServer`方法实际创建并启动Netty服务器：

```java
private ProtocolServer createServer(URL url) {
    // 1. 配置服务端默认参数（若URL中未显式指定）
    url = URLBuilder.from(url)
            // 服务端关闭时发送只读事件（默认启用），通知客户端连接即将关闭
            .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
            // 启用心跳机制（默认60秒），维持长连接活性
            .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
            // 指定编解码器为Dubbo协议专用编解码器
            .addParameter(CODEC_KEY, DubboCodec.NAME)
            .build();
    // 2. 校验服务端传输实现（如netty、mina）
    String transporter = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);
    if (StringUtils.isNotEmpty(transporter)
            && !url.getOrDefaultFrameworkModel()
                    .getExtensionLoader(Transporter.class)
                    .hasExtension(transporter)) {
        throw new RpcException("Unsupported server type: " + transporter + ", url: " + url);
    }
    // 3. 绑定服务端地址并启动监听（核心步骤）
    ExchangeServer server;
    try {
        // 使用Exchangers工具类根据URL协议和配置创建并绑定服务端
        // requestHandler负责处理接收到的请求（如解码、调用服务方法、编码响应）
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    // 4. 校验客户端传输实现（用于服务端主动调用客户端的场景，如回调）
    transporter = url.getParameter(CLIENT_KEY);
    if (StringUtils.isNotEmpty(transporter)
            && !url.getOrDefaultFrameworkModel()
                    .getExtensionLoader(Transporter.class)
                    .hasExtension(transporter)) {
        throw new RpcException("Unsupported client type: " + transporter);
    }
    // 5. 封装原生服务端为Dubbo协议服务端
    DubboProtocolServer protocolServer = new DubboProtocolServer(server);
    // 加载服务端属性（如线程池配置、连接管理策略等）
    loadServerProperties(protocolServer);
    return protocolServer;
}
```

Dubbo通过层层调用，最终创建基于Netty的高性能网络服务器：

```java
public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    // 若未指定编解码器，默认使用"exchange"编解码器
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // 获取对应的Exchanger实现并绑定（通过SPI机制，默认HeaderExchanger）
    return getExchanger(url).bind(url, handler);
}
```

```java
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    ExchangeServer server;
    // 判断是否启用端口统一模式（同一端口支持多种协议）
    boolean isPuServerKey = url.getParameter(IS_PU_SERVER_KEY, false);
    if (isPuServerKey) {
        // 端口统一模式：使用PortUnificationExchanger绑定，支持多协议自动识别
        server = new HeaderExchangeServer(
            PortUnificationExchanger.bind(
                url, 
                // 责任链：DecodeHandler（解码）→ HeaderExchangeHandler（消息交换）
                new DecodeHandler(new HeaderExchangeHandler(handler))
            )
        );
    } else {
        // 普通模式：通过Transporters绑定传输层，使用默认协议
        server = new HeaderExchangeServer(
            Transporters.bind(
                url, 
                // 责任链：DecodeHandler（解码二进制数据）→ HeaderExchangeHandler（处理业务逻辑）
                new DecodeHandler(new HeaderExchangeHandler(handler))
            )
        );
    }
    return server;
}
```

```java
public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handlers == null || handlers.length == 0) {
        throw new IllegalArgumentException("handlers == null");
    }
    // 若有多个处理器，包装为ChannelHandlerDispatcher（分发请求到多个处理器）
    ChannelHandler handler;
    if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        handler = new ChannelHandlerDispatcher(handlers);
    }
    // 获取对应的Transporter实现并绑定（通过SPI机制，默认NettyTransporter）
    return getTransporter(url).bind(url, handler);
}

public RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException {
    // 创建NettyServer，绑定URL与处理器，启动Netty服务端
    return new NettyServer(url, handler);
}
```

```java
public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
    // 调用父类构造器，同时通过ChannelHandlers.wrap包装处理器
    // 包装逻辑：添加日志、监控、编码解码等通用拦截器
    super(url, ChannelHandlers.wrap(handler, url));
}

public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
    super(url, handler);
    // 获取线程池仓库实例（用于管理服务端线程池）
    executorRepository = ExecutorRepository.getInstance(url.getOrDefaultApplicationModel());
    // 本地服务地址（对外暴露的地址，如192.168.1.100:20880）
    localAddress = getUrl().toInetSocketAddress();

    // 解析绑定地址参数（支持指定绑定的IP和端口，默认使用对外暴露的地址）
    String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
    int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
    // 若配置ANYHOST（0.0.0.0）或绑定IP无效，自动绑定到所有网络接口
    if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
        bindIp = ANYHOST_VALUE; // ANYHOST_VALUE即"0.0.0.0"，表示监听所有网卡
    }
    // 最终绑定的地址（如0.0.0.0:20880）
    bindAddress = new InetSocketAddress(bindIp, bindPort);
    // 最大并发连接数（默认8192，通过accepts参数配置）
    this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);

    try {
        // 调用子类实现的doOpen()，启动具体的网络服务（如Netty）
        doOpen();
        // 打印启动日志，包含服务类型、绑定地址和暴露地址
        if (logger.isInfoEnabled()) {
            logger.info("/*...*/");
        }
    } catch (Throwable t) {
        // 启动失败时包装异常并抛出
        throw new RemotingException("/*...*/");
    }

    // 初始化服务端线程池（从线程池仓库获取或创建，线程名格式为"DubboServerHandler-端口-序号"）
    executors.add(
            executorRepository.createExecutorIfAbsent(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
}
```

Netty服务器创建与初始化：


```java
protected void doOpen() throws Throwable {
    // 1. 创建Netty服务端引导类（ServerBootstrap是Netty服务端的核心启动类）
    bootstrap = new ServerBootstrap();

    // 2. 创建Netty线程池（bossGroup负责接收客户端连接，workerGroup负责处理IO读写）
    bossGroup = createBossGroup();    // 主线程组（默认1个线程）
    workerGroup = createWorkerGroup();// 工作线程组（默认CPU核心数*2个线程）

    // 3. 创建Netty服务端处理器，管理所有客户端通道（Channel）
    final NettyServerHandler nettyServerHandler = createNettyServerHandler();
    // 保存通道集合，用于后续管理（如关闭所有连接）
    channels = nettyServerHandler.getChannels();

    // 4. 初始化ServerBootstrap（配置通道类型、选项、处理器等）
    initServerBootstrap(nettyServerHandler);

    // 5. 绑定服务地址并启动服务
    try {
        // 绑定到指定地址（如0.0.0.0:20880）
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        // 同步等待绑定完成（不响应中断，确保绑定操作完成）
        channelFuture.syncUninterruptibly();
        // 获取绑定后的通道（代表服务端监听通道）
        channel = channelFuture.channel();
    } catch (Throwable t) {
        // 绑定失败时关闭线程池等资源
        closeBootstrap();
        throw t;
    }
}
```

###### 连接到注册中心

本地导出服务完成后，最后一步是将服务注册到注册中心，使消费者能够发现它。首先根据先前生成的url获取注册中心实例

```java
protected Registry getRegistry(final URL registryUrl) {
    // 通过SPI机制加载RegistryFactory的自适应扩展实现（如ZookeeperRegistryFactory）
    RegistryFactory registryFactory = ScopeModelUtil.getExtensionLoader(
                    RegistryFactory.class, registryUrl.getScopeModel())
            .getAdaptiveExtension();
    // 委托工厂创建注册中心实例
    return registryFactory.getRegistry(registryUrl);
}

```

`RegistryFactory`实现通过SPI机制动态加载，根据协议类型（如zookeeper）创建相应的注册中心客户端：

```java
public Registry getRegistry(URL url) {
    if (registryManager == null) {
        throw new IllegalStateException("/*...*/");
    }

    // 若注册中心已销毁，返回默认的空实现（避免NPE）
    Registry defaultNopRegistry = registryManager.getDefaultNopRegistryIfDestroyed();
    if (null != defaultNopRegistry) {
        return defaultNopRegistry;
    }

    // 标准化URL（设置路径为RegistryService接口名，移除临时参数）
    url = URLBuilder.from(url)
            .setPath(RegistryService.class.getName())
            .addParameter(INTERFACE_KEY, RegistryService.class.getName())
            .removeParameter(TIMESTAMP_KEY)
            .removeAttribute(EXPORT_KEY)
            .removeAttribute(REFER_KEY)
            .build();

    // 生成缓存键（基于URL）
    String key = createRegistryCacheKey(url);
    Registry registry = null;
    // 是否强制检查注册中心可用性（默认true）
    boolean check = url.getParameter(CHECK_KEY, true) && url.getPort() != 0;

    // 加锁确保注册中心单例
    registryManager.getRegistryLock().lock();
    try {
        // 双重检查，防止并发创建
        defaultNopRegistry = registryManager.getDefaultNopRegistryIfDestroyed();
        if (null != defaultNopRegistry) {
            return defaultNopRegistry;
        }

        // 从缓存获取
        registry = registryManager.getRegistry(key);
        if (registry != null) {
            return registry;
        }

        // 创建新的注册中心实例（通过SPI/IOC）
        registry = createRegistry(url);
        if (check && registry == null) {
            throw new IllegalStateException("无法创建注册中心: " + url);
        }

        if (registry != null) {
            // 缓存新创建的注册中心
            registryManager.putRegistry(key, registry);
        }
    } catch (Exception e) {
        if (check) {
            throw new RuntimeException("/*...*/");
        } else {
            // 非强制检查时记录警告日志，允许后续重试
            LOGGER.warn("/*...*/");
        }
    } finally {
        registryManager.getRegistryLock().unlock();
    }

    return registry;
}

public Registry createRegistry(URL url) {
    // 创建ZooKeeper注册中心，依赖ZooKeeper传输器实现（如Curator框架）
    return new ZookeeperRegistry(url, zookeeperTransporter);
}
```

ZooKeeper注册中心实现：

```java
public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
    super(url);

    // 验证URL有效性
    if (url.isAnyHost()) {
        throw new IllegalStateException("/*...*/");
    }

    // 解析注册中心根路径（默认/dubbo）
    String group = url.getGroup(DEFAULT_ROOT);
    if (!group.startsWith(PATH_SEPARATOR)) {
        group = PATH_SEPARATOR + group;
    }

    this.root = group;
    // 连接ZooKeeper服务器（通过传输器创建ZooKeeper客户端）
    this.zkClient = zookeeperTransporter.connect(url);

    // 注册连接状态监听器，处理ZooKeeper连接状态变化
    this.zkClient.addStateListener((state) -> {
        if (state == StateListener.RECONNECTED) {
            // 重连成功（会话未过期），获取最新服务地址
            logger.warn("/*...*/");
            ZookeeperRegistry.this.fetchLatestAddresses();
        } else if (state == StateListener.NEW_SESSION_CREATED) {
            // 新会话创建（原会话已过期），重新注册和订阅
            logger.warn("/*...*/");
            try {
                ZookeeperRegistry.this.recover();
            } catch (Exception e) {
                logger.error("/*...*/");
            }
        } else if (state == StateListener.SESSION_LOST) {
            // 会话丢失，等待新会话创建后自动恢复
            logger.warn("/*...*/");
        }
        // 其他状态（如SUSPENDED、CONNECTED）暂不处理
        else if (state == StateListener.SUSPENDED) {

        } else if (state == StateListener.CONNECTED) {

        }
    });
}
```

ZooKeeper连接创建：

```java
public ZookeeperClient connect(URL url) {
    ZookeeperClient zookeeperClient;
    // 解析URL中的ZooKeeper地址列表: {[username:password@]address}
    List<String> addressList = getURLBackupAddress(url);
    
    // 尝试从缓存获取已连接的客户端（首次检查，无锁）
    if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null
            && zookeeperClient.isConnected()) {
        logger.info("从缓存中找到可用的ZooKeeper客户端，地址：" + url);
        return zookeeperClient;
    }
    
    // 加锁避免并发创建过多客户端连接
    synchronized (zookeeperClientMap) {
        // 二次检查缓存（防止加锁期间其他线程已创建客户端）
        if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null
                && zookeeperClient.isConnected()) {
            logger.info("从缓存中找到可用的ZooKeeper客户端，地址：" + url);
            return zookeeperClient;
        }
        
        // 缓存无可用客户端，创建新客户端
        zookeeperClient = createZookeeperClient(url);
        logger.info("缓存中未找到可用的ZooKeeper客户端，为URL创建新客户端：" + url);
        // 将新客户端写入缓存，供后续复用
        writeToClientMap(addressList, zookeeperClient);
    }
    return zookeeperClient;
}

public Curator5ZookeeperClient(URL url) {
    super(url);
    try {
        // 解析连接超时时间（默认3000ms）
        int timeout = url.getParameter(TIMEOUT_KEY, DEFAULT_CONNECTION_TIMEOUT_MS);
        // 解析会话超时时间（默认60000ms）
        int sessionExpireMs = url.getParameter(SESSION_KEY, DEFAULT_SESSION_TIMEOUT_MS);
        // 是否启用ZooKeeper集群地址追踪（默认true，支持动态感知集群地址变化）
        boolean ensembleTracker = url.getParameter(ZOOKEEPER_ENSEMBLE_TRACKER_KEY, DEFAULT_ENSEMBLE_TRACKER);
        
        // 构建Curator客户端
        CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                .connectString(url.getBackupAddress()) // 设置ZooKeeper连接字符串（如host1:port1,host2:port2）
                .retryPolicy(new RetryNTimes(1, 1000)) // 重试策略：最多重试1次，间隔1000ms
                .connectionTimeoutMs(timeout) // 连接超时时间
                .ensembleTracker(ensembleTracker) // 启用集群地址追踪
                .sessionTimeoutMs(sessionExpireMs); // 会话超时时间
        
        // 处理ZooKeeper认证信息（若URL包含用户名密码，格式：username:password@host:port）
        String userInformation = url.getUserInformation();
        if (userInformation != null && userInformation.length() > 0) {
            // 设置digest认证（ZooKeeper默认认证方式）
            builder = builder.authorization("digest", userInformation.getBytes());
            // 设置ACL权限（创建者拥有所有权限）
            builder.aclProvider(new ACLProvider() {
                @Override
                public List<ACL> getDefaultAcl() {
                    return ZooDefs.Ids.CREATOR_ALL_ACL;
                }
                @Override
                public List<ACL> getAclForPath(String path) {
                    return ZooDefs.Ids.CREATOR_ALL_ACL;
                }
            });
        }
        
        // 创建Curator客户端实例
        client = builder.build();
        // 注册连接状态监听器（处理连接建立、断开、重连等事件）
        client.getConnectionStateListenable().addListener(new CuratorConnectionStateListener(url));
        // 启动客户端（异步连接ZooKeeper）
        client.start();
        
        // 阻塞等待连接成功（最多等待timeout毫秒）
        boolean connected = client.blockUntilConnected(timeout, TimeUnit.MILLISECONDS);
        if (!connected) {
            // 连接失败，抛出异常并记录错误日志
            IllegalStateException illegalStateException =
                    new IllegalStateException("ZooKeeper连接失败，地址：" + url);
            logger.error(
                    CONFIG_FAILED_CONNECT_REGISTRY,
                    "ZooKeeper服务器离线",
                    "",
                    "连接ZooKeeper失败",
                    illegalStateException);
            throw illegalStateException;
        }
        
    } catch (Exception e) {
        // 其他异常（如参数错误、权限不足）封装后抛出
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

简单来说，`getRegistry`方法的作用是创建了一个连接到注册中心的客户端。

###### 创建节点

最后，将服务信息注册到ZooKeeper：


```java
// RegistryProtocol#register
private static void register(Registry registry, URL registeredProviderUrl) {
    // 获取应用部署器，用于统计服务刷新状态
    ApplicationDeployer deployer =
            registeredProviderUrl.getOrDefaultApplicationModel().getDeployer();
    try {
        // 增加服务刷新计数（用于判断服务是否处于刷新状态）
        deployer.increaseServiceRefreshCount();
        
        // 获取注册中心名称（用于指标统计）
        String registryName = Optional.ofNullable(registry.getUrl())
                .map(u -> u.getParameter(
                        RegistryConstants.REGISTRY_CLUSTER_KEY,
                        // 根据URL类型决定使用的名称（服务发现URL使用registry参数，否则使用协议名）
                        UrlUtils.isServiceDiscoveryURL(u) ? u.getParameter(REGISTRY_KEY) : u.getProtocol()))
                .filter(StringUtils::isNotEmpty)
                .orElse("unknown");
        
        // 发布注册事件到指标总线，统计注册耗时和结果
        MetricsEventBus.post(
                RegistryEvent.toRsEvent(
                        registeredProviderUrl.getApplicationModel(),
                        registeredProviderUrl.getServiceKey(),
                        1,  // 注册操作计数
                        Collections.singletonList(registryName)),
                () -> {
                    // 执行实际注册操作
                    registry.register(registeredProviderUrl);
                    return null;
                });
    } finally {
        // 减少服务刷新计数，标记刷新完成
        deployer.decreaseServiceRefreshCount();
    }
}
```

首先通过`RegistryProtocol#register`对服务调用等指标进行记录之后，最终通过`Registry#register`执行注册。之前我们提到过`FailbackRegistry`为默认的FailbackRegistry

```java
// FailbackRegistry#register
public void register(URL url) {
    // 检查注册中心是否接受该协议类型的服务（如ZooKeeper注册中心不接受http协议）
    if (!acceptable(url)) {
        logger.info("/*...*/");
        return;
    }
    
    // 调用父类注册逻辑（如添加注册状态标记）
    super.register(url);
    // 移除失败列表中的注册/注销记录（如果之前注册失败，现在重试成功）
    removeFailedRegistered(url);
    removeFailedUnregistered(url);
    
    try {
        // 执行具体注册逻辑（由子类实现，如ZooKeeper注册中心创建节点）
        doRegister(url);
    } catch (Exception e) {
        Throwable t = e;
        
        // 是否强制检查注册结果（默认true，启动时检查注册是否成功）
        boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                && url.getParameter(Constants.CHECK_KEY, true)
                && (url.getPort() != 0);
        
        // 是否跳过失败重试（如明确指定不重试的异常）
        boolean skipFailback = t instanceof SkipFailbackWrapperException;
        
        if (check || skipFailback) {
            // 强制检查或跳过重试时，直接抛出异常阻止服务启动
            if (skipFailback) {
                t = t.getCause();
            }
            throw new IllegalStateException("/*...*/");
        } else {
            // 非强制检查时记录错误日志，将注册操作加入失败列表等待重试
            logger.error("/*...*/");
        }
        
        // 记录失败的注册请求，用于定时重试
        addFailedRegistered(url);
    }
}
```

这里也是做完了参数判断之后委托给`doRegister`进行实际的注册。由于`doRegister`是个抽象方法，我们以`ZookeeperRegistry`为例。

```java
// ZookeeperRegistry#doRegister
public void doRegister(URL url) {
    try {
        // 检查注册中心是否已销毁
        checkDestroyed();
        // 将URL转换为ZooKeeper路径，并创建节点
        // 动态参数决定是否创建临时节点（默认true，服务下线时自动删除节点）
        zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true), true);
    } catch (Throwable e) {
        throw new RpcException("/*...*/");
    }
}
```



```java
public void create(String path, boolean ephemeral, boolean faultTolerant) {
    // 处理持久节点的存在性缓存（避免重复检查）
    if (!ephemeral) {
        if (persistentExistNodePath.contains(path)) {
            return;
        }
        if (checkExists(path)) {
            persistentExistNodePath.add(path);
            return;
        }
    }
    
    // 递归创建父节点
    int i = path.lastIndexOf('/');
    if (i > 0) {
        create(path.substring(0, i), false, true);
    }
    
    // 根据类型创建临时或持久节点
    if (ephemeral) {
        createEphemeral(path, faultTolerant);
    } else {
        createPersistent(path, faultTolerant);
        persistentExistNodePath.add(path);
    }
}

public void createEphemeral(String path, boolean faultTolerant) {
    try {
        // 使用Curator创建临时节点
        client.create().withMode(CreateMode.EPHEMERAL).forPath(path);
    } catch (NodeExistsException e) {
        if (faultTolerant) {
            // 节点已存在，可能是旧会话延迟删除导致，尝试删除并重试
            logger.info("/*...*/)");
            deletePath(path);
            createEphemeral(path, true);
        } else {
            logger.warn("/*...*/");
            throw new IllegalStateException(e.getMessage(), e);
        }
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

服务导出的流程可以总结为：本地启动netty服务器，在注册中心上创建节点。

##### 订阅 override 数据

在`RegistryProtocol#export`方法里有订阅 override 数据的逻辑，现在一起来看看这个方法

```java
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        URL registryUrl = getRegistryUrl(originInvoker);
        // url to export locally
        URL providerUrl = getProviderUrl(originInvoker);
        
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        ConcurrentHashMap<URL, Set<NotifyListener>> overrideListeners =
                getProviderConfigurationListener(overrideSubscribeUrl).getOverrideListeners();
        ConcurrentHashMapUtils.computeIfAbsent(overrideListeners, overrideSubscribeUrl, k -> new ConcurrentHashSet<>())
                .add(overrideSubscribeListener);

        /.../
        if (applicationModel
            .modelEnvironment()
            .getConfiguration()
            .convert(Boolean.class, ENABLE_26X_CONFIGURATION_LISTEN, true)) {
        if (!registry.isServiceDiscovery()) {
            // Deprecated! Subscribe to override rules in 2.6.x or before.
            registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        }
        /.../
    }
```

首先通过Invoker获得`overrideSubscribeUrl`，创建监听器后注册到注册中心。当收到服务变更通知时触发`OverrideListener#notify`方法：

```java
public synchronized void notify(List<URL> urls) {
    // 记录原始URL列表用于调试，包含所有推送的配置规则
    if (logger.isDebugEnabled()) {
        logger.debug("original override urls: " + urls);
    }

    // 筛选出与当前订阅URL匹配的配置规则
    // 匹配逻辑基于服务分组、应用名、接口名等维度
    List<URL> matchedUrls = getMatchedUrls(urls, subscribeUrl);
    if (logger.isDebugEnabled()) {
        logger.debug("subscribe url: " + subscribeUrl + ", override urls: " + matchedUrls);
    }

    // 若无匹配规则则直接返回，避免无效处理
    if (matchedUrls.isEmpty()) {
        return;
    }

    // 将匹配的URL转换为配置器链
    // classifyUrls方法使用UrlUtils::isConfigurator作为谓词筛选配置器类型URL
    // toConfigurators方法将URL解析为可执行的配置器实例
    // orElse确保转换失败时保留原有配置器，实现优雅降级
    this.configurators = Configurator.toConfigurators(classifyUrls(matchedUrls, UrlUtils::isConfigurator))
            .orElse(configurators);

    // 获取应用部署器，用于协调服务刷新流程
    ApplicationDeployer deployer =
            subscribeUrl.getOrDefaultApplicationModel().getDeployer();

    // 使用计数器模式保护服务刷新操作
    try {
        // 增加服务刷新计数，标记服务状态变更
        deployer.increaseServiceRefreshCount();
        // 执行实际的服务覆盖逻辑（模板方法，由子类实现）
        doOverrideIfNecessary();
    } finally {
        // 减少服务刷新计数，确保状态一致性
        deployer.decreaseServiceRefreshCount();
    }
}
```

实际的服务更新在`doOverrideIfNecessary`方法里

```java
public synchronized void doOverrideIfNecessary() {
    // 1. 获取原始Invoker，处理可能的代理包装
    final Invoker<?> invoker;
    if (originInvoker instanceof InvokerDelegate) {
        invoker = ((InvokerDelegate<?>) originInvoker).getInvoker();
    } else {
        invoker = originInvoker;
    }
    
    // 2. 获取服务元信息与导出上下文
    URL originUrl = RegistryProtocol.this.getProviderUrl(invoker); // 服务原始URL
    String providerUrlKey = getProviderUrlKey(originInvoker);      // 服务提供者标识
    String registryUrlKey = getRegistryUrlKey(originInvoker);      // 注册中心标识
    
    // 3. 获取当前服务导出器
    Map<String, ExporterChangeableWrapper<?>> exporterMap = bounds.get(providerUrlKey);
    if (exporterMap == null) {
        logger.error(ERROR_CODE, "导出器映射为空，状态异常", new IllegalStateException());
        return;
    }
    
    ExporterChangeableWrapper<?> exporter = exporterMap.get(registryUrlKey);
    if (exporter == null) {
        logger.error(ERROR_CODE, "导出器为空，状态异常", new IllegalStateException());
        return;
    }
    
    // 4. 获取当前导出URL与配置合并后的新URL
    Invoker<?> exporterInvoker = exporter.getInvoker();
    URL currentUrl = exporterInvoker == null ? null : exporterInvoker.getUrl();
    
    // 5. 逐层应用配置覆盖规则（配置优先级：系统级 > 应用级 > 服务级）
    URL newUrl = getConfiguredInvokerUrl(configurators, originUrl); // 应用服务级配置
    newUrl = getConfiguredInvokerUrl(
        getProviderConfigurationListener(originUrl).getConfigurators(), newUrl); // 应用应用级配置
    newUrl = getConfiguredInvokerUrl(
        serviceConfigurationListeners.get(originUrl.getServiceKey()).getConfigurators(), newUrl); // 应用系统级配置
    
    // 6. 判断是否需要重新导出服务
    if (!newUrl.equals(currentUrl)) {
        if (newUrl.getParameter(Constants.NEED_REEXPORT, true)) {
            // 当配置变更超过阈值时触发服务重导出（关闭旧服务，导出新服务）
            RegistryProtocol.this.reExport(originInvoker, newUrl);
        }
        logger.info("服务导出URL变更: 原始[" + originUrl + "] → 旧[" + currentUrl + "] → 新[" + newUrl + "]");
    }
}
```

**服务重导出决策**：

- 比较合并后的新 URL 与当前导出 URL
- 通过`NEED_REEXPORT`参数控制是否需要物理重导出
- 记录详细的配置变更日志，便于问题追踪

```java
public <T> void reExport(final Invoker<T> originInvoker, URL newInvokerUrl) {
    // 获取服务标识和导出上下文
    String providerUrlKey = getProviderUrlKey(originInvoker);
    String registryUrlKey = getRegistryUrlKey(originInvoker);
    Map<String, ExporterChangeableWrapper<?>> registryMap = bounds.get(providerUrlKey);
    
    // 验证导出上下文是否存在
    if (registryMap == null || (exporter = (ExporterChangeableWrapper<T>) registryMap.get(registryUrlKey)) == null) {
        logger.error(INTERNAL_ERROR, "导出上下文缺失，无法重新导出服务", new IllegalStateException());
        return;
    }
    
    // 获取当前注册的URL和注册中心配置
    URL registeredUrl = exporter.getRegisterUrl();
    URL registryUrl = getRegistryUrl(originInvoker);
    
    // 定制新的服务提供者URL（应用注册中心特定配置）
    URL newProviderUrl = customizeURL(newInvokerUrl, registryUrl);

    // 1. 更新本地服务导出器
    Invoker<T> invokerDelegate = new InvokerDelegate<>(originInvoker, newInvokerUrl);
    exporter.setExporter(protocol.export(invokerDelegate));

    // 2. 更新注册中心（仅当新URL与已注册URL不同时）
    if (!newProviderUrl.equals(registeredUrl)) {
        try {
            // 执行注册中心更新操作（先取消注册旧URL，再注册新URL）
            doReExport(originInvoker, exporter, registryUrl, registeredUrl, newProviderUrl);
        } catch (Exception e) {
            // 注册中心更新失败时的处理逻辑
            ReExportTask oldTask = reExportFailedTasks.get(registeredUrl);
            if (oldTask != null) {
                return; // 已有重试任务，避免重复创建
            }
            
            // 创建重试任务，使用定时机制进行失败重试
            ReExportTask task = new ReExportTask(
                () -> doReExport(originInvoker, exporter, registryUrl, registeredUrl, newProviderUrl),
                registeredUrl,
                null);
                
            if (reExportFailedTasks.putIfAbsent(registeredUrl, task) == null) {
                // 启动定时重试（默认5秒后重试）
                retryTimer.newTimeout(
                    task,
                    registryUrl.getParameter(REGISTRY_RETRY_PERIOD_KEY, DEFAULT_REGISTRY_RETRY_PERIOD),
                    TimeUnit.MILLISECONDS);
            }
        }
    }
}
```

```java
private <T> void doReExport(
        final Invoker<T> originInvoker,
        ExporterChangeableWrapper<T> exporter,
        URL registryUrl,
        URL oldProviderUrl,
        URL newProviderUrl) {
    
    // 仅处理已注册的服务
    if (exporter.isRegistered()) {
        Registry registry;
        try {
            // 获取注册中心实例（支持多注册中心配置）
            registry = getRegistry(getRegistryUrl(originInvoker));
        } catch (Exception e) {
            // 注册中心获取失败，抛出可跳过回退的包装异常
            throw new SkipFailbackWrapperException(e);
        }

        // 1. 取消旧服务URL的注册
        logger.info("Try to unregister old url: " + oldProviderUrl);
        registry.reExportUnregister(oldProviderUrl);

        // 2. 注册新服务URL
        logger.info("Try to register new url: " + newProviderUrl);
        registry.reExportRegister(newProviderUrl);
    }
    
    // 3. 更新本地服务导出器的注册状态
    try {
        // 获取带注册状态的URL对象（包含注册中心和提供者信息）
        ProviderModel.RegisterStatedURL statedUrl = getStatedUrl(registryUrl, newProviderUrl);
        statedUrl.setProviderUrl(newProviderUrl);  // 更新提供者URL
        exporter.setRegisterUrl(newProviderUrl);   // 更新导出器的注册URL
    } catch (Exception e) {
        // 状态更新失败，抛出可跳过回退的包装异常
        throw new SkipFailbackWrapperException(e);
    }
}
```

```java
default void reExportRegister(URL url) {
    register(url);
}

default void reExportUnregister(URL url) {
    unregister(url);
}
```

## 总结

本篇文章详细分析了 Dubbo 服务导出过程，包括配置检测，URL 组装，Invoker 创建过程、导出服务以及注册服务等等。篇幅比较长，需要大家耐心阅读。
