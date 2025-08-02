本文描述一种 rocketmq 实现的一种线程协同/通信机制。

rocketmq 中有大量的生产者、消费者模型。通过这种异步模型由多个线程来完成某个功能，从而提高系统整体性能。同时这种模型也比较灵活，既能
实现同步操作，也能实现同步操作：
- 异步操作：

    将消息扔到队列中，同时生成一个 Future 对象，生产者逻辑结束。再之后消费者处理完毕消息后，更新 Future，调用 callback 逻辑。

- 同步操作：

    同样生产一个 Future 对象，但在 Future 对象同步阻塞，直到消费者处理完毕消息。


这里有个问题就是消费者如何知道有数据需要处理呢？有两种思路：

- 定时执行。消费者定时判断有没有数据需要处理。

  - 优点
    
    实现比较简单。

  - 缺点
    
    不够及时。即使有新的消息达到，也需要等到定时时间达到后处理消息。

- 线程通信机制，由生产者告诉消费者有消息达到

  - 优点

    处理消息及时

  - 缺点

    实现比较复杂。


在 rmq 中，很多地方都用到了这种通信机制：

- 同步刷盘操作。当有新的消息达到时，需要唤醒刷盘线程，完成刷盘操作。
- dledger 模型中，当 raft group 中超过一半节点同步某个消息后，leader 需要更新 commitIndex，需要及时唤醒 quorumAckChecker

这些场景的共同点就是需要消费者及时处理消息。如果不是及时场景，用定时轮询模型即可。

如何实现这种通信机制呢？

- 生产端

    站在生产端考虑，需要在生产完数据后，唤醒消费者。

- 消费者

    有数据就继续处理。没有数据的时候，进入睡眠前，需要判断是否有唤醒信息，如果有继续处理。如果没有，则阻塞。同时阻塞期间，可以被生产者唤醒。

其实，在 go 中很容易实现上述机制，通过一个大小为 1 的 chan 即可实现。生产者非阻塞发送信号。消费者在 chan 阻塞，接收到信号后唤醒。

我们来看看 rocketmq 如何实现的。

```java
    public void wakeup() {
        // 已经发送过信号，不再发送
        // 否则发送信号，唤醒消费者
        if (hasNotified.compareAndSet(false, true)) {
            waitPoint.countDown(); // notify
        }
    }

    protected void waitForRunning(long interval) {
        // 睡眠前，先看期间有没有信号，有的话，直接返回。
        if (hasNotified.compareAndSet(true, false)) {
            this.onWaitEnd();
            return;
        }

        try {
            // 否则阻塞，直到被唤醒
            waitPoint.await(interval, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            log.error("Interrupted", e);
        } finally {
            // 唤醒后重置唤醒标记
            hasNotified.set(false);
        }
    }
```

    