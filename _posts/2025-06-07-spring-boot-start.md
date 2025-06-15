---
title: SpringBoot启动流程详解
date: 2025-06-07 12:00:00 +0800
categories: [spring boot]
tags: [源码阅读, 启动流程]
---

# SpringBoot启动流程详解

在微服务架构盛行的今天，SpringBoot已成为Java开发者构建应用的首选框架。它简化了Spring应用的初始搭建和开发过程，让开发者能够更专注于业务逻辑而非配置细节。本文将深入分析SpringBoot 2.7.18版本的启动流程，帮助读者理解其内部工作机制。

## 一、启动入口

每个SpringBoot应用都从一个带有`@SpringBootApplication`注解的主类开始：

```java
@SpringBootApplication
public class BootApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootApplication.class, args);
    }
}
```

这个看似简单的`run`方法，实际上隐藏了复杂的启动流程。让我们逐步剖析这个过程。

## 二、启动流程概览

SpringBoot的启动可以分为以下几个主要阶段：

1. **SpringApplication初始化**
2. **运行准备阶段**
3. **环境准备**
4. **应用上下文创建**
5. **刷新上下文**
6. **后置处理**

接下来，我们将详细分析每个阶段的源码实现。

## 三、源码分析

### 1. 启动入口方法

首先，让我们看看`SpringApplication.run`方法的调用链：

```java
// org.springframework.boot.SpringApplication#run(java.lang.Class<?>, java.lang.String...)
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class<?>[] { primarySource }, args);
}

// org.springframework.boot.SpringApplication#run(java.lang.Class<?>[], java.lang.String[])
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

这里可以看到，`run`方法首先创建了一个`SpringApplication`实例，然后调用了该实例的`run`方法。

### 2. SpringApplication初始化

`SpringApplication`的构造过程如下：

```java
// org.springframework.boot.SpringApplication#SpringApplication(java.lang.Class<?>...)
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

// org.springframework.boot.SpringApplication#SpringApplication(org.springframework.core.io.ResourceLoader, java.lang.Class<?>...)
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new ArrayList<>(
        getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    // 配置应用程序启动前的初始化对象
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 配置应用程序启动前的监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

在初始化过程中，SpringApplication完成了以下工作：

1. **设置资源加载器**：可以为null，后续会根据需要创建
2. **设置主要源**：通常是主配置类（带有@SpringBootApplication的类）
3. **推断Web应用类型**：根据classpath判断是SERVLET、REACTIVE还是NONE
4. **加载初始化器**：从spring.factories中加载BootstrapRegistryInitializer实例
5. **设置应用上下文初始化器**：从spring.factories中加载ApplicationContextInitializer实例
6. **设置应用监听器**：从spring.factories中加载ApplicationListener实例
7. **推断主应用类**：通过分析调用栈找到main方法所在的类

其中，`getSpringFactoriesInstances`方法是SpringBoot的核心机制之一，它通过SPI机制从classpath中的`META-INF/spring.factories`文件加载指定类型的实例：

```java
// org.springframework.boot.SpringApplication#getSpringFactoriesInstances(java.lang.Class<T>)
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

// org.springframework.boot.SpringApplication#getSpringFactoriesInstances(java.lang.Class<T>, java.lang.Class<?>[], java.lang.Object...)
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

#### 2.1 深入理解SPI机制

SPI（Service Provider Interface）是一种服务发现机制，允许程序在运行时发现和加载实现了特定接口的类。SpringBoot通过自己的SPI实现（`SpringFactoriesLoader`）来加载各种组件，这是SpringBoot自动配置和扩展的核心机制。

`SpringFactoriesLoader`的工作原理如下：

1. 搜索classpath中所有的`META-INF/spring.factories`文件
2. 读取这些文件中的键值对配置
3. 根据指定的接口类型（键）找到对应的实现类名（值）
4. 通过反射实例化这些实现类

以下是`SpringFactoriesLoader.loadFactoryNames`方法的核心实现：

```java
// org.springframework.core.io.support.SpringFactoriesLoader#loadFactoryNames
	public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}
		String factoryTypeName = factoryType.getName();
		return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
	}

// org.springframework.core.io.support.SpringFactoriesLoader#loadSpringFactories
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    // 尝试从缓存中获取
    Map<String, List<String>> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    result = new HashMap<>();
    try {
        // 查找所有META-INF/spring.factories文件
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                String[] factoryImplementationNames = 
                    StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                for (String factoryImplementationName : factoryImplementationNames) {
                    result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                          .add(factoryImplementationName.trim());
                }
            }
        }
        
        // 对结果进行排序和缓存
        result.replaceAll((factoryType, implementations) -> implementations.stream()
                .distinct()
                .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
        cache.put(classLoader, result);
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" + 
                                          FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
    return result;
}
```

这种机制使得SpringBoot能够在不需要显式配置的情况下，自动发现并加载各种组件，如自动配置类、初始化器、监听器等。

### 3. 运行阶段

初始化完成后，SpringApplication实例调用`run`方法启动应用：

```java
// org.springframework.boot.SpringApplication#run(java.lang.String...)
public ConfigurableApplicationContext run(String... args) {
    long startTime = System.nanoTime();
    // 创建引导上下文
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    ConfigurableApplicationContext context = null;
    // 配置headless属性
    configureHeadlessProperty();
    // 获取并启动应用监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        // 创建应用上下文
        context = createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
        // 准备应用上下文
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 刷新上下文
        refreshContext(context);
        // 后置处理
        afterRefresh(context, applicationArguments);
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
        }
        listeners.started(context, timeTakenToStartup);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }
    try {
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        listeners.ready(context, timeTakenToReady);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

这个方法是SpringBoot启动流程的核心，我们可以将其分解为以下几个关键步骤：

#### 3.1 创建引导上下文

```java
DefaultBootstrapContext bootstrapContext = createBootstrapContext();
```

引导上下文是一个轻量级的上下文，用于在主应用上下文创建前存储对象。

#### 3.2 配置headless属性

```java
configureHeadlessProperty();
```

这一步设置`java.awt.headless`系统属性，用于在没有显示设备、键盘或鼠标的环境中运行Java应用。

```java
// org.springframework.boot.SpringApplication#configureHeadlessProperty
private void configureHeadlessProperty() {
    System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
                       System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}
```

#### 3.3 获取并启动应用监听器

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting(bootstrapContext, this.mainApplicationClass);
```

这一步获取所有的`SpringApplicationRunListener`实例，并通知它们应用正在启动。

```java
// org.springframework.boot.SpringApplicationRunListeners#starting
void starting(ConfigurableBootstrapContext bootstrapContext, Class<?> mainApplicationClass) {
    doWithListeners("spring.boot.application.starting", (listener) -> listener.starting(bootstrapContext),
                    (step) -> {
                        if (mainApplicationClass != null) {
                            step.tag("mainApplicationClass", mainApplicationClass.getName());
                        }
                    });
}
```

#### 3.4 准备环境

```java
ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
```

这一步创建并配置`Environment`，加载应用的配置属性（如application.properties或application.yml）。

```java
// org.springframework.boot.SpringApplication#prepareEnvironment
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                   DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
    // 创建并配置环境
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(bootstrapContext, environment);
    DefaultPropertiesPropertySource.moveToEnd(environment);
    Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
                 "Environment prefix cannot be set via properties.");
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
        environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

#### 3.5 打印Banner

```java
Banner printedBanner = printBanner(environment);
```

这一步在控制台打印SpringBoot的Banner。

#### 3.6 创建应用上下文

```java
context = createApplicationContext();
```

根据应用类型（SERVLET、REACTIVE或NONE）创建相应的应用上下文。

```java
// org.springframework.boot.SpringApplication#createApplicationContext
protected ConfigurableApplicationContext createApplicationContext() {
    return this.applicationContextFactory.create(this.webApplicationType);
}
```

#### 3.7 准备上下文

```java
prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
```

这一步为应用上下文做准备工作，包括设置环境、应用初始化器、注册单例Bean等。

```java
// org.springframework.boot.SpringApplication#prepareContext
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
                            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
                            ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
    listeners.contextPrepared(context);
    bootstrapContext.close(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // 添加特定的单例Bean
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
        ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
        if (beanFactory instanceof DefaultListableBeanFactory) {
            ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    // 加载源
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
```

#### 3.8 刷新上下文

```java
refreshContext(context);
```

这是Spring核心的步骤，完成Bean的实例化、依赖注入、初始化等工作。

```java
// org.springframework.boot.SpringApplication#refreshContext
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
        shutdownHook.registerApplicationContext(context);
    }
    refresh(context);
}
```

##### 3.8.1 深入解析Context刷新原理

`refresh`方法是Spring容器的核心方法，它完成了以下工作：

```java
// org.springframework.context.support.AbstractApplicationContext#refresh
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // 准备刷新上下文
        prepareRefresh();

        // 告诉子类刷新内部Bean工厂
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 准备Bean工厂，以便在此上下文中使用
        prepareBeanFactory(beanFactory);

        try {
            // 允许在上下文子类中对Bean工厂进行后处理
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // 调用在上下文中注册为Bean的工厂处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册拦截Bean创建的Bean处理器
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();

            // 为此上下文初始化消息源
            initMessageSource();

            // 为此上下文初始化事件多播器
            initApplicationEventMulticaster();

            // 在特定上下文子类中初始化其他特殊Bean
            onRefresh();

            // 检查监听器Bean并注册它们
            registerListeners();

            // 实例化所有剩余的（非延迟初始化）单例
            finishBeanFactoryInitialization(beanFactory);

            // 最后一步：发布相应的事件
            finishRefresh();
        }
        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("在上下文初始化期间遇到异常 - " +
                            "取消刷新尝试: " + ex);
            }

            // 销毁已创建的单例以避免资源悬挂
            destroyBeans();

            // 重置'active'标志
            cancelRefresh(ex);

            // 将异常传播给调用者
            throw ex;
        }
        finally {
            // 重置Spring核心中的常见内省缓存，因为我们可能不再需要单例Bean的元数据...
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

让我们详细分析这个过程中的关键步骤：

###### (1) prepareRefresh()

准备刷新上下文，包括设置启动时间、活动标志、初始化属性源等。

```java
// org.springframework.context.support.AbstractApplicationContext#prepareRefresh
protected void prepareRefresh() {
    // 设置启动日期和活动标志
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    // 初始化属性源
    initPropertySources();

    // 验证所有必须存在的属性
    getEnvironment().validateRequiredProperties();

    // 存储早期的应用事件
    if (this.earlyApplicationEvents == null) {
        this.earlyApplicationEvents = new LinkedHashSet<>();
    }
    /.../
}
```

###### (2) obtainFreshBeanFactory()

获取一个新的或现有的BeanFactory。对于XML配置，这一步会加载XML文件中的Bean定义；对于注解配置，这一步相对简单。

```java
// org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}
```

###### (3) prepareBeanFactory(beanFactory)

准备Bean工厂，设置类加载器、表达式解析器、属性编辑器等，并注册一些特殊的Bean。

```java
// org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    
    // 设置表达式解析器
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    
    // 添加属性编辑器注册器
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加Bean后处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    
    // 忽略特定接口的自动装配
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 注册特殊依赖
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 注册早期后处理器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 检测LoadTimeWeaver并准备编织
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 注册默认环境Bean
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

###### (4) invokeBeanFactoryPostProcessors(beanFactory)

调用所有注册的`BeanFactoryPostProcessor`。这是Spring配置元数据处理的关键步骤，包括配置类的处理、Bean定义的加载等。

SpringBoot的自动配置主要是在这一步通过`ConfigurationClassPostProcessor`完成的。

```java
// org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
}
```

`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors`方法会按照优先级顺序执行所有的`BeanFactoryPostProcessor`，其中最重要的是`ConfigurationClassPostProcessor`，它负责处理`@Configuration`类，包括：

- 解析`@ComponentScan`注解并扫描包
- 解析`@Import`注解并导入配置
- 解析`@Bean`方法并注册Bean定义
- 处理`@PropertySource`注解加载属性文件
- 处理`@ImportResource`注解导入XML配置

在SpringBoot中，自动配置类就是通过`@EnableAutoConfiguration`注解和`@Import(AutoConfigurationImportSelector.class)`引入的。`AutoConfigurationImportSelector`使用SPI机制从`META-INF/spring.factories`中加载自动配置类。

###### (5) registerBeanPostProcessors(beanFactory)

注册所有的`BeanPostProcessor`，它们将在Bean实例化和初始化过程中被调用。

```java
// org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

###### (6) finishBeanFactoryInitialization(beanFactory)

完成Bean工厂的初始化，实例化所有非延迟加载的单例Bean。

```java
// org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化转换服务
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 注册嵌入值解析器
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }
    
    // 尽早初始化 LoadTimeWeaverAware bean，以便尽早注册其转换器。
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }
    // 停止使用临时 ClassLoader 进行类型匹配。
    beanFactory.setTempClassLoader(null);
	// 允许缓存所有 bean 定义元数据，而不期望进一步的更改。
    beanFactory.freezeConfiguration();
    // 预实例化单例
    beanFactory.preInstantiateSingletons();
}
```

其中，`preInstantiateSingletons`方法是实例化所有非延迟加载单例Bean的关键：

```java
// org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
public void preInstantiateSingletons() throws BeansException {
    // 获取所有Bean定义名称
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // 触发所有非延迟加载单例Bean的初始化
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                // 如果是FactoryBean，则加上"&"前缀获取工厂本身
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    FactoryBean<?> factory = (FactoryBean<?>) bean;
                    if (factory.isSingleton()) {
                        // 获取FactoryBean创建的对象
                        getBean(beanName);
                    }
                }
            }
            else {
                // 普通Bean直接获取
                getBean(beanName);
            }
        }
    }

    // 触发所有实现了SmartInitializingSingleton接口的Bean的afterSingletonsInstantiated方法
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            smartSingleton.afterSingletonsInstantiated();
        }
    }
}
```

这里的`getBean`方法会触发Bean的创建、依赖注入和初始化过程。

###### (7) finishRefresh()

完成刷新过程，发布上下文刷新事件。

```java
// org.springframework.context.support.AbstractApplicationContext#finishRefresh
protected void finishRefresh() {
    // 清除资源缓存
    clearResourceCaches();

    // 初始化生命周期处理器
    initLifecycleProcessor();

    // 调用生命周期处理器的onRefresh方法
    getLifecycleProcessor().onRefresh();

    // 发布上下文刷新事件
    publishEvent(new ContextRefreshedEvent(this));

    // 注册MBean
    LiveBeansView.registerApplicationContext(this);
}
```

通过以上步骤，Spring容器完成了Bean的定义、实例化、依赖注入和初始化，最终构建了一个完整的应用上下文。

#### 3.9 后置处理

```java
afterRefresh(context, applicationArguments);
```

这一步执行自定义的上下文刷新后逻辑，默认为空实现。

#### 3.10 启动完成通知

```java
Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
if (this.logStartupInfo) {
    new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
}
listeners.started(context, timeTakenToStartup);
```

这一步记录启动时间并通知监听器应用已启动。

#### 3.11 调用Runners

```java
callRunners(context, applicationArguments);
```

这一步调用所有实现了`ApplicationRunner`或`CommandLineRunner`接口的Bean。

#### 3.12 就绪通知

```java
Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
listeners.ready(context, timeTakenToReady);
```

这一步通知监听器应用已准备就绪，可以开始接收请求。

## 四、启动流程总结

SpringBoot的启动流程可以概括为以下几个关键步骤：

1. **创建SpringApplication实例**
   - 设置资源加载器
   - 推断应用类型
   - 加载应用上下文初始化器
   - 加载应用监听器
   - 推断主应用类

2. **运行SpringApplication**
   - 创建引导上下文
   - 准备环境（加载配置文件）
   - 创建应用上下文
   - 准备上下文
   - 刷新上下文（实例化Bean、依赖注入等）
   - 执行Runners
   - 发布就绪事件

## 五、总结

通过对SpringBoot启动流程的分析，我们可以看到SpringBoot通过精心设计的启动机制，实现了"约定大于配置"的理念，大大简化了Spring应用的开发。其核心特性包括：

1. **自动配置**：通过SPI机制和条件注解，根据classpath中的依赖自动配置应用
2. **外部化配置**：支持多种配置源，如属性文件、YAML文件、环境变量等
3. **生产就绪特性**：内置度量、健康检查、外部化配置等

理解SpringBoot的启动流程，不仅有助于我们更好地使用SpringBoot，还能帮助我们在遇到问题时更快地定位和解决。同时，这些设计理念和模式也值得我们在自己的项目中借鉴和应用。

## 六、实践指南：编写自己的Spring Factory

了解了SpringBoot的启动流程和SPI机制后，我们可以利用这些知识来扩展SpringBoot的功能。以下是编写自己的Spring Factory的步骤：

### 6.1 创建自定义组件

首先，定义一个接口或抽象类作为扩展点：

```java
public interface MyService {
    void doSomething();
}
```

然后，实现这个接口：

```java
public class MyServiceImpl implements MyService {
    @Override
    public void doSomething() {
        System.out.println("Doing something...");
    }
}
```

### 6.2 创建自动配置类

创建一个自动配置类，用于注册你的组件：

```java
@Configuration
@ConditionalOnClass(MyService.class)
public class MyServiceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

这里使用了几个关键注解：
- `@Configuration`：标识这是一个配置类
- `@ConditionalOnClass`：只有当指定的类存在于classpath时，才应用这个配置
- `@ConditionalOnMissingBean`：只有当没有指定类型的Bean时，才创建这个Bean

### 6.3 注册到spring.factories

在项目的`src/main/resources/META-INF`目录下创建`spring.factories`文件，内容如下：

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.spud.demo.config.MyServiceAutoConfiguration
```

这样，当SpringBoot应用启动时，会自动加载并应用你的自动配置类。

### 6.4 使用条件注解控制自动配置

SpringBoot提供了丰富的条件注解，可以用来控制自动配置的应用条件：

- `@ConditionalOnClass`：当指定的类存在于classpath时
- `@ConditionalOnMissingClass`：当指定的类不存在于classpath时
- `@ConditionalOnBean`：当指定的Bean存在时
- `@ConditionalOnMissingBean`：当指定的Bean不存在时
- `@ConditionalOnProperty`：当指定的属性有特定值时
- `@ConditionalOnResource`：当指定的资源存在时
- `@ConditionalOnWebApplication`：当应用是Web应用时
- `@ConditionalOnNotWebApplication`：当应用不是Web应用时
- `@ConditionalOnExpression`：当SpEL表达式为true时

例如：

```java
@Configuration
@ConditionalOnProperty(prefix = "myservice", name = "enabled", havingValue = "true", matchIfMissing = true)
public class MyServiceAutoConfiguration {
    // ...
}
```

这样，只有当`myservice.enabled=true`或者没有设置这个属性时，才会应用这个配置。

### 6.5 设置自动配置顺序

有时，自动配置之间可能存在依赖关系，需要控制它们的执行顺序。可以使用`@AutoConfigureBefore`、`@AutoConfigureAfter`和`@AutoConfigureOrder`注解：

```java
@Configuration
@AutoConfigureAfter(AnotherAutoConfiguration.class)
public class MyServiceAutoConfiguration {
    // ...
}
```

### 6.6 提供默认属性

可以使用`@ConfigurationProperties`注解来绑定配置属性：

```java
@Configuration
@EnableConfigurationProperties(MyServiceProperties.class)
public class MyServiceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyServiceProperties properties) {
        return new MyServiceImpl(properties.getProperty1(), properties.getProperty2());
    }
}

@ConfigurationProperties(prefix = "myservice")
public class MyServiceProperties {
    private String property1 = "defaultValue1";
    private int property2 = 42;
    
    // getters and setters
}
```

这样，用户可以在`application.properties`或`application.yml`中配置这些属性：

```properties
myservice.property1=customValue
myservice.property2=100
```

### 6.7 打包和发布

将你的自动配置打包成一个JAR文件，并发布到Maven仓库。其他项目只需要添加这个依赖，就能自动使用你的组件：

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-service-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 6.8 最佳实践

1. **遵循命名约定**：
   - 自动配置类以`AutoConfiguration`结尾
   - Starter项目以`-spring-boot-starter`结尾

2. **提供文档**：
   - 说明你的Starter提供了什么功能
   - 列出可配置的属性及其默认值
   - 提供使用示例

3. **测试你的Starter**：
   - 编写集成测试，确保自动配置在各种条件下正常工作
   - 测试属性绑定、条件注解等功能

4. **避免依赖冲突**：
   - 明确声明你的依赖版本
   - 使用`<optional>true</optional>`标记可选依赖

通过以上步骤，就能创建自己的SpringBoot Starter，扩展SpringBoot的功能，并与社区分享你的组件。

