---
title: "1. Dubbo SPI 源码解析"
date: 2025-06-15 12:00:00 +0800
categories: [从零开始的Dubbo源码阅读]
tags: [dubbo, 源码阅读]
---
# Dubbo SPI 源码解析

> 参考链接：[dubbo2.6源码分析](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/dubbo-spi)

## 1. SPI 机制概述

SPI (Service Provider Interface) 是一种服务发现机制，允许应用在运行时动态地发现和加载实现类。SPI 的核心思想是将接口实现类的全限定名配置在特定文件中，通过服务加载器读取这些配置文件，从而加载并实例化相应的实现类。这种机制使得框架可以实现更好的扩展性和灵活性。

Dubbo 框架的核心就是通过 SPI 机制加载其各个功能组件。**值得注意的是，Dubbo 没有直接使用 Java 原生的 SPI 机制，而是对其进行了增强**，以满足更复杂的需求，如依赖注入、AOP 等特性。如果想深入理解 Dubbo 的源码设计，彻底掌握 SPI 机制是必不可少的基础。

**特别说明：本文及系列其他文章分析的源码版本均为 Dubbo-3.3.1**。

## 2. SPI 实现对比

### 2.1 Java SPI 实例演示

首先，让我们通过一个简单示例来理解 Java 原生 SPI 的工作方式。

定义一个接口：

```java
public interface Robot {
    void sayHello();
}
```

实现两个具体实现类：

```java
public class OptimusPrime implements Robot {
    
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}

public class Bumblebee implements Robot {

    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
```

在 META-INF/services 目录下创建一个文件，文件名为接口的全限定名：`org.apache.spi.Robot`，内容为实现类的全限定名：

```
org.apache.spi.OptimusPrime
org.apache.spi.Bumblebee
```

测试代码：

```java
public class JavaSPITest {

    @Test
    public void sayHello() throws Exception {
        ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
        System.out.println("Java SPI");
        serviceLoader.forEach(Robot::sayHello);
    }
}
```

测试结果：
```
Java SPI
Hello, I am Optimus Prime.
Hello, I am Bumblebee.
```

### 2.2 Dubbo SPI 实例演示

Dubbo SPI 对 Java SPI 进行了显著增强。首先需要注意，**在 Dubbo 3 中，SPI 接口类必须添加 `@SPI` 注解**。

```java
@SPI
public interface Robot {
    void sayHello();
}
```

Dubbo SPI 配置文件需要放在 META-INF/dubbo 路径下，与 Java SPI 不同的是，Dubbo 采用键值对的方式进行配置：

```
optimusPrime = org.apache.spi.OptimusPrime
bumblebee = org.apache.spi.Bumblebee
```

测试代码：

```java
public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<Robot> extensionLoader = 
            ExtensionLoader.getExtensionLoader(Robot.class);
        // 按名称加载指定实现
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

测试结果：
```
Hello, I am Optimus Prime.
Hello, I am Bumblebee.
```

Dubbo SPI 相比 Java SPI 具有以下几点优势：
1. **按需加载**：可以按名称获取指定的实现类，而不是一次性加载所有实现
2. **依赖注入**：支持自动注入其他扩展点
3. **自适应扩展**：能够根据运行时参数动态选择实现类
4. **扩展点包装**：支持对扩展点进行 AOP 增强

## 3. Dubbo SPI 源码分析

了解了基本用法后，我们开始深入分析 Dubbo SPI 的源码实现。从使用示例可以看出，获取扩展实例需要两个步骤：

1. 通过 `ExtensionLoader.getExtensionLoader(Class<T>)` 获取扩展加载器
2. 通过 `extensionLoader.getExtension(String)` 获取具体扩展实例

我们直接从核心方法 `getExtension` 开始分析：

```java
public T getExtension(String name) {
    T extension = getExtension(name, true);
    if (extension == null) {
        throw new IllegalArgumentException("Not find extension: " + name);
    }
    return extension;
}

@SuppressWarnings("unchecked")
public T getExtension(String name, boolean wrap) {
    checkDestroyed();
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) {
        // 获取默认的拓展实现类
        return getDefaultExtension();
    }
    String cacheKey = name;
    if (!wrap) {
        cacheKey += "_origin";
    }
    // Holder，用于持有目标对象，提高并发性能
    final Holder<Object> holder = getOrCreateHolder(cacheKey);
    Object instance = holder.get();
    // 双重检查锁，保证线程安全
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建拓展实例
                instance = createExtension(name, wrap);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

整个获取过程采用了经典的双重检查锁模式，确保在多线程环境下的线程安全和性能。关键逻辑在 `createExtension` 方法中：

```java
@SuppressWarnings("unchecked")
private T createExtension(String name, boolean wrap) {
    // 从配置文件中加载所有的拓展类，获取"配置项名称"到"配置类"的映射关系
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null || unacceptableExceptions.contains(name)) {
        throw findException(name);
    }
    try {
        T instance = (T) extensionInstances.get(clazz);
        if (instance == null) {
            // 1. 创建实例
            extensionInstances.putIfAbsent(clazz, createExtensionInstance(clazz));
            instance = (T) extensionInstances.get(clazz);
            
            // 2. 注入依赖
            instance = postProcessBeforeInitialization(instance, name);
            injectExtension(instance);
            instance = postProcessAfterInitialization(instance, name);
        }

        // 3. AOP 增强：包装扩展点
        if (wrap) {
            List<Class<?>> wrapperClassesList = new ArrayList<>();
            if (cachedWrapperClasses != null) {
                wrapperClassesList.addAll(cachedWrapperClasses);
                wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                Collections.reverse(wrapperClassesList);
            }

            if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                // 循环创建 Wrapper 实例
                for (Class<?> wrapperClass : wrapperClassesList) {
                    Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                    boolean match = (wrapper == null)
                        || ((ArrayUtils.isEmpty(wrapper.matches())
                             || ArrayUtils.contains(wrapper.matches(), name))
                            && !ArrayUtils.contains(wrapper.mismatches(), name));
                    
                    // 匹配的包装类才会被应用
                    if (match) {
                        // 将当前实例作为参数传给包装类的构造方法，创建包装实例
                        instance = injectExtension(
                            (T) wrapperClass.getConstructor(type).newInstance(instance));
                        instance = postProcessAfterInitialization(instance, name);
                    }
                }
            }
        }

        // 4. 生命周期初始化
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        // 异常处理
        // ...
    }
}
```

`createExtension` 方法包含了 Dubbo SPI 的核心逻辑，主要分为四个步骤：
1. **实例创建**：通过反射创建扩展类实例
2. **依赖注入**：向扩展实例注入依赖（Dubbo IOC）
3. **包装增强**：应用 Wrapper 类进行包装（Dubbo AOP）
4. **生命周期初始化**：调用初始化方法

接下来，我们重点分析 `getExtensionClasses()` 方法，它负责加载所有扩展类配置。

### 3.1 扩展类加载过程

```java
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        // 使用 ReentrantLock 实现同步，比 synchronized 更灵活
        loadExtensionClassesLock.lock();
        try {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        } finally {
            loadExtensionClassesLock.unlock();
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() throws InterruptedException {
    checkDestroyed();
    // 缓存 SPI 注解中的默认扩展名
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();
    // 按策略加载不同路径下的配置文件
    for (LoadingStrategy strategy : strategies) {
        loadDirectory(extensionClasses, strategy, type.getName());

        // 兼容旧版 ExtensionFactory
        if (this.type == ExtensionInjector.class) {
            loadDirectory(extensionClasses, strategy, ExtensionFactory.class.getName());
        }
    }

    return extensionClasses;
}
```

Dubbo 3 中定义了多种加载策略（LoadingStrategy），按优先级顺序包括：
1. **DubboInternalLoadingStrategy**：从 `META-INF/dubbo/internal/` 加载 Dubbo 内部实现
2. **DubboLoadingStrategy**：从 `META-INF/dubbo/` 加载用户自定义实现
3. **ServicesLoadingStrategy**：从 `META-INF/services/` 加载，兼容 Java SPI

`loadDirectory` 方法负责读取和解析配置文件：

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, LoadingStrategy strategy, String type)
    throws InterruptedException {
    loadDirectoryInternal(extensionClasses, strategy, type);
    // Dubbo 2.x 兼容处理
    if (Dubbo2CompactUtils.isEnabled()) {
        try {
            String oldType = type.replace("org.apache", "com.alibaba");
            if (oldType.equals(type)) {
                return;
            }
            // 尝试加载旧版本路径资源
            ClassUtils.forName(oldType);
            loadDirectoryInternal(extensionClasses, strategy, oldType);
        } catch (ClassNotFoundException classNotFoundException) {
            // 忽略异常
        }
    }
}
```

`loadDirectoryInternal` 方法使用多个 ClassLoader 加载资源：

```java
private void loadDirectoryInternal(
    Map<String, Class<?>> extensionClasses, LoadingStrategy loadingStrategy, String type)
    throws InterruptedException {
    // 构造配置文件路径
    String fileName = loadingStrategy.directory() + type;
    try {
        List<ClassLoader> classLoadersToLoad = new LinkedList<>();

        // 优先使用 ExtensionLoader 的类加载器
        if (loadingStrategy.preferExtensionClassLoader()) {
            ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
            if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                classLoadersToLoad.add(extensionLoaderClassLoader);
            }
        }

        // 特殊 SPI 加载策略处理
        if (specialSPILoadingStrategyMap.containsKey(type)) {
            // ...处理特殊策略...
        } else {
            // 从作用域模型获取类加载器
            Set<ClassLoader> classLoaders = scopeModel.getClassLoaders();

            if (CollectionUtils.isEmpty(classLoaders)) {
                // 使用系统类加载器
                Enumeration<java.net.URL> resources = ClassLoader.getSystemResources(fileName);
                if (resources != null) {
                    while (resources.hasMoreElements()) {
                        loadResource(
                            extensionClasses,
                            null,
                            resources.nextElement(),
                            loadingStrategy.overridden(),
                            loadingStrategy.includedPackages(),
                            loadingStrategy.excludedPackages(),
                            loadingStrategy.onlyExtensionClassLoaderPackages());
                    }
                }
            } else {
                classLoadersToLoad.addAll(classLoaders);
            }
        }

        // 异步并行加载资源提高性能
        Map<ClassLoader, Set<java.net.URL>> resources =
            ClassLoaderResourceLoader.loadResources(fileName, classLoadersToLoad);
        
        // 遍历资源URL，加载类
        resources.forEach(((classLoader, urls) -> {
            loadFromClass(
                extensionClasses,
                loadingStrategy.overridden(),
                urls,
                classLoader,
                loadingStrategy.includedPackages(),
                loadingStrategy.excludedPackages(),
                loadingStrategy.onlyExtensionClassLoaderPackages());
        }));
    } catch (InterruptedException e) {
        throw e;
    } catch (Throwable t) {
        // 异常处理
    }
}
```

Dubbo 3 中资源加载采用异步并行方式，提高了加载性能：

```java
public static Map<ClassLoader, Set<URL>> loadResources(String fileName, Collection<ClassLoader> classLoaders)
    throws InterruptedException {
    Map<ClassLoader, Set<URL>> resources = new ConcurrentHashMap<>();
    // 使用 CountDownLatch 实现并行加载
    CountDownLatch countDownLatch = new CountDownLatch(classLoaders.size());
    
    for (ClassLoader classLoader : classLoaders) {
        GlobalResourcesRepository.getGlobalExecutorService().submit(() -> {
            resources.put(classLoader, loadResources(fileName, classLoader));
            countDownLatch.countDown();
        });
    }
    countDownLatch.await();
    return Collections.unmodifiableMap(new LinkedHashMap<>(resources));
}
```

拿到资源 URL 后，通过 `loadResource` 方法解析配置：

```java
private void loadResource(
    Map<String, Class<?>> extensionClasses,
    ClassLoader classLoader,
    java.net.URL resourceURL,
    boolean overridden,
    String[] includedPackages,
    String[] excludedPackages,
    String[] onlyExtensionClassLoaderPackages) {
    try {
        // 获取配置文件内容
        List<String> newContentList = getResourceContent(resourceURL);
        String clazz;
        for (String line : newContentList) {
            try {
                String name = null;
                int i = line.indexOf('=');
                // 解析键值对
                if (i > 0) {
                    name = line.substring(0, i).trim();
                    clazz = line.substring(i + 1).trim();
                } else {
                    clazz = line;
                }
                
                // 检查包名过滤规则
                if (StringUtils.isNotEmpty(clazz)
                    && !isExcluded(clazz, excludedPackages)
                    && isIncluded(clazz, includedPackages)
                    && !isExcludedByClassLoader(clazz, classLoader, onlyExtensionClassLoaderPackages)) {

                    // 加载类并处理
                    loadClass(
                        classLoader,
                        extensionClasses,
                        resourceURL,
                        Class.forName(clazz, true, classLoader),
                        name,
                        overridden);
                }
            } catch (Throwable t) {
                // 记录加载异常
                exceptions.put(line, e);
            }
        }
    } catch (Throwable t) {
        // 异常处理
    }
}
```

最后，`loadClass` 方法处理加载到的类：

```java
private void loadClass(
    ClassLoader classLoader,
    Map<String, Class<?>> extensionClasses,
    java.net.URL resourceURL,
    Class<?> clazz,
    String name,
    boolean overridden) {
    // 检查类型是否匹配
    if (!type.isAssignableFrom(clazz)) {
        // 类型不匹配，抛出异常
    }

    // 检查类是否可用（Activate 注解条件判断）
    boolean isActive = loadClassIfActive(classLoader, clazz);
    if (!isActive) {
        return;
    }
    
    // 处理不同类型的扩展点
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        // 1. 自适应扩展点
        cacheAdaptiveClass(clazz, overridden);
    } else if (isWrapperClass(clazz)) {
        // 2. 包装类扩展点
        cacheWrapperClass(clazz);
    } else {
        // 3. 普通扩展点
        if (StringUtils.isEmpty(name)) {
            // 从类名或注解推断扩展名
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName()
                                                + " in the config " + resourceURL);
            }
        }
        
        // 支持一个实现类对应多个扩展名
        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            // 缓存激活条件
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                // 双向缓存：类到名称、名称到类
                cacheName(clazz, n);
                saveInExtensionClass(extensionClasses, clazz, n, overridden);
            }
        }
    }
}
```

### 3.2 Dubbo IOC 依赖注入

Dubbo 的依赖注入是通过 setter 方法实现的。`injectExtension` 方法负责这一过程：

```java
private T injectExtension(T instance) {
    if (injector == null) {
        return instance;
    }

    try {
        // 遍历目标类的所有方法
        for (Method method : instance.getClass().getMethods()) {
            // 判断是否是 setter 方法：以 set 开头，只有一个参数，且为 public
            if (!isSetter(method)) {
                continue;
            }
            
            // 检查是否禁用注入 (@DisableInject)
            if (method.isAnnotationPresent(DisableInject.class)) {
                continue;
            }

            // 排除特殊接口的 setter 方法
            if (method.getDeclaringClass() == ScopeModelAware.class) {
                continue;
            }
            if (instance instanceof ScopeModelAware || instance instanceof ExtensionAccessorAware) {
                if (ignoredInjectMethodsDesc.contains(ReflectUtils.getDesc(method))) {
                    continue;
                }
            }
            
            // 获取参数类型，跳过基本类型
            Class<?> pt = method.getParameterTypes()[0];
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }

            try {
                // 从方法名推断属性名，如 setName -> name
                String property = getSetterProperty(method);
                // 从注入器获取依赖实例
                Object object = injector.getInstance(pt, property);
                if (object != null) {
                    // 通过反射调用 setter 方法
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                // 注入失败处理
            }
        }
    } catch (Exception e) {
        logger.error(COMMON_ERROR_LOAD_EXTENSION, "", "", e.getMessage(), e);
    }
    return instance;
}
```

Dubbo 3 中依赖注入主要由 `ExtensionInjector` 接口实现，具体的注入器包括：

1. **SpiExtensionInjector**：从 Dubbo SPI 系统中获取依赖
2. **SpringExtensionInjector**：从 Spring 容器中获取依赖

这种设计使 Dubbo SPI 能够与 Spring 等 IoC 容器无缝集成，增强了框架的灵活性。

## 4. Dubbo SPI 的核心特性

通过源码分析，我们可以总结 Dubbo 3 SPI 的几个核心特性：

### 4.1 按需加载

与 Java SPI 一次性加载所有实现不同，Dubbo SPI 通过键值对配置，实现了按名称的按需加载，提高了资源利用效率。

### 4.2 IOC 依赖注入

通过自动扫描和注入 setter 方法，实现了类似 Spring 的依赖注入能力，使扩展点之间可以形成依赖关系。

### 4.3 AOP 包装增强

通过 Wrapper 机制，Dubbo SPI 能在不修改原始代码的情况下，对扩展点实现进行包装和增强，实现面向切面的功能。

### 4.4 自适应扩展

Dubbo SPI 的一大创新是自适应扩展机制，能够根据请求参数在运行时动态选择实现类。这是通过 `@Adaptive` 注解和动态代码生成实现的，将在下一篇文章详细分析。

### 4.5 扩展点激活机制

通过 `@Activate` 注解，Dubbo 可以根据不同条件自动激活特定的扩展实现，使配置更加灵活和场景化。

## 5. 总结

本文详细分析了 Dubbo 3.3.1 中 SPI 机制的实现原理和源码结构。作为 Dubbo 最核心的基础设施，SPI 机制通过精巧的设计提供了强大的扩展能力，使 Dubbo 具备了高度的可扩展性和灵活性。

Dubbo SPI 相比 Java SPI 具有明显优势，它不仅支持按需加载，还融合了 IOC、AOP 等现代编程范式，使框架具备更强的可定制性。尤其是自适应扩展机制，为服务框架提供了运行时动态选择实现的能力，这在微服务架构中尤为重要。

在下一篇文章中，我们将深入分析 Dubbo SPI 中的自适应扩展机制，了解它如何通过字节码生成技术实现运行时的动态适配。

欢迎通过 GitHub Issue 或 Pull Request 参与 Dubbo 社区讨论和贡献，共同完善这个优秀的开源项目。

