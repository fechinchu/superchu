# Zookeeper02

# 1.Zookeeper集群

## 1.1.Zookeeper集群搭建参数

1. `initLimit`

   集群中的follow服务器与leader服务器之间完成初始化同步连接时能容忍的最多心跳数.如果zk集群环境数量确实很大,同步数据的时间就会变长,因此这种情况可以适当调大该参数;

2. `syncLimit`

   集群中的follow服务器与leader服务器与leader服务器之间请求和应答之间能容忍的最多tickTime;

3. 集群节点

   `server.[id] = [host]:[port]:[port]`

   id:通过在各自的dataDir目录下创建一个名为myid的文件来为每台机器赋予一个服务器id;

   两个端口号:第一个跟随者用来连接到领导者,第二个用来选举领导者;

   注意:myid文件中一行只包含机器id的文本,id在集成中必须是唯一的,其值应该在1到255之间.如服务器1的id为"1";

## 1.2.Zookeeper集群-ZAB协议

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406140444264.png" alt="image-20210406140444264" style="zoom: 33%;" />

ZAB协议(Zookeeper Atomic Broadcast),Zookeeper原子消息广播协议是专为zookeeper设计的数据一致性协议;

1. 所有事务请求转发给leader;
2. Leader分配全局单调递增事务id(Zxid),广播事务提议;
3. Follower处理提议,做出反馈;
4. Leader收到过半数反馈,广播commit;
5. Leader做出响应;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406141207932.png" alt="image-20210406141207932" style="zoom: 33%;" />

## 1.3.Zookeeper集群-ZAB协议-崩溃恢复

Leader服务器出现崩溃,或者说由于网络原因导致Leader服务器失去了与过半Follower的联系,那么就会进入崩溃恢复模式;

* ZAB协议规定如果一个事务Proposal在一台机器上被处理成功,那么应该在所有的机器上都被处理成功,哪怕机器出现故障崩溃;
* ZAB协议确保那些已经在Leader服务器上提交的事务最终被所有服务器提交;
* ZAB协议确保丢弃那些只在Leader服务器上被提出的事务;

ZAB协议需要设计的选举算法应该满足:确保提交已经被Leader提交的事务Proposal,同时丢弃已经被跳过的事务Proposal;

* 如果让Leader选举算法能够保证新选举出来的Leader服务器拥有集群中所有机器最高ZXID的事务Proposal,那么久就可以保证这个新选举出来的Leader一定具有所有已提交的Proposal;
* 如果让具有最高编号事务Proposal机器来称为Leader,就可以省去Leader服务器检查Proposal的提交和丢弃工作的这一步操作;

## 1.4.Zookeeper集群-ZAB协议-数据同步

Leader选举出来后,需完成Followers与Leader的数据同步,当半数的Followers完成同步,则可以开始提供服务.同步过程如下:

* Leader服务器会为每个Follower服务器准备一个队列,并将那些没有被各Follower服务器同步的事务以Proposal消息的形式发送给Follower服务器,并在每一个Proposal消息后面紧接着再发送一个Commit消息,以表示该事务已经被提交;
* Follower服务器将所有其尚未同步的事务Proposal都从Leader服务器上同步过来并成功应用到本地数据库中后,Leader服务器就会将该Follower服务器加入到真正可用Follower列表中,并开始之后的其他流程;

## 1.5.Zookeeper集群-ZAB协议-丢弃事务Proposal处理

在ZAB协议的事务编号ZXID设计中,ZXID是一个64位数字.

* 低32位是一个简单的单调递增的计数器,针对客户端的每个事务请求,Leader服务器在产生一个新的事务Proposal的时候,都会对该计数器进行加1操作;
* 高32位代表了Leader周期纪元的编号,每当选举产生一个新的Leader服务器,就会从这个Leader服务器上取出其本地日志最大事务Proposal的ZXID,并从ZXID中解析出对应的纪元值,然后对其进行加1操作,之后就会以此编号作为新的纪元,并将低32位置0来开始生成新的ZXID;

基于这样的策略,当一个包含了上一个Leader周期中尚未提交过的事务Proposal的服务器启动加入到集群中,发现此集群中已经存在Leader,将自身以Follower角色连接上Leader服务器之后,Leader服务器会根据自己服务器上最后被提交的Proposal来和Follower服务器的proposal进行对比,发现follower上有上一个leader周期的事务Proposal时候,Leader会要求Follower进行一个回退操作(回退到一个确实已经被集群中过半机器提交的最新的事务Proposal);

## 1.6.Zookeeper集群-Leader选举

对选举算法的要求:

* 选出Leader节点上要持有最高的zxid;
* 过半数节点同意;

内置实现的选举算法:

* LeaderElection;
* FastLeaderElection(默认);
* AuthFastLeaderElection;

选举机制中的概念:

* 服务器id myid;
* 事务id,服务器中存放的最大Zxid;
* 逻辑时钟,发起的投票轮数计数;
* 选举状态:
  * Looking:竞选状态;
  * Following:随从状态,同步Leader状态,参与投票;
  * Observing:观察状态,同步Leader状态,不参与投票;
  * Leading,领导者状态;

选举算法:

1. 每个服务实例均发起选举自己为领导者的投票(自己的投给自己);
2. 其他服务实例收到投票邀请时,比较发起者的数据事务ID是否比自己最新的事务ID大,大则给它投票,小则不投票给它,相等则比较发起者的服务器ID,大则投票给它;
3. 发起者收到大家的投票反馈后,看投票数(含自己的)是否大于集群的半数,大于则胜出,担任领导者,未超过半数且领导者未选出,则再次发起投票;

# 2.分布式一致性算法

## 2.1.2PC

2PC:将提交的更新操作分为提交请求(投票),提交(完成)两个阶段来达成一致性算法.主要用于事务管理,分布式一致性;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406153947671.png" alt="image-20210406153947671" style="zoom:33%;" />

1. 协调者询问各参与者是否可以提交该更新;等待所有参与者给出响应;
2. 参与者执行事务操作到等待提交指令的点(这个过程中记录redo,undo日志);
3. 参与者响应是否准备好提交的结果给协同者,并阻塞等待协调者的下一步指令;
4. 协调者接收所有参与者的响应,如果超时未收到响应,当成abort处理,有一个abort,下一步是回滚;
5. 协调者向所有参与者发出提交或回滚消息;
6. 参与者执行提交/回滚,释放占用的锁等资源,并作出响应;
7. 结束;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406154928696.png" alt="image-20210406154928696" style="zoom: 33%;" />

2PC存在的不足:

* 上述第3步,阻塞等待协调者的下一步指令,准备完成时,如果协调者宕机,所有参与者将一直阻塞;
* 上述第5步,协调者向所有参与者发出提交或回滚消息,如果参与者宕机,将接受不到提交消息,会出现不一致,需要人工干预;

## 2.2.3PC

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406160103725.png" alt="image-20210406160103725" style="zoom: 33%;" />

 在doCommit阶段，如果参与者无法及时接收到来自协调者的doCommit或者rebort请求时，会在等待超时之后，会继续进行事务的提交。（其实这个应该是基于概率来决定的，当进入第三阶段时，说明参与者在第二阶段已经收到了PreCommit请求，那么协调者产生PreCommit请求的前提条件是他在第二阶段开始之前，收到所有参与者的CanCommit响应都是Yes。（一旦参与者收到了PreCommit，意味他知道大家其实都同意修改了）所以，一句话概括就是，当进入第三阶段时，由于网络超时等原因，虽然参与者没有收到commit或者abort响应，但是他有理由相信：成功提交的几率很大。 ）;

相对于2PC，3PC主要解决的单点故障问题，并减少阻塞，因为一旦参与者无法及时收到来自协调者的信息之后，他会默认执行commit。而不会一直持有事务资源并处于阻塞状态。但是这种机制也会导致数据一致性问题，因为，由于网络原因，协调者发送的abort响应没有及时被参与者接收到，那么参与者在等待超时之后执行了commit操作。这样就和其他接到abort命令并执行回滚的参与者之间存在数据不一致的情况。

## 2.3.Paxos

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406164126293.png" alt="image-20210406164126293" style="zoom:33%;" />

* Proposer:提议者,负责提议,提出想要达成一致的value的提案;
* Acceptor:接收者,对提案投票,决定是否接受此value提案;
* Learner:学习者,不参与投票,学习投票决定的提案;

一个节点既可以是提议者,也可以是接受者;

### 2.3.1.Paxos流程

#### 2.3.1.1.Paxos写流程

1. 提案:
   1. 提案编号:唯一,保证全局递增,体现提案的先后顺序;
   2. 更新值
2. 写流程,两阶段:
   1. 准备阶段(投票阶段):
      1. 提议者提出提案给接收者;
      2. 提议者如果接收该提案,做出响应,并不再Accept比当前提案号低的提案;
      3. 提议者收到超过半数的响应,则进入下一阶段,否则丢弃提案;
   2. 接受变更阶段(提交阶段);
      1. 提议者向接收者发送Accept消息;
      2. 接收者比较Accpet消息中的提案号,如果比自己当前已接收的提案号低,则不Accept,否则Accept;

 #### 2.3.1.2.Paxos读流程

1. 接收客户端请求的节点,向集群广播获取大家的当前值;
2. 接收到过半数相同的值,则返回该值,如果本地的值不同,则更新为多数值;
3. 如果得不到过半数的相同的值,则读取失败;

### 2.3.2.基于Paxos算法的协议

1. ZAB:参考上文
2. raft:https://raft.github.io/

