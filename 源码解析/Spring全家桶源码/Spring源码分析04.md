# Spring源码分析04

# 1.Spring全家桶分析套路

* `AbstractBeanDefinition`的构造器打上断点,看看何时给容器注入了什么组件;
* `AbstractApplicationContext`初始化完成后,监控容器中多了哪些后置处理器,并分析后置处理器在什么时候调用,做了什么功能;
* 每一个功能的开启,要么事写配置,要么是写注解`@EnableXXX`开启XXX功能的注解,这个注解很重要;

# 2.AOP原理

## 2.1.`@EnableAspectJAutoProxy`

给`@EnableAspectJAutoProxy`注解中的`@Import(AspectJAutoProxyRegistrar.class)`类中`registerBeanDefinitions()`方法打上断点

![image-20220103152541317](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103152541317.png)

![image-20220103152556814](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103152556814.png)

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103152612618.png" alt="image-20220103152612618" style="zoom:50%;" />

Step INTO到`registerAspectJAnnotationAutoProxyCreatorIfNessary()`方法中.

![image-20220103152832436](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103152832436.png)

观察到`AnntationAwareAspectJAutoProxyCreateor.class`分析该类的UML图,它是一个后置处理器;

![image-20220103153421173](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103153421173.png)

## 2.2.`AnnotationAwareAspectJAutoProxyCreator`

如何观察`AnnotationAwareAspectJAutoProxyCreator`这个后置处理器干了什么,可以跟着继承树打`@Override`的方法.

![image-20220103155702331](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103155702331.png)

我们先给上述三个`@Override`的方法打上三个断点

同样因为也给`AbstractAutoProxyCreator`类的后置处理方法`postProcessBeforInstantiation()`和`postProcessAfterInitialization()`打上断点.

![image-20220105210212003](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220105210212003.png)

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103155740598.png" alt="image-20220103155740598" style="zoom:50%;" />

### 2.2.1.`initBeanFactory()`

执行到第一个方法`initBeanFactory()`

通过栈引用我们可以看到:在执行`registerBeanPostProcessors()`的调用`getBean()`,再执行`initializeBean()`,再执行`invokeAwareMethods()`,如下图;

![image-20220103160400010](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103160400010.png)

最后走到了`AnnotationAwateAspectJAutoProxyCreator.initBeanFactory()`

![image-20220103160632471](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220103160632471.png)

在代码中,创建并保存了`ReflectiveAspectJAdvisorFactory`和`BeanFactoryAspectJAdvisorsBuilderAdapter`;

### 2.2.2.`AbstractAutoProxyCreator.postProcessBeforeInstantiation()`

![image-20220105211526437](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220105211526437.png)

判断当前类是否有`@Aspect`或者实现切面的接口.

![image-20220105211611372](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220105211611372.png)

在后续的`getBean()`的方法调用中,所有的方法都要去执行`applyBeanPostProcessorsBeforeInstantiation()`方法.

后续:

STEP INTO `AspectJAwareAdvisorAutoProxyCreator.shouldSkip()`;

STEP INTO`AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors()`;

STEP INTO`BeanFactoryAspectJAdvisorsBuilder,buildAspectJAdvisors()`;该方法是核心

![image-20220106192610696](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106192610696.png)

`this.advisorFactory.isAspect(beanType)`判断是不是@Aspect,如果是则进入.并通过解析@Aspect类获取到所有的advisors

![image-20220106193224311](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106193224311.png)

并将这些advisors放到`advisorsCache`缓存中.后续获取advisor直接从缓存中获取即可;

![image-20220106194000452](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106194000452.png)

当获取到这些advisors的时候实际上还没有创建@Aspect类的对象,是通过`BeanFactoryUtils`获取到所有的beanName,并去解析获取到advisors;

![image-20220106194646365](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106194646365.png)

### 2.2.3.`AbstractAutoProxyCreator.postProcessAfterInitialization()`

当后续`getBean()`获取Bean的时候执行完初始化之后就会执行如下方法

![image-20220106202723208](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106202723208.png)

这里面的关键方法是:`warpIfNecessary()`;

![image-20220106202745952](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106202745952.png)

其中`getAdvicesAndAdvisorsForBean()`方法是获取当前对象所有使用的Advisor,如果获取到的增强器不为空,则执行`createProxy()`方法,如下:

![image-20220106203302278](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106203302278.png)

最后执行`proxyFactory.getProxy()`方法,如下:

![image-20220106214342378](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106214342378.png)

创建完,因为我们代码是由接口的,所有用的是JDK的方式创建代理对象.代理对象如下:其中`advisors`第0个的拦截器是在中途代码中`extendAdvisors(eligibleAdvisors);`加入的用于暴露的拦截器;

![image-20220106214648825](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220106214648825.png)

## 2.3.执行AOP方法

Spring有两种执行AOP的方法,一种是JDK的动态代理,一种是基于CGLIB,下面的案例是基于CGLIB;

整体流程如下:

* 正常情况下:前置通知->目标切点->返回通知->后置通知;
* 异常情况下:前置通知->目标切点->异常通知->后置通知;

![image-20220107141134836](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107141134836.png)

![image-20220107102118474](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107102118474.png)

当断点到了`bean.sayHello()`时,STEP INTO进入到`DynamicAdvisedInterceptor`的`intercept()`方法

![image-20220107104858183](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107104858183.png)

>advise增强器只是用来保存信息,拦截器能真正执行目标方法,我们需要把增强器转成拦截器;

得到拦截器链如下:

![image-20220107133734733](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107133734733.png)

后续一直执行到`new CglibMethodInvocation().proceed()`方法.

![image-20220107133947863](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107133947863.png)

STEP INTO后发现代码执行的是`super.proceed()`方法

![image-20220107134118157](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134118157.png)

STEP INTO到`ReflectiveMethodInvocation.proceed()`方法,该代码就是从chain中按照下标从0开始一个个获取Interceptor,并执行`invoke()`方法

![image-20220107134522056](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134522056.png)

### 2.3.1.`ExposeInvocationInterceptor.invoke()`

![image-20220107134639661](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134639661.png)

### 2.3.2.`MethodBeforeAdviceInterceptor.invoke()`

![image-20220107134750977](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134750977.png)

### 2.3.3.`AspectJAfterAdvice.invoke()`

![image-20220107134845714](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134845714.png)

### 2.3.4.`AfterReturningAdviceInterceptor.invoke()`

![image-20220107134902829](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134902829.png)

### 2.3.5.`AspectJAfterThrowingAdvice.invoke()`

![image-20220107134955816](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107134955816.png)

最后执行到切点:

![image-20220107135140080](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107135140080.png)

# 3.监听器原理

![image-20220107152654074](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107152654074.png)

其中在Spring容器的this阶段()就往Spring的BeanDefinitionRegistry中注册了两个与Event相关的组件

* `EventListenerMethodProcessor`;
* `DefaultEventListenerFactory`;

## 3.1.监听器是如何创建的

![image-20220107153520567](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107153520567.png)

从这里我们就可以看到`EventListenerMethodProcessor`在两个地方执行

* 工厂后置处理环节;
* 所有Bean都创建完成后的`SmartInitializingSingleton.afterSingletonsInstantiated()`执行;

![image-20220107161158847](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107161158847.png)

我们可以从上面的源码中了解到,找到组件中所有的`@EventListener`标注的方法,拿到`DefaultEventLisatenerFactory`,把beanName等封装到适配器中`ApplicationListenerMethodAdapter(beanName,beanType,method)`;把这些适配器(监听器)放到容器中和事件多播器中.

可以看到Spring工厂中的3个applicationListener:

![image-20220107161552340](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107161552340.png)

## 3.2.监听器是如何感知事件的

![image-20220107162353104](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107162353104.png)