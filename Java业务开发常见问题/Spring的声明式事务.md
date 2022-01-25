# Spring的声明式事务常见问题总结

## 1.事务有可能没有生效

@Transactional生效原则:

1. 除非特殊配置(比如使用AspectJ静态织入实现AOP),否则**只有定义在public方法上的@Transactional才能生效**.原因是Spring默认通过动态代理的方式实现AOP,对目标方法进行增强,private方法无法代理到,Spring自然也无法动态增强事务处理逻辑;

2. ~~~java
   public int createUserWrong2(String name) {
       try {
           this.createUserPublic(new UserEntity(name));
       } catch (Exception ex) {
           log.error("create user failed because {}", ex.getMessage());
       }
     return userRepository.findByName(name).size();
   }
   
   //标记了@Transactional的public方法
   @Transactional
   public void createUserPublic(UserEntity entity) {
       userRepository.save(entity);
       if (entity.getName().contains("test"))
           throw new RuntimeException("invalid username!");
   }
   ~~~

   这里的方法事务同样不生效.**必须通过代理过的类从外部调用目标方法才能生效**,Spring通过AOP技术对方法进行增强,要调用增强过的方法必然是调用代理后的对象,我们可以尝试修改代码,注入一个self,然后再通过self实例调用标记有@Transactional注解的createUserPublic方法.设置断点可以看到,self是由Spring通过CGLIB方式增强过的类;

   * CGLIB通过继承方式实现代理类,private方法在子类不可见,自然也就无法进行事务增强;
   * this指针代表对象自己,Spring不可能注入this,所以通过this访问方法必然不是代理;

   ![image-20200504193733824](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200504193733824.png)

虽然在UserService内部注入自己调用自己的createUserPublic可以正确实现事务,但更合理的实现方式,让Controller直接调用之前的UserService的createUserPublic方法;

## 2.事务即便生效也不一定能回滚

通过AOP实现事务处理可以理解为,使用try...catch...来包裹标记了@Transactional注解的方法,当方法出现了异常并且满足一定的条件的时候,在catch里面我们可以设置事务回滚,没有异常则直接提交事务.

一定条件包含:

1. **只有异常传播出了标记了@Transactional注解的方法,事务才能回滚.**,在Spring的TransactionAspectSupport里面有一个invokeWithinTransaction方法,里面就是处理事务的逻辑.可以看到,只有捕获到异常才能进行后续事务处理;
2. **默认情况下,出现RuntimeException(非受检异常)或Error的时候,Spring才会回滚事务**

![image-20200504201101821](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200504201101821.png)

下面例子:    

```java
@Service
@Slf4j
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    //异常无法传播出方法，导致事务无法回滚
    @Transactional
    public void createUserWrong1(String name) {
        try {
            userRepository.save(new UserEntity(name));
            throw new RuntimeException("error");
        } catch (Exception ex) {
            log.error("create user failed", ex);
        }
    }

    //即使出了受检异常也无法让事务回滚
    @Transactional
    public void createUserWrong2(String name) throws IOException {
        userRepository.save(new UserEntity(name));
        otherTask();
    }

    //因为文件不存在，一定会抛出一个IOException
    private void otherTask() throws IOException {
        Files.readAllLines(Paths.get("file-that-not-exist"));
    }
}
```
* 在createUserWrong1方法中会抛出一个RuntimeException,但由于方法内catch了所有异常,异常无法从方法传播出去,事务自然无法回滚;
* 在createUserWrong2方法中,注册用户的同时会有一次otherTask文件读取操作,如果文件读取失败,希望用户注册的数据库操作会滚.虽然这里没有捕获异常,但因为otherTask方法抛出的是受检异常,createUserWrong2传播出去的也是受检异常,事务同样不会会滚.

解决方法:手动设置让当前事务处于回滚状态:

~~~java
@Transactional
public void createUserRight1(String name) {
    try {
        userRepository.save(new UserEntity(name));
        throw new RuntimeException("error");
    } catch (Exception ex) {
        log.error("create user failed", ex);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
~~~

在注解中声明,期望遇到所有的Exception都回滚事务

~~~java
@Transactional(rollbackFor = Exception.class)
public void createUserRight2(String name) throws IOException {
    userRepository.save(new UserEntity(name));
    otherTask();
}
~~~



