# 三大存储文件

## commitLog

### 刷盘

刷盘指的是如何将 commitLog 内容持久化到磁盘。刷盘分为两种方式：同步刷盘、异步刷盘。

rmq 的刷盘操作也是通过后台线程实现的，整个模型还是经典的生产者-队列-消费者模型。

生产者将刷盘请求提交到队列中，对于同步刷盘，提交队列后，会激活刷盘线程。而对于异步刷盘，
刷盘线程定时处理队列中的刷盘请求。

#### 同步刷盘

同步刷盘指的是：在将消息写入到 mappedFile 后，同步等待后台刷盘线程将消息持久化到磁盘。

如果刷盘操作超时了，怎么通知写入线程呢？刷盘线程本身在做持久化工作，没法监控刷盘操作是否超时。为此，
rmq 又引入了一个刷盘是否超时的监控线程，会依次便利刷盘请求队列中的请求，根据当前时间判断刷盘请求处理
是否超时，如果超时，返回错误。

需要注意的是：即使是同步刷盘也无法保证数据不丢失。因为单机可能异常退出，无法重启，磁盘数据也无法查看。
要保证数据不丢失还是得依赖多个节点(主备)来完成。

#### 异步刷盘

后台刷盘线程定时持久化 commitLog.

## consumeQueue

在一个 broker 中，所有的消息都是写入到 commitLog 中。这样的好处是：写消息很方便。但是读就不好读了，不同 topic + queue 的消息混在一起。

怎么办呢？这就是 consumeQueue 存在的意义。当消息写入到 commitLog 会有后台任务将消息从 commitLog 按照 topic + queue 分发到 consumeQueue
中。当然，consumeQueue 中存的只是 commitLog 中消息的索引信息，具体来说就是 offset + size。这样对于 consumer 而言，只需要从 consumeQueue
读取消息即可。

## indexFile

indexFile 文件的目的是提供根据 key 查找消息的功能。消息可能很多，所以需要多个 indexFile 文件，每个 indexFile
索引连续的一部分消息。

### 文件内容

indexFile 文件内容是个哈希表。文件分三部分：header、slots、elems。

header 中存储一些元信息，包括被索引的第一条消息写入 commitLog 的时间以及在 commitLog 的偏移量，也包含最后一条消息的这两个信息。

indexFile 的 slots 数量是固定的，这样实现简单，也不用考虑 rehash，这样 slots 足够大即可。

elems 中存储真正的索引数据。

文件写入流程也比较简单，根据消息 key 确定 slot，然后通过 slot 找到该 slot 第一条消息，然后通过头插法更改新消息的 ptr，然后将新消息
追加到 elems 即可。

### 持久化

在不断的 dispatch commitLog 中的消息时，消息会写入到 indexFile 中。每个 indexFile 大小是固定的，当某个 indexFile 写满后，就会
将文件持久化，并将持久化信息存储到 checkPoint 文件中，同时创建下一个新的 indexFile.

## 消息 dispatch 机制

这里的消息 dispatch 机制指的是将消息从 commitLog 分发到 consumeQueue 和 indexFile。

我们看看并发 dispatch 的实现。

整体流程上，还是通过若干个定时任务之间协作完成消息的 dispatch.

- ConcurrentReputMessageService 

    该任务从 commitLog 获取数据构造 BatchDispatchRequest 并发送到队列中。

- MainBatchDispatchRequestService 

    该任务从上面的队列中获取 BatchDispatchRequest，并提交给线程池，每个线程负责将一个 BatchDispatchRequest 
    转化为 List<DispatchRequest>，并将结果发送到一个**有序队列**中。这一步可能比较重，所以通过线程池的方式实现。

- DispatchService 

    该任务从有序队列中获取 List<DispatchRequest> 然后分发到 consumeQueue 和 indexFile

这里特别需要注意的是：从 commitLog 分发消息到 consumeQueue 和 indexFile 一定是按照消息有序分发的。但是上面的流程
中使用了线程池，如何保证 DispatchRequest 的有序呢？rmq 的解决思路如下：

- 首先每一个 BatchDispatchRequest 内的消息是有序的，每个 BatchDispatchRequest 会分配一个严格递增的 id，表示
BatchDispatchRequest 的顺序。
- 线程池中的线程在将 BatchDispatchRequest 转化为 List<DispatchRequest> 后，会严格按照 id，写入到**有序队列**对应的位置。
- DispatchService 从 **有序队列**中获取消息时，严格按照 id 顺序获取，当出现空洞(中间一部分 List<DispatchRequest> 还未生成，
但是后面的 List<DispatchRequest> 已经生成)时，停止获取数据。这样就保证了一定是有序的。

我们最后看看 rmq 中这个**有序队列**的实现。

```java
    class DispatchRequestOrderlyQueue {

        DispatchRequest[][] buffer;

        long ptr = 0; // 第一条消息位置

        AtomicLong maxPtr = new AtomicLong(); // 最后一条消息位置

        public DispatchRequestOrderlyQueue(int bufferNum) {
            this.buffer = new DispatchRequest[bufferNum][];
        }

        public void put(long index, DispatchRequest[] dispatchRequests) {
            // 队列满等待
            while (ptr + this.buffer.length <= index) {
                synchronized (this) {
                    try {
                        this.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }
            // 生产数据
            int mod = (int) (index % this.buffer.length);
            this.buffer[mod] = dispatchRequests;
            maxPtr.incrementAndGet();
        }

        public DispatchRequest[] get(List<DispatchRequest[]> dispatchRequestsList) {
            synchronized (this) {
                // 从 ptr 开始严格按照顺序获取
                for (int i = 0; i < this.buffer.length; i++) {
                    int mod = (int) (ptr % this.buffer.length);
                    DispatchRequest[] ret = this.buffer[mod];
                    // 遇到空的，直接返回，即使后面还有数据
                    if (ret == null) {
                        this.notifyAll();
                        return null;
                    }
                    dispatchRequestsList.add(ret);
                    this.buffer[mod] = null;
                    ptr++;
                }
            }
            return null;
        }

        public synchronized boolean isEmpty() {
            return maxPtr.get() == ptr;
        }

    }
```

# mappedFile

在 rmq 中，为了实现文件的高效操作，都是通过 mappedFile 的方式进行文件读写的。但是这种方式也有问题，如果发生缺页中断，还是需要从磁盘中加载页
到内存中，性能还是会有抖动。

rmq 通过下面两种方式来保证页在内存中。

## 内存预热

当创建了一个新的 mappedFile 时，我们知道磁盘页并没有加载到内存中。等到后面读写文件的时候，就会发生缺页中断。 rmq 在每次创建新的 mappedFile
时，会进行 **内存预热**，即将磁盘中的页加载到内存中，具体实现方式为：每一个磁盘页(4k) 读取一个字节，这样就会将磁盘页加载到内存中。

同时为了性能考虑，rmq 都是通过提前创建 mappedFile 的方式避免预热操作影响正常的逻辑。

## 内存锁定

即使将磁盘页加载到内存中，当内存空间紧张的时候，还是有可能被换出的，这样下次访问还是会出现缺页中断问题。

rmq 通过 mlock() 系统调用，会将这些页锁住，这些页就不会被换出。

# 日志清理

默认情况下，一个 commitLog 文件大小为 1GB。随着消息的不断写入，占用的磁盘空间会越来越大。这就需要磁盘清理机制。

rmq 的思路还是启动一个后台定时任务，不断去检查是否达到删除 commitLog 文件的要求：
- 如果磁盘空间占比大于 > 85% 则立即删除
- 如果文件最后更改时间超过 72h(默认值)，且时间则凌晨 3-5 点(这很细节，为了避免删除任务对正常业务的影响，选择业务低峰时段进行)

删除操作就是遍历一个个的 mappedFile，关闭并删除对应的文件。为了将删除操作对业务影响降至最小，rmq 删除思路如下：
- 每一轮最多删除 10 个 mappedFile
- 删除 mappedFile 时有特定的间隔，避免连续删除对正常业务的影响

此外，对于一个 mappedFile 而言，在我们要删除的时候，可能还有其他使用者在使用。那怎么平衡无法删除以及删除后影响其他逻辑之间的矛盾呢？

首先，怎么知道某个 mappedFile 是不是还有其他使用者呢？这就涉及到 **引用资源计数器：ReferenceResource**。简而言之，就是为目标对象
维护一个计数器，使用的时候加1，释放的时候减1. 计数器还有个 Avaliable 的状态，当执行 Shutdown 操作后，无法对计数器执行加1操作。计数器
初始化的时候值为1，执行 Shutdown 的时候，会减去 1，如果引用计数为 0，表示没有其他执行流引用该对象，便可以释放该对象(cleanUp)，反之，
如果计数器不为 0，说明还有其他执行流引用该对象，怎么办呢？

rmq 的思路是定时重试，但不会无限重试。默认如果在执行 Shutdown 操作后 120s mappedFile 的引用计数还是不为0，则强制释放对象。

引用计数器的这种思路很多地方都有使用，比如 Python 的 gc 就是使用的这种思路。在涉及到对象生命周期管理的时候，我们就可以考虑这种方式。

# dledger

传统的 broker 的主从模式，实际上与 mysql 的主从模式类似，写请求到主节点，然后通过日志复制到从节点。

但这种方式有个严重的问题：如果主节点挂了，没法自动化选择新的节点作为主节点。

为此 rocketmq 引入了 dledger 模式，核心思路是通过 raft 协议实现分布式共识，能够实现：leader 故障时的重新选举。

关于 raft 协议，核心内容包括以下部分：

- leader 探活以及选举、投票机制
- 日志复制以及冲突检测

在 dledger 模式下，消息写入到 `DLedgerCommitLog`，当消息在大多数的 broker(raft group) 中成功复制后，更新 DLedgerCommitLog
的 `committedPos` 成员。**只有 committedPos 之前的消息，才能被 dispatch 到 consumeQueue 和 indexFile.**

# 日志恢复

commitLog、consumeQueue、indexFile 各自有各自的定时任务来完成持久化，在持久化的时候，同时会更新并持久化 checkpoint 信息。
后续恢复可以基于 checkpoint 进行，加速恢复。

这里其实就有一个一致性的问题：比如 commitLog 和 consumeQueue 这两个文件，因为他们的持久化不是一致的，导致最终恢复出来的数据也不是一致的(不在一个 offset)，
所以还需要调谐：如果 commitLog 恢复的 commitOffset 小于 consumeQueue 恢复的 maxPhyOffset，则需要删除 consumeQueue 中多余的文件。

恢复逻辑的核心思想如下：

- 首先会记录各个需要持久化对象的 checkpoint，这样恢复就不用从头开始，只需要从 checkpoint 开始恢复即可，加快恢复速度。
- 其次，多个持久化对象之间是不一致的，需要从一个对象出发来构建其他对象。在 rmq 中，所有的恢复都是从 commitLog 开始。
commitLog 就是一生二、二生三、三生万物中的一。

