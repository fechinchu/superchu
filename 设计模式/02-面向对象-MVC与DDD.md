# 02-面向对象-MVC与DDD

# 1.什么是基于贫血模型的传统开发模式?

现在很多Web或者App项目都是前后端分离的,后端负责暴露接口给前端调用.这种情况下,一般将后端项目分为Repository层,Service层,Controller层.其中,Repository是负责数据访问,Service负责业务逻辑,Controller负责暴露接口.目前几乎所有的业务后端系统,都是基于贫血模型的.

~~~java
////////// Controller+VO(View Object) //////////
public class UserController {
  private UserService userService; //通过构造函数或者IOC框架注入
  
  public UserVo getUserById(Long userId) {
    UserBo userBo = userService.getUserById(userId);
    UserVo userVo = [...convert userBo to userVo...];
    return userVo;
  }
}

public class UserVo {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
}

////////// Service+BO(Business Object) //////////
public class UserService {
  private UserRepository userRepository; //通过构造函数或者IOC框架注入
  
  public UserBo getUserById(Long userId) {
    UserEntity userEntity = userRepository.getUserById(userId);
    UserBo userBo = [...convert userEntity to userBo...];
    return userBo;
  }
}

public class UserBo {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
}

////////// Repository+Entity //////////
public class UserRepository {
  public UserEntity getUserById(Long userId) { //... }
}

public class UserEntity {//省略其他属性、get/set/construct方法
  private Long id;
  private String name;
  private String cellphone;
}
~~~

平时开发Web后端项目的时候,基本上都是这样组织代码.其中UserEntity和UserRepository组成了数据访问层;UserBo和UserService组成了业务逻辑层,UserVo和UserController在这里属于接口层;

从代码中,可以发现UserBo是一个纯粹的数据结构,只包含数据结构,不包含任何业务逻辑.业务逻辑集中在UserService中.我们通过UserService来操作UserBo.换句话说,Service层的数据和业务逻辑,被分割为BO和Serice两个类中.像UserBo这样质保函数据,不包含业务逻辑的类,叫做贫血模型.同样,UserEntity,UserVo都是基于贫血模型设计的.这种贫血模型将数据与操作分离,破坏了面向对象的封装特性,是一种典型的面向过程的编程风格;

# 2.什么是基于充血模型的DDD开发模式?

在贫血模型中,数据和业务逻辑被分割到不同的类中.充血模型正好相反,数据和对应的业务逻辑被封装到同一个类中.这种充血模型满足面向对象的封装特性.

领域驱动设计,即DDD,主要用来指导如何解耦业务系统,划分业务模块,定义业务领域模型及其交互.微服务有一个很重要的工作,就是针对公司的业务,合理地做微服务拆分.而领域驱动设计恰好就是用来指导划分服务的.

基于充血模型的DDD开发模式实现的代码也是按照MVC三层架构分层的,Controller成还是负责暴露接口,Repository层还是负责数据存取,Service层负责核心业务逻辑,它与基于贫血模型的传统开发模式的区别在于Service层;

* 在基于贫血模型的传统开发模式中,Service层包含Service类和BO类两部分,BO是贫血模型,只包含数据,不包含具体的业务逻辑.业务逻辑集中在Service类中.
* 在基于充血模型的DDD开发模式中.Service层包含Service类和Domain类两部分.Domain就相当于贫血模型中的BO.不过,Domain与BO的区它是基于充血模型开发的,既包含数据也包含业务逻辑.而Service层这非常单薄.

总结一句话就是:**基于贫血模型的传统开发模式,重Service轻BO,基于充血模型的DDD开发模式,轻Service重Domain**;

# 3.如何基于充血模型的DDD开发一个虚拟钱包系统?

## 3.1.钱包业务背景

先限定钱包暂时只支持充值,提现,支付,查询余额,查询交易流水这五个核心功能.

* 充值:用户通过三方支付渠道,将自己银行卡账户的钱,充值到虚拟钱包账号中.整个过程中,可以分解为三个操作:
  * 1.从用户的银行卡账户转账到应用的公共银行卡账户;
  * 2.将用户的充值金额夹到虚拟钱包余额上;
  * 3.记录刚刚这笔交易流水;
* 支付:用户用钱包内的余额,支付购买应用内的商品.
  * 1.从用户的虚拟钱包账户花钱到商家的虚拟钱包账户上;
  * 2.记录这笔支付的交易流水;
* 体现:用户将虚拟钱包中的余额体现到自己的银行卡中;
  * 1.扣减用户虚拟钱包中的余额;
  * 2.触发真正的银行转账操作,从应用的公共银行账户转钱到银行账户;
  * 3.记录这笔提现的交易流水;
* 查询余额:查询余额功能就是查看虚拟钱包的余额;
* 查询交易流水:因为我们会记录相应的交易信息,在需要查询的时候,只需要将之前记录的交易流水,按照时间,类型等条件过滤之后,显示出来;

## 3.2.钱包系统的设计思路

把整个系统的业务划分为两个部分,一部分单纯跟应用内的虚拟钱包账户打交道,另一部分跟银行账户打交道.基于这样的业务划分,将整个系统拆分为:虚拟钱包系统和三方支付系统.

## 3.3.基于贫血模型的传统开发模式

如下是Controller层

~~~java
public class VirtualWalletController {
  // 通过构造函数或者IOC框架注入
  private VirtualWalletService virtualWalletService;
  
  public BigDecimal getBalance(Long walletId) { ... } //查询余额
  public void debit(Long walletId, BigDecimal amount) { ... } //出账
  public void credit(Long walletId, BigDecimal amount) { ... } //入账
  public void transfer(Long fromWalletId, Long toWalletId, BigDecimal amount) { ...} //转账
  //省略查询transaction的接口
}
~~~

如下是Service层和BO

~~~java

public class VirtualWalletBo {//省略getter/setter/constructor方法
  private Long id;
  private Long createTime;
  private BigDecimal balance;
}

public Enum TransactionType {
  DEBIT,
  CREDIT,
  TRANSFER;
}

public class VirtualWalletService {
  // 通过构造函数或者IOC框架注入
  private VirtualWalletRepository walletRepo;
  private VirtualWalletTransactionRepository transactionRepo;
  
  public VirtualWalletBo getVirtualWallet(Long walletId) {
    VirtualWalletEntity walletEntity = walletRepo.getWalletEntity(walletId);
    VirtualWalletBo walletBo = convert(walletEntity);
    return walletBo;
  }
  
  public BigDecimal getBalance(Long walletId) {
    return walletRepo.getBalance(walletId);
  }

  @Transactional
  public void debit(Long walletId, BigDecimal amount) {
    VirtualWalletEntity walletEntity = walletRepo.getWalletEntity(walletId);
    BigDecimal balance = walletEntity.getBalance();
    if (balance.compareTo(amount) < 0) {
      throw new NoSufficientBalanceException(...);
    }
    VirtualWalletTransactionEntity transactionEntity = new VirtualWalletTransactionEntity();
    transactionEntity.setAmount(amount);
    transactionEntity.setCreateTime(System.currentTimeMillis());
    transactionEntity.setType(TransactionType.DEBIT);
    transactionEntity.setFromWalletId(walletId);
    transactionRepo.saveTransaction(transactionEntity);
    walletRepo.updateBalance(walletId, balance.subtract(amount));
  }

  @Transactional
  public void credit(Long walletId, BigDecimal amount) {
    VirtualWalletTransactionEntity transactionEntity = new VirtualWalletTransactionEntity();
    transactionEntity.setAmount(amount);
    transactionEntity.setCreateTime(System.currentTimeMillis());
    transactionEntity.setType(TransactionType.CREDIT);
    transactionEntity.setFromWalletId(walletId);
    transactionRepo.saveTransaction(transactionEntity);
    VirtualWalletEntity walletEntity = walletRepo.getWalletEntity(walletId);
    BigDecimal balance = walletEntity.getBalance();
    walletRepo.updateBalance(walletId, balance.add(amount));
  }

  @Transactional
  public void transfer(Long fromWalletId, Long toWalletId, BigDecimal amount) {
    VirtualWalletTransactionEntity transactionEntity = new VirtualWalletTransactionEntity();
    transactionEntity.setAmount(amount);
    transactionEntity.setCreateTime(System.currentTimeMillis());
    transactionEntity.setType(TransactionType.TRANSFER);
    transactionEntity.setFromWalletId(fromWalletId);
    transactionEntity.setToWalletId(toWalletId);
    transactionRepo.saveTransaction(transactionEntity);
    debit(fromWalletId, amount);
    credit(toWalletId, amount);
  }
}
~~~

## 3.4.基于充血模型的DDD开发模式

~~~java

public class VirtualWallet { // Domain领域模型(充血模型)
  private Long id;
  private Long createTime = System.currentTimeMillis();;
  private BigDecimal balance = BigDecimal.ZERO;
  
  public VirtualWallet(Long preAllocatedId) {
    this.id = preAllocatedId;
  }
  
  public BigDecimal balance() {
    return this.balance;
  }
  
  public void debit(BigDecimal amount) {
    if (this.balance.compareTo(amount) < 0) {
      throw new InsufficientBalanceException(...);
    }
    this.balance = this.balance.subtract(amount);
  }
  
  public void credit(BigDecimal amount) {
    if (amount.compareTo(BigDecimal.ZERO) < 0) {
      throw new InvalidAmountException(...);
    }
    this.balance = this.balance.add(amount);
  }
}

public class VirtualWalletService {
  // 通过构造函数或者IOC框架注入
  private VirtualWalletRepository walletRepo;
  private VirtualWalletTransactionRepository transactionRepo;
  
  public VirtualWallet getVirtualWallet(Long walletId) {
    VirtualWalletEntity walletEntity = walletRepo.getWalletEntity(walletId);
    VirtualWallet wallet = convert(walletEntity);
    return wallet;
  }
  
  public BigDecimal getBalance(Long walletId) {
    return walletRepo.getBalance(walletId);
  }
  
  @Transactional
  public void debit(Long walletId, BigDecimal amount) {
    VirtualWalletEntity walletEntity = walletRepo.getWalletEntity(walletId);
    VirtualWallet wallet = convert(walletEntity);
    wallet.debit(amount);
    VirtualWalletTransactionEntity transactionEntity = new VirtualWalletTransactionEntity();
    transactionEntity.setAmount(amount);
    transactionEntity.setCreateTime(System.currentTimeMillis());
    transactionEntity.setType(TransactionType.DEBIT);
    transactionEntity.setFromWalletId(walletId);
    transactionRepo.saveTransaction(transactionEntity);
    walletRepo.updateBalance(walletId, wallet.balance());
  }
  
  @Transactional
  public void credit(Long walletId, BigDecimal amount) {
    VirtualWalletEntity walletEntity = walletRepo.getWalletEntity(walletId);
    VirtualWallet wallet = convert(walletEntity);
    wallet.credit(amount);
    VirtualWalletTransactionEntity transactionEntity = new VirtualWalletTransactionEntity();
    transactionEntity.setAmount(amount);
    transactionEntity.setCreateTime(System.currentTimeMillis());
    transactionEntity.setType(TransactionType.CREDIT);
    transactionEntity.setFromWalletId(walletId);
    transactionRepo.saveTransaction(transactionEntity);
    walletRepo.updateBalance(walletId, wallet.balance());
  }

  @Transactional
  public void transfer(Long fromWalletId, Long toWalletId, BigDecimal amount) {
    //...跟基于贫血模型的传统开发模式的代码一样...
  }
}
~~~

## 3.5.辩证的看待DDD开发模式

### 3.5.1.Service类并没有因为DDD变得单薄而去掉,为什么?

在基于充血模型的DDD开发模式中,将业务逻辑移动到Domain中,Service类变得很薄,但在平常的代码设计与实现中,并没有完全将Service类去掉,为什么?

1. Service类负责与Repository交流,在设计与代码实现中,VirtualWalletService类负责与Repository层打交道.让Service类与Repository打交道不是让Domain与Repository打交道.是因为想要保持领域模型的独立性.不与其他层的代码或开发框架耦合在一起.让流程性的代码逻辑(比如从DB中取数据,映射数据)与领域模型的业务逻辑解耦.让领域模型更加复用;
2. Service类负责跨领域模型的业务聚合功能,VirtualWalletService中的`transfer()`转账会设计两个钱包的操作,暂且将转账业务放在VirtualWalletService类中.虽然功能演进,转账业务变得复杂起来之后,我们可以将转账业务抽取出来,设计成独立的领域模型;
3. Service类负责一些非功能性即三方系统交互的工作,比如:幂等,事务,发邮件,发消息,记录日志,调用其他系统的RPC接口等;

### 3.5.2.Controller层和Repository层还是贫血模型,是否有必要进行充血领域建模?

没有必要。Controller 层主要负责接口的暴露，Repository 层主要负责与数据库打交道，这两层包含的业务逻辑并不多,如果业务逻辑比较简单，就没必要做充血建模，即便设计成充血模型，类也非常单薄，看起来也很奇怪。