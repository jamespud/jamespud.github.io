---
title: 消费者确认与发布者确认
date: 2025-02-06 12:00:00 +0800
categories: [rabbitmq文档]
tags: [doc]
---
# 消费者确认与发布者确认

> [原文](https://www.rabbitmq.com/docs/confirms)

版本：4.0
## 消费者确认与发布者确认
### 概述
本指南涵盖了与数据安全相关的两个特性：消费者确认和发布者确认，具体包括：
- 确认机制存在的原因
- 手动和自动确认模式
- 确认的API，包括批量确认和消息重新入队
- 连接丢失或通道关闭时的自动重新入队
- 通道预取及其对吞吐量的影响
- 最常见的客户端错误
- 发布者确认以及相关的发布者数据安全主题等

在使用消息传递的应用程序中，消费者和发布者端的确认对于数据安全都非常重要。
### 基础知识
使用像RabbitMQ这样的消息代理的系统本质上是分布式的。由于发送的协议方法（消息）不能保证到达对等方或被其成功处理，发布者和消费者都需要一种机制来确认消息的投递和处理情况。RabbitMQ支持的几种消息传递协议都提供了这样的特性。本指南主要介绍AMQP 0 - 9 - 1协议中的这些特性，不过在其他受支持的协议中，基本概念大致相同。

从消费者到RabbitMQ的投递处理确认在消息传递协议中被称为确认；而代理对发布者的确认是一种协议扩展，称为发布者确认。这两个特性基于相同的理念，并且都受到TCP协议的启发。

它们对于从发布者到RabbitMQ节点，以及从RabbitMQ节点到消费者的可靠消息投递至关重要。换句话说，它们对于数据安全至关重要，在这方面，应用程序和RabbitMQ节点同样负有责任。
### 发布者确认与消费者投递确认有关吗？
发布者确认和消费者投递确认是非常相似的特性，它们在不同的场景中解决相似的问题：
- 消费者确认，顾名思义，涵盖了RabbitMQ与消费者之间的通信确认。
- 发布者确认则涵盖了发布者与RabbitMQ之间的通信确认。

然而，这两个特性是完全独立的，彼此互不影响。
- 发布者确认并不关心消费者：它们仅涉及发布者与所连接的节点，以及队列（或流）的主副本之间的交互。
- 消费者确认也不关心发布者：其目的是向RabbitMQ节点确认已成功接收并处理了某个投递的消息，以便可以标记已投递的消息，供后续删除。

有时，发布和消费应用程序需要通过请求和响应进行通信，并且需要对方的明确确认。RabbitMQ教程#6展示了基本的实现方式，而“直接回复”功能则提供了一种无需声明大量短期临时响应队列的实现方法。

不过，本指南不涉及这种类型的通信，提及它只是为了与本指南中重点介绍的消息传递协议特性进行对比。
### （消费者）投递确认
当RabbitMQ将消息投递到消费者时，它需要知道何时可以认为消息已成功发送。哪种逻辑最为合适取决于具体的系统，因此这主要是由应用程序来决定。在AMQP 0 - 9 - 1协议中，这个决定是在使用`basic.consume`方法注册消费者或使用`basic.get`方法按需获取消息时做出的。

如果你更喜欢以示例为导向的分步讲解内容，RabbitMQ教程#2也介绍了消费者确认相关知识。
#### 投递标识符：投递标签
在讨论其他主题之前，解释一下如何标识投递（以及确认是如何对应各自的投递）非常重要。当注册一个消费者（订阅）时，RabbitMQ会使用`basic.deliver`方法推送消息。该方法携带一个`delivery tag`（投递标签），它在一个通道上唯一标识一次投递。因此，投递标签是按通道进行作用域划分的。

投递标签是单调递增的正整数，客户端库也是这样呈现它们的。用于确认投递的客户端库方法会将投递标签作为参数。

由于投递标签是按通道作用域划分的，所以必须在接收消息的同一通道上进行确认。在不同的通道上确认会导致“未知投递标签”的协议异常，并关闭该通道。
#### 消费者确认模式和数据安全考量
当节点将消息投递到消费者时，它必须决定该消息是否应被视为已被消费者处理（或至少已被接收）。由于多种情况（客户端连接、消费者应用程序等）都可能出现故障，这个决定涉及数据安全问题。消息传递协议通常提供一种确认机制，允许消费者向其连接的节点确认消息投递情况。是否使用该机制是在消费者订阅时决定的。

根据所使用的确认模式，RabbitMQ可以在消息发送出去（写入TCP套接字）后立即认为消息已成功投递，也可以在收到客户端明确的（“手动”）确认后才这么认为。手动发送的确认可以是肯定的，也可以是否定的，并使用以下协议方法之一：
- `basic.ack`用于肯定确认。
- `basic.nack`用于否定确认（注意：这是RabbitMQ对AMQP 0 - 9 - 1协议的扩展）。
- `basic.reject`也用于否定确认，但与`basic.nack`相比有一个限制。

下面将讨论这些方法在客户端库API中的使用方式。

肯定确认只是指示RabbitMQ将消息记录为已投递，并可以丢弃。使用`basic.reject`进行的否定确认也有相同的效果。它们的主要区别在于语义：肯定确认表示消息已成功处理，而否定确认则表明消息未被成功处理，但仍应被删除。

在自动确认模式下，消息在发送后立即被视为已成功投递。这种模式以牺牲消息投递和消费者处理的安全性为代价，换取更高的吞吐量（前提是消费者能够跟上处理速度）。这种模式通常被称为“即发即弃”模式。与手动确认模式不同，如果在消息成功投递之前，消费者的TCP连接或通道关闭，服务器发送的消息将会丢失。因此，自动消息确认应被视为不安全的，并不适用于所有的工作负载。

在使用自动确认模式时，另一个需要重点考虑的因素是消费者过载。手动确认模式通常会结合有限的通道预取设置，以限制通道上未完成（“进行中”）的投递数量。然而，在自动确认模式下，从定义上来说没有这样的限制。因此，消费者可能会被消息投递的速率压垮，潜在地在内存中积压大量未处理的消息，导致堆内存耗尽，甚至被操作系统终止进程。一些客户端库会应用TCP背压机制（在未处理的投递积压量下降到特定限制以下之前，停止从套接字读取数据）。因此，自动确认模式仅适用于能够高效且稳定地处理消息投递的消费者。
#### 肯定确认投递
用于确认投递的API方法通常作为客户端库中通道上的操作来提供。Java客户端用户会使用`Channel#basicAck`和`Channel#basicNack`分别执行`basic.ack`和`basic.nack`操作。下面是一个Java客户端示例，展示了肯定确认的用法：
```java
// 此示例假设已有一个通道实例
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "a-consumer-tag",
    new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties properties,
                                   byte[] body) throws IOException {
            long deliveryTag = envelope.getDeliveryTag();
            // 肯定确认单个投递，消息将被丢弃
            channel.basicAck(deliveryTag, false);
        }
    });
```
在.NET客户端中，对应的方法分别是`IModel#BasicAck`和`IModel#BasicNack`。下面是一个使用该客户端进行肯定确认的示例：
```csharp
// 此示例假设已有一个通道（IModel）实例
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (ch, ea) =>
{
    var body = ea.Body.ToArray();
    // 肯定确认单个投递，消息将被丢弃
    channel.BasicAck(ea.DeliveryTag, false);
};
String consumerTag = channel.BasicConsume(queueName, false, consumer);
```
#### 一次性确认多个投递
手动确认可以进行批量操作，以减少网络流量。这是通过将确认方法（见上文）的`multiple`字段设置为`true`来实现的。需要注意的是，从历史上看，`basic.reject`方法没有这个字段，这就是为什么RabbitMQ作为协议扩展引入了`basic.nack`方法。

当`multiple`字段设置为`true`时，RabbitMQ会确认所有未确认的投递标签，直到并包括确认中指定的标签。与所有与确认相关的内容一样，这也是按通道进行作用域划分的。例如，假设在通道`Ch`上有投递标签5、6、7和8未被确认，当在该通道上收到一个确认帧，其中`delivery_tag`设置为8且`multiple`设置为`true`时，从5到8的所有标签都将被确认。如果`multiple`设置为`false`，那么投递5、6和7仍将保持未确认状态。

要使用RabbitMQ Java客户端批量确认多个投递，可以将`Channel#basicAck`的`multiple`参数设置为`true`：
```java
// 此示例假设已有一个通道实例
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "a-consumer-tag",
    new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties properties,
                                   byte[] body) throws IOException {
            long deliveryTag = envelope.getDeliveryTag();
            // 肯定确认到此投递标签为止的所有投递
            channel.basicAck(deliveryTag, true);
        }
    });
```
.NET客户端的原理基本相同：
```csharp
// 此示例假设已有一个通道（IModel）实例
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (ch, ea) =>
{
    var body = ea.Body.ToArray();
    // 肯定确认到此投递标签为止的所有投递
    channel.BasicAck(ea.DeliveryTag, true);
};
String consumerTag = channel.BasicConsume(queueName, false, consumer);
```
#### 否定确认和消息重新入队
有时，一个消费者无法立即处理某个投递的消息，但其他实例可能能够处理。在这种情况下，可能需要将消息重新入队，让其他消费者接收并处理。`basic.reject`和`basic.nack`就是用于此目的的两个协议方法。

这些方法通常用于否定确认一次投递。这样的投递消息可以被代理丢弃、发送到死信队列或重新入队。这种行为由`requeue`字段控制。当该字段设置为`true`时，代理会将指定投递标签的消息（或多个消息，稍后会解释）重新入队。或者，当该字段设置为`false`时，如果配置了死信交换器，消息将被路由到死信交换器，否则将被丢弃。

这两个方法通常作为客户端库中通道上的操作来提供。Java客户端用户会使用`Channel#basicReject`和`Channel#basicNack`分别执行`basic.reject`和`basic.nack`操作：
```java
// 此示例假设已有一个通道实例
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "a-consumer-tag",
    new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties properties,
                                   byte[] body) throws IOException {
            long deliveryTag = envelope.getDeliveryTag();
            // 否定确认，消息将被丢弃
            channel.basicReject(deliveryTag, false);
        }
    });
// 此示例假设已有一个通道实例
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "a-consumer-tag",
    new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties properties,
                                   byte[] body) throws IOException {
            long deliveryTag = envelope.getDeliveryTag();
            // 将消息重新入队
            channel.basicReject(deliveryTag, true);
        }
    });
```
在.NET客户端中，对应的方法分别是`IModel#BasicReject`和`IModel#BasicNack`：
```csharp
// 此示例假设已有一个通道（IModel）实例
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (ch, ea) =>
{
    var body = ea.Body.ToArray();
    // 否定确认，消息将被丢弃
    channel.BasicReject(ea.DeliveryTag, false);
};
String consumerTag = channel.BasicConsume(queueName, false, consumer);
// 此示例假设已有一个通道（IModel）实例
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (ch, ea) =>
{
    var body = ea.Body.ToArray();
    // 将消息重新入队
    channel.BasicReject(ea.DeliveryTag, true);
};
String consumerTag = channel.BasicConsume(queueName, false, consumer);
```
当一条消息被重新入队时，如果可能的话，它将被放置在队列中的原始位置。如果不行（由于多个消费者共享队列时，其他消费者的并发投递和确认），消息将被重新入队到更靠近队列头部的位置。

重新入队的消息可能会根据其在队列中的位置和活动消费者通道使用的预取值，立即准备好进行重新投递。这意味着，如果所有消费者都因为临时条件无法处理某个投递而将其重新入队，就会创建一个重新入队/重新投递循环。这样的循环在网络带宽和CPU资源方面的成本都很高。消费者实现可以跟踪重新投递的次数，对于无法处理的消息，永久性地拒绝（丢弃）它们，或者延迟后再安排重新入队。

可以使用`basic.nack`方法一次性拒绝或重新入队多个消息。这是它与`basic.reject`的区别。它接受一个额外的参数`multiple`。下面是一个Java客户端示例：
```java
// 此示例假设已有一个通道实例
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "a-consumer-tag",
    new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag,
                                   Envelope envelope,
                                   AMQP.BasicProperties properties,
                                   byte[] body) throws IOException {
            long deliveryTag = envelope.getDeliveryTag();
            // 将到此投递标签为止的所有未确认投递重新入队
            channel.basicNack(deliveryTag, true, true);
        }
    });
```
.NET客户端的工作方式非常相似：
```csharp
// 此示例假设已有一个通道（IModel）实例
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (ch, ea) =>
{
    var body = ea.Body.ToArray();
    // 将到此投递标签为止的所有未确认投递重新入队
    channel.BasicNack(ea.DeliveryTag, true, true);
};
String consumerTag = channel.BasicConsume(queueName, false, consumer);
```
#### 通道预取设置（QoS）
消息是异步投递（发送）给客户端的，在任何给定时刻，一个通道上可能有多个消息处于“传输中”状态。客户端的手动确认本质上也是异步的，但方向相反。

这就形成了一个未确认投递的滑动窗口。

对于大多数消费者来说，限制这个窗口的大小是有意义的，以避免在消费者端出现无限制的缓冲区（堆内存）增长问题。这可以通过使用`basic.qos`方法设置“预取计数”值来实现。该值定义了通道上允许的未确认投递的最大数量。当未确认投递的数量达到配置的计数时，RabbitMQ将停止在该通道上投递更多消息，直到至少有一个未完成的投递被确认。

值为`0`表示“无限制”，允许任意数量的未确认消息。

例如，假设在通道`Ch`上有四个投递标签为5、6、7和8的消息未被确认，并且通道`Ch`的预取计数设置为4，那么RabbitMQ将不会在`Ch`上推送更多的投递，除非至少有一个未完成的投递被确认。

当在该通道上收到一个确认帧，其中`delivery_tag`设置为5（或6、7、8）时，RabbitMQ会注意到并再投递一个消息。一次性确认多个消息将使多个消息可供投递。

值得重申的是，投递和客户端手动确认的流程是完全异步的。因此，如果在已有消息处于传输中时更改预取值，就会自然产生竞争条件，通道上可能会暂时出现超过预取计数的未确认消息。
#### 每个通道、每个消费者和全局预取
QoS设置可以针对特定通道或特定消费者进行配置。《消费者预取指南》解释了这种作用域的影响。
#### 预取和轮询消费者
QoS预取设置对使用`basic.get`（“拉取API”）获取的消息没有影响，即使在手动确认模式下也是如此。
#### 消费者确认模式、预取和吞吐量
确认模式和QoS预取值对消费者吞吐量有显著影响。一般来说，增加预取值会提高消息投递到消费者的速率。自动确认模式能产生最高的投递速率。然而，在这两种情况下，已投递但尚未处理的消息数量也会增加，从而增加消费者的内存消耗。

自动确认模式或无限制预取的手动确认模式应谨慎使用。消费大量消息而不进行确认的消费者会导致其连接的节点内存消耗增加。找到合适的预取值需要反复试验，并且因工作负载而异。通常，100到300之间的值能提供最佳吞吐量，并且不会有使消费者或节点内存溢出的风险。

### 消费者确认：常见客户端错误
以下是一些常见的与消费者确认相关的客户端错误：
- **在错误的通道上确认**：如前所述，投递标签是按通道作用域划分的。这意味着确认必须在接收消息的同一通道上进行。在不同的通道上确认会导致“未知投递标签”的协议异常，并关闭该通道。
- **确认已确认的投递**：再次确认已确认的投递是一种编程错误，通常表明客户端代码中存在逻辑问题。虽然这不会导致协议错误，但会增加不必要的网络流量。
- **未确认投递**：忘记确认投递是一个常见的错误，特别是在使用手动确认模式时。这会导致消息在队列中无限期保留，占用内存，并可能导致节点内存耗尽。
- **重新入队循环**：如前所述，如果所有消费者都因为临时条件无法处理某个投递而将其重新入队，就会创建一个重新入队/重新投递循环。这样的循环在网络带宽和CPU资源方面的成本都很高。消费者实现应该跟踪重新投递的次数，对于无法处理的消息，永久性地拒绝（丢弃）它们，或者延迟后再安排重新入队。

### 发布者确认
发布者确认（也称为“确认”）是一种允许发布者了解其消息是否已被RabbitMQ成功接收的机制。这对于确保消息不会在传输过程中丢失非常重要，特别是在网络不稳定或不可靠的环境中。

在启用发布者确认之前，发布者无法确定其消息是否已被RabbitMQ接收。例如，当发布者将消息写入TCP套接字时，它不能保证消息已被RabbitMQ处理，因为消息可能在传输过程中丢失，或者RabbitMQ可能在处理消息之前崩溃。

发布者确认通过为每个已发布的消息分配一个唯一的序列号来工作。当RabbitMQ成功接收并处理消息时，它会向发布者发送一个确认，其中包含该消息的序列号。发布者可以使用这个确认来跟踪哪些消息已被成功接收，哪些需要重新发布。

### 启用发布者确认
要启用发布者确认，发布者必须在通道上设置`confirm.select`标志。这可以通过调用客户端库中的相应方法来完成。例如，在RabbitMQ Java客户端中，可以使用以下代码启用发布者确认：
```java
Channel channel = connection.createChannel();
channel.confirmSelect();
```
在.NET客户端中，可以使用以下代码启用发布者确认：
```csharp
IModel channel = connection.CreateModel();
channel.ConfirmSelect();
```
### 处理确认
一旦启用了发布者确认，发布者就可以开始接收确认。确认可以是单个确认或批量确认，具体取决于RabbitMQ的实现。

单个确认是指RabbitMQ为每个已成功接收的消息发送一个单独的确认。批量确认是指RabbitMQ在一定时间间隔或一定数量的消息成功接收后，发送一个包含多个消息序列号的确认。

发布者可以通过注册一个确认监听器来处理确认。确认监听器是一个回调函数，当RabbitMQ发送确认时，客户端库会调用该函数。例如，在RabbitMQ Java客户端中，可以使用以下代码注册一个确认监听器：
```java
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        // 处理肯定确认
    }

    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        // 处理否定确认
    }
});
```
在上述代码中，`handleAck`方法用于处理肯定确认，`handleNack`方法用于处理否定确认。`deliveryTag`参数包含已确认消息的序列号，`multiple`参数表示该确认是否是批量确认。如果`multiple`为`true`，则表示确认了从1到`deliveryTag`的所有消息。

在.NET客户端中，可以使用以下代码注册一个确认监听器：
```csharp
channel.ConfirmSelect();
channel.Confirm += (sender, ea) => {
    if (ea.Ack)
    {
        // 处理肯定确认
    }
    else
    {
        // 处理否定确认
    }
};
```
### 发布者确认和事务
发布者确认和事务是两种不同的机制，用于确保消息的可靠投递。事务提供了一种原子性操作，允许发布者在一个事务中发布多个消息，并在事务提交时确保所有消息都被成功处理。如果事务失败，所有已发布的消息将被回滚。

虽然事务提供了很强的可靠性保证，但它们的性能开销比发布者确认要高。这是因为事务需要在RabbitMQ节点上进行额外的协调和日志记录操作。因此，在性能要求较高的场景中，通常建议使用发布者确认而不是事务。

### 发布者确认和数据安全
发布者确认对于确保消息的可靠投递至关重要，特别是在需要保证数据安全的场景中。通过使用发布者确认，发布者可以确保其消息已被RabbitMQ成功接收，从而避免消息丢失。

然而，发布者确认并不能保证消息一定会被消费者成功处理。这是因为消息在被RabbitMQ接收后，可能会在投递到消费者的过程中丢失，或者消费者可能无法处理消息。因此，在需要确保消息被消费者成功处理的场景中，还需要结合消费者确认机制。

### 总结
消费者确认和发布者确认是RabbitMQ中确保数据安全的两个重要特性。消费者确认允许RabbitMQ了解消息是否已被消费者成功处理，而发布者确认允许发布者了解其消息是否已被RabbitMQ成功接收。

通过正确使用这些特性，应用程序可以确保消息在从发布者到消费者的传输过程中不会丢失，从而提高系统的可靠性和稳定性。然而，在使用这些特性时，需要注意它们的性能影响，并根据具体的应用场景进行适当的配置。