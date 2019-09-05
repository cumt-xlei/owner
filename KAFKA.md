## Producer


## Consumer
**1、Kafka 0.10.0.0及更高版本的session.timeout.ms和max.poll.interval.ms之间的差异**

&emsp;&emsp;在KIP-62之前(即Kafka 0.10.0及更早版本)，只有`session.timeout.ms`参数，KKIP-62开始引入参数`max.poll.interval.ms`；KIP-62通过后台心跳线程将heartbeats与poll()的`调用解耦`，这允许比心跳间隔更长的处理时间(即两次连续轮询之间的时间)。

&emsp;&emsp;假设处理消息需要1分钟，如果耦合了心跳和轮询(即在KIP-62之前)则需要将session.timeout.ms设置为大于1分钟以防止消费者超时；但是如果消费者死亡，检测失败的消费者也需要超过1分钟。

&emsp;&emsp;KIP-62解耦轮询和心跳，允许在两次连续的民意调查中发送心跳。现在你有两个线程在运行，心跳线程和处理线程，因此KIP-62为每个线程引入了一个超时，session.timeout.ms用于心跳线程,而max.poll.interval.ms用于处理线程。

&emsp;&emsp;假设您设置了session.timeout.ms = 30000，消费者心跳线程必须在此时间到期之前向代理发送心跳；另一方面，如果处理单个消息需要1分钟，则可以将max.poll.interval.ms设置为大于1分钟，以便为处理线程提供更多时间来处理消息。

&emsp;&emsp;如果处理线程死掉,则需要max.poll.interval.ms来检测它，但是如果整个消费者死亡(并且一个垂死的处理线程很可能崩溃包括心跳线程在内的整个消费者)，则只需要session.timeout.ms来检测它。

> 这个想法是,即使处理本身需要很长时间,也可以快速检测出失败的消费者。
> `heartbeat.interval.ms`：心跳发送频率。

**2、拉取数据方式**
- assign：手动分配topic的partition，分配之后，消费者客户端不会感知到其他事件的触发。
- subscribe：订阅topic，自动分配到topic的partition，以及该groupid之前消费到的offset，当然也可以自动感知到kafka的rebalance，并获取到相关事件。

**3、拉取数据流程**
- 1）确保kafka协调者认可了此次消费，并初始化和协调者的连接。认可很多层次的含义，包括kafka集群是否正常，安全认证是否通过之类。
- 2）确保分区被分配，除了手动assgin的topic，partition和offset，自动subscribe需要从kafka协调者获取相关元数据，也是发生重平衡事件的来源。
- 3）确保已经获取拉取的offset，否则为从协调者那重新获取对应groupid的offset，如果获取失败（比如这是一个新的groupid），那么会重置offset，根据配置用最旧或者最新来代替。参考`ConsumerCoordinator`
- 4）拉取数据，通过拉取每个partition的leader，基于NIO思路拉取数据缓存在内存中；参考`Fetcher`。
- 5）提交offset，如果开启自动提交offset的功能，那么消费者会在两个情况同步提交offset。①重平衡或者和broker心跳超时；②消费者关闭时。如果是手动提交的话可以采用异步或者同步两种提交方式。

&emsp;&emsp;具体说来就是：每个consumer 都会根据 heartbeat.interval.ms 参数指定的时间周期性地向group coordinator发送 hearbeat，group coordinator会给各个consumer响应，若发生了 rebalance，各个consumer收到的响应中会包含 REBALANCE_IN_PROGRESS 标识，这样各个consumer就知道已经发生了rebalance，同时 group coordinator也知道了各个consumer的存活情况。

**4、Rebalance**
发生条件：
- 1）一个消费者组内的任意一个消费者退出和加入都会引发rebalance.
- 2）分区的增加
- 3）订阅主题的增加(正则订阅)

&emsp;&emsp;GroupCoordinator是众多brokers中的一台，而ConsumerLeader是众多consumers中的一个。每个consumer group都有属于各自的GroupCoordinator，负责接收consumers发送来的心跳信息等，并且会触发partition rebalance。

&emsp;&emsp;每个Consumer加入Consumer Group的时候，会先发送JoinGroup request到GroupCoordinator，第一个请求的consumer成为ConsumerLeader，该consumer会接收到consumers列表(alive consumer)，然后将按照PartitionAssignor的某个实现(默认是RangeAssignor)来进行 partition assignment，最后将结果发送到GroupCoordinator由它给其余Consumers发送。
> 这个过程发生在每次rebalance时

**5、消费组状态位**
- 1）Empty：组内无成员，但是位移信息还没有过期。这种状态只能响应JoinGroup请求。
- 2）PreparingRebalance：表明group正在准备进行group rebalance。此时group收到部分成员发送的JoinGroup请求，同时等待其他成员发送JoinGroup请求，直到所耦成员都成功加入组或超时。
- 3）AwaitingSyc:表明所有成员都已经加入组并等待leader consumer发送分区分配方案。
- 4）Stable:表明group开始正常消费，可以响应客户端发送的任何请求
- 5）Dead:表明group已经彻底废弃，group内没有任何active成员且group的所有元数据都已被删除，这种状态响应各种请求都是一个response： UNKNOWN_MEMBER_ID
> 第一次poll数据时，查询kafka中保持的消费组的offset状态时，每个partition的committed offset初始为空；如果在接下来的时间消费者退出组，组内无存活的消费者；则在rebalance结束后，①如果已经成功提交了消费点位，则消费组进入Empty状态；②如果未成功提交，则消费组进入Dead状态。

**6、消费组说明**

https://www.cnblogs.com/heidsoft/p/7697974.html

https://blog.csdn.net/qwe6112071/article/details/86680900#_319

> 消费者要安全退出消费组，最后需执行close方法

> 消费组提交的offset信息保留时间可设置，可调节offset信息

**7、提交Offset**

https://blog.csdn.net/weixin_41227335/article/details/86522041

## kafka consumer 配置详解和提交方式
https://blog.csdn.net/u012129558/article/details/80076327
