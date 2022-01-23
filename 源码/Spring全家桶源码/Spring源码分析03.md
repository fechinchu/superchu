# Spring源码分析03

# 1.FactoryBean

`FactoryBean`中的`getObject()`创建的对象并不是直接解析创建的,而是在第一次通过`ApplicationContext.getBean(Hello.class)`时候进行创建的.并且创建完之后会放在`factoryBeanOnbjectCache`中.

如下是执行完`ApplicationContext.getBean(Hello.class)`之后ApplicationContext:

![image-20220101204859491](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220101204859491.png)

当然我们也可以实现`SmartFactoryBean`接口,重写`isEagerInit()`方法

![image-20220101210703587](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220101210703587.png)

就可以提前将Bean初始化好.如下代码就是判断FactoryBean是否是`isEagerInit`如果是,直接`getBean()`;

![image-20220101210810539](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220101210810539.png)

# 2.Bean的生命周期

https://www.processon.com/embed/61c851ae0791290c9e10f06c

# 3.ApplicationContext整体流程

https://www.processon.com/embed/61d13cdfe0b34d1be7ac8cea

# 4.Spring完成工厂初始化的整体流程

https://www.processon.com/embed/61d055d8637689065dd69cd3

# 5.如何解决循环依赖

https://www.processon.com/embed/61d1b4b1e401fd7a53c504ef

 

