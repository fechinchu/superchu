# SpringCloud02-SpringCloud Netflix

# 1.Hystrix

## 1.1.Hystrix的功能

* 资源隔离:让系统中的某一块地方,在故障的情况下,不会耗尽系统所有的资源,比如线程资源;'
* 限流:高并发的流量涌入进来,比如说突然间一秒钟100万QPS,10万QPS进入系统,其他90QPS被拒绝;
* 熔断:比如系统的一些地方出了故障,比如MySQL挂了,每次请求都报错,熔断,后续的请求过来直接不接收.决绝访问,10分之后再尝试看看恢复没有;
* 降级:有限保证核心服务,非核心服务不可用或弱可用;

* 运维监控:监控+报警+优化,各种异常的情况,有问题及时报警;

## 1.2.Hystrix的设计原则

* 阻止任何一个依赖服务耗尽所有的资源，比如tomcat中的所有线程资源;

* 避免请求排队和积压，采用限流和fail fast来控制故障;
* 提供fallback降级机制来应对故障;
* 使用资源隔离技术，比如bulkhead（舱壁隔离技术），swimlane（泳道技术），circuit breaker（短路技术），来限制任何一个依赖服务的故障的影响;

* 通过近实时的统计/监控/报警功能，来提高故障发现的速度;
* 通过近实时的属性和配置热修改功能，来提高故障处理和恢复的速度;
* 保护依赖服务调用的所有故障情况，而不仅仅只是网络故障情况;



* 通过HystrixCommand或者HystrixObservableCommand来封装对外部依赖的访问请求，这个访问请求一般会运行在独立的线程中，资源隔离;
* 对于超出我们设定阈值的服务调用，直接进行超时，不允许其耗费过长时间阻塞住。这个超时时间默认是99.5%的访问时间，但是一般我们可以自己设置一下;
* 为每一个依赖服务维护一个独立的线程池，或者是semaphore，当线程池已满时，直接拒绝对这个服务的调用;
* 对依赖服务的调用的成功次数，失败次数，拒绝次数，超时次数，进行统计;
* 如果对一个依赖服务的调用失败次数超过了一定的阈值，自动进行熔断，在一定时间内对该服务的调用直接降级，一段时间后再自动尝试恢复;
* 当一个服务调用出现失败，被拒绝，超时，短路等异常情况时，自动调用fallback降级机制;
* 对属性和配置的修改提供近实时的支持;

 ## 1.3.故障蔓延原理

![image-20210726220502640](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726220502640.png)

## 1.4.电商网站的详情页架构

### 1.4.1.小型电商网站的详情页

![image-20210727095959892](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210727095959892.png)

### 1.4.2.大型电商网站的详情页

![image-20210727100049746](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210727100049746.png)

## 1.5.资源隔离

### 1.5.1.线程池隔离技术

Hystrix进行资源隔离,其实提供了一个抽象叫做command,就是说:如果要把对某个依赖服务的所有调用请求,全部隔离在同一份资源池内,对这个依赖服务的所有调用请求,全部走这个资源池内的资源,不会去调用其余资源了,这个就叫做资源隔离;

线程池隔离技术,并不是去控制类似Tomcat这种web容器的线程,更加严格意义上来说,Hystrix的线程池隔离技术,控制的是Tomcat线程的执行.线程池满后,确保的是Tomcat的线程不会因为依赖的服务的接口调用延迟或者故障被阻塞,它会快速执行fallback,快速返回;

#### 1.5.1.1.示例代码

![image-20210727174658782](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210727174658782.png)

Hystrix用上述代码的`GetProductInfoCommand`,名称对应的线程池去隔离该请求,之后就可以改造接口如下

![image-20210727174759715](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210727174759715.png)

上述使用`HystrixCommand`是一次返回一条数据,我们也可以使用`HystrixObservableCommand`一次返回多条数据.

### 1.5.2.信号量隔离技术

#### 1.5.2.1.线程池隔离技术和信号量隔离技术区别

线程池隔离技术,使用线程池的线程去执行调用的;信号量的隔离技术,是直接让tomcat的线程去调用的;

* 线程池:适合大多数的场景.对依赖服务的网络请求的调用和访问的请求;
* 信号量:适合不是对外部依赖的访问,而是对内部一些比较复杂的业务逻辑的访问,避免内部复杂的低效率代码,导致大量的线程被卡住;

#### 1.5.2.2.示例代码

![image-20210727195253343](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210727195253343.png)

## 1.6.服务降级

服务器支持的线程和并发数有限,请求一直阻塞,会导致服务器资源耗尽,从而导致其它服务都不可用,形成雪崩效应.

Hystrix为每个依赖服务调用分配一个小的线程池,如果线程池已满调用将被立即拒绝,默认不采用排队,加速失败判定时间.用户的请求将不再直接访问服务,而是通过线程池中的空闲线程来访问服务,如果线程池已满或者请求超时,则会进行降级处理.

1. 添加依赖 

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
~~~

2. 在启动类上添加注解:@EnableCircuitBreaker或者使用@EnableHystrix,或者在启动类上添加 @SpringCloudApplication的注解,@SpringCloudApplication注解中组合了@EnableCiruitBreaker注解.

   ![image-20210726173406258](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726173406258.png)

3. 最后是编写降级逻辑,我们通过HystixCommond来完成.

~~~java
@HystrixCommand(fallbackMethod = "fallbackMethod01") @GetMapping(path = "/findAll",name = "查找所有")
public String findAll() {
    List userList = restTemplate.getForObject(PREFIX_URL + "/findAll", List.class);
    return userList.toString();
}

public String fallbackMethod01(){ 
    return "对不起,网络繁忙";
}
~~~

要注意,因为降级逻辑方法必须与正常逻辑方法保证:相同的参数和返回值声明.是失败逻辑中返回User对象没有太大意 义.一般会返回友好提示.

把fallback写在了某个业务方法上,如果这样的方法很多,那就要写很多,所以我们可以把Fallback配置加载类上,实现默认fallback.

~~~java
@RestController
@RequestMapping("/user/consumer")
@DefaultProperties(defaultFallback = "defaultFallBack")
public class UserServiceConsumer {
  
    private final static String PREFIX_URL = "http://USER-SERVICE/user";
  
    @Autowired
    private RestTemplate restTemplate;
  
    @Autowired
    private UserClient userClient;
  
	 @HystrixCommand
	 @GetMapping(path = "/findAll", name = "查找所有") 
    public String findAll() {
        List userList = restTemplate.getForObject(PREFIX_URL + "/findAll", List.class);
        return userList.toString();
    }
  	
	 public String defaultFallBack() { 
     	return "默认提示:网络繁忙";
	 } 
}
~~~

请求在超过1s后都会返回错误信息,这是因为Hystix的默认超时时长为1,我们可以通过配置修改这个值.

~~~yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 1500
~~~

## 1.7.服务熔断

![image-20210726174139482](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726174139482.png)

状态机有3个状态

* Closed:关闭状态(熔断器关闭),所有请求都能正常访问;
* Open:打开状态(熔断器打开),所有请求都会被降级.Hystrix会对请求计数,当一定时间内失败请求百分比达到阀值, 则触发熔断.默认失败比例的阀值是50%,请求次数最少不低于20次.
* Half Open:半开状态,open状态不是永久的,打开后会进入休眠时间(默认5s).随后熔断器会自动进入半开状态,此 时会释放1次请求通过,若这个请求时健康的,则会关闭断路器,否则继续保持打开,再进行5秒休眠计时.

~~~yaml
circuitBreaker:
  requestVolumeThreshold: 10
  sleepWindowInMilliseconds: 10000
  errorThresholdPercentage: 50
~~~

* requestVolumeThreshold:触发熔断的最小请求次数;
* 默认20 errorThresholdPercentage:触发熔断的失败请求最小占比,默认50% 
* sleepWindowInMilliseconds:休眠时长，默认是5000毫秒

## 1.8.Hystrix整体流程

![image-20210728143814075](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210728143814075.png)

# 2.Feign

1. 导入依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
~~~

2. 编写Feign接口
   1. 首先这是一个接口,Feign会通过动态代理,帮我们生成实现类 
   2. @FeignClient,声明这是一个Feign客户端,同时通过value属性指定服务名称
   3. 接口中的定义方法,完全采用SrpingMVC的注解,Feign会根据注解帮我们生成URL,并访问获取结果改造原来的调用逻辑

~~~java
@FeignClient(value = "USER-SERVICE",fallback = UserClientFallback.class)
@RequestMapping("/user")
public interface UserClient {
   //这是服务的提供者暴露的RestAPI 
   @GetMapping(path = "/find/{id}")
	User findById(@PathVariable("id") String id);
}
~~~

3. 使用Feign接口

~~~java
@Autowired
private UserClient userClient;

@RequestMapping(path = "/query/{id}")
public String findByIdOnfeign(@PathVariable("id") String id){
    return userClient.findById(id).toString();
}
~~~

4. 开启Feign功能

~~~java
@SpringCloudApplication
@EnableFeignClients
public class UserServiceConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceConsumerApplication.class,args);
	 } 
}
~~~

## 2.1.Ribbon的支持

Feign中本身已经集成了Ribbon依赖和自动配置:我们不需要再注册RestTemplate对象

Fegin内置的ribbon默认设置了请求超时时长,默认是1000ms,我们可以通过手动配置来修改这个超时时长.

~~~yaml
ribbon:
	ReadTimeout: 2000 # 读取超时时长 ConnectTimeout: 1000 # 建立链接的超时时长
~~~

因为Ribbon内部有重试机制,一旦超时,会自动重新发起请求,如果不希望重试,可以添加配置

~~~yaml
ribbon:
	ConnectTimeout: 500 # 连接超时时长
	ReadTimeout: 2000 # 数据通信超时时长
	MaxAutoRetries: 0 # 当前服务器的重试次数 
	MaxAutoRetriesNextServer: 1 # 重试多少其余服务
	OkToRetryOnAllOperations: false # 是否对所有的请求方式都重试
~~~

## 2.2.Hystrix的支持

Feign默认对Hystrix也有集成.只不过我们需要下面的参数来开启

~~~yaml
feign:
  hystrix:
    enabled: true
~~~

1. 首先,我们需要定义一个类,实现刚才编写的Client接口,作为fallback的处理类

~~~java
@Component
@RequestMapping("fallback/") //这个可以防止容器中有与父类重复的requestMapping
public class UserClientFallback implements UserClient {
    @Override
    public User findById(String id) {
      User user = new User(); 
      user.setName("用户查询异常"); 
      return user;
	 } 
}
~~~

2. 在client接口中指定刚才编写的实现类

~~~java
@FeignClient(value = "USER-SERVICE",fallback = UserClientFallback.class)
@RequestMapping("/user")
public interface UserClient {
    @GetMapping(path = "/find/{id}")
    User findById(@PathVariable("id") String id);
}
~~~

## 5.3.请求压缩

SpringCloud Feign支持对请求和响应进行GZIP压缩,以减少通信过程中的性能损耗

~~~yaml
feign:
  compression:
		request:
			enabled: true # 开启请求压缩
		response:
			enabled: true # 开启响应压缩
~~~

同时我们也可以对请求的数据类型,以及触发压缩的大小下限进行设置

~~~yaml
feign:
  compression:
		request:
			enabled: true # 开启请求压缩
			mime-types: text/html,application/xml,application/json # 设置压缩的数据类型 
			min-request-size: 2048 # 设置触发压缩的大小下限
~~~

# 3.Zuul

Zuul是Netflix的微服务网关,Zuul的核心就是一系列的过滤器

1. 动态路由:新开发某个服务,动态把请求路径和服务的映射关系加载到网关中去,服务增减机器,网关自动感知;
2. 灰度发布;
3. 授权认证;
4. 限流熔断;
5. 性能监控:每个API接口的耗时,成功率,QPS;
6. 系统日志;
7. 数据缓存;

![image-20210726175658389](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726175658389.png)

## 3.1.动态路由

不管是来自于客户端的请求,还是服务内部的调用,一切对方服务的请求都会经过Zuul这个网关,然后再由网关来实现鉴 权,动态路由等等操作.Zuul就是我们服务的同一入口.

![image-20210726175730161](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726175730161.png)

1. 添加依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
~~~

2. 编写启动类

~~~java
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
public class ZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class,args);
	 } 
}
~~~

3. 编写yml配置

~~~yaml
server:
  port: 10010
spring:
  application:
    name: api-gateway
eureka:
  client:
  	service-url:
       defaultZone: http://eureka01:10086/eureka
zuul:
	prefix: /api
	routes: #路由映射:请求url与目标微服务
     user-service:
       path: /u/**
       serviceId: user-service
     user-consumer:
      path: /c/**
      serviceId: user-consumer
~~~

4. 上述的关于user-service的配置可以简化为一条

~~~yaml
zuul:
  routes:
		user-service: /user-service/** # 这里是映射路径
~~~

5. 默认的路由规则:默认情况下:一切服务的映射路径就是服务名本身. 例如服务名为: `user-service` ，则默认的映射路径就是: `/user-service/**` 如果想要禁用某个路由规则,可以这样:

~~~yaml
zuul:
  ignored-services:
    - user-service
    - user-consumer
~~~

6. 我们可以通过`zuul.prefix=/api`来指定路由前缀,这样在发起请求时,路径就要以/api开头.

> 在实际的生产环境中,如果需要修改Zuul的路由的话,不能停机修改再上线,我们对Zuul进行二次改造:
>
> * 将路由的关系配置在MySQL中,并提供Gateway路由的增删改查;
> * Zuul中定时去拉取MySQL中的配置,并进行更新路由;

## 3.2.过滤器

ZuulFilter是过滤器的顶级父类

~~~java
public abstract ZuulFilter implements IZuulFilter{
  
    abstract public String filterType();
  
	  abstract public int filterOrder();
	
		boolean shouldFilter();// 来自IZuulFilter
  
   Object run() throws ZuulException;// IZuulFilter
}
~~~

* shouldFilter :返回一个 Boolean 值，判断该过滤器是否需要执行。返回true执行，返回false不执行。
*  run :过滤器的具体业务逻辑。
* filterType :返回字符串，代表过滤器的类型。包含以下4种:
  * pre :请求在被路由之前执行
  * route:在路由请求时调用
  * post :在routing和errror过滤器之后调用 
  * error :处理请求时发生错误调用
* filterorder:通过返回的int值来定义过滤器的执行顺序,数字越小优先级越高

![image-20210726180711776](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210726180711776.png)

* 正常流程:
  * 请求到达首先会经过pre类型过滤器，而后到达routing类型，进行路由，请求就到达真正的服务提供者，执行请求，返回结果后，会到达post过滤器。而后返回响应。
*  异常流程:
  * 整个过程中，pre或者routing过滤器出现异常，都会直接进入error过滤器，再error处理完毕后，会将请求交给post过滤器，最后返回给用户。
  *  如果是error过滤器自己出现异常，最终也会进入post过滤器，而后返回。
  *  如果是post过滤器出现异常，会跳转到error过滤器，但是与pre和routing不同的时，请求不会再到达 post过滤器了。

### 3.2.1.定义过滤器类

~~~java
@Slf4j
@Component
public class AuthFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }
    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER;
    }
    @Override
    public boolean shouldFilter() {
        return true;
    }
    @Override
    public Object run() throws ZuulException {
			log.info("经过过滤器");
			// 获取请求上下文
			RequestContext ctx = RequestContext.getCurrentContext(); 
      // 获取request对象
			HttpServletRequest request = ctx.getRequest();
			// 获取请求参数
			String token = request.getParameter("access-token");
			// 判断是否存在
			if(StringUtils.isEmpty(token)){
					// 如果验证失败,将请求拦截,返回错误提示信息 
        	ctx.setSendZuulResponse(false);
					// 设置返回状态码 
        	ctx.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
			}
      return null;
    }
}
~~~

## 3.3.负载均衡和熔断

Zuul中默认就已经集成了Ribbon负载均衡和Hystix熔断机制。但是所有的超时策略都是走的默认值，比如熔断超时 时间只有1S，很容易就触发了。因此建议我们手动进行配置:

~~~yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 6000
ribbon:
  ConnectTimeout: 1000
  ReadTimeout: 2000
  MaxAutoRetries: 0
  MaxAutoRetriesNextServer: 1
~~~

## 3.4.基于网关实现灰度发布

我们可以基于Gateway实现灰度发布;我们可以做一个后台管理系统,针对哪些要需要灰度发布,哪些不需要灰度发布.针对需要进行灰度发布的,可以设置LoadBalance权重,新发布的设置少一些权重,旧版本的设置多一些权重;等运行一段时间,没问题后,再全部部署成新版本;







