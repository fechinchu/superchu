# Dubbo原理

 # 1.Dubbo的工作原理

具体Dubbo的原理可以查看官网:https://dubbo.apache.org/zh/docs/v2.7/dev/design/

![image-20210720155746434](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720155746434.png)

## 1.1.整体设计

![image-20210720164301904](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720164301904.png)

- 图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。
- 图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为 API，其它各层均为 SPI。
- 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。
- 图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调时链，紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。

![image-20210728201653413](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210728201653413.png)

## 1.2.分层

Dubbo框架的分层

1. **config 配置层**：对外配置接口，以 `ServiceConfig`, `ReferenceConfig` 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
2. **proxy 服务代理层**：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 `ServiceProxy` 为中心，扩展接口为 `ProxyFactory`
3. **registry 注册中心层**：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 `RegistryFactory`, `Registry`, `RegistryService`
4. **cluster 路由层**：封装多个提供者的路由及负载均衡，并桥接注册中心，以 `Invoker` 为中心，扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`
5. **monitor 监控层**：RPC 调用次数和调用时间监控，以 `Statistics` 为中心，扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`
6. **protocol 远程调用层**：封装 RPC 调用，以 `Invocation`, `Result` 为中心，扩展接口为 `Protocol`, `Invoker`, `Exporter`
7. **exchange 信息交换层**：封装请求响应模式，同步转异步，以 `Request`, `Response` 为中心，扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`
8. **transport 网络传输层**：抽象 mina 和 netty 为统一接口，以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`
9. **serialize 数据序列化层**：可复用的一些工具，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`

分层的说明:

- 在 RPC 中，Protocol 是核心层，也就是只要有 Protocol + Invoker + Exporter 就可以完成非透明的 RPC 调用，然后在 Invoker 的主过程上 Filter 拦截点。
- 图中的 Consumer 和 Provider 是抽象概念，只是想让看图者更直观的了解哪些类分属于客户端与服务器端，不用 Client 和 Server 的原因是 Dubbo 在很多场景下都使用 Provider, Consumer, Registry, Monitor 划分逻辑拓普节点，保持统一概念。
- 而 Cluster 是外围概念，所以 Cluster 的目的是将多个 Invoker 伪装成一个 Invoker，这样其它人只要关注 Protocol 层 Invoker 即可，加上 Cluster 或者去掉 Cluster 对其它层都不会造成影响，因为只有一个提供者时，是不需要 Cluster 的。
- Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是去掉 Proxy 层 RPC 是可以 Run 的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。
- 而 Remoting 实现是 Dubbo 协议的实现，如果你选择 RMI 协议，整个 Remoting 都不会用上，Remoting 内部再划为 Transport 传输层和 Exchange 信息交换层，Transport 层只负责单向消息传输，是对 Mina, Netty, Grizzly 的抽象，它也可以扩展 UDP 传输，而 Exchange 层是在传输层之上封装了 Request-Response 语义。
- Registry 和 Monitor 实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。

## 1.3.调用链

![image-20210720164034673](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720164034673.png)

# 2.Dubbo的通信协议

Dubbo的通信协议查考:https://dubbo.apache.org/zh/docs/v3.0/references/protocols/

# 3.Dubbo的负载均衡

Dubbo的负载均衡策略参考:https://dubbo.apache.org/zh/docs/v2.7/user/examples/loadbalance/#m-zhdocsv27userexamplesloadbalance

![image-20210720181108299](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720181108299.png)

# 4.Dubbo的集群容错

Dubbo的集群容错参考:https://dubbo.apache.org/zh/docs/v2.7/user/examples/fault-tolerent-strategy/#m-zhdocsv27userexamplesfault-tolerent-strategy

![image-20210720191443083](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720191443083.png)

# 5.Dubbo的SPI

Dubbo的SPI参考:https://dubbo.apache.org/zh/docs/v2.7/dev/source/dubbo-spi/#m-zhdocsv27devsourcedubbo-spi

SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。正因此特性，我们可以很容易的通过 SPI 机制为我们的程序提供拓展功能。SPI 机制在第三方框架中也有所应用，比如 Dubbo 就是通过 SPI 机制加载所有的组件。不过，Dubbo 并未使用 Java 原生的 SPI 机制，而是对其进行了增强，使其能够更好的满足需求。

自己如何去扩展Dubbo中的组件?

1. 自己写一个工程,需要哪种可以打成jar包的,里面的`src/main/resources`目录下,搞一个`META-INF/services`,里面放一个文件叫`com.alibaba.dubbo.rpc.Protocol`,文件里面`fechin=com.fechin.MyProtocol`.然后把Jar包放到nexus私服里
2. 自己做一个dubbo provider工程,在这个工程里面依赖自己的那个jar,然后在Spring配置文件里给个配置.`<dubbo:protocol name='fechin' port='20000'>`
3. 这个时候provider启动的时候,会加载到jar包中的`fechin=com.fechin.MyProtocol`,接着会根据自己的配置使用定义好的Myprotocol了.通过上述方式,可以替换掉大量的dubbo内部组件;

Dubbo 通过SPI拓展示例:https://dubbo.apache.org/zh/docs/v2.7/dev/impls/protocol/

![image-20210720200512261](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720200512261.png)

![image-20210720201617977](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720201617977.png)

# 6.基于Dubbo如何做服务治理,服务降级以及服务重试

## 6.1.基于Skywalking分布式链路追踪

https://dubbo.apache.org/zh/docs/v2.7/admin/ops/skywalking/

## 6.2.服务降级

https://dubbo.apache.org/zh/docs/v3.0/references/features/service-downgrade/#m-zhdocsv30referencesfeaturesservice-downgrade

## 6.3.服务重试

**https://dubbo.apache.org/zh/docs/v2.7/user/examples/%E9%87%8D%E8%AF%95%E6%AC%A1%E6%95%B0%E9%85%8D%E7%BD%AE/#m-zhdocsv27userexamplese9878de8af95e6aca1e695b0e9858de7bdae**

# 7.如何保证接口的幂等性

如下是项目中盱眙龙虾回调保证接口幂等性的代码,使用Redis

~~~java
@RequestMapping(value = "/payNotify")
    @ResponseBody
    @Transactional
    @SuppressWarnings("all")
    public ResSyncPayDTO syncPayInfo(XXX) {//参数省略
        LOGGER.info("/front/baiduMiniProgram/payNotify 方法开始了");
        //记录任务
        BaiduNotifyDto baiduNotifyDto = new BaiduNotifyDto();
        try {
            ReqSyncPayDTO reqSyncPayDTO = new ReqSyncPayDTO();
            //Bean copyProperties代码省略
            try {
                //0.保证接口幂等
                //解决rsaSign将" "替换成"+"
                rsaSign = rsaSign.replace(" ", "+");
                baiduNotifyDto.setHandleStatus(0);//处理中
                baiduNotifyService.saveNotifyMsg(baiduNotifyDto);

                Boolean idempotentFlag = redisTemplate.opsForValue().setIfAbsent(REDIS_KEY + orderId, "consumptionNot");
                if (!idempotentFlag) {
                    ResSyncPayDTO fail = new ResSyncPayDTO.Builder().setErrno(500).setMsg("已重复调用").setIsErrorOrder(1).setIsConsumed(2).build();
                    return fail;
                }
                //1.验证签名++++++++++++++++++++++
                HashMap<String, Object> signMap = new HashMap<>();
                //Map放数据代码省略

                //todo
                try {
                    boolean flag = BaiduRSASign.checkSign(signMap, BaiduRSASign.BD_PUBLIC_KEY);
                    if (!flag) {
                        //验证签名不通过
                        ResSyncPayDTO fail = new ResSyncPayDTO.Builder().setErrno(500).setMsg("验证签名不通过").setIsErrorOrder(1).setIsConsumed(2).build();
                        LOGGER.error("RSASign check fail");
                        return fail;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                    LOGGER.error("RSASign checkError", e);
                    ResSyncPayDTO exception = new ResSyncPayDTO.Builder().setErrno(500).setMsg("签名验证异常").setIsErrorOrder(1).setIsConsumed(2).build();
                    throw new RuntimeException("验证签名异常", e);
                }
                //+++++++++++++++++++++++++++++++
                //在订单表中插入userId和orderId
                //更新baiduUserId和baidRecordId
                log.info("/front/baiduMiniProgram/payNotify queryRecordIdByPayId tpOrderId"+tpOrderId);
                Long recordId = orderInfoService.queryRecordIdByPayId(tpOrderId);
                if(recordId==null){
                    log.error("/front/baiduMiniProgram/payNotify queryRecordIdByPayId tpOrderId"+recordId);
                    return new ResSyncPayDTO.Builder().setErrno(500).setMsg("查询不到订单").setIsErrorOrder(1).setIsConsumed(2).build();
                }
                log.info("/front/baiduMiniProgram/payNotify queryRecordIdByPay recordId"+recordId);
                ReqUpdateUserIdAndOrderIdDTO reqUpdateUserIdAndOrderIdDTO = new ReqUpdateUserIdAndOrderIdDTO();
                reqUpdateUserIdAndOrderIdDTO.setBaiduUserId(userId);
                reqUpdateUserIdAndOrderIdDTO.setBaiduOrderId(orderId);
                reqUpdateUserIdAndOrderIdDTO.setRecordId(recordId);
                log.info("/front/baiduMiniProgram/payNotify updateUserIdAndOrderId reqUpdateUserIdAndOrderIdDTO"+JSON.toJSONString(reqUpdateUserIdAndOrderIdDTO));
                orderInfoService.updateUserIdAndOrderId(reqUpdateUserIdAndOrderIdDTO);

                Map<String, Object> mapRR = new HashMap<String, Object>();
                mapRR.put("bossuser", ConstantResources.BOSSUSER);
                mapRR.put("id", tpOrderId);
                String servicetypeAndServiceflowid = hlLotteryService.queryServiceType(mapRR);

                NotifyYjfEntity notifyYjfEntity = new NotifyYjfEntity();
                notifyYjfEntity.setAmount(NumberHelper.div(totalMoney + "", 100 + "", 2));
                notifyYjfEntity.setContext("");
                notifyYjfEntity.setMerchOrderNo(tpOrderId);
                notifyYjfEntity.setTradeNo(orderId + "");
                notifyYjfEntity.setTradeType(payType + "");
                notifyYjfEntity.setTradeStatus("1");
                log.info("[/front/baiduMiniProgram][/payNotify]:" + JSON.toJSONString(notifyYjfEntity));
                //在订单处理中 为粉丝加积分
                    Result notifyResult = notifyYjfService.handleOrderAndBusiness(notifyYjfEntity);
                log.info("[/front/baiduMiniProgram][/payNotify]:" + JSON.toJSONString(notifyResult));
            } finally {
                //执行不管完成不完成一定要释放
                redisTemplate.delete(REDIS_KEY + orderId);
            }
            //执行成功,更新状态
            baiduNotifyDto.setHandleStatus(1);
            baiduNotifyService.updateNotifyMsg(baiduNotifyDto);
            //n.执行成功,这部分必须要等业务逻辑全部执行成功才会执行
            redisTemplate.opsForValue().set(REDIS_KEY + orderId, "consumption", 3, TimeUnit.MINUTES);
            ResSyncPayDTO success = new ResSyncPayDTO.Builder().setErrno(0).setMsg("SUCCESS").setIsConsumed(2).build();
            LOGGER.info("/front/baiduMiniProgram/payNotify 方法执行结束");
            return success;
        } catch (Exception e) {
            log.error("SyncPayController::syncPayInfo代码异常", e);
            //执行失败更新状态
            baiduNotifyDto.setHandleStatus(2);
            baiduNotifyService.updateNotifyMsg(baiduNotifyDto);
            return new ResSyncPayDTO.Builder().setErrno(500).setMsg(e.toString()).setIsErrorOrder(1).setIsConsumed(2).build();
        }
    }
~~~

还可以用数据库的unique key来确保幂等性,这种要结合具体的业务;

# 8.如何保证接口请求的顺序性

* 从业务逻辑上最好设计的这个系统不需要这种顺序性的保证,因为一旦引入顺序性保障,会导致系统复杂度上升,而且会带来效率低下,热点数据压力过大问题.

* 方案1:首先你需要用Dubbo的一致性Hash负载均衡策略,比如某一个订单id对应的请求都要给分发到某个机器上去,接着就是在那个机器上因为可能还是多线程并发执行的,可能得立即将某个订单id对应的请求扔到一个内存队列里去,强制排队.保证他们的顺序性.

* 方案2:分布式锁加上顺序标识如图

  ![image-20210720215921156](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720215921156.png)

* 引发的后续问题就很多,比如说要是某个订单对应的请求特别多,造成某台机器成为热点,解决这些问题又要开启后续一连串的复杂技术方案.

  