---
title: RabbitMQ教程 - 使用发布者确认实现可靠发布
date: 2025-02-20 12:00:00 +0800
categories: [rabbitmq文档]
tags: [doc]
---
# RabbitMQ教程 - 使用发布者确认实现可靠发布
## 发布者确认
发布者确认是RabbitMQ为实现可靠发布而推出的扩展功能。当在某个通道上启用发布者确认后，客户端发布的消息会由代理服务器进行异步确认，这意味着消息在服务器端已得到处理。
### 概述（使用Java客户端）
在本教程中，我们将使用发布者确认功能，确保发布的消息已安全到达代理服务器。我们会介绍几种使用发布者确认的策略，并说明它们的优缺点。
### 在通道上启用发布者确认
发布者确认是对AMQP 0.9.1协议的RabbitMQ扩展，因此默认是未启用的。可以通过`confirmSelect`方法在通道级别启用发布者确认：
```java
Channel channel = connection.createChannel();
channel.confirmSelect();
```
你期望使用发布者确认的每个通道都必须调用此方法。确认功能只需启用一次，而不是为每条发布的消息都启用。
### 策略1：单独发布消息
让我们从使用确认进行发布的最简单方法开始，即发布一条消息并同步等待其确认：
```java
while (thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    // 设置5秒超时
    channel.waitForConfirmsOrDie(5_000);
}
```
在上述示例中，我们像往常一样发布消息，并使用`Channel#waitForConfirmsOrDie(long)`方法等待其确认。一旦消息得到确认，该方法就会返回。如果消息在超时时间内未得到确认，或者被否定确认（即代理服务器由于某种原因无法处理该消息），该方法将抛出异常。异常处理通常包括记录错误消息和/或重试发送消息。

不同的客户端库对同步处理发布者确认的方式有所不同，因此请务必仔细阅读你正在使用的客户端文档。

这种技术非常直接，但也有一个主要缺点：它会显著降低发布速度，因为一条消息的确认会阻塞后续所有消息的发布。这种方法每秒发布的消息数量不会超过几百条。不过，对于某些应用程序来说，这可能已经足够。
### 发布者确认是异步的吗？
我们在开头提到，代理服务器是异步确认已发布消息的，但在第一个示例中，代码会同步等待，直到消息得到确认。实际上，客户端是异步接收确认信息的，并相应地解除对`waitForConfirmsOrDie`调用的阻塞。可以将`waitForConfirmsOrDie`看作是一个同步辅助方法，其底层依赖异步通知机制。
### 策略2：批量发布消息
为了改进前面的示例，我们可以批量发布消息，并等待整个批次的消息都得到确认。以下示例每次发布100条消息为一批：
```java
int batchSize = 100;
int outstandingMessageCount = 0;
while (thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    outstandingMessageCount++;
    if (outstandingMessageCount == batchSize) {
        channel.waitForConfirmsOrDie(5_000);
        outstandingMessageCount = 0;
    }
}
if (outstandingMessageCount > 0) {
    channel.waitForConfirmsOrDie(5_000);
}
```
与等待单个消息的确认相比，等待一批消息的确认可以显著提高吞吐量（如果连接远程RabbitMQ节点，吞吐量可提升20 - 30倍）。缺点是在出现故障的情况下，我们无法确切知道哪里出了问题，因此可能需要在内存中保留一整批消息，以便记录有意义的信息或重新发布这些消息。而且这个解决方案仍然是同步的，所以它会阻塞消息的发布。
### 策略3：异步处理发布者确认
代理服务器异步确认已发布的消息，我们只需在客户端注册一个回调函数，以便在收到确认时得到通知：
```java
Channel channel = connection.createChannel();
channel.confirmSelect();
channel.addConfirmListener((sequenceNumber, multiple) -> {
    // 消息确认时的代码
}, (sequenceNumber, multiple) -> {
    // 消息被否定确认时的代码
});
```
这里有两个回调函数：一个用于处理确认的消息，另一个用于处理被否定确认的消息（即代理服务器认为丢失的消息）。每个回调函数都有两个参数：
- **序列号**：用于标识已确认或被否定确认消息的编号。我们很快会了解如何将其与发布的消息关联起来。
- **multiple**：这是一个布尔值。如果为`false`，则表示只有一条消息被确认/否定确认；如果为`true`，则表示所有序列号小于或等于该序列号的消息都被确认/否定确认。

可以在发布消息前使用`Channel#getNextPublishSeqNo()`获取序列号：
```java
int sequenceNumber = channel.getNextPublishSeqNo());
ch.basicPublish(exchange, queue, properties, body);
```
一种将消息与序列号关联的简单方法是使用映射（map）。假设我们要发布字符串，因为字符串很容易转换为字节数组进行发布。以下是一个使用映射将发布序列号与消息字符串内容关联起来的代码示例：
```java
ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();
// ... 确认回调函数的代码稍后添加
String body = "...";
outstandingConfirms.put(channel.getNextPublishSeqNo(), body);
channel.basicPublish(exchange, queue, properties, body.getBytes());
```
现在，发布代码使用映射跟踪出站消息。当收到确认时，我们需要清理这个映射；当消息被否定确认时，需要进行类似记录警告信息的操作：
```java
ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();
ConfirmCallback cleanOutstandingConfirms = (sequenceNumber, multiple) -> {
    if (multiple) {
        ConcurrentNavigableMap<Long, String> confirmed = outstandingConfirms.headMap(
                sequenceNumber, true
        );
        confirmed.clear();
    } else {
        outstandingConfirms.remove(sequenceNumber);
    }
};
channel.addConfirmListener(cleanOutstandingConfirms, (sequenceNumber, multiple) -> {
    String body = outstandingConfirms.get(sequenceNumber);
    System.err.format(
            "Message with body %s has been nack-ed. Sequence number: %d, multiple: %b%n",
            body, sequenceNumber, multiple
    );
    cleanOutstandingConfirms.handle(sequenceNumber, multiple);
});
// ... 发布代码
```
前面的示例包含一个在收到确认时清理映射的回调函数。请注意，这个回调函数同时处理单个确认和多个确认的情况。当收到确认时会使用这个回调函数（作为`Channel#addConfirmListener`的第一个参数）。被否定确认消息的回调函数会获取消息内容并发出警告，然后重用前面的回调函数清理未确认消息的映射（无论消息是被确认还是被否定确认，其在映射中的相应条目都必须删除）。
### 如何跟踪未确认消息？
我们的示例使用`ConcurrentNavigableMap`来跟踪未确认消息。这种数据结构很方便，原因有几个。它可以轻松地将序列号与消息（无论消息数据是什么）关联起来，并且可以轻松清理到给定序列号的条目（以处理多个确认/否定确认）。最后，它支持并发访问，因为确认回调函数是在客户端库所属的线程中调用的，该线程应与发布线程不同。

除了使用复杂的映射实现外，还有其他方法可以跟踪未确认消息，比如使用简单的并发哈希映射和一个变量来跟踪发布序列的下限，但这些方法通常更复杂，不属于本教程的范畴。

总之，异步处理发布者确认通常需要以下步骤：
1. 提供一种将发布序列号与消息关联起来的方法。
2. 在通道上注册确认监听器，以便在发布者确认/否定确认到达时得到通知，从而执行相应的操作，比如记录日志或重新发布被否定确认的消息。在此步骤中，序列号与消息的关联机制可能也需要进行一些清理工作。
3. 在发布消息前跟踪发布序列号。
### 重新发布被否定确认的消息？
在相应的回调函数中重新发布被否定确认的消息可能看起来很诱人，但应该避免这种做法，因为确认回调函数是在I/O线程中调度的，而通道不应该在这个线程中执行操作。更好的解决方案是将消息放入内存队列中，由发布线程进行轮询。像`ConcurrentLinkedQueue`这样的类是在确认回调函数和发布线程之间传递消息的不错选择。
### 总结
在某些应用程序中，确保发布的消息到达代理服务器至关重要。发布者确认是RabbitMQ的一项功能，有助于满足这一要求。发布者确认本质上是异步的，但也可以同步处理。实现发布者确认并没有固定的方法，这通常取决于应用程序和整个系统的限制。典型的技术包括：
- 单独发布消息，同步等待确认：简单，但吞吐量非常有限。
- 批量发布消息，同步等待一批消息的确认：简单，吞吐量合理，但在出现问题时难以排查。
- 异步处理：性能最佳，资源利用率高，在出错时能更好地控制操作，但正确实现可能比较复杂。
### 整合
`PublisherConfirms.java`类包含了我们介绍的这些技术的代码。我们可以编译并直接执行它，查看每种技术的性能表现：

```bash
javac -cp $CP PublisherConfirms.java
java -cp $CP PublisherConfirms
```
输出结果可能如下：
```
Published 50,000 messages individually in 5,549 ms
Published 50,000 messages in batch in 2,331 ms
Published 50,000 messages and handled confirms asynchronously in 4,054 ms
```
如果客户端和服务器在同一台机器上，你计算机上的输出应该与此类似。单独发布消息的性能如预期的那样差，但与批量发布相比，异步处理的结果有点令人失望。

发布者确认功能与网络密切相关，所以我们最好使用远程节点进行测试，因为在生产环境中，客户端和服务器通常不在同一台机器上，这样更符合实际情况。可以轻松修改`PublisherConfirms.java`以使用非本地节点：
```java
static Connection createConnection() throws Exception {
    ConnectionFactory cf = new ConnectionFactory();
    cf.setHost("remote-host");
    cf.setUsername("remote-user");
    cf.setPassword("remote-password");
    return cf.newConnection();
}
```
重新编译该类，再次执行，并等待结果：
```
Published 50,000 messages individually in 231,541 ms
Published 50,000 messages in batch in 7,232 ms
Published 50,000 messages and handled confirms asynchronously in 6,332 ms
```
我们可以看到，现在单独发布消息的性能变得非常糟糕。但考虑到客户端和服务器之间存在网络延迟，批量发布和异步处理的性能现在表现相似，而异步处理发布者确认略占优势。

请记住，批量发布实现起来很简单，但在出现否定发布者确认的情况下，很难确定哪些消息未能到达代理服务器。异步处理发布者确认的实现更复杂，但在处理被否定确认的消息时，能提供更细粒度的控制和更好的操作管理。 