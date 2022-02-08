# DDD架构模式下的电商履约系统

# 1.电商履约的流程

1. 收到订单;
2. 生成履约单;
3. 预分仓;
4. 风控拦截:预防刷单,黄牛等;
5. 合并处理:在一段时间内同时下了很多单,看情况是否能够同一批次发货;
6. 人工审核;
7. 重分仓;
8. 分物流;
9. 下库房;
10. 生成拣货单;
11. 打印拣货单;
12. 人工拣货;
13. 复核打包;
14. 发货;
15. 物流揽件;
16. 物流运输;
17. 物流派件;
18. 物流签收;

## 1.1.预分仓和DDD引入

一般来说,预分仓的流程如下:

1. 寻找跟订单收获地址距离最近的仓库;
2. 检验该仓库的指定商品的库存,是否足够该订单发货;
3. 锁定该仓库中指定商品的对应数量的库存;
4. 将订单履约分配给该仓库,完成预分仓的过程;

如果我们用传统的面向数据库设计的日常编写的伪代码如下:

```java
public void allocateWarehouse(OrderFulFill orderFulFill){
  List<Warehouse> warehouses = warehouseDao.find();
  Warehouse selectedWarehouse = null;
  for(Warehouse warehouse : warehouses){
    //编写业务逻辑;
    seletedWarehouse = warehouse;
  }
  //检验库存,做数据库的查询操作,再做比较和检验;
  //锁定仓库里的库存;
  WarehouseInventory warehouseInventory = warehouseInventoryDao.find(seletedWarehouse);
  warehouseInventory.set(xxx);
  warehouseInventoryDao.update(warehouseInventory);
  //做订单跟仓库的匹配逻辑;
  orderAllcatedWarehouseDao.save(xxx);
}
```

如果我们用DDD实现伪代码如下:

```java
public void dddAllocateWarehouse(OrderFulFill orderFulFill){
  //获取所有的仓库;
  Warehouses warehouses = warehouseRepository.getAll();
  //搜寻最近的仓库;
  Warehouse warehouse = warehouses.searchNearest(orderFulFill);
  //检验库存;
  Boolean checkInventoryResult = warehouse.checkInventory(orderFulfill);
  //锁定库存
  warehouse.lockInventory(orderFulfill);
  //预先分仓;
  orderFulfill.allocateTO(warehouse);
}
```

# 2.战略设计

**电商系统可以看成一个领域,一个领域包含多个子域(有界上下文),**

战略设计包含:

* **有界上下文的划分**;(有界上下文跟子域一一对应起来)
* **上下文映射**:
  * `separate way`:完全没有关系;
  * `customer-supplier`:是一种上下游的关系.如果你有一些变化,我有一些需求,大家可以商量一下;
  * `publish-subscribe`:发布-订阅,有一方发布一个事件,另外一方监听事件以及处理这个事件;
  * `anti corruption layer`:acl,防腐层.不是一个孤立,单独使用的映射关系,一般是跟其他的映射关系一起进行使用;有可能字段的名称,类型,可选的值变化了.防腐层一般来说是把调用后的东西,做一个适配和转化;
  * `conformist`:u-d,对方提供接口,你要用,就直接调用,并且必须遵守他们的调用模式,完全没法商量;
  * `partnership`:强耦合,合作,适用于强而紧凑的系统;一般是在中小型的系统;
  * `open host service+publish language`:HOS+PL:对于第三方支付平台,第三方物流平台,SaaS云平台提供的API接口,开放平台接口;
  * `shared kernel`:两个有界上下文之间,共享一个数据模型,数据模型是两个团队一起维护,不常见;

* **子域类型的确定**;
  * 核心子域;
    * 对于一个电商领域而言,用于完成用户购物这个核心需求,所需要具备的一些核心子域,这就是属于核心子域.
    * 商品之域,订单子域,支付子域,履约子域,会员子域,营销子域都是属于核心子域;
  * 支持子域;
    * 负责支持核心子域,他的作用是属于支持别的核心子域的功能实现,属于执行性的子域;
    * 仓储子域,物流子域用于支持履约子域;财务子域,报表子域;
  * 通用子域;
    * 第三方的支付平台,第三方的物流平台;
* **通用语言的确定**:命名的规则和规范,都确定下来,这就是一套通用语言;

# 3.战术设计

战术设计,牵扯到了类层面的设计,设计你的上下文里有哪些类,这些类如何配合,可以实现上下文里要解决的各种各样的问题;

## 3.1.贫血模型与充血模型

* **贫血模型**:以前我们设计domain包里的class,一般都是贫血模型,因为这些class,都是一些数据模型,包含的仅仅是数据,方法只有getter和setter方法,没别的方法;
* **充血模型**:根据业务业务语义,放入对应的一些业务逻辑的行为;

## 3.2.实体.值对象,聚合

* **实体**:就是我们之前在项目中的Domain类,它有唯一的标识,并且它的数据是允许变化的;
* **值对象**:ValueObject,没有唯一的标识,并且数据是不允许变化的;
* **聚合**:Aggregate:多个Class绑定在一起就叫聚合如下
  * 一个聚合里,聚合根(aggregate root),Order就是聚合根;
  * 聚合是有边界概念的,聚合里面的classes生命周期是一致的

```java
public class Order{
  private Long orderId;
  //....省略Order自己的字段;
  
  List<OrderItem> orderItems;
  
  DeliveryAddress deliveryAddress;
}
```

## 3.3.领域服务与业务组件

* **领域服务**:有一些业务行为是无法放在聚合,或者实体,或者值对象中的.比如Order,如果说要对多个Order一起进行运算的业务逻辑,是不可能放在Order里面去设计的,这时候就会有**领域服务**;

```java
public class XXXService{
  //包含了一些没法直接放在聚合里的业务行为,针对多个聚合执行的业务行为;
}
```

* **业务组件**:有一些业务语义放在**业务组件**中,比如订单状态机,它不是聚合,不是领域服务,其实是一个业务组件;

```java
public class OrderStateMachine{
  //贴合业务语义的业务组件;
}
```

## 3.4.命令和领域事件

* **命令**:Command一般来说对应的都是更新数据一类的动作,如果要进行一些查询的动作,一般叫做Query;

```java
public class FulfillService{
  
  //这里的AuditOrderCommand就是命令
  public void auditOrder(AuditOrderCommand command){
    //集合聚合,业务组件,其他领域服务完成业务;
  }
}
```

* **领域事件**:Event,OrderPayedEvent(订单已支付的事件),驱动和发起履约流程,履约上下文,可以去订阅这个订单已经支付的事件,一旦订单被支付了,就可以开始进行履约;

```java
public class FulfillService{
  
  //领域事件,监听
  public OrderFulfill receivePayedOrder(OrderPayedEvent orderPayedEvent){
    //领域事件,要去接收订单,存储数据到自己履约上下文里面;
  }
  
  public void preAllocateWarehouse(OrderFulfill orderFulfill){
    //业务逻辑...
  }
}
```

## 3.5.仓储

**repository**:区别于DAO,如果我们底层要用上缓存,ES,NoSQL,MQ以及其他持久层的技术,都是封装到仓储的底层;

# 4.分层

## 4.1.接口层(表现层)

* 我们对外提供服务的系统,接口层是用来对接用户界面发送的一些命令,command或者query.对用户界面提供对应的http接口;
* 我们的有界上下文,是对其他的有界上下文提供映射和交互的,提供一个内部调用的接口,一般来说,内部的接口,都命名为XXXApi,通过 Spring Cloud Netflix(openfeign)或者是Spring Cloud Alibaba(dubbo)RPC框架,对外提供一套API接口;
* 别的有界上下文会发布一个领域事件,发布到MQ中,我们就需要去进行领域事件的监听,这个负责领域事件监听的减轻器也算是对外的接口层,是基于MQ的事件的消费接口;

当我们接口层收到一个Http请求,XXXApi收到了一个RPC调用,Listener收到了Event事件之后,此时每个请求(Command或者Query),RPC调用,Event事件,都对应了业务流程去执行,业务链路去执行;  

当接口层收到了东西(http请求,RPC调用请求,Event事件)做一些转化(防腐层,把外部传递进来的东西转化为内部的东西,比如说把Http Request转化为Command或者是Query),然后给到应用服务层.Application service layer;

## 4.2.应用服务层

应用服务层,就是用于进行业务流程编排,由ApplicationService来进行编排,由它来基于仓储,聚合,领域服务,执行各种的动作,把这个业务链路/链路里的各个环节和动作都完成;

## 4.3.domain层

聚合,领域服务,业务组件,仓储;
