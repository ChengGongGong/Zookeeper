# Zookeeper简介
    一个开源的分布式协调服务，为我们提供了高可用、高性能、稳定的分布式数据一致性解决方案，通常被用于实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、
    集群管理、Master 选举、分布式锁和分布式队列等功能；
    
    此外，ZooKeeper 将数据保存在内存中，在“读”多于“写”的应用程序中尤其地高性能，因为“写”会导致所有的服务器间同步状态。
    
    特点：
    1. 顺序一致性，从同一客户端发起的事务请求，最终将会严格地按照顺序被应用到 ZooKeeper 中去；
    
    2. 原子性：要么整个集群中所有的机器都成功应用了某一个事务，要么都没有应用；
    
    3. 单一系统映像：无论客户端连到哪一个zk服务器上，其看到的服务器数据模型都是一致的；
    
    4. 可靠性：一旦一次更改请求被应用，更改的结果会被持久化，直到下一次更改覆盖； 

# Zookeeper的角色
    
    领导者(leader)：主节点，为客户端提供读和写服务，负责进行投票的发起和决议，更新系统状态；
    
    学习者(learner)：从节点，为客户端提供读服务，如果是写服务则转发给leader。包括：
                    跟随者(Follower),用于接收客户端请求并向客户端返回结果，在选主过程中参与投票，
                    观察者(Observer),可以接收客户端连接，将写请求转发给leader，但Observer不参加投票过程，只同步leader的状态，目的是为了扩展系统，提高读取速度；
    
    客户端(client)：一个节点，通过 ZkClient 或是其他 ZooKeeper 客户端与 ZooKeeper 集群中的一个 Server 实例维持长连接，并定时发送心跳。
    
    server之间的同步：基于Zab协议的原子广播，包括恢复模式(选主)和广播模式(同步)，当服务启动或者在领导者崩溃之后，Zab进入恢复模式，当领导者被选举出后，
    且大多数Server完成了和leader的状态同步后，恢复模式结束，状态同步保证了leader和server具有相同的系统状态。
      
    选举过程：
      1. 选举阶段，节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 leader；
      
      2. 发现阶段，followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议;
      
      3. 同步阶段，利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本。同步完成之后，准leader 才会成为真正的leader；
      
      4. 广播阶段，zk集群正式对外提供事务服务，并且leader进行消息广播，如果有新节点加入，需对新节点进行同步。
    
    事务一致性：zk采用递增的事务id号(zxid)来标识事务，所有的提议都被在被提出来的时候加上了zxid，其中zxid是一个64位的数字，高32位用于标识leader关系是否改变，低32位用于递增计数。
    
    server状态：
      looking：当前server不知道leader是谁，正在搜寻；
      leading：当前server即为选举出来的leader；
      following：leader已经选举出来，当前server与之同步；
      observing：Observer 状态，对应节点为 Observer，该节点不参与 Leader 选举。
 # Zookeeper的重要概念
 ## 数据模型(Data model)：
 ![image](https://user-images.githubusercontent.com/41152743/141439330-d948af6b-924c-4b78-a0b2-35fc9419c539.png)
 
    采用层次化的多叉树形结构，每个节点上都可以存储数据，这些数据可以是数字、字符串或者二级制序列，
    并且每个节点还可以拥有N个子节点,最上层的根节点以“/”代表；
    每个数据节点在zk中被称为znode,它是zk中数据的最小单元，且每个znode都有一个唯一的路径标识，
    每个节点的数据大小最大是1M，因为zk主要是用于协调服务的而不是存储业务数据的。
    
 ## 数据节点(znode)：包含4类
      持久节点(persistent)：一旦创建就一直存在，即使zk集群宕机，直到将其删除；
      
      持久顺序节点(persistent_sequential)节点：除了具有持久节点的特性之外，子节点的名称还具有顺序性；
      
      临时节点(ephemeral)：临时节点的生命周期与客户端会话绑定的，会话消失则节点消失，并且临时节点只能做叶子节点，不能创建子节点；
      
      临时顺序节点(ephemeral_sequential)：除了具有临时节点的特性之外，节点的名称还具有顺序性
      
      数据结构：包含两部分stat状态信息、data节点存放的数据的具体内容，其中stat状态信息如下所示：
![image](https://user-images.githubusercontent.com/41152743/141439640-8bb943ec-e351-4f67-aaf5-483b64d70026.png)

      cZxid	-> create ZXID，即该数据节点被创建时的事务 id
      ctime	-> create time，即该节点的创建时间
      mZxid	-> modified ZXID，即该节点最终一次更新时的事务 id
      mtime ->	modified time，即该节点最后一次的更新时间
      pZxid	-> 该节点的子节点列表最后一次修改时的事务 id，只有子节点列表变更才会更新 pZxid，子节点内容变更不会更新
      cversion ->	子节点版本号，当前节点的子节点每次变化时值增加 1
      dataVersion	 -> 数据节点内容版本号，节点创建时为 0，每更新一次节点内容(不管内容有无变化)该版本号的值增加 1
      aclVersion ->	节点的 ACL 版本号，表示该节点 ACL 信息变更次数
      ephemeralOwner	-> 创建该临时节点的会话的 sessionId；如果当前节点为持久节点，则 ephemeralOwner=0
      dataLength	-> 数据节点内容长度
      numChildren	-> 当前节点的子节点个数
 ## 权限控制(ACL)
    采用 ACL（Access Control Lists）策略来进行权限控制，类似于 UNIX 文件系统的权限控制,主要包括5种：
    CREATE :能创建子节点
    READ ：能获取节点数据和列出其子节点
    WRITE : 能设置/更新节点数据
    DELETE : 能删除子节点
    ADMIN : 能设置节点 ACL 的权限
## 事件监听器(Watcher)
    允许用户在指定节点上注册一些watcher，并且在一些特定事件触发的时候，服务端会将事件通知到感兴趣的客户端上去。特点：
    主动推送：当watcher被触发时，由zk主动将更新推送给客户端，而不需要客户端轮询；
    一次性：数据变化时，Watcher 只会被触发一次，如果客户端想得到后续更新的通知，必须要在 Watcher 被触发后重新注册一个 Watcher；
    可见性：更新通知先于更新结果；
    顺序性：如果多个更新触发了多个 Watcher ，那 Watcher 被触发的顺序与更新顺序一致
    
# Zab协议(消息广播和崩溃恢复)
    总结：
      1. 使用主从复制模式，所有的写操作都要由Leader主导完成，而读操作可通过任意节点完成；
      
      2. 虽然使用主从复制模式，同一时间只有一个Leader，但是Failover机制保证了集群不存在单点失败（SPOF）的问题；
      
      3. 保证了Failover过程中的数据一致性,leader commit过的数据不丢失，未commit过的消息对客户端不可见；
      
      4. 服务器收到数据后先写本地文件再进行处理，保证了数据的持久性
      
    该协议保证所有的写操作都必须通过Leader完成，Leader写入本地日志后再复制到所有的Follower节点。
    一旦leader节点无法工作，Zab协议能够自动从follower节点中重新选出一个合适的替代者，即为新的leader。
## 1. 写操作
![image](https://user-images.githubusercontent.com/41152743/141733873-742e284e-29be-4cd5-80a2-4f8b6f1fcdc6.png)

### 写leader
    客户端直接向leader发起写请求，leader将写请求以proposal的形式发给所有Follower，并等待ACK，得到超过一半的ACK向所有的follower和observer发送commit，将处理结果返回给客户端。
    (注：此处ack信号，leader自身默认有一个ack)
### 写follower/observer
    可以接受写请求，不能直接处理，将写请求转发给leader进行处理
## 2.读操作
    Leader/Follower/Observer都可直接处理读请求，从本地内存中读取数据并返回给客户端即可。读请求不需要服务器直接的交互，Follower/Observer越多，整体可处理的读请求量越大，也即读性能越好。
## 3. leader选举算法
    具体流程见：https://dbaplus.cn/news-141-1875-1.html
    
    可通过electionAlg配置项设置ZooKeeper用于领导选举的算法，其中：
    
    0 基于UDP的LeaderElection

    1 基于UDP的FastLeaderElection

    2 基于UDP和认证的FastLeaderElection

    3 基于TCP的FastLeaderElection
    
    每个服务器在进行领导选举时，发送的信息如下：
    logicClock:每个服务器会维护一个自增的整数，名为logicClock，它表示这是该服务器发起的第多少轮投票

    state:当前服务器的状态

    self_id:当前服务器的myid(与hostname必须一一对应)

    self_zxid：当前服务器上所保存的数据的最大zxid

    vote_id：被推举的服务器的myid

    vote_zxid：被推举的服务器上所保存的数据的最大zxid
    
# 一致性问题
    分布式系统遇到的问题：CAP定理，即由于分区容忍性(partition tolerance)的存在,需要在系统可用性(availabilty)和数据一致性(consistency)中做出权衡。
    例子：Eureka的处理方式保证了AP(可用性)、zookeeper的处理方式保证了CP（数据一致性）
## 一致性算法
  所有一致性算法的前提是安全可靠的消息通道，发出的信号不会被篡改
### 1.2PC(两阶段提交)
    两个角色：协调者与参与者
    
    第一阶段：当要执行一个分布式事务的时候，事务发起者首先向协调者发起事务请求，协调者会给所有参与者发送 prepare 请求，
              参与者收到prepare消息后，开始执行任务(但不提交)，并将Undo和Redo信息记入事务日志中，然后向协调者反馈。
              
    第二阶段：协调者根据参与者反馈的情况来决定接下来是否可以进行事务的提交操作，即提交事务或者回滚事务
    
    问题：
    单点故障问题：协调者挂了那么整个系统都处于不可用的状态了；
    
    阻塞问题：参与者收到协调者的prepare请求进行事务的处理但并不提交，会一直占用着资源不释放，如果此时协调者挂了，那么这些资源都不会再释放了；
    
    数据不一致问题：协调者提交事务时，只发送了一部分commit请求就挂掉了，没收到commit请求的不会提交事务，造成数据不一致
 
 ### 1.3PC(三阶段提交)
    两个角色：协调者与参与者
    
    1.CanCommit阶段:协调者向所有参与者发送CanCommit请求，参与者根据自身情况查看是否能执行事务，返回YES或NO响应并进入预备状态；
    
    2.PreCommit阶段：如果协调者收到的都是YES响应，则向所有参与者发送PreCommit预提交请求，参与者收到之后，进行事务的操作，并将Undo和Redo信息写入事务日志中，并向协调者反馈；
    如果协调者收到任何一个NO响应或者在一定时间内没有收到所有参与者的响应，则中断事务，向参与者发送中断请求，参与者收到后立即中断事务；
    或者参与者在一段时间内没有收到协调者的请求，它也会中断事务(超时中断)；
    
    3.DoCommit阶段：如果协调者收到的都是YES响应，则向所有参与者发送DoCommit请求，参与者收到后进行事务的提交工作，并向协调者反馈；
    反之，收到了任何一个NO响应或者在一定时间内没有收到所有参与者的响应，发送中断请求，参与者收到后通过记录的回滚日志进行事务的回滚，并向协调者反馈；
    如果参与者在一定时间内没有收到协调者的请求，它会进行事务的提交。
    
    总结：
      超时机制缓解了阻塞问题，但最终一致性没有得到根本的解决，例如在preCommit阶段，当一个参与者收到请求后其他参与者和协调者挂掉了或者出现了网络分区，一定时间后，
      收到消息的参与者会进行事务的提交，造成了数据不一致性问题。
### 3. paxos算法简介
    概念：基于消息传递且具有高度容错特性的一致性算法，是目前公认的解决分布式一致性问题最有效的算法之一，其解决的问题就是在分布式系统中如何就某个值（决议）达成一致。
    
    需要解决的问题：常见的分布式系统中，机器宕机或网络异常（包括消息的延迟、丢失、重复、乱序，还有网络分区）等情况发生时，能够快速且正确地在集群内部对某个数据的值达成一致。
    
    典型场景：在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。
    
    节点通讯模型：消息传递，前提是假设信道是安全可靠的，发出的信号不会被篡改；
    
    工程实现：ZAB协议、Raft协议
    
    三个角色：Proposer提案者、Acceptor表决者、Learner学习者
    
    1. prepare阶段：proposer将具有全局唯一性的、递增的提案编号N发送给所有的Acceptor,Acceptor接收到提案后，会将该提案编号记录在本地，
    每个Acceptor会仅会接收编号大于自己本地的提案，并将之前接受过的最大编号反馈给proposer。
    ![Image_text](https://camo.githubusercontent.com/958acf97caf9cb2405b7004dceb966a4c587f43bcc78af3bf150c932dbdf53e8/68747470733a2f2f696d672d626c6f672e6373646e696d672e636e2f696d675f636f6e766572742f32326538643531326439353436373662646630636339326432303061663865662e706e67)
    
    2.accept阶段：当一个提案被Propersoer提出后，如果收到了超过半数的Acceptor的批准(Properser本身同意)，则向所有的Acceptor发送真正的提案[N,V]，包括提案的内容和编号，
    其中V就是收到的想要值编号最大的提案的value，如果响应中不包含任何提案，则V由Properser自己决定；
    此时，Acceptor收到一个[N,V]的提案，如果N大于本地的提案编号，则接受提案并执行，并反馈给Proposer。
    如果没有超过半数的accept，就递增此次提案的编号，重新进入prepare阶段。
    具体流程如下：
    ![Image_text](https://upload-images.jianshu.io/upload_images/1752522-44c5a422f917bfc5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    3.Learner学习被选定的value：
    
    ![Image_text](https://upload-images.jianshu.io/upload_images/1752522-0fab48ed2bdf358a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    问题：存在死循环问题，两个Proposer依次提出编号递增的提案，都无法完成第二阶段，没有value被选定。
    解决：选择一个主Proposer，只有主proposer可以提出提案。
    
    
    
    
    
    
    
    
    
    
    
    
