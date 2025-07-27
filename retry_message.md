rocketmq 天然支持消息的重试，当消费消息失败时，broker 根据消息的重试次数，发送一条延时消息，<br>
延时消息到期后，发送到 consumer group 对应的重试队列中，consumer 可以直接读取重试队列中的消息，进而实现消息的重试。

