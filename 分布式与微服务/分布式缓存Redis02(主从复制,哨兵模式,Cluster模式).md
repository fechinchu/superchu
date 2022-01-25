#  分布式缓存Redis02

# 1.Redis主从复制

## 1.1.主从复制核心机制

1. redis采用异步方式复制数据到slave节点,不过redis2.8开始,slave node会周期性地确认自己每次复制的数据量;
2. 一个master node是可以配置多个slave node的,slave node也可以连接其他的slave node;
3. slave node做复制的时候,是不会block master node的正常工作的;
4. slave node在做复制的时候,也不会block对自己的查询操作,它会用旧的数据来提供服务,但是复制完成的时候,需要删除旧数据集,这个时候就会暂停对外服务了;
5. slave node主要用来横向扩容,做读写分离,扩容的slave node可以提高读的吞吐量;

## 1.2.主从复制流程

1. 当启动一个slave node的时候,slave node会发送一个PSYNC命令给master node;

2. 如果这是slave node重新连接master node,那么master node仅仅会复制给slave部分缺少的数据;否则如果是slave node第一次连接master node,那么会触发一次full resynchronization;

3. 开始full resynchronization的时候,master会启动一个后台线程,开始生成一份RDB快照文件,同时还会将从客户端收到的所有写命令缓存在内存中.RDB文件生成完毕后,master会将这个RDB发送给slave,slave会先写入本地磁盘,然后再从本地磁盘加载到内存中.然后master会将内存中新缓存的写命令直接发送到slave,slave也会同步这些数据;

4. slave node如果跟master node有网络故障,断开了连接,会自动重连;

5. master 如果有多个slave node,仅仅会启动一个rdb save操作,用一份数据服务所有slave;

6. 从redis 2.8开始,就支持主从复制的断点续传,如果主从复制过程中,网络连接断掉了,可以接着上次复制的地方,继续复制下去;

7. redis支持无磁盘化复制,master在内存中 直接创建rdb,然后发送给slave,不会在自己本地落地磁盘了.在conf文件中做如下修改:

   ```shell
   repl-diskless-sync yes #开启无磁盘化复制
   repl-diskless-sync-delay 5 #等待一定时长再开始复制,因为要等待更多slave重新
   ```

8. slave不会过期key,只会等待master过期key.如果master过期了一个key,或者通过LRU淘汰了一个key,那么会模拟一条del命令发送给slave;

## 1.3.主从复制的细节概念

1. master和salve都会维护一个offset

   master会在自身不断累加offset,slave也会在自身不断累加offset.slave每秒都会上报自己的offset给master,同时master也会保存每个slave的offset;

2. backlog

   master node有一个backlog,默认是1MB大小,master node给slave node复制数据时,也会将数据在backlog中同步写一份
   backlog主要是用来做全量复制中断候的增量复制的;

3. runid

   如果根据host+ip定位master node,是不靠谱的,如果master node重启或者数据出现了变化,那么slave node应该根据不同的run id区分,run id不同就做全量复制.

### 1.3.1.全量复制

1. master执行bgsave,在本地生成一份rdb快照文件;
2. master node将rdb快照文件发送给salve node,如果rdb复制时间超过60秒(repl-timeout),那么slave node就会认为复制失败,可以适当调节大这个参数;
3. 对于千兆网卡的机器,一般每秒传输100MB,6G文件,很可能超过60s;
4. master node在生成rdb时,会将所有新的写命令缓存在内存中,在salve node保存了rdb之后,再将新的写命令复制给salve node;
5. client-output-buffer-limit slave 256MB 64MB 60,如果在复制期间,内存缓冲区持续消耗超过64MB,或者一次性超过256MB,那么停止复制,复制失败;
6. slave node接收到rdb之后,清空自己的旧数据,然后重新加载rdb到自己的内存中;
7. 如果slave node开启了AOF,那么会立即执行BGREWRITEAOF,重写AOF;

### 1.3.2.增量复制

1. 如果全量复制过程中,master-slave网络连接断掉,那么salve重新连接master时,会触发增量复制;
2. master直接从自己的backlog中获取部分丢失的数据,发送给slave node,默认backlog就是1MB;
3. msater就是根据slave发送的psync中的offset来从backlog中获取数据的;

### 1.3.3.异步复制

master每次接收到写命令之后,先在内部写入数据,然后异步发送给slave node;

## 1.4.主从复制与读写分离  

* 主从复制可以用来支持读写分离;
* slave服务器设定为只读,可以用在数据安全的场景下;
* 可以使用主从复制来避免master持久化造成的开销.master关闭持久化,slave配置为RDB或是启用AOF.(注意:重新启动master程序将从一个空数据集开始,如果一个slave试图与它同步,那么这个slave也会被清空);

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526163746659.png" alt="image-20200526163746659" style="zoom: 33%;" />

## 1.5.主从复制的问题

* 读写分离场景:
  * 数据复制延迟导致读到过期数据或者读不到数据(网络原因,slave阻塞);
* 全量复制的情况下:
  * 第一次建立主从关系或者runid不匹配会导致全量复制;
  * 故障转移的时候也会出现全量复制;
* 复制风暴:
  * master故障重启,如果slave节点较多,所有slave都要复制,对服务器的性能,网络的压力都有很大影响;
* 写能力有限:
  * 主从复制只有一台master,提供的写服务能力有限;
* master故障情况下:
  * 如果master无持久化,slave开启持久化来保留数据的场景,建议不要配置redis的自动重启;
  * 如果redis自动重启,master启动后,无备份数据,可能导致集群数据丢失的情况;

# 2.Redis高可用哨兵模式(Sentinel)

## 2.1.哨兵介绍

sentinel,中文哨兵.主要功能如下:

1. 集群监控:负责监控redis master和slave进程是否正常工作;
2. 消息通知:如果某个redis实例有故障,那么哨兵负责发送消息作为报警通知给管理员; 
3. 故障转移:如果master node挂了,会自动转移到slave node上;
4. 配置中心,如果故障转移发生了,通知client客户端新的master地址;  

## 2.2.哨兵部署

* 哨兵至少需要3个实例,来保证自己的健壮性;
* 哨兵+redis主从的部署架构是不会保证数据零丢失的,只能保证redis集群的高可用性;

> 1. 为什么redis哨兵集群只有2个节点无法正常工作?
>
>    哨兵集群必须部署2个以上节点.如果哨兵集群仅仅部署了2个哨兵实例,quorum=1;
>
>    <img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210718185117321.png" alt="image-20210718185117321" style="zoom: 50%;" />
>
>    Configuration:`quorum=1`;master宕机,s1和s2中只要有1个哨兵认为master宕机就可以进行切换,同时s1和s2中会选举出一个哨兵来执行故障转移;同时这个时候,需要majority,也就是大多数哨兵都是运行的,2个哨兵的majority就是2(2的majority=2,3的majority=2,4的majority=2,5的majority=3),2个哨兵都运行着,就可以允许执行故障转移,如果整个M1和S1宕机了,那么哨兵只有1个,只是就没有majority来允许执行故障转移.
>
> 2. 3节点哨兵集群;
>
>    <img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210718185720889.png" alt="image-20210718185720889" style="zoom:50%;" />
>
>    Configuration:`quorum=2,majority=2`,如果M1所在的机器宕机了,那么三个哨兵还剩下2个,S2和S3可以一致认为master宕机,然后选举出来一个执行故障转移;同时majority是2,还剩下2个哨兵运行着,就可以允许执行故障转移;

## 2.3.quorum和majority

* 每次一个哨兵要做主备切换,首先需要quorum数量的哨兵认为odown,然后选举出一个哨兵来做切换,这个哨兵还得得到majority哨兵的授权,才能正式执行切换;
* 如果quorum < majority,比如5个哨兵,majority就是3,quorum设置为2,那么就3个哨兵授权就可以执行切换;
* 但是如果quorum >= majority,那么必须quorum数量的哨兵都授权,比如5个哨兵,quorum是5,那么必须5个哨兵都同意授权,才能执行切换;

## 2.4.哨兵sdown和odown

启动命令`redis-server /path/to/sentinel.conf --sentinel`;

配置文件启动时指定,运行过程中会自动变更,记录哨兵的监测结果;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526204137815.png" alt="image-20200526204137815" style="zoom: 33%;" />

* 哨兵如何知道Redis主从信息?
  * 哨兵配置文件中,保存着主从集群中master的信息,可以通过info命令,进行主从信息自动发现;
* 什么是主观下线(sdown)?
  * 主观下线:单个哨兵自身认为Redis实例已经不能提供服务;
  * 监测机制:哨兵向Redis发送PING请求,+PONG,-LOADING,-MASTERDOWN这三种情况视为正常,其他回复均视为无效;
* 什么是客观下线(odown)?
  * 客观下线:一定数量值的哨兵认为master已经下线;
  * 监测机制:则会通过`SENTINEL is-master-down-by-addr`命令询问其他哨兵是否认为master已经下线,如果达成共识(达到quorum个数),就会认为master节点客观下线,开始故障转移流程;

## 2.5.哨兵和slave集群的自动发现机制

* 哨兵互相之间的发现,是通过redis的pub/sub系统实现的,每个哨兵都会往__sentinel__:hello这个channel里发送一个消息,这时候所有其他哨兵都可以消费到这个消息,并感知到其他的哨兵的存在;

* 每隔两秒钟,每个哨兵都会往自己监控的某个master+slaves对应的__sentinel__:hello channel里发送一个消息,内容是自己的host、ip和runid还有对这个master的监控配置;

* 每个哨兵也会去监听自己监控的每个master+slaves对应的__sentinel__:hello channel,然后去感知到同样在监听这个master+slaves的其他哨兵的存在;

* 每个哨兵还会跟其他哨兵交换对master的监控配置,互相进行监控配置的同步;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526210158691.png" alt="image-20200526210158691" style="zoom: 33%;" />

## 2.6.哨兵领导选举机制

基于Raft算法实现的选举机制,流程如下:

1. 拉票阶段:每个哨兵节点希望自己成为领导者;
2. sentinel节点收到拉票命令后,如果没有收到或同意过其他sentinel节点的请求,就同意该sentinel节点的请求(每个sentinel只持有一个同意票数);
3. 如果sentinel节点发现自己的票数已经超过一般的数值,那么它将成为领导者,去执行故障转移;
4. 投票结束后,如果超过failover-timeout的时间内,没有进行实际的故障转移操作,则重新拉票选举;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526211208801.png" alt="image-20200526211208801" style="zoom: 33%;" />

## 2.7.Slave->Master选举算法

 如果一个master被认为odown了,而且majority哨兵都允许了准备切换,此时首先要选举一个slave来.会考虑slave的一些信息;如下顺序有优先级

1. 如果一个slave根master断开时间已经超过了`(down-after-milliseconds*10)+milliseconds_since_master_is_in_SDOWN_state`即down-after-milliseconds的10倍,外加master宕机的时长,那么slave就被认为不适合选举为master.
2. `slave-priority`值越小,优先级越高;
3. 如果`slave-priority`相同,看replica offset,哪个slave复制了越多的数据,offset越靠后,优先级就越高;
4. 如果上述条件都相同,选择 run id比较小的slave;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526211950920.png" alt="image-20200526211950920" style="zoom: 33%;" />

最终主从切换过程
* 针对即将成为master的slave节点,将其撤出主从集群,自动执行:`slaveof no one`;
* 针对其他slave节点,使他们成为新master的从属,自动执行:`slaveof new_master_host new_master_port`;

## 2.8.哨兵模式数据丢失和解决方案

### 2.8.1.数据丢失的情况

1. 异步复制导致的数据丢失;

   因为master --> slave的复制是异步的,所以可能有部分数据还没复制到slave,master就宕机了,此时这些部分数据就丢失了;

2. 脑裂导致的数据丢失;

   某个master所在机器突然脱离了正常的网络,跟其他slave机器不能连接,但是实际上master还运行着.此时哨兵可能就会认为master宕机了,然后开启选举,将其他slave切换成了master.这个时候,集群里就会有两个master,脑裂.此时虽然某个slave被切换成了master,但是可能client还没来得及切换到新的master,还继续写向旧master的数据可能也丢失了.因此旧master再次恢复的时候,会被作为一个slave挂到新的master上去,自己的数据会清空,重新从新的master复制数据;

### 2.8.2.减少异步复制和脑裂导致的数据丢失

~~~shell
min-slaves-to-write 1 
min-slaves-max-lag 10
#要求至少有1个slave,数据复制和同步的延迟时间不能超过10s;如果说一旦所有的slave,数据复制和同步的延迟都超过了10s,那么这个时候,master就不会再接收任何请求了.
~~~

1. 减少异步复制的数据丢失:

   有了min-slaves-max-lag这个配置,一旦slave复制数据和ack延时太长,就认为可能master宕机后损失的数据太多了,那么就拒绝写请求,这样可以把master宕机时由于部分数据未同步到slave导致的数据丢失降低的可控范围内;

2. 减少脑裂的数据丢失:

   如果一个master出现了脑裂,跟其他slave丢了连接,如果不能继续给指定数量的slave发送数据,而且slave超过10秒没有给自己ack消息,那么就直接拒绝客户端的写请求,这样脑裂后的旧master就不会接受client的新数据,也就避免了数据丢失,上面的配置就确保了,如果跟任何一个slave丢了连接,在10秒后发现没有slave给自己ack,那么就拒绝新的写请求,因此在脑裂场景下,最多就丢失10秒的数据.


# 3.Redis高可用Cluster模式

Redis cluster支撑N个redis master node,每个master node都可以挂载多个salve node.对于每个master来说,写就写到master,读就到从master对应的slave去读;

## 3.1.官方集群方案

redis cluster是Redis的分布式集群解决方案,在3.0版本推出后有效地解决了redis分布式方面的需求,实现了数据在多个Redis节点之间自动分片,故障自动转移,扩容机制等功能;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526214113624.png" alt="image-20200526214113624" style="zoom: 33%;" />

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526220401387.png" alt="image-20200526220401387" style="zoom: 33%;" />

## 3.2.Hash算法

对key进行计算hash值,然后对master节点数量取模.最大的问题,任意一个master宕机,那么大量的数据就需要重新计算写入缓存,风险极大;

## 3.3.一致性Hash算法

可参考https://www.cnblogs.com/lpfuture/p/5796398.html;

## 3.4.Hash Slot算法

* redis cluster有固定的16384个hash slot，对每个key计算CRC16值，然后对16384取模，可以获取key对应的hash slot;
* redis cluster中每个master都会持有部分slot，比如有3个master，那么可能每个master持有5000多个hash slot;
* hash slot让node的增加和移除很简单，增加一个master，就将其他master的hash slot移动部分过去，减少一个master，就将它的hash slot移动到其他master上去,移动hash slot的成本是非常低的;
* 如果有3个master节点,任何一台机器宕机,另外两个节点,不影响,因为key找的是hash slot,找的不是机器;
* 客户端的api，可以对指定的数据，让他们走同一个hash slot，通过hash tag来实现;

## 3.5.节点间的内部通信机制

Redis cluster节点采用gossip协议进行通信;跟集中式不同,不是将集群元数据集中存储在节点上,而是互相之间不断通信,保持整个集群所有节点都是完整的.

维护集群的元数据一般有两种:

1. 集中式:好处在于，元数据的更新和读取，时效性非常好，一旦元数据出现了变更，立即就更新到集中式的存储中，其他节点读取的时候立即就可以感知到; 不好在于，所有的元数据的跟新压力全部集中在一个地方，可能会导致元数据的存储有压力;
2. gossip:好处在于，元数据的更新比较分散，不是集中在一个地方，更新请求会陆陆续续，打到所有节点上去更新，有一定的延时，降低了压力; 缺点，元数据更新有延时，可能导致集群的一些操作会有一些滞后;

## 3.6.面向cluster集群的jedis内部实现原理

* 基于重定向的客户端
  * 请求重定向
    * 客户端可能回挑选任意一个redis实例去发送命令,每个redis实例接收到命令,都会计算对应的hash slot;如果在本地就在本地处理,否则返回moved给客户端,让客户端重定向;
    * `cluster keyslot mykey`可以查看一个key对应的hash slot是什么
    * 用redis-cli的时候可以加`-c`参数,支持自动的请求重定向,redis-cli接收到moved之后,会自动重定向到对应的节点执行命令;
  * 计算hash slot
    * 计算hash slot的算法,就是根据key计算CRC16值,然后对16384取模,拿到对应的hash slot;
    * 用hash tag可以手动指定key对应的slot,同一个hash tag下的key,都会在一个hash slot中,比如`set mykey1:{100}`和`set mykey2:{100}`;
  * hash slot查找
    * 节点间通过gossip协议进行数据交换,就知道每个hash slot在哪个节点上;

* smart jedis

  * smart jedis工作原理
    * 基于重定向的客户端,很消耗网络IO,因为大部分情况下,可能都会出现一次请求重定向,才能找到正确的节点;
    * 大部分的客户端,如java redis客户端jedis就是smart的;
    * 本地维护一份hashslot -> node的映射表,缓存,大部分情况下,直接走本地缓存就可以找到hashslot -> node;

  * JedisCluster的工作原理
    * 在JedisCluster初始化的时候，就会随机选择一个node，初始化hashslot -> node映射表，同时为每个节点创建一个JedisPool连接池.每次基于JedisCluster执行操作，首先JedisCluster都会在本地计算key的hashslot，然后在本地映射表找到对应的节点.如果那个node正好还是持有那个hashslot，那么就ok; 如果说进行了reshard这样的操作，可能hashslot已经不在那个node上了，就会返回moved.如果JedisCluter API发现对应的节点返回moved，那么利用该节点的元数据，更新本地的hashslot -> node映射表缓存.重复上面几个步骤，直到找到对应的节点，如果重试超过5次，那么就报错，JedisClusterMaxRedirectionException.jedis老版本，可能会出现在集群某个节点故障还没完成自动切换恢复时，频繁更新hash slot，频繁ping节点检查活跃，导致大量网络IO开销.jedis最新版本，对于这些过度的hash slot更新和ping，都进行了优化，避免了类似问题.
  * hashslot迁移和ask重定向
    * 如果hash slot正在迁移，那么会返回ask重定向给jedis;
    * jedis接收到ask重定向之后，会重新定位到目标节点去执行，但是因为ask发生在hash slot迁移过程中，所以JedisCluster API收到ask是不会更新hashslot本地缓存
    * 已经可以确定说，hashslot已经迁移完了，moved是会更新本地hashslot->node映射表缓存的

## 3.7.高可用性与主备切换的原理

redis cluster的高可用的原理,几乎与哨兵类似:

* 判断节点宕机
  * 如果一个节点认为另外一个节点宕机，那么就是pfail，主观宕机
  * 如果多个节点都认为另外一个节点宕机了，那么就是fail，客观宕机，跟哨兵的原理几乎一样，sdown，odown
  * 在cluster-node-timeout内，某个节点一直没有返回pong，那么就被认为pfail
  * 如果一个节点认为某个节点pfail了，那么会在gossip ping消息中，ping给其他节点，如果超过半数的节点都认为pfail了，那么就会变成fail
* 从节点过滤
  * 对宕机的master node，从其所有的slave node中，选择一个切换成master node
  * 检查每个slave node与master node断开连接的时间，如果超过了`cluster-node-timeout * cluster-slave-validity-factor`，那么就没有资格切换成master
  * 这个也是跟哨兵是一样的，从节点超时过滤的步骤

* 从节点选举
  * 哨兵：对所有从节点进行排序，slave priority，offset，run id
  * 每个从节点，都根据自己对master复制数据的offset，来设置一个选举时间，offset越大（复制数据越多）的从节点，选举时间越靠前，优先进行选举
  * 所有的master node开始slave选举投票，给要进行选举的slave进行投票，如果大部分master node（N/2 + 1）都投票给了某个从节点，那么选举通过，那个从节点可以切换成master
  * 从节点执行主备切换，从节点切换为主节点

* 与哨兵比较
  * 整个流程跟哨兵相比，非常类似，所以说，redis cluster功能强大，直接集成了replication和sentinal的功能



