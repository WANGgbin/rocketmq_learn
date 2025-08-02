# 发送给谁

producer 与 consumer 一样也是通过与 nameserver 获取 topic 的路由信息，获取 topic 的所有 messageQueue 信息。

一般通过轮询的方式选择 messageQueue。

不过在发送顺序消息的时候，也可以通过 MessageQueueSelector 自定义 messageQueue 选择逻辑。

# 发送方式

producer 提供了 3 种方式：

- sync

    同步发送。发送，直到收到响应，才返回。
    这种方式消息可靠性高，但是性能最差。

- async

    异步发送，发送后，返回。当收到响应后，执行毁掉函数。
    这种方式消息可靠性较低，性能较好。

- oneWay

    发送就不管了，不关心发送成功还是失败。
    性能最好，但是可靠性最差。

# 消息重投

对于 sync 模式发送的消息，rmq producer 自动重投机制。即在发送失败的时候，重新发送。默认重试次数为 2 次。可配置。

# 顺序消息

producer 支持发送顺序消息，send api 会指定两个参数(MessageQueueSelector selector, Object arg)。producer 内部会通过这两个参数
决定消息发往的 messageQueue。

通过顺序消息，我们可以保证一些消息发往同一个 messageQueue，从而保证了消息的顺序性。当然后消费者也要按照顺序模式消费。

# 事务消息

# 如何提高吞吐

为了尽可能的提高消息吞吐，producer 支持以下特性：
- 批量发送

    发一个消息进行一次网络调用，吞吐太低。producer 支持批量发送(可选)。将消息缓存到 accumulator 中，同时阻塞等待。
    当 accumulator 中累计的消息达到一定大小或者一定时间，才会批量发送消息。获取到发送结果后，同时激活之前阻塞的线程。

- 消息压缩

    当消息大小超过 4kb 的时候，会压缩消息。