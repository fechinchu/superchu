# SpringMVC源码分析01

# 1.Servlet

## 1.1.ServletContainerInitializer

![image-20220107213815931](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107213815931.png)

ServletContainerInitializer是Servlet规范下的接口.

![image-20220107215415826](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107215415826.png)

![image-20220107215523373](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220107215523373.png)

在spring-web包下利用**SPI机制**,加载了`SpringServletContainerInitializer`

![image-20220110103609132](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110103609132.png)

`SpringServletContainerInitializer`的`@HandleTypes`注解也是Servlet规范的注解,它的作用就是传入该注解声明的继承和实现的类;

可以看到在`onStartUp()`方法的`webAppInitializerClasses`形参中传递了如下实参.其中就有我们实现的`MyWebApplicationInitializer`

![image-20220110104036697](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110104036697.png)

`MyWebApplicationInitializer`代码是根据Spring官网编写的

```java
/**
 * 只要写了这个,就相当于配置了SpringMVC的DispatcherServlet
 *
 */
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        //registration.addMapping("/app/*");
        registration.addMapping("/");
    }
}
```

## 1.2.WebApplicationInitializer

![image-20220118155754984](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118155754984.png)

`ServletContainerInitializer`通过SPI机制会执行`SpringServletContainerInitializer`中的`onStartup()`方法,先拿到所有`WebApplicationInitializer`的实现类,包括`MyWebApplicationInitializer`,循环遍历并调用`WebApplicationInitializer`的实现类的`onStartup()`方法;

```java
/**
 * 只要写了这个,就相当于配置了SpringMVC的DispatcherServlet
 *
 */
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        //registration.addMapping("/app/*");
        registration.addMapping("/");
    }
}
```

上述代码是准备一个空的IOC容器,(容器此时并没有刷新);创建一个DispatcherServlet,传入IOC容器并注册到ServletContext.

## 1.3.DispatcherServlet

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220111215544148.png" alt="image-20220111215544148" style="zoom:50%;" />

需要注意的是:DispatcherServlet也是一个Servlet,它必定遵循Servlet规范:

* Tomcat在启动的时候,会给每一个Servlet创建对象;
* Tomcat自然会给DispatcherServlet创建对象;
* DispatcherServlet一定会初始化,整个初始化调用过来以后就会启动IOC容器;

![image-20220110143842505](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110143842505.png)

我们根据这个结构树从`HttpServletBean`开始看`HttpServletBean`重写了`init()`方法,并采用模板模式,留了一个`initServletBean()`抽象方法给子类去实现;

![image-20220110145933869](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110145933869.png)

如下是FrameworkServlet的`initServletBean()`方法,当中可以看到`initWebApplicationContext()`是初始化IOC容器,**当然这里初始化的是MVC的容器**

![image-20220110150224044](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110150224044.png)

# 2.Web容器

## 2.1.父子容器

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110153921410.png" alt="image-20220110153921410" style="zoom:80%;" />

* 父容器:Spring配置文件进行包扫描并保存所有组件的容器;
* 子容器:SpringMVC配置文件进行包扫描并保存所有组件的容器;

### 2.1.1.XML版的父子容器

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

上面是XML版的WEB项目:

* 在web.xml中配置ContextLoaderListener,指定Spring配置文件的位置;
* 在web.xml中配置DispatcherServlet,指定SpringMVC配置文件位置;
* 这样就会产生父子容器;

在`FrameworkServlet`的`initWebApplicationContext()`方法中有父子容器的构建过程如下:

![image-20220110154401259](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110154401259.png)

### 2.1.2.代码版的父子容器

![image-20220110160019483](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220110160019483.png)

我们可以直接实现`AbstractAnnotationConfigDispatcherServletInitializer`抽象类

```java
public class MyAbstractAnnotationConfigDispatcherServletInitializer
    extends AbstractAnnotationConfigDispatcherServletInitializer {
  @Override
  // 父容器,也就是Spring的容器
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] {AppConfig.class};
  }

  @Override
  // 子容器,也就是SpringMVC的容器
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] {SpringMvcConfig.class};
  }

  //映射路径
  @Override
  protected String[] getServletMappings() {
    return new String[] {"/"};
  }
}
```

### 2.1.3.父子容器启动时机

下图是父子容器的启动时机,监听器保存了Spring的根容器(父容器),DispatcherServlet保存了SpringMVC的容器(子容器).当前这两个容器里面还没有初始化.

* 当Tomcat触发监听器,初始化Spring的根容器;
* 当Tomcat调用DispatcherServlet的初始化方法`init()`方法后执行; 

![SpringMVC父子容器初始化过程](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/SpringMVC%E7%88%B6%E5%AD%90%E5%AE%B9%E5%99%A8%E5%88%9D%E5%A7%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png)

#### 2.1.3.1.父容器启动

通过监听器刷新Spring的父容器;

![image-20220113112330996](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113112330996.png)

#### 2.1.3.2.子容器启动

通过容器启动后执行Servlet的`init()`方法,刷新子容器;

![image-20220113112656130](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113112656130.png)

并在如下代码中设置父子容器

![image-20220113112921171](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113112921171.png)

容器属性完成之后就形成了父子容器

![image-20220113161713763](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113161713763.png)

### 2.1.4.父子容器,组件是先搜子容器还是父容器

![image-20220118114445601](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118114445601.png)

在SpringMVC中寻找组件的时候,很多时候会使用`BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, Object.class)`,观察源码如下:

![image-20220118122305105](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118122305105.png)

先去子容器找,再从父容器找;

# 3.DispatcherServlet

## 3.1.DispatcherServlet的9大组件

参考官网:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-servlet-special-bean-types

### 3.1.1.何时初始化DispatcherServlet组件

Tomcat启动完成之后,一定会调用`servlet`的`init()`;

![image-20220113111218764](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113111218764.png)

1. `HttpServletBean.init()`

![image-20220111220036527](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220111220036527.png)

2. `FrameworkServlet.initServletBean()`

![image-20220111220145718](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220111220145718.png)

3. `FrameworkServlet.initWebApplicationContext()`

![image-20220113162500496](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113162500496.png)

4. `AbstractApplicationContext.publishEvent()`

![image-20220113162757115](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113162757115.png)

5. `ContextRefreshListener`是`FrameworkServlet`的一个内部类,它实现了`ApplicationListener`监听到事件.

![image-20220113162834693](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113162834693.png)

6. 初始化组件`DispatcherServlet.onRefresh()`

![image-20220113163006497](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113163006497.png)

大部分的组件都是先从Spring容器中获取我们编写的组件,如果没有,就从`DispatcherServlet.properties`文件中去生成默认的组件,如下是文件.

![image-20220111222033386](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220111222033386.png)

## 3.2.`DispatcherServlet.doDispatch()`何时执行

![image-20220111221220866](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220111221220866.png)

当我们访问controller层的时候,是调用的`Servlet.service()`方法,通过层层调用.最后到达`DispatcherServlet.doDispatch()`;

# 4.HandlerMapping

![image-20220113215354084](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113215354084.png)

在`DispatcherServlet`的`doDispatch()`方法中获取处理器链

![image-20220113220632240](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113220632240.png)

## 4.1.HandlerMapping如何确定用哪个实现类

* `BeanNameUrlHandlerMapping`:扫描所有名字以`/`开始的组件,注册到url映射中;

![image-20220116203931601](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116203931601.png)

* `RequestMappingHandlerMapping`:扫描所有`@Controller`有标注了`@RequestMapping`注解的,注册到url映射中;

## 4.2.RequestMappingHandlerMapping

`RequestMappingHandlerMapping`是比较常用的`HandlerMapping`,当在调用`DispatcherServlet.getHandler()`的时候,可以看到`RequestMappingHandlerMapping`中有属性`mappingRegistry`里面已经保存了url与controller的映射,如下图

![image-20220113220926294](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113220926294.png)

### 4.1.1.mappingRegistry属性是什么时候构建的

那么是什么时候就已经解析完controller,并把数据保存到`mappingRegistry`中呢.给`AbstractHandlerMethodMapping`的`registerHandlerMethod()`打上断点;

![image-20220113221423369](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113221423369.png)

![image-20220113221602040](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113221602040.png)

通过调用栈就可以知道,在我们执行`DispatcherServlet.initStrategies()`的时候,会创建`RequestMappingHandlerMapping`,创建完后执行初始化方法`afterPropertiesSet()`

最终层层调用到`RequestMappingHandlerMapping.createRequestMappingInfo(element)`;

![image-20220113223046351](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113223046351.png)

代码就是通过注解`@RequestMapping`进行处理;

### 4.1.2.处理器链是如何构建的

`AbstractHandlerMapping.getHandler()`

![image-20220113225213282](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113225213282.png)

上述方法中执行`getHandlerExecutionChain(handler,request)`;

![image-20220113225625674](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113225625674.png)

那么这些拦截器是什么时候加入到RequestMappingHandlerMapping中的?

在`AbstractHandlerMapping.initApplicationContext()`方法中,设置了拦截器
![image-20220113230211368](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113230211368.png)

`initApplicationContext()`是重写的`ApplicationObjectSupport.initApplicationContext()`方法.而该方法是通过重写`ApplicationContextAware.setApplicationContext()`方法实现;

![image-20220113230337486](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113230337486.png)

在`AbstractHandlerMapping`中从IOC容器中寻找并获取

![image-20220113230700704](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220113230700704.png)

# 5.HandlerAdapter

## 5.1.HandlerAdapter如何确定用哪个实现类

* `HttpRequestHandlerAdapter`:判断`Handler`是否实现`HttpRequestHandler`接口;

![image-20220116204243564](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116204243564.png)

* `SimpleControllerHandlerAdapter`:判断`Handler`是否实现`Controller`接口;

![image-20220116204308799](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116204308799.png)

* `RequestMappingHandlerAdapter`:判断`Handler`是否实现了`HandlerMethod`;

![image-20220116204416847](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116204416847.png)

## 5.2.RequestMappingHandlerAdapter

### 5.2.1.argumentResolvers与returnValueHandlers

* `argumentResolvers`:参数解析器,用来反射解析目标方法中的每一个参数;
* `returnValueHandlers`:返回值处理器,未来用来处理目标方法执行后的返回值,无论目标方法返回什么,都会转成`ModelAndView`;

官网中可以查看支持的参数类型和返回值类型:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-methods

#### 5.2.1.1.argumentResolvers与returnValueHandlers属性是什么时候构建的

与RequestMappingHandlerMapping的mappingRegistry属性一样,它们都是通过实现`InitializingBean`并重写`afterPropertiesSet()`方法来实现

![image-20220116220006111](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116220006111.png)

#### 5.2.1.2.如何自定义argumentResolvers与returnValueHandlers属性

我们可以参考`RequestMappingHandlerAdapterTests`,先调用`setArgumentResolvers()`,再调用`afterPropertiesSet()`方法;

![image-20220116220701180](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116220701180.png)

#### 5.2.1.3.默认的argumentResolvers和returnValueHandlers

![image-20220116224904756](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116224904756.png)

![image-20220116224925323](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116224925323.png)

### 5.2.2.`RequestMappingHandlerAdapter.invokeHandlerMethod()`

#### 5.2.2.1.解析参数

`InvocableHandlerMethod.getMethodArgumentValues()`

![image-20220116223838306](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116223838306.png)

这里的`resolvers`设计的很巧妙,里面有两个属性.一个用来存储解析器,另一个用来存储解析器缓存;

![image-20220116224113348](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116224113348.png)

调用`resolvers.supportsParameter(parameter)`方法层层调用到`HandlerMethodArgumentResolverComposite.getArgumentResolver()`,先看缓存有没有,没有就遍历解析器然后放入缓存,提高了程序的效率;

![image-20220116224341254](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116224341254.png)

#### 5.2.2.2.处理返回值

![image-20220116231135541](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116231135541.png)

selectHandler()方法如下:

![image-20220116231430765](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116231430765.png)

那么如果我们在方法上加上`@response`注解的话,我们返回的是JSON数据,如果不加的话是跳转页面.这是如何实现的?

![image-20220116231721363](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220116231721363.png)

上图的`RequestResponseBodyMethodProcessor`是用来解析`@ResponseBody`的,`ViewNameMehodReturnValueHandler`是用来跳转字符串页面的.我们可以看到在这个解析返回值处理器的集合里,有优先级顺序.如果谁在前面并且匹配到的话就不再使用下面的返回值处理器了.

# 6.HandlerExceptionResolver

![image-20220117181923741](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117181923741.png)

通过断点调试我们可以知道有3个默认的`HandlerExceptionResolver`

* `ExceptionHandlerExceptionResolver`;
* `ResponseStatusExceptionResolver`;
* `DefaultHandlerExceptionResolver`;

## 6.1.异常执行流程

`DispatcherServlet.processDispatchResult()`:如果有异常的话是不会执行下面的`render()`渲染的;

![image-20220117191034819](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117191034819.png)

`DispatcherServlet.processHandlerException()`:可以看如下的中文注释:

![image-20220117191334459](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117191334459.png)

## 6.2.三种异常测试代码

```java
@RestController
public class HelloController {

  @Autowired private HelloService helloService;

  public HelloController() {
    System.out.println("HelloController.construct()");
  }

  @GetMapping("/hello")
  public String sayHello(@RequestParam("name") String name, @RequestParam("num") Integer num) {
	int result = 10 / num;
	if("456".equals(name)){
		throw new ArithmeticException("给爷整笑了");
	}
	if("123".equals(name)){
      /*
       * @ResponseStatus(value = HttpStatus.CONFLICT,reason = "非法用户") public class
       * InvalidUserException extends RuntimeException{ static final long serialVersionUID  = -7034897190745766932L; }
       */
      throw new InvalidUserException();
	}
    helloService.sayHello();
    System.out.println("HelloController.sayHello()");
    return "HelloController.sayHello()";
  }
}
```

## 6.3.DefaultHandlerExceptionResolver

该异常处理器是默认的用于处理SpringMVC异常的处理器.它能够处理的异常如下:

![image-20220117183024098](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117183024098.png)

调用测试代码,并且不传递参数就可以复现`DefaultHandlerExceptionResolver`解析器处理异常的流程

![image-20220117192241736](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117192241736.png)

判断出异常为`MissingServletRequestParameterException`

## 6.4.ResponseStatusExceptionResolver

编写自定义异常

```java
@ResponseStatus(value = HttpStatus.CONFLICT,reason = "非法用户")
public class InvalidUserException extends RuntimeException{
	static final long serialVersionUID = -7034897190745766932L;
}
```

调用测试代码,传入`name = 123`即可复现.调用到`ResponseStatusExceptionResolver.doResolveException()`可以发现Spring会根据注解获取到错误信息

![image-20220117192736229](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117192736229.png)

## 6.5.ExceptionHandlerExceptionResolver

编写`ControllerAdvice`

```java
@ControllerAdvice
public class MyExceptionHandler {

	@ResponseBody
	@ExceptionHandler(value = {ArithmeticException.class})
	public String handleZeroException(Exception e){
		//这里的参数能够传递什么参考:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-exceptionhandler-args
		//这里的返回值能够写什么参考:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-exceptionhandler-return-values
		//异常处理器的功能如何增强?参数解析器,返回值处理器
		return "ERROR";
	}
}
```

编写上述代码.

这里的参数能够传递什么参考:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-exceptionhandler-args;

这里的返回值能够写什么参考:https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-exceptionhandler-return-values

调用测试代码:`name = 456`进入到`ExceptionHandlerExceptionResolver.doResolveHandlerMethodException()`方法如下

![image-20220117195341184](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117195341184.png)

里面的`argumentResolvers`参数解析器,`returnValueHandlers`返回值处理器都是在`afterPropertiesSet()`的时候初始化好的;

### 6.5.1.什么时候初始化的

![image-20220117210059950](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117210059950.png)

`ExceptionHandlerExceptionResolver.afterPropertiesSet()`代码如上.并且在`initExceptionHandlerAdviceCache()`中就把`ControllerAdvice`分析好了并放在缓存中.

![image-20220117213034020](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117213034020.png)

# 7.ViewResolver

整体流程如下:

**在Controller中返回`index.jsp`,这是逻辑视图,通过ViewResolver视图解析器获取到View对象,再执行`View.render()`渲染出真正的页面**

![image-20220117203830463](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220117203830463.png)

# 8.SpringMVC的自动配置

SpringMVC的所有九大组件都是可以进行定制的.因为在`initStrategies()`的时候,是先从IOC容器中去寻找,找不到再创建默认的组件.我们可以编写好自己的组件并`@Component`到IOC容器中即可;

还有一种方式可以实现自动配置,代码如下,为了添加自己的`ViewResolver`;

```java
@Configuration
@EnableWebMvc
public class MvcExtendConfiguration implements WebMvcConfigurer {

	@Override
	public void configureViewResolvers(ViewResolverRegistry registry) {
		registry.viewResolver(new MyViewResolver());
	}
}
```

`@EnableWebMvc`注解中导入了类`DelegatingWebMvcConfiguration`

![image-20220118144143383](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118144143383.png)

`DelegatingWebMvcConfiguration`有一个方法,`@Autowired`就是将所有的`WebMvcConfigurer`注入到类中,包括我们自己写的类

![image-20220118144318320](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118144318320.png)

继续看`DelegatingWebMvcConfiguration`的父类`WebMvcConfigurationSupport`,里面的方法大量使用`@Bean`注解,往Spring容器中注入组件,下面开始往容器中注入`ViewResolver`

![image-20220118144724329](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118144724329.png)

其中留了一个模板方法`configureViewResolvers(registry)`给子类实现.实际上最后也是调用的我们自己的`MvcExtendConfiguration`给IOC容器中注入自己的组件;

![image-20220118144907103](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118144907103.png)

# 9.写一个简陋的SpringBoot

1. 导入依赖:里面包含嵌入式的Tomcat

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.2.8.RELEASE</version>
    </dependency>

    <dependency>
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-core</artifactId>
        <version>8.5.64</version>
    </dependency>

    <dependency>
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-jasper</artifactId>
        <version>8.5.64</version>
    </dependency>
</dependencies>
```

2. 编写一个启动类,如下:

```java
public class Main {

  public static void main(String[] args) throws LifecycleException {
    // 自己写Tomcat的启动
    Tomcat tomcat = new Tomcat();
    tomcat.setPort(8888);
    tomcat.setHostname("localhost");
    tomcat.setBaseDir(".");
    tomcat.addWebapp("/boot", System.getProperty("user.dir") + "/src/main");

    tomcat.start();
    tomcat.getServer().await();
  }
}
```

3. SPI机制创建IOC容器,配置`DispatcherServlet`

```java
public class MyAbstractAnnotationConfigDispatcherServletInitializer
    extends AbstractAnnotationConfigDispatcherServletInitializer {
  @Override
  // 父容器,也就是Spring的容器
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] {AppConfig.class};
  }

  @Override
  // 子容器,也就是SpringMVC的容器
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] {SpringMvcConfig.class};
  }

  //映射路径
  @Override
  protected String[] getServletMappings() {
    return new String[] {"/"};
  }
}
```

启动容器,就可以访问Controller;
