# SpringBoot源码 

# 1.配置

可以设置的配置参考如下:

https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties

# 2.依赖管理

## 2.1.版本控制

![image-20220119151159465](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119151159465.png)

父项目使用来做版本的依赖管理,父项目中还有父项目如下:

![image-20220119151250621](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119151250621.png)

里面用`<properties>`和`<dependencyManagement>`已经将能够用到的版本全部设置好.

![image-20220119151449408](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119151449408.png)

![image-20220119151503881](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119151503881.png)

## 2.2.Starter场景启动器

可以写什么样的Starter,可以参考官网如下:官方的starter命名一般都是`spring-boot-starter-xxxx`;非官方的starter命名一般都是`xxx-spring-boot-starter-xxx`;

https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.build-systems.starters

所有的starter的最底层的依赖是:这是SpringBoot的自动配置依赖

```xml
		<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.6.2</version>
      <scope>compile</scope>
    </dependency>
```

# 3.关键注解

## 3.1.`@Configuration`

`@Configuration`中有一个方法`proxyBeanMethods()`需要注意;

 *      Full模式`(proxyBeanMethods = true)`保证每个@Bean方法被调用多少次返回的组件都是单实例的;
 *      Lite模式`(proxyBeanMethods = false)`每个@Bean方法被调用多少次返回的组件都是新创建的;
 *      组件依赖必须使用Full模式默认,其他默认是否Lite模式;

```java
@Configuration(proxyBeanMethods = true) //告诉SpringBoot这是一个配置类 == 配置文件
public class MyConfig {

    /**
     * Full:外部无论对配置类中的这个组件注册方法调用多少次获取的都是之前注册容器中的单实例对象
     * @return
     */
    @Bean //给容器中添加组件。以方法名作为组件的id。返回类型就是组件类型。返回的值，就是组件在容器中的实例
    public User user01(){
        User zhangsan = new User("zhangsan", 18);
        //user组件依赖了Pet组件
        zhangsan.setPet(tomcatPet());
        return zhangsan;
    }

    @Bean("tom")
    public Pet tomcatPet(){
        return new Pet("tomcat");
    }
}
```

## 3.2.`@Conditional`

![image-20220119171123629](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119171123629.png)

这边需要注意:`@ConditionOnBean`与`@ConditionOnClass`的区别:

* `@ConditionOnBean`:Spring容器中需要有;
* `@ConditionOnBean`:项目中或者导的包中需要有;

 ## 3.3.`@ImportResource`

假如第三方包或者之前的非SpringBoot项目需要迁移或重构,我们完全可以通过该注解,导入配置文件;例如:`@ImportResource("classpath:beans.xml")`;

## 3.4.配置绑定

### 3.4.1.`@ConfigurationProperties`

这里是通过`@ConfigurationProperties`+`@Component`来实现属性的与类的映射配置

```java
@Component//只有在容器中的组件,才会拥有`@ConfigurationProperties`提供的功能;
@ConfigurationProperties(prefix = "mycar")
@Data
public class CarProperties {
    private String brand;
    private Integer price;
}
```

### 3.4.2.`@EnableConfigurationProperties`

在配置类中`@EnableConfigurationProperties(User.class)`,指定属性类为`User`;

```java
@Configuration
@EnableConfigurationProperties(User.class)
public class SpringBootConfiguration {

}
```

给`User`类加上`@ConfigurationProperties(prefix = "user")`

```java
@ConfigurationProperties(prefix = "user")
public class User {

  private String name;
}
```

# 4.自动配置原理

## 4.1.`@SpringBootApplication`

该注解的层次结构如下:

* `@SpringBootApplication`
  * `@SpringBootConfiguration`
    * `@Configuration`
  * `@EnableAutoConfiguration`
    * `@AutoConfigurationPackage`
      * `@Import(AutoConfigurationPackages.Registrar.class)`
    * `@Import(AutoConfigurationImportSelector.class)`
  * `@ComponentScan`

从上面的层次结构可以看出:如果加上`@SpringBootApplication`,那么就是一个加上`@Configuration`的配置类,并且导入了两个类`AutoConfigurationPackages.Registrar.class`和`AutoConfigurationImportSelector.class`,并且进行了包扫描;

## 3.2.`AutoConfigurationPackages.Registrar`

我们给`Registrar`这个内部类打上断点

![image-20220118213354486](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118213354486.png)

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118213420918.png" alt="image-20220118213420918" style="zoom:50%;" />

`new PackageImports(metadata).getPackageNames()`得到的路径实际上就是启动类所在的路径;

STEP INTO到`register`方法中

![image-20220118213648068](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118213648068.png)

**在这边将SpringBoot启动类所在路径下的所有类的BeanDefinition信息进行注册**;

如下图是堆栈信息

![image-20220118214108435](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220118214108435.png)

## 3.3.`AutoConfigurationImportSelector`

`AutoConfigurationImportSelector.getAutoConfigurationEntry()`;

![image-20220119194759248](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119194759248.png)

`AutoConfigurationImportSelector.getCandidateConfigurations()`;

![image-20220119194920057](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119194920057.png)

`SpringFactoriesLoader.loadFactoryNames()`

![image-20220119195031513](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119195031513.png)

`SpringFactoriesLoader.loadSpringFactories()`

![image-20220119195128459](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119195128459.png)

利用工厂加载`loadSpringFactories(classLoader)`得到所有的组件;如下图:`factoryTypeName`实际上就是`org.springframework.boot.autoconfigure.EnableAutoConfiguration`

![image-20220119224242769](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119224242769.png)

**默认扫描当前系统里面所有`META-INF/spring.factories`位置的文件**;并且找的是`org.springframework.boot.autoconfigure.EnableAutoConfiguration`属性下的所有类;

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.r2dbc.R2dbcDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.r2dbc.R2dbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration,\
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration,\
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration,\
org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration,\
org.springframework.boot.autoconfigure.influx.InfluxDbAutoConfiguration,\
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration,\
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration,\
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\
org.springframework.boot.autoconfigure.jooq.JooqAutoConfiguration,\
org.springframework.boot.autoconfigure.jsonb.JsonbAutoConfiguration,\
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration,\
org.springframework.boot.autoconfigure.availability.ApplicationAvailabilityAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.embedded.EmbeddedLdapAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration,\
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\
org.springframework.boot.autoconfigure.neo4j.Neo4jAutoConfiguration,\
org.springframework.boot.autoconfigure.netty.NettyAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.quartz.QuartzAutoConfiguration,\
org.springframework.boot.autoconfigure.r2dbc.R2dbcAutoConfiguration,\
org.springframework.boot.autoconfigure.r2dbc.R2dbcTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketRequesterAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketServerAutoConfiguration,\
org.springframework.boot.autoconfigure.rsocket.RSocketStrategiesAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.rsocket.RSocketSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.saml2.Saml2RelyingPartyAutoConfiguration,\
org.springframework.boot.autoconfigure.sendgrid.SendGridAutoConfiguration,\
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.reactive.ReactiveOAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.resource.reactive.ReactiveOAuth2ResourceServerAutoConfiguration,\
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\
org.springframework.boot.autoconfigure.sql.init.SqlInitializationAutoConfiguration,\
org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration,\
org.springframework.boot.autoconfigure.task.TaskSchedulingAutoConfiguration,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration,\
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration,\
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.HttpHandlerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.ReactiveMultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.WebSessionIdResolverAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.error.ErrorWebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.function.client.ClientHttpConnectorAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.function.client.WebClientAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.client.WebServiceTemplateAutoConfiguration
```

但是并不是说Spring Boot就会加载所有这些类;

![image-20220119224451823](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119224451823.png)

进入到`AutoConfigurationImportSelector,filter()`方法;

![image-20220119225202259](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119225202259.png)

发现在这里就是通过`@Conditional`及其派生注解进行过滤;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119203925088.png" alt="image-20220119203925088" style="zoom:50%;" />

在`org.springframework.boot.autoconfigure`包下有大量的自动配置类,以`AopAutoConfiguration`为例;

![image-20220119204159226](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220119204159226.png)

内部类`AspectJAutoProxyingConfiguration`需要在有`Advice.class`的情况下才能生效;

**SpringBoot写好所有全场景的自动配置类,并根据具体的需求导入进来后,接下来就是Spring的事情了**;

# 5.SpringBoot Web启动流程

## 5.1.`ServletWebServerFactoryAutoConfiguration`

![image-20220120100029243](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120100029243.png)

导入了4个类:

* `ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class`;
* `ServletWebServerFactoryConfiguration.EmbeddedTomcat`;
* `ServletWebServerFactoryConfiguration.EmbeddedJetty`;
* `ServletWebServerFactoryConfiguration.EmbeddedUndertow`;

上面导入了三个服务器工厂:Tomcat,Jetty,Undertow;根据`@Conditional`判断是否导入  

![image-20220120102559126](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120102559126.png)

默认导入的是`TomcatServletWebServerFactory`,该类有`getWebServer()`方法如下:实际上就是创建Tomcat. 参数可以看到`ServletContextInitializer`

![image-20220120104631918](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120104631918.png)

## 5.2.`DispatcherServletAutoConfiguration`

`DispatcherServletAutoConfiguration`中主要做了两件事情.

第一件是往IOC容器中放入DispatcherServlet

![image-20220120142037838](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120142037838.png)

第二件是往IOC容器获取`DispatcherServlet`并往容器中放入`DispatcherServletRegistrationBean`

![image-20220120142131578](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120142131578.png)

观察`DispatcherServletRegistrationBean`,实现了`ServletContextInitializer`接口;

![image-20220120142358133](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120142358133.png)

## 5.3.`AnnotationConfigServletWebServerApplicationContext`

SpringBoot使用的容器是`AnnotationConfigServletWebServerApplicationContext`

我们可以给`ServletWebServerFactoryConfiguration`的`tomcatServletWebServerFactory()`方法打上断点:

![image-20220120113110353](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120113110353.png)

启动SpringBoot,调用链如下:

![image-20220120113130056](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120113130056.png)

在`ServletWebServerApplicationContext`中的`onRefresh()`方法中`createWebServer()`;

![image-20220120113336715](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120113336715.png)

STEP INTO

![image-20220120113856979](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120113856979.png)

这里面的`getWebServerFactory()`STEP INTO

![image-20220120113951343](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120113951343.png)

从IOC容器中获取`ServletWebServerFactory`.调用到最后其实就是调用的`@Bean`创建对象,并放在IOC容器中;

![image-20220120134758633](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120134758633.png)

回过头来看`createWebServer()`,创建好`ServletWebServerFactory`之后,通过工厂创建WebServer;

![image-20220120135456881](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120135456881.png)

STEP INTO,里面创建了一个Tomcat并设置了一些基本参数

![image-20220120135759151](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120135759151.png)

STEP INTO `prepareContext()`在该方法中`configureContext(context,initializersToUse)`定制Tomcat容器;

![image-20220120135905945](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120135905945.png)

之后调用`TomcatServletWebServerFactory.getTomcatWebServer(tomcat)`;

![image-20220120140506793](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120140506793.png)

里面创建了一个`TomcatWebServer`,STEP INTO之后可以看到内部方法中调用了`initialize()`方法,该方法中`tomcat.start()`启动了Tomcat;

![image-20220120140601691](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220120140601691.png)

