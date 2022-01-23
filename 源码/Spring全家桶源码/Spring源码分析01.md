# Spring源码分析01

# 1.Gradle

mac通过brew进行安装:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211205230718542.png" alt="image-20211205230718542" style="zoom:50%;" />

之后给Gradle安装目录下init.d文件夹,放一个init.gradle文件,配置阿里云镜像,内容如下:

```groovy
gradle.projectsLoaded {
    rootProject.allprojects {
        buildscript {
            repositories {
                def JCENTER_URL = 'https://maven.aliyun.com/repository/jcenter'
                def GOOGLE_URL = 'https://maven.aliyun.com/repository/google'
                def NEXUS_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
                all { ArtifactRepository repo ->
                    if (repo instanceof MavenArtifactRepository) {
                        def url = repo.url.toString()
                        if (url.startsWith('https://jcenter.bintray.com/')) {
                            project.logger.lifecycle "Repository ${repo.url} replaced by $JCENTER_URL."
                            println("buildscript ${repo.url} replaced by $JCENTER_URL.")
                            remove repo
                        }
                        else if (url.startsWith('https://dl.google.com/dl/android/maven2/')) {
                            project.logger.lifecycle "Repository ${repo.url} replaced by $GOOGLE_URL."
                            println("buildscript ${repo.url} replaced by $GOOGLE_URL.")
                            remove repo
                        }
                        else if (url.startsWith('https://repo1.maven.org/maven2')) {
                            project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                            println("buildscript ${repo.url} replaced by $REPOSITORY_URL.")
                            remove repo
                        }
                    }
                }
                jcenter {
                    url JCENTER_URL
                }
                google {
                    url GOOGLE_URL
                }
                maven {
                    url NEXUS_URL
                }
            }
        }
        repositories {
            def JCENTER_URL = 'https://maven.aliyun.com/repository/jcenter'
            def GOOGLE_URL = 'https://maven.aliyun.com/repository/google'
            def NEXUS_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
            all { ArtifactRepository repo ->
                if (repo instanceof MavenArtifactRepository) {
                    def url = repo.url.toString()
                    if (url.startsWith('https://jcenter.bintray.com/')) {
                        project.logger.lifecycle "Repository ${repo.url} replaced by $JCENTER_URL."
                        println("buildscript ${repo.url} replaced by $JCENTER_URL.")
                        remove repo
                    }
                    else if (url.startsWith('https://dl.google.com/dl/android/maven2/')) {
                        project.logger.lifecycle "Repository ${repo.url} replaced by $GOOGLE_URL."
                        println("buildscript ${repo.url} replaced by $GOOGLE_URL.")
                        remove repo
                    }
                    else if (url.startsWith('https://repo1.maven.org/maven2')) {
                        project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                        println("buildscript ${repo.url} replaced by $REPOSITORY_URL.")
                        remove repo
                    }
                }
            }
            jcenter {
                url JCENTER_URL
            }
            google {
                url GOOGLE_URL
            }
            maven {
                url NEXUS_URL
            }
        }
    }
}
```

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211205231229921.png" alt="image-20211205231229921" style="zoom:50%;" />

# 2.Spring注解驱动开发

| 注解             | 功能                                                         |
| ---------------- | ------------------------------------------------------------ |
| @Bean            | 容器中注册组件                                               |
| @Primary         | 同类组件如果有多个,标注主组件                                |
| @DependsOn       | 组件之间声明依赖关系                                         |
| @Lazy            | 组件懒加载(最后使用的时候才创建)                             |
| @Scope           | 声明组件的作用范围(SCOPE_PROTOTYPE,SCOPE_SINGELETON)         |
| @Configuration   | 声明这是一个配置类,替换以前配置文件                          |
| @Component       | @Controller,@Service,@Repository                             |
| @Indexed         | 加速组件,标注了@Indexed的组件,直接会启动快速加载             |
| @Order           | 数字越小优先级越高,越先工作                                  |
| @ComponentScan   | 包扫描                                                       |
| @Import          | 导入第三方Jar包组件,或定制批量导入组件逻辑                   |
| @ImportResource  | 导入以前的xml配置文件,让其生效                               |
| @Profile         | 基于多环境激活                                               |
| @PropertySource  | 外部properties配置文件和JavaBean进行绑定.结合ConfigurationProperties |
| @PropertySources | @PropertySource组合注解                                      |
| @Autowired       | 自动装配                                                     |
| @Qualifier       | 精确指定                                                     |
| @Value           | 取值,列如:`@Value(“${xx}”)`                                  |
| @Lookup          | 单例组件依赖非单例组件，非单例组件获取需要使用方法           |

## 2.1.组件注册

### 2.1.1.`@ComponentScan`

`@ComponentScan`可以自定义规则.

```java
//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类
@ComponentScans(
		value = {
				@ComponentScan(value="com.atguigu",includeFilters = {
/*						@Filter(type=FilterType.ANNOTATION,classes={Controller.class}),
						@Filter(type=FilterType.ASSIGNABLE_TYPE,classes={BookService.class}),*/
						@Filter(type=FilterType.CUSTOM,classes={MyTypeFilter.class})
				},useDefaultFilters = false)	
		}
		)
//@ComponentScan  value:指定要扫描的包
//excludeFilters = Filter[] ：指定扫描的时候按照什么规则排除那些组件
//includeFilters = Filter[] ：指定扫描的时候只需要包含哪些组件
//FilterType.ANNOTATION：按照注解
//FilterType.ASSIGNABLE_TYPE：按照给定的类型；
//FilterType.ASPECTJ：使用ASPECTJ表达式
//FilterType.REGEX：使用正则指定
//FilterType.CUSTOM：使用自定义规则
public class MainConfig {
	
	//给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id
	@Bean("person")
	public Person person01(){
		return new Person("lisi", 20);
	}
}
```

```java
public class MyTypeFilter implements TypeFilter {

	/**
	 * metadataReader：读取到的当前正在扫描的类的信息
	 * metadataReaderFactory:可以获取到其他任何类信息的
	 */
	@Override
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException {
		//获取当前类注解的信息
		//AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		//获取当前正在扫描的类的类信息
		ClassMetadata classMetadata = metadataReader.getClassMetadata();
		//获取当前类资源（类的路径）
		//Resource resource = metadataReader.getResource();
		String className = classMetadata.getClassName();
		System.out.println("--->"+className);
		if(className.contains("er")){
			return true;
		}
		return false;
	}
}
```

### 2.1.2.`@Scope`

* prototype:多实例的:ioc容器启动并不会去调用方法创建对象放在容器中,每次获取的时候才会调用方法创建对象;

   *                  singleton:单实例的(默认值):ioc容器启动会调用方法创建对象放到ioc容器中,以后每次获取就是直接从容器中拿;
   *                  request:同一次请求创建一个实例;
   *                  session:同一个session创建一个实例;

### 2.1.3.`@Condition`

按照一定的条件进行判断,满足条件的话才给容器中注册Bean

```java
/**
	 * @Conditional({Condition}) ： 按照一定的条件进行判断，满足条件给容器中注册bean
	 * 
	 * 如果系统是windows，给容器中注册("bill")
	 * 如果是linux系统，给容器中注册("linus")
	 */
	@Bean("bill")
	public Person person01(){
		return new Person("Bill Gates",62);
	}
	
	@Conditional(LinuxCondition.class)
	@Bean("linus")
	public Person person02(){
		return new Person("linus", 48);
	}
```

```java
//判断是否linux系统
public class LinuxCondition implements Condition {

	/**
	 * ConditionContext：判断条件能使用的上下文（环境）
	 * AnnotatedTypeMetadata：注释信息
	 */
	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		// TODO是否linux系统
		//1、能获取到ioc使用的beanfactory
		// ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		//2、获取类加载器
		// ClassLoader classLoader = context.getClassLoader();
		//3、获取当前环境信息
		Environment environment = context.getEnvironment();
		//4、获取到bean定义的注册类
		// BeanDefinitionRegistry registry = context.getRegistry();
		
		String property = environment.getProperty("os.name");
		
		//可以判断容器中的bean注册情况，也可以给容器中注册bean
		boolean definition = registry.containsBeanDefinition("person");
		if(property.contains("linux")){
			return true;
		}
		return false;
	}
}
```

### 2.1.4.`@Import`

一般用在`@Configuration`注解的配置类上;

```java
@Import({Color.class,Red.class,MyImportSelector.class,MyImportBeanDefinitionRegistrar.class})
//@Import导入组件，id默认是组件的全类名
//如果是使用的@Import(Color.class)那么默认就是用Color的无参构造器导入到Spring容器中;
```

`@Import`注解使用`MyImportSelector.class`

```java
//自定义逻辑返回需要导入的组件
public class MyImportSelector implements ImportSelector {

	//返回值，就是到导入到容器中的组件全类名
	//AnnotationMetadata:当前标注@Import注解的类的所有注解信息
	@Override
	public String[] selectImports(AnnotationMetadata importingClassMetadata) {
		//importingClassMetadata
		//方法不要返回null值
		return new String[]{"com.atguigu.bean.Blue","com.atguigu.bean.Yellow"};
	}
}
```

`@Import`注解使用`MyImportBeanDefinitionRegistrar.class`

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

	/**
	 * AnnotationMetadata：当前类的注解信息
	 * BeanDefinitionRegistry:BeanDefinition注册类；
	 * 		把所有需要添加到容器中的bean；调用BeanDefinitionRegistry.registerBeanDefinition手工注册进来
	 */
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		
		boolean definition = registry.containsBeanDefinition("com.fechin.bean.Red");
		boolean definition2 = registry.containsBeanDefinition("com.fechin.bean.Blue");
		if(definition && definition2){
			//指定Bean定义信息；（Bean的类型，Bean。。。）
			RootBeanDefinition beanDefinition = new RootBeanDefinition(RainBow.class);
			//注册一个Bean，指定bean名
			registry.registerBeanDefinition("rainBow", beanDefinition);
		}
	}
}
```

### 2.1.5.`@LookUp`

```java
	@Lookup  //去容器中找。@Bean的这种方式注册的Person @Lookup不生效
	public Cat getCat() {
		return cat;
	}
```

```java
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
//容器就应该给Cat再创建一个对象
@Component
public class Cat {

	private String name;

	public void setName(String name) {
		this.name = name;
	}

	public String getName() {
		return name;
	}
}
```

### 2.1.6.`FactoryBean`

```java
//创建一个Spring定义的FactoryBean
public class ColorFactoryBean implements FactoryBean<Color> {

	//返回一个Color对象，这个对象会添加到容器中
	@Override
	public Color getObject() throws Exception {
		// TODO Auto-generated method stub
		System.out.println("ColorFactoryBean...getObject...");
		return new Color();
	}

	@Override
	public Class<?> getObjectType() {
		// TODO Auto-generated method stub
		return Color.class;
	}

	//是单例？
	//true：这个bean是单实例，在容器中保存一份
	//false：多实例，每次获取都会创建一个新的bean；
	@Override
	public boolean isSingleton() {
		// TODO Auto-generated method stub
		return false;
	}
}
```

在配置类中将`ColorFactoryBean`放入IOC容器

```java
@Bean
	public ColorFactoryBean colorFactoryBean(){
		return new ColorFactoryBean();
	}
```

获取`applicationContext.getBean("colorFactoryBean")`实际上是获取`ColorFactoryBean`中的`getObject()`方法中的对象

```java
//工厂Bean获取的是调用getObject创建的对象
		Object bean2 = applicationContext.getBean("colorFactoryBean");
		Object bean3 = applicationContext.getBean("colorFactoryBean");
		System.out.println("bean的类型："+bean2.getClass());
		System.out.println(bean2 == bean3);
		//加上`&`就能获取ColorFactoryBean对象本身;
		Object bean4 = applicationContext.getBean("&colorFactoryBean");
		System.out.println(bean4.getClass());
```

>   * 给容器中注册组件的四种方式
>        * 包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
>        * @Bean[导入的第三方包里面的组件]
>        * @Import[快速给容器中导入一个组件]
>             * @Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
>             * ImportSelector:返回需要导入的组件的全类名数组；
>             * ImportBeanDefinitionRegistrar:手动注册bean到容器中
>        * 使用Spring提供的 FactoryBean（工厂Bean）;
>             * 默认获取到的是工厂bean调用getObject创建的对象
>             * 要获取工厂Bean本身，我们需要给id前面加一个&(即&colorFactoryBean)

## 2.2.生命周期

> * 指定初始化和销毁方法；
>   * 通过`@Bean`指定`init-method`和`destroy-method`；列如:`@Bean(initMethod="init",destroyMethod="detory")`
> * 实现`InitializingBean`和`DisposableBean`接口
>   * 通过让`Bean`实现`InitializingBean`(实现`afterPropertiesSet()`方法,定义初始化逻辑).
>   * 通过让`Bean`实现`DisposableBean`(实现`destroy()`方法,定义销毁逻辑);
> * 可以使用JSR250；
>   * `@PostConstruct`:在bean创建完成并且属性赋值完成;来执行初始化方法;
>   * `@PreDestroy`:在容器销毁bean之前通知我们进行清理工作;
> * 实现`BeanPostProcessor`的接口,在bean初始化前后进行一些处理工作；
>   * `postProcessBeforeInitialization()`:在初始化之前工作
>   * `postProcessAfterInitialization()`:在初始化之后工作

```java
@Component
public class Car implements InitializingBean, DisposableBean {

  public Car() {
    System.out.println("car constructor...");
  }

  public void init() {
    System.out.println("car ... init...");
  }

  public void detory() {
    System.out.println("car ... detory...");
  }

  @Override
  public void afterPropertiesSet() throws Exception {
    System.out.println("car InitializingBean.afterPropertiesSet()");
  }

  @Override
  public void destroy() throws Exception {
    System.out.println("car DisposableBean.destroy()");
  }

  @PostConstruct
  public void postConstruct(){
    System.out.println("postConstruct()");
  }

  public void preDestroy(){
    System.out.println("preDestroy()");
  }
  
  @Override
  public String toString() {
    return "Car{}";
  }
  
}
```

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// TODO Auto-generated method stub
		System.out.println("postProcessBeforeInitialization..."+beanName+"=>"+bean);
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		// TODO Auto-generated method stub
		System.out.println("postProcessAfterInitialization..."+beanName+"=>"+bean);
		return bean;
	}
}
```

得到结果如下:

```shell
car constructor...
postProcessBeforeInitialization...car=>Car{}
postConstruct()
car InitializingBean.afterPropertiesSet()
car ... init...
postProcessAfterInitialization...car=>Car{}
car DisposableBean.destroy()
car ... detory...
```

![image-20211211225904706](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211211225904706.png)

注意BeanProcessor是一个非常关键的接口,框架源码中很多地方都实现了该接口,

观察Spring源码可以发现如下:

```java
populateBean(beanName, mbd, instanceWrapper);//给bean进行属性赋值
initializeBean()
{
	applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);//postProcessBeforeInitialization
	invokeInitMethods(beanName, wrappedBean, mbd);//执行自定义初始化
	applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);//postProcessAfterInitialization
}
```

## 2.3.属性赋值

### 2.3.1.`@Value`与`@PropertySource`

```java
//使用@PropertySource读取外部配置文件中的k/v保存到运行的环境变量中;加载完外部的配置文件以后使用${}取出配置文件的值
@PropertySource(value={"classpath:/person.properties"})
@Configuration
public class MainConfigOfPropertyValues {
	
	@Bean
	public Person person(){
		return new Person();
	}
}
```

```java
public class Person {
	//使用@Value赋值；
	//1:基本数值;
	//2:可以写SpEL:#{};
	//3:可以写${};取出配置文件properties中的值（在运行环境变量里面的值）
	@Value("张三")
	private String name;
	@Value("#{20-2}")
	private Integer age;
	@Value("${person.nickName}")
	private String nickName;
  
}
```

```properties
person.nickName=李四
```

## 2.4.自动装配

>* `@Autowired`:自动注入:
>  * 默认优先按照类型去容器中找对应的组件:`applicationContext.getBean(XXX.class)`;找到就赋值;
>  * 如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查`applicationContext.getBean("xxxBeanName")`;
>  * 自动装配默认一定要将属性赋值好,没有就会报错;可以使用`@Autowired(required=false)`;
>* `@Qualifier("xxxBeanName")`:使用@Qualifier指定需要装配的组件的id，而不是使用属性名;
>  * 它给类成员注入时候不能单独使用,需要配合`@Autowired`,但是给方法参数注入时候可以;
>* `@Primary`:让Spring进行自动装配的时候，默认使用的首选的bean.也可以继续使用`@Qualifier`指定需要装配的bean的名字
>
>
>
>* `@Resource`:
>  * 可以和`@Autowired`一样实现自动装配功能；默认是按照组件名称进行装配的；
>  * 没有能支持`@Primary`功能没有支持`@Autowired(reqiured=false)`;
>* `@Inject`:
>  * 需要导入`javax.inject`的包，和`@Autowired`的功能一样。没有`required=false`的功能；
>
>
>
>* ` @Autowired`:构造器,参数,方法,属性;都是从容器中获取参数组件的值;
>  * **标注在方法位置**: `@Bean`+方法参数;参数从容器中获取;默认不写`@Autowired`效果是一样的;都能自动装配;
>  * **标在在构造器位置**如果组件只有一个有参构造器,这个有参构造器的`@Autowired`可以省略,参数位置的组件还是可以自动从容器中获取;
>  * **标注在参数位置**

`AutowiredAnnotationBeanPostProcessor`:解析完成自动装配功能；    

# 3.Spring的整体概览

## 3.1.核心组件接口

* 基础接口:
  * `Resource`+`ResourceLoader`;
  * `BeanFactory`;
  * `BeanDefinition`;
  * `BeanDefinitionReader`;
  * `BeanDefinitionRegistry`;
  * `ApplicationContext`;
  * `Aware`;
* 生命周期-后置处理器:
  * `BeanFactoryPostProcessor`;
  * `InitializingBean`;
  * `BeanPostProcessor`;

##  3.2.`Resource`和`ResourceLoader`

Resource层次结构目录如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211214204330656.png" alt="image-20211214204330656" style="zoom:50%;" />

ResourceLoader层次结构目录如下:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211214204924421.png" alt="image-20211214204924421" style="zoom: 50%;" />

ResourceLoader是用来获取Resource的,

## 3.3.`BeanFactory`

查看BeanFactory的继承的所有接口以及`AnnotationConfigApplicationContext`可以看到如下图:

![image-20211214214328777](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211214214328777.png)

* `BeanFactory`
  * `AutowireCapableBeanFactory`:提供自动装配能力;
    * `GenericApplicationContext` 是`AnnotationApplicationContext`的父类,也是`AbstractApplicationContext`的子类,它组合了一个`DefaultListableBeanFactory`,`AnnotationConfigApplicationContext`继承于`GenericApplicationContext`,那么`AnnotationConfigApplicationContext`也有了自动装配能力;
  * `HierachicalBeanFactory`:定义了父子工厂,父子容器;
  * `ListableBeanFactory`:实现是`DefaultListableBeanFactory`,保存了ioc容器中的核心信息;

观察BeanFactory下的抽象类`AbstractApplicationContext`,有属性`ResourcePatternResolver`,那么`AbstractApplicationContext`实际上也组合`ResourceLoader`资源加载类;

![image-20211214215955871](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211214215955871.png)

`GenericApplicationContext` 是`AnnotationApplicationContext`的父类,也是`AbstractApplicationContext`的子类,它组合了一个`DefaultListableBeanFactory`

![image-20211220194006873](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220194006873.png)

`DefaultListableBeanFactory`是一个非常重要的保存`BeanDefinition`的地方,它实现了`BeanDefinitionRegistry`;

![image-20211220200730021](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220200730021.png)

如下是`DefaultListableBeanFactory`中重要的`BeanDifinition`池

![image-20211220201344065](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220201344065.png)

## 3.4.`BeanDefinition`放入档案馆的过程

![image-20211225211920030](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225211920030.png)

在`DefaultListableBeanFactory`的`registerBeanDefinition()`中,可以看到将`beanDefinition`放入到`beanDefinitionMap`的档案馆中;

![image-20211220211911522](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220211911522.png)

给上述代码打上断点,debug调试可以得到如下图的栈帧信息,从而得到整个流程,如下是XML的方式,如果是注解的方式是不一样的;

![image-20211220212950197](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220212950197.png)

1. `AbstractApplicationContext.refresh()`:刷新容器;

![image-20211225202259817](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225202259817.png)

1. `AbstractRefreshableApplicationContext.refreshBeanFactory()`:在方法中创建`DefaultListableBeanFactory`即档案馆并加载`beanDefinition`;

![image-20211220213717189](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220213717189.png)

3. `AbstractXmlApplicationContext.loadBeanDefinitions()`:创建了一个`XmlBeanDefinitionReader`,该类组合了一个`ResourceLaoder`,并用该类去加载`beanDefinition`;

![image-20211220214058561](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220214058561.png)

4. `AbstractBeanDefinitionReader`的`loadBeanDefinitions()`方法通过ResourceLoader获取Resource,得到Resource之后,再根据Resource加载BeanDefini

![image-20211220214904062](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220214904062.png)

5. `XmlBeanDefinitionReader.doLoadBeanDefinitions()`通过dom解析得到Document;

![image-20211220215542309](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220215542309.png)

7. `DefaultBeanDefinitionDeocumentReader.doRegisterBeanDefinitions()`解析Documment

![image-20211220220458280](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220220458280.png)

`this.delegate`是`BeandDefinitionParserDelegate`;

8. `BeanDefinitionReaderUtils.registerBeanDefinition()`中`BeanDefintionRegistry.registerBeanDefinition()`;

![image-20211220221328076](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220221328076.png)

9.`DefaultListableBeanFactory.registerBeanDefinition()`;

## 3.5.`ApplicationContext`

![image-20211220223429897](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220223429897.png)

ApplicationContext包含:

1. IOC事件派发器;
2. 国际化解析功能;
3. 资源解析功能;
4. Bean工厂功能;

## 3.6.`Aware`与`BeanPostProcessor`

![image-20211220223834665](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220223834665.png)

![image-20211220225628612](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220225628612.png)

![image-20211220225702699](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220225702699.png)

1. `AbstractApplicationContext.refresh()`,初始化所有的不是懒加载的单实例bean

![image-20211225212927002](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225212927002.png)

2. `DefaultListableBeanFactory.preInstantiateSingletons()`:循环遍历beanDefinitionNames来获取bean;

![image-20211220233639995](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220233639995.png)

3. `AbstractBeanFactory.doGetBean()`:先检查IOC容器中有没有,然后再创建Bean,在创建Bean之前还看了该Bean是否依赖于其它Bean,这里的依赖专指deponds-on,如果依赖了就创建依赖的Bean;

![image-20211225214241364](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225214241364.png)

4. `AbstractBeanFactory.doCreateBean()`;创建Bean的关键;
5. `DefaultSingletonBeanRegistry.getSingleton()`;

![image-20211220234137566](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220234137566.png)

其中`singletonObjects`就是单例对象缓存池

![image-20211220234216429](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220234216429.png)

6. `AbstractAutowireCapableBeanFactory.doCreateBean()`

![image-20211225215549961](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225215549961.png)

7. `AbstractAutowireCapableBeanFactory.createBeanInstance()`

![image-20211225215751075](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225215751075.png)

8. `AbstractAutowireCapableBeanFactory.instantiateBean()`获取初始化策略创建对象;

![image-20211225220129074](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225220129074.png)

9. `SimpleInstantiationStrategy.instantiate()`创建对象;

上述都是创建Person对象的过程,释放断点到`setApplicationContext()`

![image-20211220235612819](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211220235612819.png)

10. `AbstractAutowireCapableBeanFactory.doCreateBean()`,对象创建完之后,开始执行`initializeBean()`;

![image-20211225220713214](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225220713214.png)

11. `AbstractAutowireCapableBeanFactory.initializeBean()`

![image-20211225220850021](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225220850021.png)

12. `AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization()`;

13. `ApplicationContextAwareProcessor.postProcessBeforeInitialization()`;`ApplicationContextAwareProcessor`是`BeanPostProcessor`的实现类;

![image-20211225221121894](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211225221121894.png)

14. `ApplicationContextAwareProcessor.invokeAwareInterfaces()`;执行了`Aware`接口的回调;  

![image-20211221000240522](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211221000240522.png)

**Bean的功能增强全部都是`BeanPostProcessor`+`InitializingBean`完成的**;

  ## 3.7.`@Autowired`与`BeanPostProcessor`

断点放到`setPerson()`使用`@Autowired`注入的方法上;

![image-20211226172036771](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226172036771.png)

![image-20211226170004492](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226170004492.png)

1. `AbstractApplicationContext.refresh()`

![image-20211221210910131](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211221210910131.png)

2. `DefaultListableBeanFactory.preInstantiateSingletons()`预先初始化所有的非懒加载的单实例bean;

`MyBatis`和`Spring`的整合用的`SqlSessionFactoryBean`,就是一个`FactoryBean`,它注入到IOC容器并不是`SqlSessionFactoryBean`,而是`SqlSessionFactory`

![image-20211226170333232](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226170333232.png)

3. `AbstractAutowireCapableBeanFactory.populateBean()`进行属性的赋值;

![image-20211226172509205](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226172509205.png)

当我们使用`@Autowired`进行属性赋值的时候,我们是使用的`AutowiredAnnotationBeanPostProcessor`后置处理器;

![image-20211226171221725](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226171221725.png)

4. 最终debug查到如果方法或者属性上有`@Autowired`,`@Value`,`@Inject`注解就会执行该后置处理器;

![image-20211226174547159](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226174547159.png)



断点放到新的类的`setName()`方法上,采用的是xml的属性赋值

![image-20211226172347393](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226172347393.png)

![image-20211226172358150](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226172358150.png)

1. `AbstractAutowireCapableBeanFactory.populateBean()`

![image-20211226172634338](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211226172634338.png)

