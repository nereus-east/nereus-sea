## 基于redis的延迟、阻塞队列

#### 使用场景

1. 订单 **超时** 未支付，要取消或者提醒用户
2. 下单未付款，**超过12小时** 则取消订单释放库存
3. 消费者为保证高可用，收到message后先缓存，消费成功后删除缓存，消费失败则过 **一段时间** 再重试，因为如果基础设施故障，立马重试大概率也是失败的。所以需要延迟
4. 短信平台设定了 **凌晨0点** 准时发送营销短信

#### 开发背景以及Q&A
1. 分布式，J.U.C DelayQueue虽好，可不是分布式的
2. 避免轮询，减少不必要的开销
3. 为什么不用Rabbit MQ？不纯粹...就想要一个纯粹的延迟队列
4. 为什么用Redis？性能好，且有天然的BLPOP这样的天然阻塞命令
5. 为什么用BRPOPLPUSH？为了保证消息不丢失，高可用

#### 设计思路
1. **延迟**
    * 通过ZSET中的Score实现，Score设定为期望执行时间，获取到期的任务则很简单，只要获取Score>=当前时间即可
2. **阻塞**
    * 通过List中的BRPOPLPUSH实现，做到有job立即响应
3. **消息流转**
    * 结合ZSET + List，同时job元数据存放在hash中，固整个流程如图
![消息流转图例](https://github.com/nereus-east/nereus-sea/blob/master/redis-blocking-delay-queue/%E5%BB%B6%E8%BF%9F%E9%98%9F%E5%88%97Redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%9B%BE.jpg?raw=true)
4. **避免轮询**
    * 安排一个线程，单独处理搬运（将job从Waiting搬至Ready），每次搬运已经到期的job
    * 通过wait/notify实现唤醒，服务启动时搬运一次，并peek搬运后Waiting中的队首job，计算wait的时间（即计算下一次搬运的时间）。
    * 设定类型为AtomicLong的变量nextTransferTime,在put/add job的时候，将job的时间与nextTransferTime比较，如果job的期望时间早于已经记录的nextTransferTime，则通知搬运线程重新wait。
5. **搬运兜底**
    * 上面说到按需搬运，但是如果最近搬运的线程所在的实例发生了故障怎么办？
    * 这里我们采用兜底机制，既另起一根线程，每隔1min或者指定的时间，去唤醒搬运线程，这样可以保证最多延迟误差在可控范围之内。
6. **如何消费**
    * 通过BRPOPLPUSH命令可以实现阻塞，但是每次仅能获取一个job
    * 这里通过两根线程协同的操作方式，如果获取到一个job，则启用批量获取job的线程，直到批量获取的数量小于预设值，则重新启用BRPOPLPUSH线程。
7. **简化设计** 通过 **双线程消费**
    * 获取到的job，再批量获取job的Metadata，然后放入内部 BlockingQueue中，用户线程通过BlockingQueue实现阻塞
8. **高可用、重试+超时**
    * 由于job在消费的同时，会记录在Back中，所以这里要求消费成功后调用complate api，如果消费失败调用retry api。那么问题来了，消费失败了，但是服务挂了也没调用retry怎么办？
    * 这里再起一根线程，每隔指定的时间去back queue中检查 **队尾** 的job（即right），这里有一个设计技巧，因为back 中的job是BRPOPLPUSH命令入队的，所以设计中可以保证 back 中job的顺序，并且同一个队列中，job都是类似的，所以超时时间也是一样。
    * 故每次仅需要检查 back中的 right 数据（通过LRANGE命令），如果队尾超时，则下一次多拿出一些right job判定，若还有生命值则放入ZSET同时更新Metadata（使用lua脚本模拟compareAndUpdate，更新成功才放入ZSET，避免重复消费）。否则wait指定间隔。因为是兜底重试，所以频率无需太高。
9. **并发安全**
    * 上述操作皆通过lua脚本操作，保证操作的原子性
10. **优雅停机**
    * a.停止双线程消费 2、停止搬运 3、停止兜底 4、停止retry
    * b.停止内部线程池
    * c.等待内部队列消费完成，等待指定时间&&内部队列为空

#### 简单图示
![数据流转](https://github.com/nereus-east/nereus-sea/blob/master/redis-blocking-delay-queue/%E5%BB%B6%E8%BF%9F%E9%98%9F%E5%88%97%EF%BC%8C%E6%95%B0%E6%8D%AE%E6%B5%81%E8%BD%AC.jpg?raw=true)

#### 设计优点
1. 性能好，性能基本上以用户的消费速率决定
2. 开销不大，避免了轮询
3. 在分布式的应用下可保证高可用

#### 设计不足
1. 支持多实例，但是无法做到多实例平均消费
2. Redis集群由于lua脚本的关系，实际上是伪集群（建议仅使用哨兵模式）
3. 由于redis的持久化策略，无法做到100%的消息不丢失（具体根据redis的持久化策略有关）
