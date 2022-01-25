#  分布式缓存Redis01

# 1. Redis的常用数据类型

* String
* List
* Hash
* Set
* ZSet

~~~java
public class JedisTests {

    // ------------------------ jedis 工具直连演示
    // jedis和redis命令名称匹配度最高,最为简洁,学习难度最低

    // 列表~ 集合数据存储~ java.util.List,java.util.Stack
    // 生产者消费者(简单MQ)
    public static void list() {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 插入数据1 --- 2 --- 3
        jedis.rpush("queue_1", "1");
        jedis.rpush("queue_1", "2", "3");
        //这里的参数start:0,stop:-1表示全部数据
        // 可以通过Irange命令,从某个元素开始读取多少个元素,可以基于list实现分页查询.
        List<String> strings = jedis.lrange("queue_1", 0, -1);
        for (String string : strings) {
            //1,2,3
            System.out.println(string);
        }

        // 消费者线程简例
        while (true) {
            String item = jedis.lpop("queue_1");
            if (item == null) break;
            //1,2,3
            System.out.println(item);
        }

        jedis.close();
    }

    // 类似：在redis里面存储一个hashmap
    // 推荐的方式,无特殊需求是,一般的缓存都用这个
    public static void hashTest() {
        HashMap<String, Object> user = new HashMap<>();
        user.put("name", "tony");
        user.put("age", 18);
        user.put("userId", 10001);
        //{name=tony, userId=10001, age=18}
        System.out.println(user);

        Jedis jedis = new Jedis("127.0.0.1", 6379);
        jedis.hset("user_10001", "name", "tony");
        jedis.hset("user_10001", "age", "18");
        jedis.hset("user_10001", "userId", "10001");

        // {name=tony, userId=10001, age=18}
        System.out.println(jedis.hgetAll("user_10001"));
        jedis.close();
    }

    // 用set实现(交集 并集)
    // 交集示例： 共同关注的好友
    // 并集示例：
    public static void setTest() {
        // 取出两个人共同关注的好友
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 每个人维护一个set
        jedis.sadd("user_A", "userC", "userD", "userE");
        jedis.sadd("user_B", "userC", "userE", "userF");
        // 取出共同关注
        Set<String> sinter = jedis.sinter("user_A", "user_B");
        //[userC, userE]
        System.out.println(sinter);

        // 检索给某一个帖子点赞/转发的
        jedis.sadd("trs_tp_1001", "userC", "userD", "userE");
        jedis.sadd("star_tp_1001", "userE", "userF");
        // 取出共同人群
        //[userC, userD, userE, userF]
        Set<String> union = jedis.sunion("star_tp_1001", "trs_tp_1001");
        System.out.println(union);

        jedis.close();
    }

    // 游戏排行榜
    public static void zsetTest() {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        String ranksKeyName = "exam_rank";
      	//zadd第二个参数是score,用score进行的排序
        jedis.zadd(ranksKeyName, 100.0, "tony");
        jedis.zadd(ranksKeyName, 82.0, "allen");
        jedis.zadd(ranksKeyName, 90, "mengmeng");
        jedis.zadd(ranksKeyName, 96, "netease");
        jedis.zadd(ranksKeyName, 89, "ali");
        //start:0,stop:2
        Set<String> stringSet = jedis.zrevrange(ranksKeyName, 0, 2);
        System.out.println("返回前三名:");
        //tony,netease,mengmeng
        for (String s : stringSet) {
            System.out.println(s);
        }

        Long zcount = jedis.zcount(ranksKeyName, 85, 100);
        //min 85,max:100 超过85分的数量为4
        System.out.println("超过85分的数量 " + zcount);

        jedis.close();
    }
}
~~~

# 2.持久化的方式

* **RDB持久化方式**能够在指定的时间间隔对数据进行快照存储;
* **AOF(append only file)持久化方式**记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据;

## 2.1.RDB方式

客户端直接通过命令BGSAVE或者SAVE来创建一个内存快照

* BGSAVE调用fork来创建一个子进程,子进程负责将快照写入磁盘,而父进程仍然继续处理命令;
* SAVE执行SAVE命令的过程中,不再响应其他命令;

在redis.conf中调整save配置选项,当在规定时间内,redis发生了写操作的个数满足条件会触发BGSAVE命令;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526140434229.png" alt="image-20200526140434229" style="zoom:50%;" />

### 2.1.1.RDB的优点

1. RDB会生成多个数据文件，每个数据文件都代表了某一个时刻中redis的数据，这种多个数据文件的方式，非常适合做冷备，可以将这种完整的数据文件发送到一些远程的安全存储上去，比如说Amazon的S3云服务上去，在国内可以是阿里云的ODPS分布式存储上，以预定好的备份策略来定期备份redis中的数据;

2. RDB对redis对外提供的读写服务，影响非常小，可以让redis保持高性能，因为redis主进程只需要fork一个子进程，让子进程执行磁盘IO操作来进行RDB持久化即可;

3. 相对于AOF持久化机制来说，直接基于RDB数据文件来重启和恢复redis进程，更加快速;

### 2.2.2.RDB的缺点

1. 如果想要在redis故障时，尽可能少的丢失数据，那么RDB没有AOF好。一般来说，RDB数据快照文件，都是每隔5分钟，或者更长时间生成一次，这个时候就得接受一旦redis进程宕机，那么会丢失最近5分钟的数据;

2. RDB每次在fork子进程来执行RDB快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒;

## 2.2.AOF方式

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526140652123.png" alt="image-20200526140652123" style="zoom:50%;" />

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200526140732823.png" alt="image-20200526140732823" style="zoom:50%;" />

开启AOF持久化:

* `appendonly yes`

AOF策略调整:

* `appendfsync always`:每次有数据修改发生时都会写入AOF文件
* `appendfsync everysec`:每秒钟同步一次,该策略为AOF的缺省策略;
* `appendfsync no`:从不同步,高效但是数据不会被持久化;

### 2.2.1.AOF的优点

1. AOF可以更好的保护数据不丢失，一般AOF会每隔1秒，通过一个后台线程执行一次fsync操作，最多丢失1秒钟的数据
2. AOF日志文件以append-only模式写入，所以没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复
3. AOF日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在rewrite log的时候，会对其中的指导进行压缩，创建出一份需要恢复数据的最小日志出来。再创建新日志文件的时候，老的日志文件还是照常写入。当新的merge后的日志文件ready的时候，再交换新老日志文件即可。
4. AOF日志文件的命令通过非常可读的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用flushall命令清空了所有数据，只要这个时候后台rewrite还没有发生，那么就可以立即拷贝AOF文件，将最后一条flushall命令给删了，然后再将该AOF文件放回去，就可以通过恢复机制，自动恢复所有数据

### 2.2.2.AOF的缺点

1. 对于同一份数据来说，AOF日志文件通常比RDB数据快照文件更大
2. AOF开启后，支持的写QPS会比RDB支持的写QPS低，因为AOF一般会配置成每秒fsync一次日志文件，当然，每秒一次fsync，性能也还是很高的
3. 以前AOF发生过bug，就是通过AOF记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似AOF这种较为复杂的基于命令日志/merge/回放的方式，比基于RDB每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有bug。不过AOF就是为了避免rewrite过程导致的bug，因此每次rewrite并不是基于旧的指令日志进行merge的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。

## 2.3.RDB和AOF如何选择

综合使用AOF和RDB两种持久化机制，用AOF来保证数据不丢失，作为数据恢复的第一选择; 用RDB来做不同程度的冷备，在AOF文件都丢失或损坏不可用的时候，还可以使用RDB来进行快速的数据恢复.

# 3.Redis线程模型

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210712201356714.png" alt="image-20210712201356714" style="zoom: 33%;" />

## 3.1.文件事件处理器

Redis基于Reactor模式开发了网络事件处理器,这个处理器叫做文件事件处理器,file event handler.**这个文件事件处理器是单线程的**,Redis才叫做单线程模型,采用IO多路复用机制同时监听多个socket,根据socket上的事件来选择对应的事件处理器来处理这个事件.

文件事件处理器的结构:

* 多个socket;
* IO多路复用程序;
* 文件事件分派器;
* 事件处理器(连接应答处理器,命令请求处理器,命令回复处理器);

## 3.2.一次客户端与Redis的完整通信过程

**建立连接**

1. 首先,redis 服务端进程初始化的时候,会将 server socket 的 AE_READABLE 事件与连接应答处理器关联。
2. 客户端 socket01 向 redis 进程的 server socket 请求建立连接,此时 server socket 会产生一个 AE_READABLE 事件,IO 多路复用程序监听到 server socket 产生的事件后,将该 socket 压入队列中。
3. 文件事件分派器从队列中获取 socket,交给连接应答处理器。
4. 连接应答处理器会创建一个能与客户端通信的 socket01,并将该 socket01 的 AE_READABLE 事件与命令请求处理器关联。

**执行一个set请求**

1. 客户端发送了一个 set key value 请求,此时 redis 中的 socket01 会产生 AE_READABLE 事件,IO 多路复用程序将 socket01 压入队列,
2. 此时事件分派器从队列中获取到 socket01 产生的 AE_READABLE 事件,由于前面 socket01 的 AE_READABLE 事件已经与命令请求处理器关联,
3. 因此事件分派器将事件交给命令请求处理器来处理。命令请求处理器读取 socket01 的 key value 并在自己内存中完成 key value 的设置。
4. 操作完成后,它会将 socket01 的 AE_WRITABLE 事件与命令回复处理器关联。
5. 如果此时客户端准备好接收返回结果了,那么 redis 中的 socket01 会产生一个 AE_WRITABLE 事件,同样压入队列中,
6. 事件分派器找到相关联的命令回复处理器,由命令回复处理器对 socket01 输入本次操作的一个结果,比如 ok,之后解除 socket01 的 AE_WRITABLE 事件与命令回复处理器的关联。

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210712203348619.png" alt="image-20210712203348619" style="zoom: 33%;" />

## 3.3.为什么Redis单线程还能支持高并发

1. 存内存操作;
2. 核心是基于非阻塞的IO多路复用机制;
3. 单线程避免了多线程的频繁上下文切换问题;

# 4.Redis内存管理

## 4.1.过期数据的处理策略

* 主动处理(redis主动触发检测key是否过期),redis默认100ms执行一次,过程如下;

  1. 从具有相关过期的密钥集中测试20个随机密钥;

  2. 删除找到的所有密钥已过期;
  3. 如果超过25%的密钥已过期,从步骤一开始;

* 被动处理:
  * 每次访问key的时候,发现超时后被动过期,清理掉;

那么**数据恢复阶段过期数据的处理策略**:

* RDB方式
  * 过期的key不会被持久化到文件中;
  * 载入时过期的key,会通过redis的主动和被动方式清理掉;
* AOF方式
  * 当redis使用AOF方式持久化,每次遇到过期的key,redis会追加一条DEL命令到AOF文件;也就是说只要我们顺序载入执行AOF命令文件就会删除过期的键;

## 4.2.Redis内存淘汰策略

1. noeviction:当内存不足以容纳新写入数据时,新写入操作会报错,这个一般没人用;
2. allkeys-lru:当内存不足以容纳新写入数据时,在键空间中,移除最近最少使用的key;(这个是最常用的);
3. allkeys-random:当内存不足以容纳新写入数据时,在键空间中,随机移除某个key,这个一般没人用;
4. volatile-lru:当内存不足以容纳新写入数据时,在设置了过期时间的键空间中,移除最近最少使用的key(这个一般不太合适);
5. volatile-random:当内存不足以容纳新写入数据时,在设置了过期时间的键空间中,随机移除某个key;
6. volatile-ttl:当内存不足以容纳新写入数据时,在设置了过期时间的键空间中,有更早过期时间的key优先移除;

### 4.2.1.LRU算法

LRU(Least recently used,最近最少使用):根据数据的历史访问记录来进行淘汰数据;

* 核心思想:如果数据最近被访问过,那么将来被访问的几率也更高;
* 注意:Redis的LRU算法并非完整的思想,完整的LRU实现需要太多的内存;
* 方法:通过对少量keys进行取样(50%),然后回收其中一个最好的key.配置方式:`maxmemory-samples 5`;

### 4.2.2.LFU算法

LFU(Least Frequently Used)根据数的历史访问频率来淘汰数据;

* 核心思想:如果数据过去被访问多次,那么将来被访问的频率也更高;
* Redis实现的是近似的实现,每次对key进行访问时,用基于概率的对数计算器来记录访问次数,同时这个计数器会随着时间推移而减小;
* 启用LFU算法后,可以使用热点数据分析功能`redis-cli --hotkeys`;

# 5.缓存雪崩,缓存穿透,缓存击穿

## 5.1.缓存雪崩

* 事前:Redis高可用,避免全盘崩溃;
* 事中:本地ehache缓存+hystrix限流&降级,避免MySQL被打死;
* 事后:Redis持久化,快速恢复缓存数据;

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200527151531365.png" alt="image-20200527151531365" style="zoom: 33%;" />

## 5.2.缓存穿透

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200527152119221.png" alt="image-20200527152119221" style="zoom: 33%;" />

## 5.3.缓存击穿

缓存击穿是指缓存中没有但数据库中有的数据,一般是缓存时间到期,这时由于并发用户特别多,同时读缓存没读到数据,又同时去数据库去取数据,引起数据库压力瞬间增大,造成过大压力

* 解决方法
  1. 设置热点数据永不过期;
  2. 加互斥锁(如果缓存中不存在数据,去请求数据库的时候加上互斥锁);

# 6.缓存与数据库双写一致问题 

数据的一致性包含两种情况:

* 缓存中有数据.那么,缓存的数据值需要和数据库中的值相同;
* 缓存中本身没有数据,那么,数据库中的值必须是最新值;

不属于这两种情况的,就属于缓存和数据库的数据不一致问题了;

对于读写缓存来说,如果要对数据进行增删改,就需要在缓存中进行,同时还要根据采取的写会策略,决定是否同步写到数据库中,策略如下:

* 同步直写策略:写缓存时,也同步写数据库,缓存和数据库中的数据一致;
* 异步写回策略:写缓存时不同步写缓存,等到数据从缓存中淘汰时,再写回数据库.使用这种策略时候,如果数据还没有写会数据库,缓存就发生了故障.那么此时数据库就没有最新的数据了;

对于只读缓存来说，如果有数据新增，会直接写入数据库；而有数据删改时，就需要把只读缓存中的数据标记为无效。这样一来，应用后续再访问这些增删改的数据时，因为缓存中没有相应的数据，就会发生缓存缺失。此时，应用再从数据库中把数据读入缓存，这样后续再访问数据时，就能够直接从缓存中读取了。

出现问题如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720103237371.png" alt="image-20210720103237371" style="zoom: 33%;" />

## 6.1.先删除缓存,再更新数据库情况

假设线程A删除缓存值后,还没有来得及更新数据库(比如有网络延迟),线程B就开始读数据了,那么这个时候,线程B会发现缓存缺失,就只能去数据库读取,带来两个问题;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720110637676.png" alt="image-20210720110637676" style="zoom: 33%;" />

**解决方案:在线程A更新完数据库值后,可以让它先sleep一小段时间,再进行一次缓存删除**;

之所以要加上 sleep 的这段时间，就是为了让线程 B 能够先从数据库读取数据，再把缺失的数据写入缓存，然后，线程 A 再进行删除。所以，线程 A sleep 的时间，就需要大于线程 B 读取数据再写入缓存的时间。这个时间怎么确定呢？建议你在业务程序运行的时候，统计下线程读数据和写缓存的操作时间，以此为基础来进行估算。这样一来，其它线程读取数据时，会发现缓存缺失，所以会从数据库中读取最新值。因为这个方案会在第一次删除缓存值后，延迟一段时间再次进行删除，所以我们也把它叫做“延迟双删”。

## 6.2.先更新数据库值,再删除缓存值

如果线程 A 删除了数据库中的值，但还没来得及删除缓存值，线程 B 就开始读取数据了，那么此时，线程 B 查询缓存时，发现缓存命中，就会直接从缓存中读取旧值。不过，在这种情况下，如果其他线程并发读缓存的请求不多，那么，就不会有很多请求读取到旧值。而且，线程 A 一般也会很快删除缓存值，这样一来，其他线程再次读取时，就会发生缓存缺失，进而从数据库中读取最新值。所以，这种情况对业务的影响较小。

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720111052314.png" alt="image-20210720111052314" style="zoom: 33%;" />

**解决方案:重试缓存删除**

重试机制,把删除的缓存自或者是要更新的数据库值展示存到消息队列中,当引用没有成功删除缓存或者是更新数据库值时候,可以从消息队列中读取这些值,然后再次进行删除或更新;如果成功地删除或更新,需要把这些值从消息队列中删除,意面重复操作.否则还需要进行重试,如果重试超过一定的次数,还是没有成功,就需要向业务层发送报错信息了.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720105204818.png" alt="image-20210720105204818" style="zoom: 33%;" />

## 6.3.解决方案总结

对于读写缓存来说，如果我们采用同步写回策略，那么可以保证缓存和数据库中的数据一致。只读缓存的情况比较复杂，如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720111502401.png" alt="image-20210720111502401" style="zoom: 33%;" />

如果业务层要求必须读取一致的数据,那么我们就需要在更新数据库的时候,先在Redis缓存客户端暂存并发请求,等数据库更新完,缓存值删除后,再读取数据,保证数据一致性.

# 7.Redis的并发竞争问题

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200527171057865.png" alt="image-20200527171057865" style="zoom: 33%;" />



