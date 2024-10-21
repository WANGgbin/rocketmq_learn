rocketmq 的一个重要的特性是支持事务消息，rocketmq 的事务是满足最终一致性的。

# 流程

- producer 先发送一个 half 消息
- 发送成功后，执行本地事务
- 根据本地事务执行的结果，发送 commit/rollback 消息
- 如果 rocketmq 在特定的时间内没有接收到 commit/rollback 消息，会通过 producer 回查事务状态，据此 commit/rollback 消息，如果查询不到状态，则会尝试。如果超过一定重试次数，还没查询到 producer 事务状态，则 rollback half 消息


# half 消息

当 broker 结束到 prepare half 消息后，会改写消息中的 topic(专门存储 half 消息的 topic)、queueID(0)，并保存原始的 topic + queueID，然后把消息追加到 commit log 中。<br>
broker 中分发消息的任务，会把该消息分发到 half topic 中，所以此消息对 consumer 不可见。

当 producer 发送 commit 消息时候，commit 消息中包含 prepare half 消息在 commit log 中的 offset。broker 根据此读取 prepare half 消息，然后恢复原始的消息并追加到 commit log 中，<br>
这样 consumer 就可以看到此消息了。同时，broker 还会往 commit log 中追加一个 topic 为 op_topic 的消息，为什么要追加这个消息呢？<br>
主要用来判断某个 prepare half 消息是否已经达到终态，这类消息不再进行回查。

# 回查

broker 中模拟了一个消费者专门用来消费 half topic 和 op topic 中的消息。回查的时候，先读取 op topic 中的消息，op topic 的每个消息包含对应的 half 消息在 queue 中的 offset。<br>
然后消费 half topic 中的消息，判断是否有对应的 op 消息，如果没有则把该 half 消息再次添加到 commit log，同时向 producer 发起回查。