新版本的延时消息通过时间轮实现，其原理大概如下：

- 发送延时消息
- 延时消息投递到 delay topic
- 消费 delay topic 消息，添加到 time wheel
- 消息到期后，添加到 commit log，客户端可以消费

rocketmq 从 5.0 开始支持基于 time wheel 的延时消息。我们来看看 rocketmq 的 time wheel 的实现原理。

支持任意时长的延时消息，需要考虑以下几个方面：

- 延时消息如何持久化
- time wheel 结构如何维护
- 如何尽可能提高性能
- 如何异常恢复

# 延时消息如何持久化

延时消息可能很多，所以需要持久化机制。

在 rocketmq 中，producer 发送过来的所有消息都会写入到 commitLog，commitLog 承担消息的持久化机制，包括延时消息。

对于延时消息在未达到预定时间的时候，消息是不能被 consumer 消费的，因此在持久化到 commitLog 之前，broker 会将 msg 的
topic 修改为 time wheel，将 queueID 修改为 0，并保留实际的 topic 和 queueID。

随后跟普通消息处理一样，会把延时消息从 commitLog 分发到对应的 consumeQueue。

# time wheel 如何维护

因为延时消息可能很多，如果我们将 time wheel 直接维护在内存中，内存可能放不下。此外，在 broker 异常退出重启的时候，我们需要
恢复 time wheel 上下文，因此对于 time wheel 上下文需要有持久化机制。

在实现上，时间轮是一个环状的数组，每个数组代表一个 slot，即一个时间点。每个 slot 对应一个列表，存储所有对应时间点的消息。

在 rmq 中，通过两个文件来实现时间轮。time wheel 文件来保存所有的 slot，rmq 并不是支持无限跨度的延时消息，最大支持延后 3 天的消息。
时间精度为 1s，即每个 slot 跨度为 1s。所有的延时消息(消息内容实际存储在 commitLog 中，这里存储的是消息的索引)存储在 timeLog 文件中。
每个 slot 都有个first、last 指针指向 timeLog 中 对应的消息，每个 timeLog 中的成员都通过指针指向前一个成员，从而实现链表。


# 如何尽可能提高性能

整个延时消息的处理包括以下几步：

- 从 consumeQueue 获取消息存放到时间轮
- 从时间轮获取消息并从 commitLog 读取消息内容构建新的 msg(恢复实际 topic、queueID)
- 将新的 msg 重新加入到 commitLog

rmq 为了提高性能，将整个流程划分为多个阶段，每个阶段都是 producer -> queue -> consumer 模式。通过多线程来提高延时消息的处理性能。

除此之外，因为时间轮通过两个磁盘文件来维护，为了尽可能提高读写文件的效率，rmq 通过 mmap 的方式读写磁盘文件。

# 如何异常恢复

如果系统异常退出了，重启后需要恢复上下文，对于延时消息，需要恢复的上下文包括：

- timeLog 文件
- time wheel 文件
- time wheel 的消费进度： commitReadTimeMs，这样重启后便可以接着从对应的 slot 消费消息
- consumeQueue 的消费进度

为了帮助恢复，rmq 有个定时的 flush 任务(默认 1s) 将 timeLog、timeWheel 持久化到磁盘中。同时有个 checkPoint 文件保存：
timeLog flush position、commitReadTimeMs 等信息。这个 checkPoint 就可以理解为是一个一致性的元信息，每刷新一次，保存
最新的一致性信息。之后的 recover 流程都是基于 checkPoint 这个文件进行的。

整体恢复流程如下：

- 因为 timeLog 文件和 timeWheel 文件刷新的时候无法保证一致性，比如 timeLog 持久化了但是 timeWheel 还没持久化，又或者
消息先写入到 timeLog，还没有更新 timeWheel 此时系统异常退出了，timeLog 和 timeWheel 就不一致了。 所以第一步是根据 timeLog 来矫正(revise) timeWheel.
- rmq 保证重启后最多消费近 7 天的消息，第二步会比较 checkPoint 中的 commitReadTimeMs 以及 前 7 天的时间点，取较大者，
后续从这个时间点开始消费 timeWheel
- timeLog 和 timeWheel 恢复后，通过 timeLog 最后一个元素保留的消息在 commitLog 中的偏移量与 consumeQueue 中的元素比较
确定 consumeOffset.

## commitReadTimeMs

前面说整个延时消息的处理流程分为了多个阶段。只有当延时消息重新写入到 commitLog 后，才需要更新 commitReadTimeMs。那么
rmq 是如何实现在 slot 的所有消息写入到 commitLog 后，才更新的呢？

rmq 通过 CountLatch 来实现。从 timeWheel 某个 slot 消费消息时，创建一个大小为 slot 中元素个数的 CountLatch，每个
消息写入到 commitLog 后执行 countDown 操作，这样当所有消息写入喉，更新 commitReadTimeMs。

# 延时消息如果太多了，怎么半？

可以指定每个 slot 最大元素数量，超过最大值的时候，写入会失败。