# Spring源码分析02

![Bean的生命周期](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/Bean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

# 1.BeanFactoryPostProcessor

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226191125704.png" alt="image-20211226191125704" style="zoom:50%;" />

* BeanFactoryPostProcessor;增强的是beanFactory;

![image-20211226190827111](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226190827111.png)

* BeanDefinitionRegistryPostProcessor;增强的是档案馆;

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226190738356.png)

# 2.BeanPostProcessor

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226191043436.png" alt="image-20211226191043436" style="zoom:50%;" />

* BeanPostProcessor;

![image-20211226192227942](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226192227942.png)

* DestructionAwareBeanPostProcessor;(销毁接口先暂不看)
* InstantiantionAwareBeanPostProcessor;

![image-20211226194258663](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226194258663.png)

* MergedBeanDefinitionPostProcessor;

![image-20211226194322806](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226194322806.png)

* SmartInstantiationAwareBeanPostProcessor; 

![image-20211226194345571](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226194345571.png)

#3.InitializingBean,DisposableBean

![image-20211223185233475](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211223185233475.png)

# 4.整体流程

## 4.1.BeanDefinitionRegistryPostProcessor创建与执行

给`MyBeanDefinitionRegistryPostProcessor`打上断点

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226213303277.png" alt="image-20211226213303277" style="zoom:50%;" />

1. `AbstractApplicationContext.refresh()`

![image-20211226213452516](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226213452516.png)

2. `PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessor()`在该方法中执行`getBean()`方法.

![image-20211226215012614](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226215012614.png)

断点放开,执行到`postProcessBeanDefinitionRegistry()`

![image-20211226220320702](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226220320702.png)

![image-20211226220421119](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226220421119.png)

断点放开,执行到`postProcessBeanFactory()`

![image-20211226221008198](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226221008198.png)

## 4.2.BeanFactoryPostProcessor创建与执行

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226221721113.png" alt="image-20211226221721113" style="zoom:50%;" />

![image-20211226221810545](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226221810545.png)

## 4.3.BeanPostProcessor接口的所有实现类的创建

`PostProcessor.registerBeanPostProcessors()`,将所有的BeanPostProcessor放入到`AbstractBeanFactory`的属性`List<BeanPostProcessor> beanPostProcessors = new BeanPostProcessorCacheAwareList()`中;

![image-20211226232212021](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226232212021.png)

## 4.4.BeanPostProcessor的执行

### 4.4.1.`SmartInstantiationAwareBeanProcessor.predictBeanType()`

#### 4.4.1.1.在注册监听器的时候调用

![image-20211226233057133](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226233057133.png)

![image-20211226233050009](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226233050009.png)

在注册监听器的时候,Spring框架想要根据Class类型去获取所有的监听器.因为要判断所有的组件是不是类型匹配到`ApplicationListener.class`,在这里调用了`SmartInstantiationAwareBeanProcessor.redictBeanType()`去改变了Bean的Type.

![image-20211230155345541](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230155345541.png)

![image-20211230155713045](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230155713045.png)

#### 4.4.1.2.在完成Bean工厂初始化的时候调用

![image-20211230164546057](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230164546057.png)

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230164644523.png)

在`finishBeanFactoryInitialization()`根据类型获取bean的名字,在判断类型的时候同样使用到了`SmartInstantiationAwareBeanProcessor.predictBeanType()`;

### 4.4.2.`InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()`

![image-20211230170357078](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230170357078.png)

1. `AbstractApplicationContext.finishBeanFactoryInitialization()`

![image-20211230170522745](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230170522745.png)

2. `DefaultListableBeanFactory.preInstantiateSingletons()`在该方法中进行创建对象

![image-20211230170814543](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230170814543.png)

3. `AbstractAutowireCapableBeanFactory.createBean()`在执行真正的`doCreateBean()`对象之前,给BeanPostProcessor一个机会返回一个代理对象.如果成功返回了一个对象,那么后续的创建对象逻辑就不再执行了.

![image-20211230171901441](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230171901441.png)

### 4.4.3.`SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()`

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230183738205.png" alt="image-20211230183738205" style="zoom:50%;" />

`AbstractAutowireCapableBeanFactory.createBeanInstance()`获取构造器,如果有构造器,那么用指定的构造器进行自动注入,否则用无参构造;

![image-20211230185134778](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230185134778.png)

### 4.4.4.创建对象

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230222301035.png" alt="image-20211230222301035" style="zoom:50%;" />

### 4.4.5.`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()`

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230222623570.png" alt="image-20211230222623570" style="zoom:67%;" />

![image-20211230222832525](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230222832525.png)

在创建完实例对象之后,紧接着调用了`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()`;

### 4.4.6.`InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()`

![image-20211230223243654](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230223243654.png)

1. `AbstractAutowireCapableBeanFactory.doCreateBean()`在创建完Bean,执行过`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()`后对执行`populateBean()`方法对属性进行赋值

![image-20211230223343649](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230223343649.png)

2. `AbstractAutowireCapableBeanFactory.populateBean()`在该方法中,在真正的属性赋值之前,先执行了该`PostProcessor`的方法

![image-20211230224308588](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230224308588.png)

### 4.4.7.`InstantiationAwareBeanPostProcessor.postProcessProperties()`

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230225058512.png" alt="image-20211230225058512" style="zoom: 67%;" />

``AbstractAutowireCapableBeanFactory.populateBean()``方法中,`@Autowired`注解的属性进行赋值就是用的`postProcessProperties()`

![image-20211230225307422](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230225307422.png)

通过`postProperties()`处理好`PropertyValues`之后会在后面的`applyPropertyValues()`方法进行赋值;

![image-20211230230356014](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230230356014.png)

### 4.4.8.`BeanPostProcessor.postProcessBeforeInitialization()`

![image-20211230232708356](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230232708356.png)

1. `AbstractAutowireCapableBeanFactory.doCreateBean()`

![image-20211230232758254](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230232758254.png)

2. `AbstractAutowireCapableBeanFactory.initializeBean()`

![image-20211230232934575](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230232934575.png)

### 4.4.9.执行对象的初始化方法

执行`afterPropertiesSet()`方法;

![image-20211230235220851](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211230235220851.png)

### 4.4.10.`BeanPostProcessor.postProcessAfterInitialization()`

![image-20211231000137164](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211231000137164.png)

# 5.如何分析Sping体系下的框架

* 在`AbstractApplicationContext`的`refresh()`方法中的最后一步`finishRefresh()`打一个断点,看ApplicationContext中有什么后置处理器.`beanPostProcessors`.这些后置处理器是干嘛用的?
* 在`getBean()`中观察后置处理器干了什么?
* 给`AbstractBeanDefinition`的构造器打上断点,看什么时候给Spring容器放入Bean的定义信息;

