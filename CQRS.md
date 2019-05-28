

## 什么是CQRS, Event Sourcing?

- Event Sourcing 是把用户直接发起的业务操作转成事件序列，然后异步的处理事件序列，好处是可以支持很高的并发，故障时可以很方便的重放，并且很自然的有了审计功能，这个做法跟关系数据库的事务日志很像。现在流行的 serverless、函数式计算跟这个做法也是暗合，而在前端的 React、ReactiveX 里也符合这个模式。
- CQRS (Command Query Responsibility Segregation) 指读取和写入两条路径的领域模型不同，做法也不同，因为读取和写入往往要求很不一样，比如读取要查一堆东西，格式化成各种格式，一般是等着返回，而写入则往往改少量数据，可以异步执行（采用 Event Sourcing 模式），返回的结果也很简单。



#### CQRS架构

CQRS最早来自于Betrand Meyer（Eiffel语言之父，[开-闭原则](http://msdn.microsoft.com/en-us/magazine/cc546578.aspx)OCP提出者）在 [Object-Oriented Software Construction](http://www.amazon.com/gp/product/0136291554) 这本书中提到的一种 [命令查询分离](http://martinfowler.com/bliki/CommandQuerySeparation.html) ([Command Query Separation](http://en.wikipedia.org/wiki/Command-query_separation),CQS) 的概念。其基本思想在于，任何一个对象的方法可以分为两大类：

- 命令(Command):不返回任何结果(void)，但会改变对象的状态。
- 查询(Query):返回结果，但是不会改变对象的状态，对系统没有副作用。

CQRS是对CQS模式的进一步改进成的一种简单模式。 它由Greg Young在[CQRS, Task Based UIs, Event Sourcing agh!](http://codebetter.com/gregyoung/2010/02/16/cqrs-task-based-uis-event-sourcing-agh/) 这篇文章中提出。“CQRS只是简单的将之前只需要创建一个对象拆分成了两个对象，这种分离是基于方法是执行命令还是执行查询这一原则来定的(这个和CQS的定义一致)”。

CRQS架构本身是一个读写分离的思想. C端负责数据存储, Q端负责数据查询. Q端的数据通过C端产生的Event来同步.

![](https://images2015.cnblogs.com/blog/27612/201603/27612-20160303112537752-153692912.png)

简单可理解为, UI层只管发送各种命令(`Command`)，触发事件(Event)，然后由EventHandler去异步处理，最终写入master DB，对于数据库的查询，则查询slave DB（注：这里的master db, slave db只是一个逻辑上的区分，可以是真正的主-从库，也可以都是一个库）。 这样的架构，很容易实现读写分离，也易于大型项目的扩展。



### CQRS与Event Sourcing的关系

CQRS中，查询方面，直接通过方法查询数据库，然后通过DTO将数据返回。在操作(Command)方面，是通过发送Command实现，由CommandBus处理特定的Command，然后由Command将特定的Event发布到EventBus上，然后EventBus使用特定的Handler来处理事件，执行一些诸如，修改，删除，更新等操作。这里，所有与Command相关的操作都通过Event实现。这样我们可以通过记录Event来记录系统的运行历史记录，并且能够方便的回滚到某一历史状态。[Event Sourcing](http://msdn.microsoft.com/en-us/library/dn589792.aspx)就是用来进行存储和管理事件的。



#### CQRS架构如何实现高性能?

要想高性能，需要尽量：1. 避开网络开销 (IO); 2. 避开海量数据; 3. 避开资源争夺; 

##### 避免资源竞争

什么是**资源争夺**？狭义理解就是多个线程同时修改同一个数据。就像阿里秒杀活动一样，秒杀开抢时，很多人同时抢一个商品，导致商品的库存会被并发更新减库存，这就是一个资源争夺的例子。

![](https://images0.cnblogs.com/blog/13665/201410/272142021446397.png)

**解决思路**: 对同一资源修改串行执行, 不同资源并发执行. 例如对MySQL Server端来说，内部总是对同一行的修改请求都排队处理；这样就能确保不会有并发产生，从而不会导致线程浪费堆积，导致数据库性能下降。[实现案例分析]()

什么是**Group Commit**技术? Group Commit就是对多个请求(特别是write) 合并为一次操作进行处理。例如: 秒杀时，大家都在购买这个商品，A买2件，B买3件，C买1件；其实我们可以把A,B,C的这三个请求合并为一次减库存操作，就是一次性减6件。这样，对于A,B,C的这三个请求，在InnoDB层我们只需要做一次减库存操作即可。

#### CQRS如何避免资源竞争?

试想事务A要修改a1,a2,a3三个聚合根；事务B要修改a2,a3,a4；事务C要修改a3,a4,a5三个聚合根。那这样，我们很容易理解，这三个事务只能串行执行，因为它们要修改相同的资源。比如事务A和事务B都要修改a2,a3这两个聚合根，那同一时刻，只能由一个事务能被执行。

##### 让一个Command总是只修改一个聚合根

缩小事务的范围, 确保一个事务一次只涉及一条记录的修改。只有单个聚合根的修改才是事务的，让聚合根成为数据强一致性的最小单位。

一个请求就是会涉及多个聚合根的修改的，这种情况怎么办呢？ **[Saga](#什么是Saga?)**是一种基于事件驱动的思想来实现业务流程的技术，通过Saga，我们可以用**最终一致性**的方式最终实现对多个聚合根的修改。对于一次涉及多个聚合根修改的业务场景，一般总是可以设计为一个业务流程，也就是可以定义出要先做什么后做什么。例如: 银行转账的场景为例子，如果是按照传统事务的做法，那可能是先开启一个事务，然后让A账号扣减余额，再让B账号加上余额，最后提交事务；如果A账号余额不足，则直接抛出异常，同理B账号如果加上余额也遇到异常，那也抛出异常即可，事务会保证原子性以及自动回滚。也就是说，数据一致性已经由DB帮我们做掉了。Saga的做法, 聚合根有：两个银行账号聚合根，一个交易（Transaction）聚合根，它用于负责存储流程的当前状态，它还会维护流程状态变更时的规则约束；然后当然还有一个流程管理器。转账开始时，我们会先创建一个Transaction聚合根，然后它产生一个TransactionStarted的事件，然后流程管理器响应事件，然后发送一个`Command`让A账号聚合根做减余额的操作；A账号操作完成后，产生领域事件；然后流程管理器响应事件，然后发送一个`Command`通知Transaction聚合根确认A账号的操作；确认完成后也会产生事件，然后流程管理器再响应，然后发送一个Command通知B账号做加上余额的操作；

##### 同一个聚合根的Command串行处理

在集群的情况下呢，也就是你不止有一台服务器在处理Command，而是有十台，那要怎么办呢？因为同一时刻，完全有可能有两个不同的Command在修改同一个聚合根。**方案**是通过`shard技术`, 对要修改聚合根的Command根据聚合根的ID进行路由，根据聚合根的ID的`hashcode`，然后和当前处理Command的服务器数目取模，就能确定当前Command要被路由到哪个服务器上处理了服务器数目不变的情况下，针对同一个聚合根实例修改的所有Command都是被路由到同一台服务器处理。然后加上我们前面在单个服务器里面内部做的排队设计，就能最终保证，对同一个聚合根的修改，同一时刻只有一个线程在进行。

> 在进行ID路由前可以采用Topic方式, 对Command按照业务分组.

### 如何避免网络开销?

**In-Memory**模式也是一种减少网络IO的一种设计，通过让所有**生命周期**还没结束的聚合根一直常驻在内存，所有持久化都异步进行。也就是说，内存的数据才是最新的，db的数据是异步持久化的，也就是某个时刻，内存中有些数据可能还没有被持久化到db。[LMAX](https://martinfowler.com/articles/lmax.html)架构就是In-Memory模式的成功案例, LMax内部采用[Distruptor](https://tech.meituan.com/2016/11/18/disruptor.html)无锁环状队列, 实现单机单核600w/s订单.

#### 避开海量数据?

整个CQRS架构中, command, Event会对底层存储做大量insert操作, 最后一定会局限于单机I / O 瓶颈, 通过[sharding]()避开大数据突破I / O瓶颈.

### 如何避免重复消费?

#### 解决Command的幂等处理

幂等处理是为了保证系统的最终一致性. 我们举个转账的例子，A账户向B账户转账. 假如A账号扣减余额的命令被重复执行了，那会导致A账号扣了两次钱。那最后就数据无法一致了。平时判重有两种方法?

1. db对某一列建唯一索引, 全局保证某一列的数据不重复
2. 通过程序保证, 比如插入前先通过select查询判断是否存在，如果不存在，则insert，否则就认为重复；

#### Event持久化的幂等处理

新增或修改聚合根的Command，总是会产生相应的领域事件（Domain Event）。聚合根有生命周期，在它的生命周期里，会经历各种事件，而事件的发生总有确定的时间顺序。所以，为了明确哪个事件先发生，哪个事件后发生，我们可以对每个事件设置一个版本号，即version。要实现事件持久化的幂等处理，也很好做了，就是db中的事件表，对`聚合根ID+聚合根当前的version建`唯一索引。

## 什么是Saga?

Saga关注的是多个系统（微服务）之间数据一致性：**如何在避免使用分布式事务情况下达到跨系统数据一致性**.

Saga 的设计有三个重点：

- 每个系统的领域对象有状态，比如 PENDING、DONE、CANCELLED，跨系统的这些对象的所有状态合起来形成一个状态机，通过状态转换达到“**最终一致性**”。比如一个订单要操作三个系统里的数据，那么三个系统里依次会有一条 PENDING 记录，最后一个系统成功后，它的记录标记为 DONE，然后反向通知前面的系统也标记成 DONE，如果失败，也要反向通知。
- 跨系统之间通过消息来推进状态切换
- 每个系统在创建 PENDING状态的记录后，立即返回记录 ID，在 UI 一侧通过轮训后端服务或者被动通知获得最终结果。



## 相关开源项目

[eventuate-tram-core](https://github.com/eventuate-tram/eventuate-tram-core) : Transactional messaging for microservices 

[Axon](https://axoniq.io/) : Framework and server for event-driven microservices

[Enode](https://github.com/tangxuehua/enode) : a framework aims to help us developing ddd, cqrs, eda, and event sourcing style applications.

## 相关资料

[Event Sourcing, CQRS, Saga 模式探讨](https://zhuanlan.zhihu.com/p/37098711)

[CQRS架构简介](https://www.cnblogs.com/netfocus/p/4055346.html) 

[DDD CQRS架构和传统架构的优缺点比较](https://www.cnblogs.com/netfocus/p/5184182.html)

[The LMAX Architecture](https://martinfowler.com/articles/lmax.html)

[浅谈命令查询职责分离(CQRS)模式](https://www.cnblogs.com/yangecnu/p/Introduction-CQRS.html)