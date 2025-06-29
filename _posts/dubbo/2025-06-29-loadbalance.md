---
title: "6. Dubbo3 负载均衡源码分析"
date: 2025-06-29 23:00:00 +0800
categories: [从零开始的Dubbo源码阅读]
tags: [dubbo, 源码阅读]
---
# 负载均衡

## 1.简介

LoadBalance 中文意思为负载均衡，它的职责是将网络请求，或者其他形式的负载“均摊”到不同的机器上。避免集群中部分服务器压力过大，而另一些服务器比较空闲的情况。通过负载均衡，可以让每台服务器获取到适合自己处理能力的负载。在为高负载服务器分流的同时，还可以避免资源浪费，一举两得。负载均衡可分为软件负载均衡和硬件负载均衡。在我们日常开发中，一般很难接触到硬件负载均衡。但软件负载均衡还是可以接触到的，比如 Nginx。在 Dubbo 中，也有负载均衡的概念和相应的实现。Dubbo 需要对服务消费者的调用请求进行分配，避免少数服务提供者负载过大。服务提供者负载过大，会导致部分请求超时。因此将负载均衡到每个服务提供者上，是非常必要的。Dubbo 提供了4种负载均衡实现，分别是基于权重随机算法的 RandomLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance，以及基于加权轮询算法的 RoundRobinLoadBalance。这几个负载均衡算法代码不是很长，但是想看懂也不是很容易，需要大家对这几个算法的原理有一定了解才行。如果不是很了解，也没不用太担心。我们会在分析每个算法的源码之前，对算法原理进行简单的讲解，帮助大家建立初步的印象。

本系列文章在编写之初是基于 Dubbo 2.6.4 的，近期，Dubbo 2.6.5 发布了，其中就有针对对负载均衡部分的优化。因此我们在分析完 2.6.4 版本后的源码后，会另外分析 2.6.5 更新的部分。其他的就不多说了，进入正题吧。

## 2.源码分析

在 Dubbo 中，所有负载均衡实现类均继承自 AbstractLoadBalance，该类实现了 LoadBalance 接口，并封装了一些公共的逻辑。所以在分析负载均衡实现之前，先来看一下 AbstractLoadBalance 的逻辑。首先来看一下负载均衡的入口方法 select，如下：

```java
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    // 如果 invokers 列表中仅有一个 Invoker，直接返回即可，无需进行负载均衡
    if (invokers.size() == 1) {
        return invokers.get(0);
    }
    
    // 调用 doSelect 方法进行负载均衡，该方法为抽象方法，由子类实现
    return doSelect(invokers, url, invocation);
}

protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
```

select 方法的逻辑比较简单，首先会检测 invokers 集合的合法性，然后再检测 invokers 集合元素数量。如果只包含一个 Invoker，直接返回该 Inovker 即可。如果包含多个 Invoker，此时需要通过负载均衡算法选择一个 Invoker。具体的负载均衡算法由子类实现，接下来章节会对这些子类一一进行详细分析。

AbstractLoadBalance 除了实现了 LoadBalance 接口方法，还封装了一些公共逻辑，比如服务提供者权重计算逻辑。具体实现如下：

```java
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    int weight;
    URL url = invoker.getUrl();
    
    // 处理集群Invoker的特殊情况，获取其注册中心URL
    if (invoker instanceof ClusterInvoker) {
        url = ((ClusterInvoker<?>) invoker).getRegistryUrl();
    }

    // 多注册中心场景处理：当服务接口为注册中心服务引用路径时
    if (REGISTRY_SERVICE_REFERENCE_PATH.equals(url.getServiceInterface())) {
        // 直接从URL参数获取权重，使用默认值当参数不存在时
        weight = url.getParameter(WEIGHT_KEY, DEFAULT_WEIGHT);
    } else {
        // 普通服务场景：从方法参数中获取权重
        String methodName = RpcUtils.getMethodName(invocation);
        weight = url.getMethodParameter(methodName, WEIGHT_KEY, DEFAULT_WEIGHT);
        
        // 当权重配置大于0时，考虑服务预热(warmup)机制
        if (weight > 0) {
            long timestamp = invoker.getUrl().getParameter(TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
                // 计算服务上线时间(毫秒)
                long uptime = System.currentTimeMillis() - timestamp;
                if (uptime < 0) {
                    // 时间戳异常时返回最小权重1
                    return 1;
                }
                
                // 获取预热期配置(毫秒)
                int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
                if (uptime > 0 && uptime < warmup) {
                    // 处于预热期时，按公式计算预热权重
                    weight = calculateWarmupWeight((int) uptime, warmup, weight);
                }
            }
        }
    }
    
    // 确保权重值非负
    return Math.max(weight, 0);
}

static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    // 随着服务运行时间 uptime 增大，权重计算值 ww 会慢慢接近配置值 weight
    int ww = (int) (uptime / ((float) warmup / weight));
    return ww < 1 ? 1 : (Math.min(ww, weight));
}
```

上面是权重的计算过程，该过程主要用于保证当服务运行时长小于服务预热时间时，对服务进行降权，避免让服务在启动之初就处于高负载状态。服务预热是一个优化手段，与此类似的还有 JVM 预热。主要目的是让服务启动后“低功率”运行一段时间，使其效率慢慢提升至最佳状态。

关于 AbstractLoadBalance 就先分析到这，接下来分析各个实现类的代码。首先，我们从 Dubbo 缺省的实现类 RandomLoadBalance 看起。

### 2.1 RandomLoadBalance

RandomLoadBalance 是加权随机算法的具体实现，它的算法思想很简单。假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3, 2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10) 区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10) 之间的随机数，然后计算这个随机数会落到哪个区间上。比如数字3会落到服务器 A 对应的区间上，此时返回服务器 A 即可。权重越大的机器，在坐标轴上对应的区间范围就越大，因此随机数生成器生成的数字就会有更大的概率落到此区间内。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。比如，经过一万次选择后，服务器 A 被选中的次数大约为5000次，服务器 B 被选中的次数约为3000次，服务器 C 被选中的次数约为2000次。

以上就是 RandomLoadBalance 背后的算法思想，比较简单。下面开始分析源码。

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // 可用Invoker数量
    int length = invokers.size();

    // 快速路径：不需要加权负载均衡时直接随机选择
    if (!needWeightLoadBalance(invokers, invocation)) {
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }

    // 权重计算与验证
    boolean sameWeight = true;       // 标记所有权重是否相同
    int[] weights = new int[length]; // 存储累计权重数组
    int totalWeight = 0;             // 总权重值
    
    // 遍历计算每个Invoker的权重
    for (int i = 0; i < length; i++) {
        int weight = getWeight(invokers.get(i), invocation);
        totalWeight += weight;       // 累加总权重
        weights[i] = totalWeight;    // 存储累计权重
        
        // 验证所有权重是否相同
        if (sameWeight && totalWeight != weight * (i + 1)) {
            sameWeight = false;
        }
    }

    // 加权随机选择逻辑
    if (totalWeight > 0 && !sameWeight) {
        // 生成0到总权重之间的随机偏移量
        int offset = ThreadLocalRandom.current().nextInt(totalWeight);
        
        // 优化策略：小规模列表使用线性查找，大规模列表使用二分查找
        if (length <= 4) {
            // 线性查找：适用于小规模列表（≤4个元素）
            for (int i = 0; i < length; i++) {
                if (offset < weights[i]) {
                    return invokers.get(i);
                }
            }
        } else {
            // 二分查找：适用于大规模列表（>4个元素）
            int i = Arrays.binarySearch(weights, offset);
            if (i < 0) {
                // 二分查找未找到时，计算插入位置
                i = -i - 1;
            } else {
                // 处理权重相同的情况，找到最后一个匹配位置
                while (weights[i + 1] == offset) {
                    i++;
                }
                i++;
            }
            return invokers.get(i);
        }
    }

    // 回退策略：权重相同或总权重为0时随机选择
    return invokers.get(ThreadLocalRandom.current().nextInt(length));
}
```

RandomLoadBalance 的算法思想比较简单，在经过多次请求后，能够将调用请求按照权重值进行“均匀”分配。当然 RandomLoadBalance 也存在一定的缺点，当调用次数比较少时，Random 产生的随机数可能会比较集中，此时多数请求会落到同一台服务器上。这个缺点并不是很严重，多数情况下可以忽略。RandomLoadBalance 是一个简单，高效的负载均衡实现，因此 Dubbo 选择它作为缺省实现。

关于 RandomLoadBalance 就先到这了，接下来分析 LeastActiveLoadBalance。

### 2.2 LeastActiveLoadBalance

LeastActiveLoadBalance 翻译过来是最小活跃数负载均衡。活跃调用数越小，表明该服务提供者效率越高，单位时间内可处理更多的请求。此时应优先将请求分配给该服务提供者。在具体实现中，每个服务提供者对应一个活跃数 active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快，此时这样的服务提供者能够优先获取到新的服务请求、这就是最小活跃数负载均衡算法的基本思想。除了最小活跃数，LeastActiveLoadBalance 在实现上还引入了权重值。所以准确的来说，LeastActiveLoadBalance 是基于加权最小活跃数算法实现的。举个例子说明一下，在一个服务提供者集群中，有两个性能优异的服务提供者。某一时刻它们的活跃数相同，此时 Dubbo 会根据它们的权重去分配请求，权重越大，获取到新请求的概率就越大。如果两个服务提供者权重相同，此时随机选择一个即可。关于 LeastActiveLoadBalance 的背景知识就先介绍到这里，下面开始分析源码。

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // 可用Invoker数量
    int length = invokers.size();
    // 所有Invoker中的最小活跃数
    int leastActive = -1;
    // 具有最小活跃数的Invoker数量
    int leastCount = 0;
    // 存储具有最小活跃数的Invoker索引
    int[] leastIndexes = new int[length];
    // 每个Invoker的权重
    int[] weights = new int[length];
    // 所有最小活跃数Invoker的权重和
    int totalWeight = 0;
    // 第一个最小活跃数Invoker的权重
    int firstWeight = 0;
    // 所有最小活跃数Invoker的权重是否相同
    boolean sameWeight = true;

    // 第一步：筛选出所有最小活跃数的Invoker
    for (int i = 0; i < length; i++) {
        Invoker<T> invoker = invokers.get(i);
        // 获取Invoker的当前活跃连接数
        int active = RpcStatus.getStatus(invoker.getUrl(), RpcUtils.getMethodName(invocation))
                .getActive();
        // 获取Invoker的权重（已考虑预热期）
        int afterWarmup = getWeight(invoker, invocation);
        // 保存权重用于后续选择
        weights[i] = afterWarmup;
        
        // 发现新的最小活跃数Invoker
        if (leastActive == -1 || active < leastActive) {
            leastActive = active;         // 更新最小活跃数
            leastCount = 1;               // 重置最小活跃数Invoker数量
            leastIndexes[0] = i;          // 记录第一个最小活跃数Invoker的索引
            totalWeight = afterWarmup;    // 重置总权重
            firstWeight = afterWarmup;    // 记录第一个最小活跃数Invoker的权重
            sameWeight = true;            // 初始假设权重相同
        } 
        // 找到与当前最小活跃数相同的Invoker
        else if (active == leastActive) {
            leastIndexes[leastCount++] = i;  // 记录最小活跃数Invoker的索引
            totalWeight += afterWarmup;      // 累加总权重
            // 检查权重是否相同
            if (sameWeight && afterWarmup != firstWeight) {
                sameWeight = false;
            }
        }
    }

    // 第二步：在最小活跃数Invoker中进行选择
    if (leastCount == 1) {
        // 只有一个最小活跃数Invoker时直接返回
        return invokers.get(leastIndexes[0]);
    }
    
    if (!sameWeight && totalWeight > 0) {
        // 权重不同且总权重>0时，按权重比例随机选择
        int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
        // 遍历最小活跃数Invoker，根据权重偏移量选择
        for (int i = 0; i < leastCount; i++) {
            int leastIndex = leastIndexes[i];
            offsetWeight -= weights[leastIndex];
            if (offsetWeight < 0) {
                return invokers.get(leastIndex);
            }
        }
    }
    
    // 第三步：权重相同或总权重为0时，随机选择
    return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
}
```

上面代码的逻辑比较多，我们在代码中写了大量的注释，有帮助大家理解代码逻辑。下面简单总结一下以上代码所做的事情，如下：

1. 遍历 invokers 列表，寻找活跃数最小的 Invoker
2. 如果有多个 Invoker 具有相同的最小活跃数，此时记录下这些 Invoker 在 invokers 集合中的下标，并累加它们的权重，比较它们的权重值是否相等
3. 如果只有一个 Invoker 具有最小的活跃数，此时直接返回该 Invoker 即可
4. 如果有多个 Invoker 具有最小活跃数，且它们的权重不相等，此时处理方式和 RandomLoadBalance 一致
5. 如果有多个 Invoker 具有最小活跃数，但它们的权重相等，此时随机返回一个即可

以上就是 LeastActiveLoadBalance 大致的实现逻辑，大家在阅读的源码的过程中要注意区分活跃数与权重这两个概念，不要混为一谈。

### 2.3 ConsistentHashLoadBalance

一致性 hash 算法由麻省理工学院的 Karger 及其合作者于1997年提出的，算法提出之初是用于大规模缓存系统的负载均衡。它的工作过程是这样的，首先根据 ip 或者其他的信息为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 232 - 1] 的圆环上。当有查询或写入请求时，则为缓存项的 key 生成一个 hash 值。然后查找第一个大于或等于该 hash 值的缓存节点，并到这个节点中查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入缓存时，为缓存项查找另一个大于其 hash 值的缓存节点即可。大致效果如下图所示，每个缓存节点在圆环上占据一个位置。如果缓存项的 key 的 hash 值小于缓存节点 hash 值，则到该缓存节点中存储或读取缓存项。比如下面绿色点对应的缓存项将会被存储到 cache-2 节点中。由于 cache-3 挂了，原本应该存到该节点中的缓存项最终会存储到 cache-4 节点中。

![img](https://cn.dubbo.apache.org/imgs/dev/consistent-hash.jpg)

下面来看看一致性 hash 在 Dubbo 中的应用。我们把上图的缓存节点替换成 Dubbo 的服务提供者，于是得到了下图：

![img](https://cn.dubbo.apache.org/imgs/dev/consistent-hash-invoker.jpg)

这里相同颜色的节点均属于同一个服务提供者，比如 Invoker1-1，Invoker1-2，……, Invoker1-160。这样做的目的是通过引入虚拟节点，让 Invoker 在圆环上分散开来，避免数据倾斜问题。所谓数据倾斜是指，由于节点不够分散，导致大量请求落到了同一个节点上，而其他节点只会接收到了少量请求的情况。比如：

![img](https://cn.dubbo.apache.org/imgs/dev/consistent-hash-data-incline.jpg)

如上，由于 Invoker-1 和 Invoker-2 在圆环上分布不均，导致系统中75%的请求都会落到 Invoker-1 上，只有 25% 的请求会落到 Invoker-2 上。解决这个问题办法是引入虚拟节点，通过虚拟节点均衡各个节点的请求量。

到这里背景知识就普及完了，接下来开始分析源码。我们先从 ConsistentHashLoadBalance 的 doSelect 方法开始看起，如下：

```java
private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<>();

protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // 生成唯一键：服务Key+方法名，用于标识不同的服务方法
    String methodName = RpcUtils.getMethodName(invocation);
    String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
    
    // 计算Invoker列表的哈希码，用于检测列表变更
    // 注：使用列表自身的哈希码，该哈希码基于列表元素计算
    int invokersHashCode = invokers.hashCode();
    
    // 从缓存中获取一致性哈希选择器
    ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
    
    // 检查选择器是否存在或Invoker列表是否变更
    if (selector == null || selector.identityHashCode != invokersHashCode) {
        // 创建新的一致性哈希选择器（包含Invoker列表、方法名和哈希码）
        selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, invokersHashCode));
        // 从缓存中重新获取（确保线程安全）
        selector = (ConsistentHashSelector<T>) selectors.get(key);
    }
    
    // 使用一致性哈希选择器执行最终的Invoker选择
    return selector.select(invocation);
}
```

如上，doSelect 方法主要做了一些前置工作，比如检测 invokers 列表是不是变动过，以及创建 ConsistentHashSelector。这些工作做完后，接下来开始调用 ConsistentHashSelector 的 select 方法执行负载均衡逻辑。在分析 select 方法之前，我们先来看一下一致性 hash 选择器 ConsistentHashSelector 的初始化过程，如下：

```java
private static final class ConsistentHashSelector<T> {

    // 使用 TreeMap 存储 Invoker 虚拟节点
    private final TreeMap<Long, Invoker<T>> virtualInvokers;

    private final int replicaNumber;

    private final int identityHashCode;

    private final int[] argumentIndex;


    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
        // 初始化虚拟节点环，使用TreeMap存储虚拟节点的哈希值到Invoker的映射
        this.virtualInvokers = new TreeMap<>();
        // 记录服务提供者列表的哈希码，用于检测列表是否变更
        this.identityHashCode = identityHashCode;

        // 获取服务提供者URL，用于读取配置参数
        URL url = invokers.get(0).getUrl();
        // 获取虚拟节点数量配置（默认160个），用于平衡负载
        this.replicaNumber = url.getMethodParameter(methodName, HASH_NODES, 160);
        // 获取参与哈希计算的参数索引配置（默认使用第1个参数）
        String[] index = COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, HASH_ARGUMENTS, "0"));
        // 转换参数索引为整数数组
        argumentIndex = new int[index.length];
        for (int i = 0; i < index.length; i++) {
            argumentIndex[i] = Integer.parseInt(index[i]);
        }

        // 为每个服务提供者创建虚拟节点并添加到哈希环
        for (Invoker<T> invoker : invokers) {
            String address = invoker.getUrl().getAddress();
            // 每个服务提供者创建replicaNumber/4组虚拟节点
            for (int i = 0; i < replicaNumber / 4; i++) {
                // 计算MD5摘要（每组生成4个哈希值）
                byte[] digest = Bytes.getMD5(address + i);
                for (int h = 0; h < 4; h++) {
                    // 从MD5摘要中提取4个长整型哈希值
                    long m = hash(digest, h);
                    // 将哈希值映射到对应的服务提供者
                    virtualInvokers.put(m, invoker);
                }
            }
        }
    }
}
```

ConsistentHashSelector 的构造方法执行了一系列的初始化逻辑，比如从配置中获取虚拟节点数以及参与 hash 计算的参数下标，默认情况下只使用第一个参数进行 hash。需要特别说明的是，ConsistentHashLoadBalance 的负载均衡逻辑只受参数值影响，具有相同参数值的请求将会被分配给同一个服务提供者。ConsistentHashLoadBalance 不 关系权重，因此使用时需要注意一下。

在获取虚拟节点数和参数下标配置后，接下来要做的事情是计算虚拟节点 hash 值，并将虚拟节点存储到 TreeMap 中。到此，ConsistentHashSelector 初始化工作就完成了。接下来，我们来看看 select 方法的逻辑。

```java
public Invoker<T> select(Invocation invocation) {
    // 将参数转为 key
    String key = toKey(invocation.getArguments());
    // 对参数 key 进行 md5 运算
    byte[] digest = md5(key);
    // 取 digest 数组的前四个字节进行 hash 运算，再将 hash 值传给 selectForKey 方法，
    // 寻找合适的 Invoker
    return selectForKey(hash(digest, 0));
}

private Invoker<T> selectForKey(long hash) {
    // 到 TreeMap 中查找第一个节点值大于或等于当前 hash 的 Invoker
    Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);
    // 如果 hash 大于 Invoker 在圆环上最大的位置，此时 entry = null，
    // 需要将 TreeMap 的头节点赋值给 entry
    if (entry == null) {
        entry = virtualInvokers.firstEntry();
    }

    // 返回 Invoker
    return entry.getValue();
}
```

如上，选择的过程相对比较简单了。首先是对参数进行 md5 以及 hash 运算，得到一个 hash 值。然后再拿这个值到 TreeMap 中查找目标 Invoker 即可。

到此关于 ConsistentHashLoadBalance 就分析完了。在阅读 ConsistentHashLoadBalance 源码之前，大家一定要先补充背景知识，不然很难看懂代码逻辑。

### 2.4 RoundRobinLoadBalance

本节，我们来看一下 Dubbo 中加权轮询负载均衡的实现 RoundRobinLoadBalance。在详细分析源码前，我们先来了解一下什么是加权轮询。这里从最简单的轮询开始讲起，所谓轮询是指将请求轮流分配给每台服务器。举个例子，我们有三台服务器 A、B、C。我们将第一个请求分配给服务器 A，第二个请求分配给服务器 B，第三个请求分配给服务器 C，第四个请求再次分配给服务器 A。这个过程就叫做轮询。轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。但现实情况下，我们并不能保证每台服务器性能均相近。如果我们将等量的请求分配给性能较差的服务器，这显然不合理。因此，这个时候我们需要对轮询过程进行加权，以调控每台服务器的负载。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将收到其中的5次请求，服务器 B 会收到其中的2次请求，服务器 C 则收到其中的1次请求。

以上就是加权轮询的算法思想，搞懂了这个思想，接下来我们就可以分析源码了。

#### 2.6.4 版本

我们先来看一下 2.6.4 版本的 RoundRobinLoadBalance。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "roundrobin";

    private final ConcurrentMap<String, AtomicPositiveInteger> sequences = 
        new ConcurrentHashMap<String, AtomicPositiveInteger>();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // key = 全限定类名 + "." + 方法名，比如 com.xxx.DemoService.sayHello
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int length = invokers.size();
        // 最大权重
        int maxWeight = 0;
        // 最小权重
        int minWeight = Integer.MAX_VALUE;
        final LinkedHashMap<Invoker<T>, IntegerWrapper> invokerToWeightMap = new LinkedHashMap<Invoker<T>, IntegerWrapper>();
        // 权重总和
        int weightSum = 0;

        // 下面这个循环主要用于查找最大和最小权重，计算权重总和等
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // 获取最大和最小权重
            maxWeight = Math.max(maxWeight, weight);
            minWeight = Math.min(minWeight, weight);
            if (weight > 0) {
                // 将 weight 封装到 IntegerWrapper 中
                invokerToWeightMap.put(invokers.get(i), new IntegerWrapper(weight));
                // 累加权重
                weightSum += weight;
            }
        }

        // 查找 key 对应的对应 AtomicPositiveInteger 实例，为空则创建。
        // 这里可以把 AtomicPositiveInteger 看成一个黑盒，大家只要知道
        // AtomicPositiveInteger 用于记录服务的调用编号即可。至于细节，
        // 大家如果感兴趣，可以自行分析
        AtomicPositiveInteger sequence = sequences.get(key);
        if (sequence == null) {
            sequences.putIfAbsent(key, new AtomicPositiveInteger());
            sequence = sequences.get(key);
        }

        // 获取当前的调用编号
        int currentSequence = sequence.getAndIncrement();
        // 如果最小权重小于最大权重，表明服务提供者之间的权重是不相等的
        if (maxWeight > 0 && minWeight < maxWeight) {
            // 使用调用编号对权重总和进行取余操作
            int mod = currentSequence % weightSum;
            // 进行 maxWeight 次遍历
            for (int i = 0; i < maxWeight; i++) {
                // 遍历 invokerToWeightMap
                for (Map.Entry<Invoker<T>, IntegerWrapper> each : invokerToWeightMap.entrySet()) {
					// 获取 Invoker
                    final Invoker<T> k = each.getKey();
                    // 获取权重包装类 IntegerWrapper
                    final IntegerWrapper v = each.getValue();
                    
                    // 如果 mod = 0，且权重大于0，此时返回相应的 Invoker
                    if (mod == 0 && v.getValue() > 0) {
                        return k;
                    }
                    
                    // mod != 0，且权重大于0，此时对权重和 mod 分别进行自减操作
                    if (v.getValue() > 0) {
                        v.decrement();
                        mod--;
                    }
                }
            }
        }
        
        // 服务提供者之间的权重相等，此时通过轮询选择 Invoker
        return invokers.get(currentSequence % length);
    }

    // IntegerWrapper 是一个 int 包装类，主要包含了一个自减方法。
    private static final class IntegerWrapper {
        private int value;

        public void decrement() {
            this.value--;
        }
        
        // 省略部分代码
    }
}
```

如上，RoundRobinLoadBalance 的每行代码都不是很难理解，但是将它们组合在一起之后，就不是很好理解了。所以下面我们举例进行说明，假设我们有三台服务器 servers = [A, B, C]，对应的权重为 weights = [2, 5, 1]。接下来对上面的逻辑进行简单的模拟。

mod = 0：满足条件，此时直接返回服务器 A

mod = 1：需要进行一次递减操作才能满足条件，此时返回服务器 B

mod = 2：需要进行两次递减操作才能满足条件，此时返回服务器 C

mod = 3：需要进行三次递减操作才能满足条件，经过递减后，服务器权重为 [1, 4, 0]，此时返回服务器 A

mod = 4：需要进行四次递减操作才能满足条件，经过递减后，服务器权重为 [0, 4, 0]，此时返回服务器 B

mod = 5：需要进行五次递减操作才能满足条件，经过递减后，服务器权重为 [0, 3, 0]，此时返回服务器 B

mod = 6：需要进行六次递减操作才能满足条件，经过递减后，服务器权重为 [0, 2, 0]，此时返回服务器 B

mod = 7：需要进行七次递减操作才能满足条件，经过递减后，服务器权重为 [0, 1, 0]，此时返回服务器 B

经过8次调用后，我们得到的负载均衡结果为 [A, B, C, A, B, B, B, B]，次数比 A:B:C = 2:5:1，等于权重比。当 sequence = 8 时，mod = 0，此时重头再来。从上面的模拟过程可以看出，当 mod >= 3 后，服务器 C 就不会被选中了，因为它的权重被减为0了。当 mod >= 4 后，服务器 A 的权重被减为0，此后 A 就不会再被选中。

以上是 2.6.4 版本的 RoundRobinLoadBalance 分析过程，2.6.4 版本的 RoundRobinLoadBalance 在某些情况下存在着比较严重的性能问题，该问题最初是在 [issue #2578](https://github.com/apache/dubbo/issues/2578) 中被反馈出来。问题出在了 Invoker 的返回时机上，RoundRobinLoadBalance 需要在`mod == 0 && v.getValue() > 0` 条件成立的情况下才会被返回相应的 Invoker。假如 mod 很大，比如 10000，50000，甚至更大时，doSelect 方法需要进行很多次计算才能将 mod 减为0。由此可知，doSelect 的效率与 mod 有关，时间复杂度为 O(mod)。mod 又受最大权重 maxWeight 的影响，因此当某个服务提供者配置了非常大的权重，此时 RoundRobinLoadBalance 会产生比较严重的性能问题。这个问题被反馈后，社区很快做了回应。并对 RoundRobinLoadBalance 的代码进行了重构，将时间复杂度优化至了常量级别。这个优化可以说很好了，下面我们来学习一下优化后的代码。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "roundrobin";

    private final ConcurrentMap<String, AtomicPositiveInteger> sequences = new ConcurrentHashMap<String, AtomicPositiveInteger>();

    private final ConcurrentMap<String, AtomicPositiveInteger> indexSeqs = new ConcurrentHashMap<String, AtomicPositiveInteger>();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int length = invokers.size();
        int maxWeight = 0;
        int minWeight = Integer.MAX_VALUE;
        final List<Invoker<T>> invokerToWeightList = new ArrayList<>();
        
        // 查找最大和最小权重
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            maxWeight = Math.max(maxWeight, weight);
            minWeight = Math.min(minWeight, weight);
            if (weight > 0) {
                invokerToWeightList.add(invokers.get(i));
            }
        }
        
        // 获取当前服务对应的调用序列对象 AtomicPositiveInteger
        AtomicPositiveInteger sequence = sequences.get(key);
        if (sequence == null) {
            // 创建 AtomicPositiveInteger，默认值为0
            sequences.putIfAbsent(key, new AtomicPositiveInteger());
            sequence = sequences.get(key);
        }
        
        // 获取下标序列对象 AtomicPositiveInteger
        AtomicPositiveInteger indexSeq = indexSeqs.get(key);
        if (indexSeq == null) {
            // 创建 AtomicPositiveInteger，默认值为 -1
            indexSeqs.putIfAbsent(key, new AtomicPositiveInteger(-1));
            indexSeq = indexSeqs.get(key);
        }

        if (maxWeight > 0 && minWeight < maxWeight) {
            length = invokerToWeightList.size();
            while (true) {
                int index = indexSeq.incrementAndGet() % length;
                int currentWeight = sequence.get() % maxWeight;

                // 每循环一轮（index = 0），重新计算 currentWeight
                if (index == 0) {
                    currentWeight = sequence.incrementAndGet() % maxWeight;
                }
                
                // 检测 Invoker 的权重是否大于 currentWeight，大于则返回
                if (getWeight(invokerToWeightList.get(index), invocation) > currentWeight) {
                    return invokerToWeightList.get(index);
                }
            }
        }
        
        // 所有 Invoker 权重相等，此时进行普通的轮询即可
        return invokers.get(sequence.incrementAndGet() % length);
    }
}
```

上面代码的逻辑是这样的，每进行一轮循环，重新计算 currentWeight。如果当前 Invoker 权重大于 currentWeight，则返回该 Invoker。下面举例说明，假设服务器 [A, B, C] 对应权重 [5, 2, 1]。

第一轮循环，currentWeight = 1，可返回 A 和 B

第二轮循环，currentWeight = 2，返回 A

第三轮循环，currentWeight = 3，返回 A

第四轮循环，currentWeight = 4，返回 A

第五轮循环，currentWeight = 0，返回 A, B, C

如上，这里的一轮循环是指 index 再次变为0所经历过的循环，这里可以把 index = 0 看做是一轮循环的开始。每一轮循环的次数与 Invoker 的数量有关，Invoker 数量通常不会太多，所以我们可以认为上面代码的时间复杂度为常数级。

重构后的 RoundRobinLoadBalance 看起来已经很不错了，但是在代码更新不久后，很快又被重构了。这次重构原因是新的 RoundRobinLoadBalance 在某些情况下选出的服务器序列不够均匀。比如，服务器 [A, B, C] 对应权重 [5, 1, 1]。进行7次负载均衡后，选择出来的序列为 [A, A, A, A, A, B, C]。前5个请求全部都落在了服务器 A上，这将会使服务器 A 短时间内接收大量的请求，压力陡增。而 B 和 C 此时无请求，处于空闲状态。而我们期望的结果是这样的 [A, A, B, A, C, A, A]，不同服务器可以穿插获取请求。为了增加负载均衡结果的平滑性，社区再次对 RoundRobinLoadBalance 的实现进行了重构，这次重构参考自 Nginx 的平滑加权轮询负载均衡。每个服务器对应两个权重，分别为 weight 和 currentWeight。其中 weight 是固定的，currentWeight 会动态调整，初始值为0。当有新的请求进来时，遍历服务器列表，让它的 currentWeight 加上自身权重。遍历完成后，找到最大的 currentWeight，并将其减去权重总和，然后返回相应的服务器即可。

上面描述不是很好理解，下面还是举例进行说明。这里仍然使用服务器 [A, B, C] 对应权重 [5, 2, 1] 的例子说明，现在有7个请求依次进入负载均衡逻辑，选择过程如下：

| 请求编号 | currentWeight 数组 | 选择结果 | 减去权重总和后的 currentWeight 数组 |
| :------- | :----------------- | :------- | :---------------------------------- |
| 1        | [5, 1, 1]          | A        | [-2, 1, 1]                          |
| 2        | [3, 2, 2]          | A        | [-4, 2, 2]                          |
| 3        | [1, 3, 3]          | B        | [1, -4, 3]                          |
| 4        | [6, -3, 4]         | A        | [-1, -3, 4]                         |
| 5        | [4, -2, 5]         | C        | [4, -2, -2]                         |
| 6        | [9, -1, -1]        | A        | [2, -1, -1]                         |
| 7        | [7, 0, 0]          | A        | [0, 0, 0]                           |

如上，经过平滑性处理后，得到的服务器序列为 [A, A, B, A, C, A, A]，相比之前的序列 [A, A, A, A, A, B, C]，分布性要好一些。初始情况下 currentWeight = [0, 0, 0]，第7个请求处理完后，currentWeight 再次变为 [0, 0, 0]。

#### 3.x 版本

dubbo3 版本又对RoundRobinLoadBalance进行了优化：使用 `ConcurrentHashMapUtils.computeIfAbsent` 替代双重检查锁，代码更简洁；移除 `updateLock` 原子锁，改用 `removeIf` 直接清理过期节点；新增 `getInvokerAddrList` 方法用于单元测试。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "roundrobin";

    private static final int RECYCLE_PERIOD = 60000;

    private final ConcurrentMap<String, ConcurrentMap<String, WeightedRoundRobin>> methodWeightMap =
            new ConcurrentHashMap<>();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // 构建方法级缓存键
        String key = invokers.get(0).getUrl().getServiceKey() + "." + RpcUtils.getMethodName(invocation);
        // 获取或创建方法对应的权重映射表
        ConcurrentMap<String, WeightedRoundRobin> map = 
            ConcurrentHashMapUtils.computeIfAbsent(methodWeightMap, key, k -> new ConcurrentHashMap<>());

        int totalWeight = 0;
        long maxCurrent = Long.MIN_VALUE;
        long now = System.currentTimeMillis();
        Invoker<T> selectedInvoker = null;
        WeightedRoundRobin selectedWRR = null;

        // 遍历所有可用Invoker，计算权重并选择当前最优节点
        for (Invoker<T> invoker : invokers) {
            String identifyString = invoker.getUrl().toIdentityString();
            int weight = getWeight(invoker, invocation);

            // 获取或创建该Invoker的权重包装对象
            WeightedRoundRobin weightedRoundRobin = ConcurrentHashMapUtils.computeIfAbsent(map, identifyString, k -> {
                WeightedRoundRobin wrr = new WeightedRoundRobin();
                wrr.setWeight(weight);
                return wrr;
            });

            // 权重发生变化时更新
            if (weight != weightedRoundRobin.getWeight()) {
                weightedRoundRobin.setWeight(weight);
            }

            // 增加当前权重：current += weight
            long cur = weightedRoundRobin.increaseCurrent();
            weightedRoundRobin.setLastUpdate(now);

            // 找出当前权重最大的Invoker
            if (cur > maxCurrent) {
                maxCurrent = cur;
                selectedInvoker = invoker;
                selectedWRR = weightedRoundRobin;
            }

            // 累加总权重
            totalWeight += weight;
        }

        // 清理过期节点（超过60秒未使用）
        if (invokers.size() != map.size()) {
            map.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
        }

        // 选择当前权重最大的节点，并调整其权重：current -= totalWeight
        if (selectedInvoker != null) {
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }

        // 通常不会执行到这里，默认返回第一个Invoker
        return invokers.get(0);
    }
}
```

权重包装类

```java
protected static class WeightedRoundRobin {
    private int weight; // 固定权重
    private final AtomicLong current = new AtomicLong(0); // 当前动态权重
    private long lastUpdate; // 最后更新时间

    // 增加当前权重并返回新值
    public long increaseCurrent() {
        return current.addAndGet(weight);
    }

    // 选择后调整权重：减去总权重
    public void sel(int total) {
        current.addAndGet(-1 * total);
    }
    
    // 省略getter/setter方法
}
```

Dubbo 3 的实现进一步简化了代码结构，提升了并发性能，同时保持了平滑加权轮询的优势，确保请求在不同性能服务器间的合理分配。

## 3.总结

本篇文章对 Dubbo 中的几种负载均衡实现进行了详细的分析，内容比较多，大家慢慢消化。理解负载均衡代码逻辑的关键之处在于对背景知识的理解，因此大家在阅读源码前，务必先了解每种负载均衡对应的背景知识。