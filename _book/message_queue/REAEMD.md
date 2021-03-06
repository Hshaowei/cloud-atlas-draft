[消息队列](https://zh.wikipedia.org/wiki/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97)是一种进程间通信或同一进程的不同线程间的通信方式。消息队列提供了异步的通信协议，消息的发送者和接收者不需要同时与消息队列交互。消息会保存在队列中，直到接收者取回它。

消息队列常常保存在链表结构中。拥有权限的进程可以向消息队列中写入或读取消息。

消息队列有很多开源的实现，包括JBoss Messaging、JORAM、Apache ActiveMQ、Sun Open Message Queue、RabbitMQ[3]、IBM MQ、Apache Qpid和HTTPSQS。

* 消息队列优点
  * 消息队列本身是异步的，它允许接收者在消息发送很长时间后再取回消息。一个进程通知另一个进程发生了一个事件，但不需要等待回应。
  * 异步处理可以减少请求响应时间和应用间解耦
  * 和信号相比，消息队列能够传递更多的信息。
  * 与管道相比，消息队列提供了有格式的数据，这可以减少开发人员的工作量。但消息队列仍然有大小限制。
  * 可以作为不同线程或进程间的缓冲，并且可以通过监控队列中消息数量来了解收发线程或进程的效率问题。
* 消息队列缺点
  * 接收者必须轮询消息队列，才能收到最近的消息。

# 消息队列的使用场景

消息队列的主要特点是异步处理，主要目的是减少请求响应时间和解耦。所以主要的使用场景就是将比较耗时而且不需要即时（同步）返回结果的操作作为消息放入消息队列。同时由于使用了消息队列，只要保证消息格式不变，消息的发送方和接收方并不需要彼此联系，也不需要受对方的影响，即解耦和。

> 软件的正常功能开发中，并不需要去刻意的寻找消息队列的使用场景，而是当出现性能瓶颈时，去查看业务逻辑是否存在可以异步处理的耗时操作，如果存在的话便可以引入消息队列来解决。否则盲目的使用消息队列可能会增加维护和开发的成本却无法得到可观的性能提升，那就得不偿失了。

当你的体系结构中有以下需求时可以考虑采用消息队列：

* 解耦

解耦是消息队列要解决的最本质问题。所谓解耦，简单点讲就是一个事务，只关心核心的流程。而需要依赖其他系统但不那么重要的事情，有通知即可，无需等待结果。

系统不是强耦合，消息接受者可以随意增加，而不需要修改消息发送者的代码。消息发送者的成功不依赖消息接受者（比如有些银行接口不稳定，但调用方并不需要依赖这些接口）。

不强依赖于非本系统的核心流程，对于非核心流程，可以放到消息队列中让消息消费者去按需消费，而不影响核心主流程。

> 虽然API可以实现不同系统之间的信息沟通，但是存在一个依赖性和扩展性问题。不管是主动推送调用其他系统的API（随着接入系统数量增加，会导致中心不堪重负）还是对外提供查询API（同样带来多系统轮询的沉重负担且浪费资源，而且无法实时），都存在规模限制。
>
> 消息队列实现了只要将消息放入队列，主系统就完全不需要关心后续业务消费消息的情况，达到了极高的响应性能。这样可以实现非常专注的精简系统。

* 最终一致性

最终一致性指的是两个系统的状态保持一致，要么都成功，要么都失败。

> 以一个银行的转账过程来理解最终一致性，转账的需求很简单，如果A系统扣钱成功，则B系统加钱一定成功。反之则一起回滚，像什么都没发生一样。

本地事务维护业务变化和通知消息，一起落地（失败则一起回滚），然后RPC到达broker，在broker成功落地后，RPC返回成功，本地消息可以删除。否则本地消息一直靠定时任务轮询不断重发，这样就保证了消息可靠落地broker。

broker往consumer发送消息的过程类似，一直发送消息，直到consumer发送消费成功确认。通过两次消息落地加补偿，下游是一定可以收到消息的。然后依赖状态机版本号等方式做判重，更新自己的业务，就实现了最终一致性。

> `注意`：**最终一致性不是消息队列的必备特性**，但确实可以依靠消息队列来做最终一致性的事情。另外，所有不保证100%不丢消息的消息队列，理论上无法实现最终一致性。

* 广播

消息队列的基本功能之一是进行广播。只需要关心消息是否送达了队列，至于谁希望订阅，是下游的事情，无疑极大地减少了开发和联调的工作量。

* 错峰与流控

如果没有消息队列，两个系统之间通过协商、滑动窗口等复杂的方案也可以实现流控。但系统复杂性指数级增长，势必在上游或者下游做存储，并且要处理定时、拥塞等一系列问题。而且每当有处理能力有差距的时候，都需要单独开发一套逻辑来维护这套逻辑。所以，利用中间系统转储两个系统的通信内容，并在下游系统有能力处理这些消息的时候，再处理这些消息，是一套相对较通用的方式。

> `注意`：消息队列不是万能的。**对于需要强事务保证而且延迟敏感的，RPC是优于消息队列的。**

* 日志处理

将消息队列用在日志处理中，比如Kafka的应用，解决大量日志传输的问题。

* 消息通讯

消息队列一般都内置了高效的通信机制，因此也可以用于单纯的消息通讯，比如实现点对点消息队列或者聊天室等。

# 消息队列的设计

一般来讲，设计消息队列的整体思路是先build一个整体的数据流,例如producer发送给broker,broker发送给consumer,consumer回复消费确认，broker删除/备份消息等。

利用RPC将数据流串起来。然后考虑RPC的高可用性，尽量做到无状态，方便水平扩展。

之后考虑如何承载消息堆积，然后在合适的时机投递消息，而处理堆积的最佳方式，就是存储，存储的选型需要综合考虑性能/可靠性和开发维护成本等诸多因素。

为了实现广播功能，我们必须要维护消费关系，可以利用zk/config server等保存消费关系。

# 参考

* [消息队列](https://zh.wikipedia.org/wiki/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97)
* [消息队列的使用场景是怎样的？](https://www.zhihu.com/question/34243607)
* [消息队列设计精要](https://tech.meituan.com/mq-design.html)