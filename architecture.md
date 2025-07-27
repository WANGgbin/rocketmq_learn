描述 rocketmq 整体架构。

# 架构

- nameserver

    保存集群元信息，集群之间通过 raft 协议达成共识。同时负责 broker 的选举任务，当某个 master broker 下线后，从 Sync State Set(3S) 中选择一个 broker 作为新的 master。


- broker cluster

    每个 broker 通过 heartbeat 上报自己的信息，并获取集群元信息


- producer/consumer

    producer/consumer 通过 nameserver 获取集群元信息，进而知道发送消息到哪儿或者从哪儿读取消息。


# broker 日志存储

发往 broker 的所有消息，即使不同的 topic 或者 queue，都会写入到同一个 commit log 中，会有后台线程不断分发 commit log 中的消息到不同的 consume queue 中。<br>
消费者通过读取 consume queue 来读取消息。

# 高可用

rocketmq broker 集群中有若干个 master，这些 master 共同承担 producer 的写请求，对于一个 topic 而言，其 queue 可能分布在多个 master broker 上。<br>
每个 master 又有若干个 slave，slave 负责从 master 同步消息。当 master 宕机后，controller 会从 slave 中选择一个作为 master。