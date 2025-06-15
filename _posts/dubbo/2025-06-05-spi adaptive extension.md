---
title: "2. SPI 自适应拓展"
date: 2025-06-15 13:00:00 +0800
categories: [从零开始的Dubbo源码阅读]
tags: [dubbo, 源码阅读]
---

# SPI 自适应拓展

> 参考链接：[dubbo2.6源码分析](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/dubbo-spi)

## 1.原理

在 Dubbo 中，很多拓展都是通过 SPI 机制进行加载的，比如 Protocol、Cluster、LoadBalance 等。有时，有些拓展并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。这听起来有些矛盾。拓展未被加载，那么拓展方法就无法被调用（静态方法除外）。拓展方法未被调用，拓展就无法被加载。对于这个矛盾的问题，Dubbo 通过自适应拓展机制很好的解决了。自适应拓展机制的实现逻辑比较复杂，首先 Dubbo 会为拓展接口生成具有代理功能的代码。然后通过 javassist 或 jdk 编译这段代码，得到 Class 类。最后再通过反射创建代理类，整个过程比较复杂。为了让大家对自适应拓展有一个感性的认识，下面我们通过一个示例进行演示。这是一个与支付相关的例子，我们有一个支付接口 PaymentService：

```java
@SPI("default")
public interface PaymentService {

    @Adaptive({"payment.type", "payment"})
    String pay(URL url, double amount);

    String query(URL url, String orderId);
}
```

PaymentService 接口的自适应实现类如下：

```java
package org.apache.dubbo.test.spi.adaptive;
import org.apache.dubbo.rpc.model.ScopeModel;
import org.apache.dubbo.rpc.model.ScopeModelUtil;

public class PaymentService$Adaptive implements org.apache.dubbo.test.spi.adaptive.PaymentService {
    
    public java.lang.String pay(org.apache.dubbo.common.URL arg0, double arg1) {
        if (arg0 == null) {throw new IllegalArgumentException("url == null");}
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("payment.type", url.getParameter("payment", "default"));
        if (extName == null) {
            throw new IllegalStateException(
                    "Failed to get extension (org.apache.dubbo.test.spi.adaptive.PaymentService) name from url ("
                            + url.toString() + ") use keys([payment.type, payment])");
        }
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.test.spi.adaptive.PaymentService.class);
        org.apache.dubbo.test.spi.adaptive.PaymentService extension = (org.apache.dubbo.test.spi.adaptive.PaymentService) scopeModel.getExtensionLoader(org.apache.dubbo.test.spi.adaptive.PaymentService.class)
                .getExtension(extName);
        return extension.pay(arg0, arg1);
    }
    
    public java.lang.String query(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        throw new UnsupportedOperationException("The method public abstract java.lang.String org.apache.dubbo.test.spi.adaptive.PaymentService.query(org.apache.dubbo.common.URL,java.lang.String) of interface org.apache.dubbo.test.spi.adaptive.PaymentService is not adaptive method!");
    }
}
```

`PaymentService$Adaptive` 是一个代理类，与传统的代理逻辑不同，`PaymentService$Adaptive` 所代理的对象是在 pay方法中通过 SPI 加载得到的。pay方法主要做了三件事情：

1. 从 URL 中获取 payment.type 或 payment 名称
2. 通过 SPI 加载具体的 PaymentService实现类
3. 调用目标方法

在运行时，假设有这样一个 url 参数传入：

```
dubbo://localhost/service?payment.type=alipay
```

pay 方法从 url 中提取 payment 或 payment.type 参数。之后再通过 SPI 加载配置名为 PaymentService 的实现类，得到具体的 WheelMaker 实例。

上面的示例展示了自适应拓展类的核心实现 —- 在拓展接口的方法被调用时，通过 SPI 加载具体的拓展实现类，并调用拓展对象的同名方法。接下来，我们深入到源码中，探索自适应拓展类生成的过程。

## 2.源码分析

在对自适应拓展生成过程进行深入分析之前，我们先来看一下与自适应拓展息息相关的一个注解，即 Adaptive 注解。该注解的定义如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```

从上面的代码中可知，Adaptive 可注解在类或方法上。当 Adaptive 注解在类上时，Dubbo 不会为该类生成代理类。注解在方法（接口方法）上时，Dubbo 则会为该方法生成代理逻辑。Adaptive 注解在类上的情况很少，在 Dubbo 中，仅有两个类被 Adaptive 注解了，分别是 AdaptiveCompiler 和 AdaptiveExtensionFactory。此种情况，表示拓展的加载逻辑由人工编码完成。更多时候，Adaptive 是注解在接口方法上的，表示拓展的加载逻辑需由框架自动生成。Adaptive 注解的地方不同，相应的处理逻辑也是不同的。注解在类上时，处理逻辑比较简单，本文就不分析了。注解在接口方法上时，处理逻辑较为复杂，本章将会重点分析此块逻辑。

测试用到的代码: 

```java
@SPI("default")
public interface PaymentService {

    @Adaptive({"payment.type", "payment"})
    String pay(URL url, double amount);

    String query(URL url, String orderId);
}
public class AliPayPaymentService implements PaymentService {

    @Override
    public String pay(URL url, double amount) {
        return "Pay " + amount + " via AliPay";
    }

    @Override
    public String query(URL url, String orderId) {
        return "Query order " + orderId + " via AliPay";
    }
}
public class WechatPayPaymentService implements PaymentService {

    @Override
    public String pay(URL url, double amount) {
        return "Pay " + amount + " via WechatPay";
    }

    @Override
    public String query(URL url, String orderId) {
        return "Query order " + orderId + " via WechatPay";
    }
}
```

测试代码:

```java
@Test
public void testAdaptiveExtension() {
    // 获取自适应扩展实例
    ExtensionDirector extensionDirector = ApplicationModel.defaultModel().getExtensionDirector();
    PaymentService adaptivePaymentService = extensionDirector.getExtensionLoader(PaymentService.class)
        .getAdaptiveExtension();

    // 创建URL并设置payment.type参数
    URL url1 = URL.valueOf("dubbo://localhost/service?payment.type=alipay");
    System.out.println(adaptivePaymentService.pay(url1, 100.0));

    // 创建URL并设置payment参数
    URL url2 = URL.valueOf("dubbo://localhost/service?payment=wechat");
    System.out.println(adaptivePaymentService.pay(url2, 200.0));

    // 创建没有相关参数的URL，将使用默认实现
    URL url3 = URL.valueOf("dubbo://localhost/service");
    System.out.println(adaptivePaymentService.pay(url3, 300.0));

    try {
        // query方法没有@Adaptive注解，调用将抛出UnsupportedOperationException
        adaptivePaymentService.query(url1, "ORDER123");
    } catch (UnsupportedOperationException e) {
        System.out.println("Expected exception: " + e.getMessage());
    }
}
```

测试结果: 

```
Pay 100.0 via AliPay
Pay 200.0 via WechatPay
Pay 300.0 via AliPay
Expected exception: The method public abstract java.lang.String org.apache.dubbo.test.spi.adaptive.PaymentService.query(org.apache.dubbo.common.URL,java.lang.String) of interface org.apache.dubbo.test.spi.adaptive.PaymentService is not adaptive method!
```

### 2.1 获取自适应拓展

getAdaptiveExtension 方法是获取自适应拓展的入口方法，因此下面我们从这个方法进行分析。相关代码如下：

```java
public T getAdaptiveExtension() {
    checkDestroyed();
    // 从缓存中获取自适应拓展
    Object instance = cachedAdaptiveInstance.get();
    // 缓存未命中
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("...");
        }

        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    // 创建自适应拓展
                    instance = createAdaptiveExtension();
                    // 设置自适应拓展到缓存中
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("...");
                }
            }
        }
    }

    return (T) instance;
}
```

getAdaptiveExtension 方法首先会检查缓存，缓存未命中，则调用 createAdaptiveExtension 方法创建自适应拓展。下面，我们看一下 createAdaptiveExtension 方法的代码。

```java
private T createAdaptiveExtension() {
    try {
        // 获取自适应拓展类，并通过反射实例化
        T instance = (T) getAdaptiveExtensionClass().newInstance();
        instance = postProcessBeforeInitialization(instance, null);
        // 注入依赖
        injectExtension(instance);
        instance = postProcessAfterInitialization(instance, null);
        // 初始化声明周期
        initExtension(instance);
        return instance;
    } catch (Exception e) {
        throw new IllegalStateException("...");
    }
}
```

createAdaptiveExtension 方法的代码比较少，但却包含了三个逻辑，分别如下：

1. 调用 getAdaptiveExtensionClass 方法获取自适应拓展 Class 对象
2. 通过反射进行实例化
3. 调用 injectExtension 方法向拓展实例中注入依赖

前两个逻辑比较好理解，第三个逻辑用于向自适应拓展对象中注入依赖。这个逻辑看似多余，但有存在的必要，这里简单说明一下。前面说过，Dubbo 中有两种类型的自适应拓展，一种是手工编码的，一种是自动生成的。手工编码的自适应拓展中可能存在着一些依赖，而自动生成的 Adaptive 拓展则不会依赖其他类。这里调用 injectExtension 方法的目的是为手工编码的自适应拓展注入依赖，这一点需要大家注意一下。关于 injectExtension 方法，前文已经分析过了，这里不再赘述。接下来，分析 getAdaptiveExtensionClass 方法的逻辑。

```java
private Class<?> getAdaptiveExtensionClass() {
    // 通过 SPI 获取所有的拓展类
    getExtensionClasses();
    // 检查缓存，若缓存不为空，则直接返回缓存
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建自适应拓展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

getAdaptiveExtensionClass 方法同样包含了三个逻辑，如下：

1. 调用 getExtensionClasses 获取所有的拓展类
2. 检查缓存，若缓存不为空，则返回缓存
3. 若缓存为空，则调用 createAdaptiveExtensionClass 创建自适应拓展类

这三个逻辑看起来平淡无奇，似乎没有多讲的必要。但是这些平淡无奇的代码中隐藏了着一些细节，需要说明一下。首先从第一个逻辑说起，getExtensionClasses 这个方法用于获取某个接口的所有实现类。比如该方法可以获取 Protocol 接口的 DubboProtocol、HttpProtocol、InjvmProtocol 等实现类。在获取实现类的过程中，如果某个实现类被 Adaptive 注解修饰了，那么该类就会被赋值给 cachedAdaptiveClass 变量。此时，上面步骤中的第二步条件成立（缓存不为空），直接返回 cachedAdaptiveClass 即可。如果所有的实现类均未被 Adaptive 注解修饰，那么执行第三步逻辑，创建自适应拓展类。相关代码如下：

```java
private Class<?> createAdaptiveExtensionClass() {
    // Adaptive Classes' ClassLoader should be the same with Real SPI interface classes' ClassLoader
    ClassLoader classLoader = type.getClassLoader();
    try {
        if (NativeDetector.inNativeImage()) {
            return classLoader.loadClass(type.getName() + "$Adaptive");
        }
    } catch (Throwable ignore) {

    }
    // 构建自适应拓展代码
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    // 获取编译器实现类
    org.apache.dubbo.common.compiler.Compiler compiler = extensionDirector
        .getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class)
        .getAdaptiveExtension();
    // 编译代码，生成 Class
    return compiler.compile(type, code, classLoader);
}
```

createAdaptiveExtensionClass 方法用于生成自适应拓展类，该方法首先会生成自适应拓展类的源码，然后通过 Compiler 实例（Dubbo 默认使用 javassist 作为编译器）编译源码，得到代理类 Class 实例。接下来，我们把重点放在代理类代码生成的逻辑上，其他逻辑大家自行分析。

### 2.2 自适应拓展类代码生成

generate 方法代码如下：

```java
public String generate(boolean sort) {
    // 若所有的方法上均无 Adaptive 注解，则抛出异常
    if (!hasAdaptiveMethod()) {
        throw new IllegalStateException("No adaptive method exist on extension " + type.getName()
                                        + ", refuse to create the adaptive class!");
    }

    StringBuilder code = new StringBuilder();
    // 生成 package 代码：package + type 所在包
    code.append(generatePackageInfo());
    // 生成 import 代码：import + ExtensionLoader 全限定名
    code.append(generateImports());
    // 生成类代码：public class + type简单名称 + $Adaptive + implements + type全限定名 + {
    code.append(generateClassDeclaration());
    // 生成类的方法
    Method[] methods = type.getMethods();
    if (sort) {
        Arrays.sort(methods, Comparator.comparing(Method::toString));
    }
    for (Method method : methods) {
        code.append(generateMethod(method));
    }
    code.append('}');

    if (logger.isDebugEnabled()) {
        logger.debug(code.toString());
    }
    return code.toString();
}
```

这里使用 ${…} 占位符代表其他代码的生成逻辑，该部分逻辑将在随后进行分析。上面代码不是很难理解，下面直接通过一个例子展示该段代码所生成的内容。以 Dubbo 的 Protocol 接口为例，生成的代码如下：

#### 2.2.1 Adaptive 注解检测

在生成代理类源码之前，generate 方法首先会通过反射检测接口方法是否包含 Adaptive 注解。对于要生成自适应拓展的接口，Dubbo 要求该接口至少有一个方法被 Adaptive 注解修饰。若不满足此条件，就会抛出运行时异常。相关代码如下：

```java
private boolean hasAdaptiveMethod() {
    // 通过反射获取所有的方法
    // 检测方法上是否有 Adaptive 注解
    return Arrays.stream(type.getMethods()).anyMatch(m -> m.isAnnotationPresent(Adaptive.class));
}
```

#### 2.2.2 生成类

通过 Adaptive 注解检测后，即可开始生成代码。代码生成的顺序与 Java 文件内容顺序一致，首先会生成 package 语句，然后生成 import 语句，紧接着生成类名等代码。整个逻辑如下：

```java
private String generatePackageInfo() {
    return String.format(CODE_PACKAGE, type.getPackage().getName());
}
private String generateImports() {
    StringBuilder builder = new StringBuilder();
    builder.append(String.format(CODE_IMPORTS, ScopeModel.class.getName()));
    builder.append(String.format(CODE_IMPORTS, ScopeModelUtil.class.getName()));
    return builder.toString();
}
private String generateClassDeclaration() {
    return String.format(CODE_CLASS_DECLARATION, type.getSimpleName(), type.getCanonicalName());
}
```

这里使用 ${…} 占位符代表其他代码的生成逻辑，该部分逻辑将在随后进行分析。上面代码不是很难理解，下面直接通过一个例子展示该段代码所生成的内容。以 PaymentService 接口为例，生成的代码如下：

```java
package org.apache.dubbo.test.spi.adaptive;
import org.apache.dubbo.rpc.model.ScopeModel;
import org.apache.dubbo.rpc.model.ScopeModelUtil;
public class PaymentService$Adaptive implements org.apache.dubbo.test.spi.adaptive.PaymentService {
    // 省略方法代码
}
```

#### 2.2.3 生成方法

一个方法可以被 Adaptive 注解修饰，也可以不被修饰。这里将未被 Adaptive 注解修饰的方法称为“无 Adaptive 注解方法”，下面我们先来看看此种方法的代码生成逻辑是怎样的。

##### 2.2.3.1 无 Adaptive 注解方法代码生成逻辑

对于接口方法，我们可以按照需求标注 Adaptive 注解。以 PaymentService 接口为例，该接口的 query 未标注 Adaptive 注解，其他方法均标注了 Adaptive 注解。Dubbo 不会为没有标注 Adaptive 注解的方法生成代理逻辑，对于该种类型的方法，仅会生成一句抛出异常的代码。生成逻辑如下：

```java
private String generateMethod(Method method) {
    String methodReturnType = method.getReturnType().getCanonicalName();
    String methodName = method.getName();
    // 生成方法逻辑代码
    String methodContent = generateMethodContent(method);
    // 生成方法参数
    String methodArgs = generateMethodArguments(method);
    // 生成方法throw
    String methodThrows = generateMethodThrows(method);
    return String.format(
        CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
}
private String generateMethodContent(Method method) {
    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // 无 Adaptive 注解方法代码生成逻辑
        return generateUnsupported(method);
    } else {
        // 省略无关逻辑
    }

    return code.toString();
}

private String generateUnsupported(Method method) {
    // 生成的代码格式如下: 
    // throw new UnsupportedOperationException("The method " + 方法签名 + " of interface " + 全限定接口名 + " is not adaptive method!");
    return String.format(CODE_UNSUPPORTED, method, type.getName());
}
```

以 PaymentService 接口的 query 方法为例，上面代码生成的内容如下：

```java
public java.lang.String query(org.apache.dubbo.common.URL arg0, java.lang.String arg1)  {
    throw new UnsupportedOperationException("The method public abstract java.lang.String org.apache.dubbo.test.spi.adaptive.PaymentService.query(org.apache.dubbo.common.URL,java.lang.String) of interface org.apache.dubbo.test.spi.adaptive.PaymentService is not adaptive method!");
}
```

##### 2.2.3.2 获取 URL 数据

前面说过方法代理逻辑会从 URL 中提取目标拓展的名称，因此代码生成逻辑的一个重要的任务是从方法的参数列表或者其他参数中获取 URL 数据。举例说明一下，我们要为 PaymentService 接口的 pay 方法生成代理逻辑。在运行时，通过反射得到的方法定义如下：

```java
public java.lang.String pay(org.apache.dubbo.common.URL arg0, double arg1)
```

对于 refer 方法，通过遍历 refer 的参数列表即可获取 URL 数据，这个还比较简单。对于 export 方法，获取 URL 数据则要麻烦一些。export 参数列表中没有 URL 参数，因此需要从 Invoker 参数中获取 URL 数据。获取方式是调用 Invoker 中可返回 URL 的 getter 方法，比如 getUrl。如果 Invoker 中无相关 getter 方法，此时则会抛出异常。整个逻辑如下：

```java
private String generateMethodContent(Method method) {
    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成逻辑}
    } else {
        // 遍历参数列表，确定 URL 参数位置
        int urlTypeIndex = getUrlTypeIndex(method);

        // urlTypeIndex != -1，表示参数列表中存在 URL 参数
        if (urlTypeIndex != -1) {
            // 1. 为 URL 类型参数生成判空代码，格式如下：
            // if (arg + urlTypeIndex == null) 
            //     throw new IllegalArgumentException("url == null");
            // 2. 为 URL 类型参数生成赋值代码，形如 URL url = arg1
            code.append(generateUrlNullCheck(urlTypeIndex));
        } else {
            // 参数列表中不存在 URL 类型参数
            // 尝试从参数对象的 getter 方法中查找返回 URL 类型的方法，并生成相应的判空和赋值代码
            code.append(generateUrlAssignmentIndirectly(method));
        }

        // 省略无关代码
    }

    return code.toString();
}

private String generateUrlAssignmentIndirectly(Method method) {
    Class<?>[] pts = method.getParameterTypes();

    Map<String, Integer> getterReturnUrl = new HashMap<>();
    // 遍历方法的所有参数类型，查找每个参数中是否有公开的、无参数、返回类型为 URL 的 getter 方法 (如 getUrl())
    // 如果找到这样的 getter 方法，则记录其方法名和参数索引
    for (int i = 0; i < pts.length; ++i) {
        for (Method m : pts[i].getMethods()) {
            String name = m.getName();
            if ((name.startsWith("get") || name.length() > 3)
                && Modifier.isPublic(m.getModifiers())
                && !Modifier.isStatic(m.getModifiers())
                && m.getParameterTypes().length == 0
                && m.getReturnType() == URL.class) {
                getterReturnUrl.put(name, i);
            }
        }
    }
	// 如果没有找到任何返回 URL 的 getter 方法，则抛出异常，说明无法为自适应类生成获取 URL 的代码
    if (getterReturnUrl.size() <= 0) {
        // getter method not found, throw
        throw new IllegalStateException("Failed to create adaptive class for interface " + type.getName()
                                        + ": not found url parameter or url attribute in parameters of method " + method.getName());
    }
	// 如果找到名为 getUrl 的方法，优先使用它，否则使用找到的第一个符合条件的 getter 方法
    Integer index = getterReturnUrl.get("getUrl");
    if (index != null) {
        return generateGetUrlNullCheck(index, pts[index], "getUrl");
    } else {
        Map.Entry<String, Integer> entry =
            getterReturnUrl.entrySet().iterator().next();
        return generateGetUrlNullCheck(entry.getValue(), pts[entry.getValue()], entry.getKey());
    }
}
```

上面代码有点多，需要耐心看一下。这段代码主要目的是为了获取 URL 数据，并为之生成判空和赋值代码。以 PaymentService 的 pay 方法为例，上面的代码为它们生成如下内容（代码已格式化）：

```java
if (arg0 == null) {
    throw new IllegalArgumentException("url == null");
}
org.apache.dubbo.common.URL url = arg0;
```

##### 2.2.3.3 获取 Adaptive 注解值

Adaptive 注解值 value 类型为 String[]，可填写多个值，默认情况下为空数组。若 value 为非空数组，直接获取数组内容即可。若 value 为空数组，则需进行额外处理。处理过程是将类名转换为字符数组，然后遍历字符数组，并将字符放入 StringBuilder 中。若字符为大写字母，则向 StringBuilder 中添加点号，随后将字符变为小写存入 StringBuilder 中。比如 LoadBalance 经过处理后，得到 load.balance。

```java
private String[] getMethodAdaptiveValue(Adaptive adaptiveAnnotation) {
    // 获取参数类型列表
    String[] value = adaptiveAnnotation.value();
    // value is not set, use the value generated from class name as the key
    if (value.length == 0) {
        String splitName = StringUtils.camelToSplitName(type.getSimpleName(), ".");
        value = new String[] {splitName};
    }
    return value;
}
```

##### 2.2.3.4 检测 Invocation 参数

此段逻辑是检测方法列表中是否存在 Invocation 类型的参数，若存在，则为其生成判空代码和其他一些代码。相应的逻辑如下：

```java
private boolean hasInvocationArgument(Method method) {
    Class<?>[] pts = method.getParameterTypes();
    // CLASS_NAME_INVOCATION = "org.apache.dubbo.rpc.Invocation"
    return Arrays.stream(pts).anyMatch(p -> CLASS_NAME_INVOCATION.equals(p.getName()));
}
```

##### 2.2.3.5 生成拓展名获取逻辑

本段逻辑用于根据 SPI 和 Adaptive 注解值生成“获取拓展名逻辑”，同时生成逻辑也受 Invocation 类型参数影响，综合因素导致本段逻辑相对复杂。本段逻辑可能会生成但不限于下面的代码：

```java
String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
```

或

```java
String extName = url.getMethodParameter(methodName, "loadbalance", "random");
```

亦或是

```java
String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
```

本段逻辑复杂之处在于条件分支比较多，大家在阅读源码时需要知道每个条件分支的意义是什么，否则不太容易看懂相关代码。下面开始分析本段逻辑。

```java
private String generateExtNameAssignment(String[] value, boolean hasInvocation) {
    // 设置默认拓展名，cachedDefaultName 源于 SPI 注解值，默认情况下，
    // SPI 注解值为空串，此时 cachedDefaultName = null
    String getNameCode = null;
    // 遍历 value，这里的 value 是 Adaptive 的注解值，2.2.3.3 节分析过 value 变量的获取过程。
    // 此处循环目的是生成从 URL 中获取拓展名的代码，生成的代码会赋值给 getNameCode 变量。注意这
    // 个循环的遍历顺序是由后向前遍历的。
    for (int i = value.length - 1; i >= 0; --i) {
        // 当 i 为最后一个元素的坐标时
        if (i == value.length - 1) {
            // 默认拓展名非空
            if (null != defaultExtName) {
                // protocol 是 url 的一部分，可通过 getProtocol 方法获取，其他的则是从
                // URL 参数中获取。因为获取方式不同，所以这里要判断 value[i] 是否为 protocol
                if (!CommonConstants.PROTOCOL_KEY.equals(value[i])) {
                    // hasInvocation 用于标识方法参数列表中是否有 Invocation 类型参数
                    if (hasInvocation) {
                        // 生成的代码功能等价于下面的代码：
                        //   url.getMethodParameter(methodName, value[i], defaultExtName)
                        // 以 LoadBalance 接口的 select 方法为例，最终生成的代码如下：
                        //   url.getMethodParameter(methodName, "loadbalance", "random")
                        getNameCode = String.format(
                            "url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                    } else {
                        // 生成的代码功能等价于下面的代码：
                        //   url.getParameter(value[i], defaultExtName)
                        getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                    }
                } else {
                    // 生成的代码功能等价于下面的代码：
                    //   ( url.getProtocol() == null ? defaultExtName : url.getProtocol() )
                    getNameCode = String.format(
                        "( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                }
                // 默认拓展名为空
            } else {
                if (!CommonConstants.PROTOCOL_KEY.equals(value[i])) {
                    if (hasInvocation) {
                        // 生成代码格式同上
                        getNameCode = String.format(
                            "url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                    } else {
                        // 生成的代码功能等价于下面的代码：
                        //   url.getParameter(value[i])
                        getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                    }
                } else {
                    // 生成从 url 中获取协议的代码，比如 "dubbo"
                    getNameCode = "url.getProtocol()";
                }
            }
        } else {
            if (!CommonConstants.PROTOCOL_KEY.equals(value[i])) {
                if (hasInvocation) {
                     // 生成代码格式同上
                    getNameCode = String.format(
                        "url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                } else {
                    // 生成的代码功能等价于下面的代码：
                    //   url.getParameter(value[i], getNameCode)
                    // 以 Transporter 接口的 connect 方法为例，最终生成的代码如下：
                    //   url.getParameter("client", url.getParameter("transporter", "netty"))
                    getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                }
            } else {
                // 生成的代码功能等价于下面的代码：
                //   url.getProtocol() == null ? getNameCode : url.getProtocol()
                // 以 Protocol 接口的 connect 方法为例，最终生成的代码如下：
                //   url.getProtocol() == null ? "dubbo" : url.getProtocol()
                getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
            }
        }
    }
    // 生成 extName 赋值代码
    return String.format(CODE_EXT_NAME_ASSIGNMENT, getNameCode);
}
```

上面代码比较复杂，不是很好理解。对于这段代码，建议大家写点测试用例，对 Protocol、LoadBalance 以及 Transporter 等接口的自适应拓展类代码生成过程进行调试。这里我以 PaymentService 接口的自适应拓展类代码生成过程举例说明。首先看一下 PaymentService 接口的定义，如下：

```java
@SPI("default")
public interface PaymentService {

    @Adaptive({"payment.type", "payment"})
    String pay(URL url, double amount);

    String query(URL url, String orderId);
}
```

下面对 connect 方法代理逻辑生成的过程进行分析，此时生成代理逻辑所用到的变量如下：

```java
String defaultExtName = "default";
boolean hasInvocation = false;
String getNameCode = null;
String[] value = [ "payment.type", "payment" ];
```

下面对 value 数组进行遍历，此时 i = 1, value[i] = “payment”，生成的代码如下：

```java
getNameCode = url.getParameter("payment", "default")
```

接下来，for 循环继续执行，此时 i = 0, value[i] = “client”，生成的代码如下：

```java
getNameCode = url.getParameter("payment.type", url.getParameter("payment", "default"))
```

for 循环结束运行，现在为 extName 变量生成赋值和判空代码 (generateExtNameNullCheck())，如下：

```java
String extName = url.getParameter("payment.type", url.getParameter("payment", "default"));
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.test.spi.adaptive.PaymentService) name from url (" + url.toString() + ") use keys([payment.type, payment])");
```

##### 2.2.3.6 生成拓展加载与目标方法调用逻辑

本段代码逻辑用于根据拓展名加载拓展实例，并调用拓展实例的目标方法。相关逻辑如下：

```java
private String generateMethodContent(Method method) {
    // 省略无关代码
    
    // 生成ScopeModel获取代码，格式如下:
	//     ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), type全限定名.class);
    code.append(generateScopeModelAssignment());
    // 生成拓展获取代码，格式如下:
    // type全限定名 extension = (typetype全限定名)scopeModel.getExtensionLoader(type全限定名.class).getExtension(extName);
    code.append(generateExtensionAssignment());

    // return statement
    code.append(generateReturnAndInvocation(method));

    return code.toString();
}

private String generateReturnAndInvocation(Method method) {
    // 如果方法返回值类型非 void，则生成 return 语句。
    String returnStatement = method.getReturnType().equals(void.class) ? "" : "return ";
    // 生成目标方法调用逻辑，格式为：
    //     extension.方法名(arg0, arg2, ..., argN);
    String args = IntStream.range(0, method.getParameters().length)
        .mapToObj(i -> String.format(CODE_EXTENSION_METHOD_INVOKE_ARGUMENT, i))
        .collect(Collectors.joining(", "));

    return returnStatement + String.format("extension.%s(%s);\n", method.getName(), args);
}
```

以 PaymentService.pay 方法举例说明，上面代码生成的内容如下：

```java
ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.test.spi.adaptive.PaymentService.class);
org.apache.dubbo.test.spi.adaptive.PaymentService extension = (org.apache.dubbo.test.spi.adaptive.PaymentService)scopeModel.getExtensionLoader(org.apache.dubbo.test.spi.adaptive.PaymentService.class).getExtension(extName);
return extension.pay(arg0, arg1);
```

##### 2.2.3.7 生成完整的方法

本节进行代码生成的收尾工作，主要用于生成方法定义的代码。相关逻辑如下：

```java
private String generateMethod(Method method) {
    String methodReturnType = method.getReturnType().getCanonicalName();
    String methodName = method.getName();
    // 添加方法体代码
    String methodContent = generateMethodContent(method);
    // 添加参数列表代码
    String methodArgs = generateMethodArguments(method);
    // 添加异常抛出代码
    String methodThrows = generateMethodThrows(method);
    // public + 返回值全限定名 + 方法名 + (
    return String.format(
        CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
}
```

以 PaymentService 的 pay方法为例，上面代码生成的内容如下：

```java
public java.lang.String pay(org.apache.dubbo.common.URL arg0, double arg1)  {
	// 方法体
}
```

## 3.总结

到此，关于自适应拓展的原理，实现就分析完了。总的来说自适应拓展整个逻辑还是很复杂的，并不是很容易弄懂。因此，大家在阅读该部分源码时，耐心一些。同时多进行调试，也可以通过生成好的代码思考代码的生成逻辑。好了，本篇文章就分析到这里。