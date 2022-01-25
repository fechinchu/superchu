#  SpringCloud01-SpringCloud Netflix

# 1.服务调用方式

## 1.1.RPC和HTTP

常见的远程调用方式有以下2种:

* RPC:(Remote Produce Call)远程过程调用,类似的还有RMI(Remote Method Invoke).自定义数据格式,基于原生 TCP通信,速度快,效率高.早期的webservice,现在热门的dubbo.都是RPC的典型代表
*  Http:Http是一种网络传输协议,基于TCP,规定了数据传输的格式.现在客户端浏览器与服务端通信的方式基本都是 采用http协议.也可以用来进行远程服务调用.缺点就是消息封装臃肿.优势是对服务的提供和调用方没有任何技术 限定,自由灵活,更符合微服务理念

## 1.2.HTTP客户端工具

Spring提供了一个RestTemplate模板工具类,对基于Http的客户端进行了封装,并且实现了对象与json的序列化和反 序列化.非常方便.RestTemplate并没有限定Http的客户端类型,而是进行了抽象.

首先在项目中注入一个RestTemplate对象,可以在启动类位置注册,在服务的消费者端直接 @Autowired 并使用即可.通 过RestTemplate的 getForObject() 方法,传递url地址及实体类的字节码,RestTemplate会自动发起请求,接收响应,并 且帮我们对响应结果进行反序列化.

# 2.Eureka

## 2.1.CAP原则

CAP原则又称CAP定理,指的是在一个分布式系统中,Consistency(一致性),Availability(可用性),Partition tolerance(分 区容错性),三者不可兼得.

Eureka遵循的的就是AP原则.

## 2.2.Eureka对比Zookeeper

作为服务注册中心,Eureka比Zookeeper好在哪里.

注明的CAP理论指出,一个分布式系统不可能同时满足C(一致性),A(可用性)和P(分区容错性).由于分区容错性P在是在分 布式系统中必须要保证的,因此我们只能在A和C之间进行权衡.

因此:Zookeeper保证的是CP,Eureka是AP

### 2.2.1.Zookeeper保证CP

在Zookeeper中会出现这样一种情况,当master节点因为网络问题故障与其他节点失去联系时候,剩余节点会重新进行 leader选举.问题在于,选举leader的时间太长,30-120s,且选举期间整个zookeeper集群都是不可用的.这就导致在选择期间注册服务瘫痪.在云部署的环境下,因网络问题使得zookeeper集群失去master节点是较大概率会发生的事.虽然 服务能够最终恢复,但是漫长的选举时间导致的注册长期不可用是不能容忍的.

### 2.2.2.Eureka保证AP

Eureka各个节点都是平等的,几个节点挂掉不会影响正常节点的工作,剩余节点依然可以提供注册和查询服务.而 Eureka的客户端在向某个Eureka注册或如果发现连接失败,则会自动切换至其他节点,只要一台Eureka还在,就能保证 注册服务可用(保证可用性),只不过查到的信息可能不是最新的(不保证强一致性).除此之外,Eureka还有一种自我保护 机制.如果在15分钟内超过85%的节点都没有正常的心跳,那么Eureka就认为客户端与注册中心出现了网络故障,此时会出现以下几种情况.

1. Eureka不再从注册列表中移除因为长时间没有收到心跳而应该过期的服务

2. Eureka仍然能够接受新服务的注册和查询请求,但是不会被同步到其他节点上(即保证当前节点依然可用).

3. 当网络稳定时,当前实例新的注册信息会被同步到其它节点中.

因此,Eureka可以很好的应对因网络故障导致部分节点失去联系的情况,而不会像Zookeeper那样使整个注册服务瘫痪.

### 2.2.3.服务注册发现的时效性

* Zookeeper:时效性更好,注册或者是挂了,由于watch监听机制,一般秒级就能感知;
* Eureka:默认配置非常糟糕,由于缓存的存在,服务发现感知一般需要几十秒,甚至分钟级别;

## 2.3.Eureka原理

![image-20210726165221994](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726165221994.png)

* 处于不同节点的eureka通过replicate(复制)进行数据同步;
*  Application Sservice为服务提供者;
*  Application Client为服务消费者;
*  Make Remote Call完成一次服务调用;

当服务启动后向Eureka注册,Eureka Server会将注册信息向其他Eureka Server进行同步,当服务消费者要调用服务提供者,则向服务注册中心获取服务提供者地址,然后会将服务提供者地址缓存在本地,下次再调用时,则直接从本地缓存中 取,完成一次调用.

当服务注册中新Eureka Server检测到服务者因为宕机,网络原因不可用时,则在服务注册中心将服务置为DOWN状态, 并把当前服务提供者向订阅者发布,订阅过的服务消费者更新本地缓存.

服务提供者在启动后,周期性(默认30s)向Eureka Server发送心跳,以证明当前服务是可用状态.Eureka Server在一定时 间(默认90s)未收到客户端的心跳,则认为服务宕机,注销该实例.

## 2.4.Eureka注册中心程序

~~~xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
~~~

~~~yaml
spring:
  profiles:
    active: singleTon
  application:
		name: eureka-server # application name是体现在注册的instance名 # eureka:
# server:
# enable-self-preservation: false 
# 关闭自我保护模式(缺省为打开)
---
# 单机版 
server:
  port: 10086
spring:
  profiles: singleTon
eureka:
  instance:
    hostname: host-eureka-server01
  client:
		register-with-eureka: false #单机版是不能够自己注册自己 
		fetch-registry: false #单机版不能够自己拉取自己 
		service-url:
			dafaultZone: http://${eureka.instance.hostname}:${server.port}/eureka #这是eureka服务 地址
---
# 集群节点1 
spring:
  profiles: replicas01
eureka:
	instance:
		hostname: eureka01 # 当前节点域名名称,体现在DS-replicas中的名字
	client:
    service-url:
      defaultZone: http://eureka02:10088/eureka, http://eureka03:10089/eureka
server:
  port: 10087
---
# 集群节点2 
spring:
  profiles: replicas02
eureka:
  instance:
    hostname: eureka02
  client:
    service-url:
      defaultZone: http://eureka01:10087/eureka, http://eureka03:10089/eureka
server:
  port: 10088
---
# 集群节点3 
spring:
  profiles: replicas03
eureka:
  instance:
    hostname: eureka03
  client:
    service-url:
      defaultZone: http://eureka01:10087/eureka, http://eureka02:10088/eureka
server:
	port: 10089
~~~

多个Eureka Server之间会互相注册为服务,当服务提供者注册到Eureka Server集群中的某个节点时,该节点会把服务 的信息同步给集群中的每个节点,从而实现高可用集群.因此,无论客户端访问到Eureka Server集群中的任意一个节点, 都可以获取到完整的服务列表信息.

之后再主启动类上添加@EnableEurekaServer,并启动该微服务.

## 2.5.服务生产者程序

### 2.5.1.服务注册

~~~xml
<!--eureka客户端--> 
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
~~~

~~~yaml
server:
  port: 8081
spring:
  application:
		name: item-service
	datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/leyou?characterEncoding=utf-8
    username: root
    password: root
    
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10086/eureka
  instance:
    prefer-ip-address: true
    ip-address: 127.0.0.1
    
mybatis:
	mapper-locations: mappers/*.xml #指定mybatis的mapper文件所放置位置 
	type-aliases-package: com.leyou.item.pojo #实体类所在的包 
	configuration:
		map-underscore-to-camel-case: true #开启驼峰命名 
logging:
  level:
    com.leyou: debug
~~~

添加一个`spring.application.name`属性来指定应用名称,将来会作为服务的id进行使用;

~~~java
@SpringBootApplication
@EnableEurekaClient
@MapperScan("com.leyou.item.mapper")
public class LyItemApplication {
    public static void main(String[] args) {
        SpringApplication.run(LyItemApplication.class,args);
} }
~~~

`@EnableEurekaClient`也可以使用`@EnableDiscoveryClient`,不过`@EnableDiscoveryClient`的使用范围更广,而 `@EnableEurekaClient`只使用于Eureka注册中心.

服务提供者在启动时,会检测配置属性中的: `eureka.client.register-with-erueka=true` 参数是否为true,默认事实 上就是true,如果值确实为true,则会向EurekaServer发起一个Rest请求,并携带自己的元数据.Eureka Server会把这些 信息保存到一个双层Map结构Map结构中.

* 第一层Map的Key就是服务id,一般是配置中的 spring.application.name 属性 
* 第二层Map的Key是服务的实例id.一般host+serviceId+port,例如: localhost:user-service:8081 
* 值则是服务的实例对象,也就是所一个服务,可以同时启动多个不同实例,形成集群.

### 2.5.2.服务续约

在注册服务完成以后,服务提供者维持一个心跳(定时向EurekaServer发起Rest请求),告诉EurekaServer"我还活着".这个我们称为服务的续约(renewal).

~~~yaml
eureka:
	instance:
  	lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
~~~

* lease-renewal-interval-in-seconds: 服务续约(renew)的间隔,默认为30s
*  lease-expiration-duration-in-seconds:服务失效时间,默认值90s

默认情况下每个30s服务会向注册中心发送一次心跳,证明自己还活着.如果超过90s没有发送心跳.EurekaServer就会 认为该服务宕机,会从服务列表中移除,这两个值在生产环境不要修改,默认即可.

### 2.5.3.获取服务列表

当服务消费者启动时,会检测`eureka.client.fetch-registry=true` 参数的值,如果为true,则会从Eureka Server服务的列表只读备份,然后缓存在本地.并且每隔30s会重新获取并更新数据,我们可以通过下面的参数来修改.

~~~yaml
eureka:
	client:
    registry-fetch-interval-seconds: 30
~~~

## 2.6.服务消费者程序

服务消费者同样也需要引入eureka-client的坐标

~~~yaml
server:
  port: 19001
spring:
  application:
		name: item-consumer #应用名称 
eureka:
  client:
    service-url:
			defaultZone: http://localhost:10086/eureka
		# register-with-eureka: false #消费者并不需要注册进eureka,当然也可以注册金eureka
~~~

在编写消费者的时候,我们可以直接通过RestTemplate去调用user-service服务

这个时候会抛出异常:java.net.UnknownHostException: USER-SERVICE,想要通过服务名称对服务进行调用的话, 我们需要对服务的消费端实现负载均衡.才能通过user-service进行对服务的调用.

~~~java
private final static String PREFIX_URL = "http://USER-SERVICE/user";

@Autowired
private RestTemplate restTemplate;
@GetMapping(path = "/findAll",name = "查找所有") public String findAll() {
    List userList = restTemplate.getForObject(PREFIX_URL + "/findAll", List.class);
    return userList.toString();
}
~~~

## 2.7.服务下线,失效剔除和自我保护

### 2.7.1.服务下线

当服务进行正常关闭操作时候,它会触发一个服务下线的REST请求给Eureka Server,告诉服务注册中性:"我要下线了". 服务中心接受到请求之后,将该服务置为下线状态.

### 2.7.2.失效剔除

有时我们的服务可能由于种种原因使服务不能正常工作,而服务注册中心并未受到"服务下线"的请求,相对于服务提供 者的"服务续约"操作,服务注册中心在启动时会创建一个定时任务,默认每隔一端时间(默认60s)将当前清单中超时(默认 为90s)没有续约的服务剔除,这个操作成为失效剔除.

我们通过` eureka.server.eviction-interval-timer-in-ms `参数对其进行修改,单位时毫秒.

### 2.7.3.自我保护

![image-20210726172735472](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726172735472.png)

我们关停一个服务.就会在Eureka面板看到

这是触发了Eureka的自我保护机制,当服务未按时进行心跳续约时,Eureka会统计服务实例最新15分钟心跳续约的比例 是否低于85%,在生产环境下,因为网络延迟等原因,心跳失败实例的比例很有可能超标.但时此时就把服务剔除列表并不 妥当,因为服务可能没有宕机.Eureka在这段时间内不会剔除任何服务实例,知道网络恢复正常,生产环境下很有效,保证 了大多数服务依然可用,不过也有可能获取到失败的服务实例,因此服务调用者必须做好服务的失败容错.

~~~yaml
eureka:
  server:
		enable-self-preservation: false # 关闭自我保护模式(缺省为打开)
~~~

## 2.8.Eureka的缓存

![image-20210729002109970](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210729002109970.png)

Eureka一共有3层缓存,第一层为只读缓存,第二层为读写缓存,第三层为服务注册表缓存.

为什么这么设计?Eureka这样设计为了读写分离.线程在写的时候,并不会影响读操作,避免了争抢资源带来的压力;

![image-20210728231911981](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210728231911981.png)

如下是Eureka的源码,这些缓存的本质还是ConcurrentHashMap,如果对ConcurrentHashMap进行高并发的读写的话,会频繁产生锁,多级缓存正式为了解决这样的问题;

## 2.9.Eureka服务发现剔除过慢问题

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210729002208590.png" alt="image-20210729002208590" style="zoom:50%;" />

上图是Eureka的默认的配置.说明如下:

* responseCacheUpdateIntervalMs:ReadWrite缓存和ReadOnly缓存之间定时同步的时间;
* registryFetchIntervalSeconds:消费者定时拉取ReadOnly缓存的时间;
* leaseRenewalIntervalInSeconds:服务生产者发送心跳的时间;
* evictionIntervalTimerInMs:后台线程定时检查心跳的时间;
* LeaseExpirationDurationInSeconds:超过多少时间没有发送心跳就剔除;

所以我们可以在生产环境进行将时间缩小进行优化;

# 3.Ribbon

在Eureka中已经帮我们集成了负载均衡组件Ribbon,所以我们无需引入新的依赖,直接修改代码即可.

~~~java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
~~~

这个时候,我们就可以直接通过微服务的service-id去调用服务了.对于集群环境默认使用的就是轮询策略. `@LoadBalance`这个注解源码的注释中我们可以看到该注解会把RestTemplate配置使用一个负载均衡的客户端.

![image-20210726173015744](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726173015744.png)

![image-20210726173033522](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726173033522.png)

我们可以知道IRule该接口下的所有的实现类,这些类都是相应的负债均衡的执行规则,默认是采用RoundRobinRule,即轮询.如果我们想要使用别的负载均衡规则,我们可以直接往Spring容器中去添加该组件即可.

~~~java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
@Bean 
public IRule randomRule(){
    return new RandomRule();
}
~~~

## 3.1.项目刚发布Timeout超时优化

每个服务第一次被请求的时候,他会去初始化Ribbon的组件,初始化这些组件需要耗费一定时间,很容易会导致超时.可以让每个服务启动时候就初始化Ribbon相关组件,避免第一次请求的时候初始化;

~~~yaml
# ribbon配置
ribbon:
    eager-load:
        enabled: true
        clients: user-service #指定服务名称

# zuul配置
zuul:
    ribbon:
        eager-load:
            enabled: true
~~~





