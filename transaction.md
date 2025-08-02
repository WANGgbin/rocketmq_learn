rocketmq 的一个重要的特性是支持事务消息，rocketmq 的事务是满足最终一致性的。

# 流程

- producer 先发送一个 half 消息
- 发送成功后，执行本地事务
- 根据本地事务执行的结果，发送 commit/rollback 消息
- 如果 rocketmq 在特定的时间内没有接收到 commit/rollback 消息，会通过 producer 回查事务状态，据此 commit/rollback 消息，如果查询不到状态，则会尝试。如果超过一定重试次数，还没查询到 producer 事务状态，则 rollback half 消息


# half 消息

当 broker 接受到 prepare half 消息后，会改写消息中的 topic(专门存储 half 消息的 topic)、queueID(0)，并保存原始的 topic + queueID，然后把消息追加到 commit log 中。
broker 中分发消息的任务，会把该消息分发到 half topic 中，所以此消息对 consumer 不可见。

当 producer 发送 commit 消息时，commit 消息中包含 prepare half 消息在 commit log 中的 offset。broker 根据此读取 prepare half 消息，然后恢复原始的消息并追加到 commit log 中，

这样 consumer 就可以看到此消息了。同时，broker 还会往 commit log 中追加一个 topic 为 op_topic 的消息，为什么要追加这个消息呢？

主要用来判断某个 prepare half 消息是否已经达到终态，这类消息不再进行回查。

# 回查

为什么要回查呢？

因为可能由于各种原因导致 producer 并没有给 broker 发送事务最终的状态，比如网络问题、producer 下线等，这样 half 消息永远无法达到终态。


那么如何回查呢？

在 broker 中会有一个 half topic 的消费者，定时(30s)消费 half topic 中的消息。这里有个问题，half 中的消息对应的事务可能已经被提交/回滚了，
这个消费者怎么知道呢？

因为 commitLog、consumeQueue 消息都是追加写的，因此在 producer commit/rollback 的时候，也不能直接更改 half 消息的信息。那怎么办呢？
rmq 的思路是在接收到 producer 的 commit/rollback 请求时，会生成一条 op 信息，存储到特定的 topic 中。

这样在 half topic consumer 消费 half 消息时，首先判断 half 消息是否有对应的 op 信息在 op topic 中，如果在说明该 half 消息已经达到终态了，
继续处理下一条消息。

这里又有个问题。怎么知道 half 消息有没有 op 消息在 op topic 中呢？

op 消息记录了 half 消息在 consumeQueue 中的偏移量。但这样还不够。因为 op 消息的顺序并不是与 half 消息顺序一样的，因为每个事务提交的时间点
是不一样的。

消费者在消费 half 消息的时候，首先会从 op topic 中拉一批消息(32)个，然后组成一个 map。然后判断 half 消息是不是在这个 map 中，如果在说明
已经被处理了。如果不在，则会继续取下一批消息加入到 map 中。直到找到对应的 op 消息，或者 op topic 为空，或者从 op topic 获取的最近一条消息
的生产时间-本轮开始消费时间 > transactionTimeout，也就是说超过一定时间，还没有对应的 op 消息及时 op topic 中还有其他消息，消费者也不在
从 op topic 获取消息，而是发起回查请求。

一直到遇到某个 half 消息还未到回查时间，或者本轮消费已经超过一定时间，则结束本轮消费。

注意查的 producer 节点不一定就是 发送 half 消息的节点，而是与发送 half 消息属于同一个 producer group 中的任一个 producer。
另外，需要注意的是：**回查操作，并不需要 broker 主动与 producer 建立 tcp 连接，broker 是通过 producer 与 broker 建立的连接发送回查请求的。**
毕竟 tcp 连接是双向的。

如果查了很多次后，还是无法获取到事务的状态，broker 会忽略这一条 half 消息。

## half 和 op topic 偏移量提交

消费者在回查 half 请求后，这一轮可能仍然无法获取到 half 消息对应事务的状态。如果这个时候停止消费 half topic，等一段时间后再进行下一轮消费，
那么整个 half topic 的消费进度就会被这个超不到状态的 half 消息阻塞，这显然是不能接受的。

所以，rmq 的处理思路是，修改这条 half 信息的回查次数++，然后再写入到 commitLog 中，这样就不会阻塞后续 half 消息的消费了。这其实就是消费者
的重试逻辑。

half topic 的位移提交比较简单。主要我们看看 op topic 偏移量的提交。我们虽然为了判断 half 消息有没有对应的 op 消息，从 op topic 获取很多消息
构造了一个 map，但最后提交的时候，并不能把这些 op 消息都提交了。op 消息能提交的标准是：其对应的 half 消息偏移量被提交了。
核心在下面这个函数：
```java
// 提交 half topic 偏移量
if (newOffset != halfOffset) {
    transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);
}
// 提交 op topic 偏移量
long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);
if (newOpOffset != opOffset) {
    transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);
}

// doneOffset 就是对应的 half 消息被成功消费的 op 消息的偏移量列表
private long calculateOpOffset(List<Long> doneOffset, long oldOffset) {
    Collections.sort(doneOffset);
    long newOffset = oldOffset;
    // doneOffset 中连续的部分才可以被提交
    for (int i = 0; i < doneOffset.size(); i++) {
        if (doneOffset.get(i) == newOffset) {
            newOffset++;
        } else {
            break;
        }
    }
    return newOffset;

}
```