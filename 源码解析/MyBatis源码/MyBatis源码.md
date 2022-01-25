# MyBatis源码

 # 1.MyBatis源码

## 1.1.MyBatis的使用流程

~~~java
public class MybatisTest {
    public static void main(String[] args) throws IOException {
        //************** 第一阶段 **************
        // 第一步，读取mybaits-config.xml配置文件
        InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");

        // 第二步，构建SqlSessionFactory
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        //************** 第二阶段 **************
        // 第三步，打开SqlSession
        SqlSession session = sqlSessionFactory.openSession();

        // 第四步，获取Mapper接口对象
        UserMapper userMapper = session.getMapper(UserMapper.class);

        //************** 第三阶段 **************
        // 第五步，调用Mapper接口对象的方法操作数据库
        List<User> users = userMapper.selectUsersByArray();

        //List<User> users = session.selectList("com.study.mybatis.mapper.UserMapper.selectUsersByArray");

        // 第六步，业务处理
        System.out.println(users.size());
    }
~~~

如下是SqlSessionFactoryBuilder的源码,我们通过InputStream返回一个DefaultSqlSessionFactory

![image-20210527161233675](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210527161233675.png)

如下是DefaultSqlSessionFactory的源码,源码中通过调用`openSessionFromDataSource`来开启事务,创建Executor,创建DefaultSqlSession;

![image-20210527162058596](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210527162058596.png)

`session.getMapper()`最终调用到`MapperRegistry`

## 1.2.实现思路

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210527142521486.png" alt="image-20210527142521486" style="zoom: 50%;" />

1. 创建SqlSessionFactory实例;
2. 实例化过程中加载配置文件创建Configuration对象;
3. 通过factory创建SqlSession;
4. 通过SqlSession获取mapper接口动态代理;
5. 动态代理回调SqlSession中的某个具体方法;
6. SqlSession将执行方法转发给Executor;
7. Executor基于jdbc访问数据库获取数据;
8. Executor通过反射将数据转成POJO返回给SqlSession;
9. 将数据返回给调用者;

## 1.3.MyBatis架构设计

![image-20210621155511464](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210621155511464.png)

## 1.4.MyBatis的层次结构

![image-20210621155813228](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210621155813228.png)

## 1.5.MyBatis核心组件

| 组件名           | 功能                                                         |
| ---------------- | ------------------------------------------------------------ |
| Configuration    | MyBatis所有的配置信息都维持在Configuration对象之中           |
| SqlSession       | 作为MyBatis工作的主要顶层API,表示和数据库交互的会话,完成必要数据库增删盖茶功能; |
| Executor         | MyBatis执行器,是MyBatis调度的核心,负责SQL语句的生成和查询缓存的维护; |
| StatementHandler | 封装了JDBC Statement操作,负责对JDBC statement的操作,如设置参数,将Statement结果集转换成List集合 |
| ParameterHandler | 负责对用传递的参数转换成JDBC Statement所需要的参数           |
| ResultSetHandler | 负责将JDBC返回的ResultSet结果集对象转成List类型的集合        |
| MappedStatement  | MappedStatement维护了一条select update delete insert节点的封装 |
| MapperProxy      | Mapper代理,使用原生的Proxy执行mapper里的方法                 |

## 1.6.源码流程

![image-20210621232248661](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210621232248661.png)

## 1.7.源码中的设计模式

| 模式         | MyBatis                                            |
| ------------ | -------------------------------------------------- |
| 建造者模式   | SessionFactoryBuilder,Environment                  |
| 工厂方法模式 | SqlSessionFactory,TransactionFactory,LogFactory    |
| 单例模式     | ErrorContext,LogFactory                            |
| 代理模式     | MapperProxy,ConnectionLogger,executor.loader包     |
| 组合模式     | SqlNode和各个子类ChooseSqlNode                     |
| 模板方法模式 | BaseExecutor和SimpleExecutor,BaseTypeHandler和子类 |
| 适配器模式   | Log的MyBatis接口和它对JDBC,Log4j等日志框架适配实现 |
| 装饰器模式   | Cache包中cache.decorators子包中的各个装饰器的实现  |
| 迭代器模式   | PropertyTokenizer                                  |

# 2.MyBatis插件

MyBatis插件Interceptor又称拦截器.

MyBatis采用责任链模式,通过动态代理组织多个插件(拦截器),通过插件可以改变MyBatis的默认行为(诸如SQL重写之类);

## 2.1.MyBatis四大核心接口对象

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210621235220217.png" alt="image-20210621235220217" style="zoom:50%;" />

MyBatis允许在已映射语句执行过程中的某一点进行拦截调用.默认情况下,MyBatis允许使用插件来拦截四大对象:

* Executor:执行增删改查操作;
* StatementHandler:处理SQL语句预编译,设置参数等相关工作;
* ParamentHandler:设置预编译参数用;
* ResultSetHandler:处理结果集;

![image-20210622153813630](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622153813630.png)

![image-20210622154236713](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622154236713.png)

## 2.2.插件Interceptor

MyBatis定义插件要实现Interceptor接口,

![image-20210622153701950](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622153701950.png)

这个接口的三个方法:

* setProperties:在Mybatis进行配置插件的时候可以配置自定义相关属性;
* plugin:插件用于封装目标对象,通过该方法可以返回目标对象本身,也可以返回一个它的代理,可以决定是否要进行拦截而决定要返回一个什么样的目标对象;
* 要进行拦截的时候执行的方法;

## 2.3.插件配置注解@Intercepts

~~~java
@Intercepts(@Signature(type = StatementHandler.class,method="prepare",args={Connection.class,Integer.class}))
~~~

* `@Intercepts`:注解;
* `@Signature`:对插件需要拦截的对象进行签名;
* `type`:表示要拦截的类型,method表示拦截类中的方法;
* `args`:是需要的参数;
* `statementHandler`:数据库会话器专门用于处理数据库会话;
* `method`:这里method的`preapre`是StatementHandler中的方法;

只有通过Intercepts注解指定的方法才会执行自定义插件的intercept方法;

## 2.4.插件开发方式

### 2.4.1.官方插件开发方式

![image-20210622160445271](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622160445271.png)

### 2.4.2.自定义插件开发方式

![image-20210622160621858](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622160621858.png)

## 2.5.自定义分页插件

~~~java
@Intercepts(@Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class}
))
public class FechinPagePlugin implements Interceptor {
    // 插件的核心业务
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        /**
         * 1、拿到原始的sql语句
         * 2、修改原始sql，增加分页  select * from t_user limit 0,3
         * 3、执行jdbc去查询总数
         */
        // 从invocation拿到我们StatementHandler对象
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        // 拿到原始的sql语句
        BoundSql boundSql = statementHandler.getBoundSql();
        String sql = boundSql.getSql();
        System.out.println("原始sql：" + sql);

        // 分页参数
        Object paramObj = boundSql.getParameterObject();

        // statementHandler 转成 metaObject
        MetaObject metaObject = SystemMetaObject.forObject(statementHandler);

        // spring context.getBean("userBean")
        MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        // 获取mapper接口中的方法名称  selectUserByPage
        String mapperMethodName = mappedStatement.getId();
        if (mapperMethodName.matches(".*ByPage$")) {
            Map<String, Object> params = (Map<String, Object>) paramObj;

            PageInfo pageInfo = (PageInfo) params.get("page"); // map.put("page", PageInfo);
            //  select * from user;
            String countSql = "select count(0) from (" + sql + ") a";
            System.out.println("查询总数的sql : " + countSql);

            // 执行jdbc操作
            Connection connection = (Connection) invocation.getArgs()[0];
            PreparedStatement countStatement = connection.prepareStatement(countSql);
            ParameterHandler parameterHandler = (ParameterHandler) metaObject.getValue("delegate.parameterHandler");
            parameterHandler.setParameters(countStatement);
            ResultSet rs = countStatement.executeQuery();
            if (rs.next()) {
                pageInfo.setTotalNumber(rs.getInt(1));
            }
            rs.close();
            countStatement.close();

            // 改造sql limit
            String pageSql = this.generaterPageSql(sql, pageInfo);
            System.out.println("分页sql：" + pageSql);

            metaObject.setValue("delegate.boundSql.sql", pageSql);

        }
        // 把执行流程交给mybatis
        return invocation.proceed();
    }

    // 把自定义的插件加入到mybatis中去执行
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    // 设置属性
    @Override
    public void setProperties(Properties properties) {

    }

    // 根据原始sql 生成 带limit sql
    public String generaterPageSql(String sql, PageInfo pageInfo) {
        StringBuffer sb = new StringBuffer();
        sb.append(sql);
        sb.append(" limit " + pageInfo.getStartIndex() + " , " + pageInfo.getTotalSelect());
        return sb.toString();
    }
}
~~~

最后需要在mybatis-config中加入插件

![image-20210622162717650](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622162717650.png)

## 2.6.MyBatis的读写分离插件

如果后台结构是Spring+MyBatis.可以通过Spring的AbstractRountingDataSource和MyBatis Plugin拦截器来实现读写分离;

### 2.6.1.MyBatis Plugin

~~~java
@Intercepts({// mybatis 执行流程
	@Signature(type = Executor.class, method = "update", args = { MappedStatement.class, Object.class }),
	@Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class }) 
})
public class DynamicPlugin implements Interceptor {
	private Logger log = LoggerFactory.getLogger(DynamicPlugin.class);

	private static final Map<String, DynamicDataSourceGlobal> cacheMap = new ConcurrentHashMap<>();

	@Override
	public Object intercept(Invocation invocation) throws Throwable {
		Object[] objects = invocation.getArgs();
		MappedStatement ms = (MappedStatement) objects[0];

		DynamicDataSourceGlobal dynamicDataSourceGlobal = null;

		if ((dynamicDataSourceGlobal = cacheMap.get(ms.getId())) == null) {
			// 读方法
			if (ms.getSqlCommandType().equals(SqlCommandType.SELECT)) { // select * from user;    update insert
				// !selectKey 为自增id查询主键(SELECT LAST_INSERT_ID() )方法，使用主库
				if (ms.getId().contains(SelectKeyGenerator.SELECT_KEY_SUFFIX)) {
					dynamicDataSourceGlobal = DynamicDataSourceGlobal.WRITE;
				} else {
					// 负载均衡，针对多个读库
					dynamicDataSourceGlobal = DynamicDataSourceGlobal.READ;
				}
			} else {
				// 写方法
				dynamicDataSourceGlobal = DynamicDataSourceGlobal.WRITE;
			}
			System.out.println("---------------------------------------");
			System.out.println("方法[{"+ms.getId()+"}] 使用了 [{"+dynamicDataSourceGlobal.name()+"}] 数据源, SqlCommandType [{"+ms.getSqlCommandType().name()+"}]..");
			//log.warn("设置方法[{}] use [{}] Strategy, SqlCommandType [{}]..", ms.getId(), dynamicDataSourceGlobal.name(), ms.getSqlCommandType().name());

			// 把id(方法名)和数据源存入map，下次命中后就直接执行
			cacheMap.put(ms.getId(), dynamicDataSourceGlobal);
		}
		// 设置当前线程使用的数据源
		DynamicDataSourceHolder.putDataSource(dynamicDataSourceGlobal);

		return invocation.proceed();
	}

	@Override
	public Object plugin(Object target) {
		if (target instanceof Executor) {
			return Plugin.wrap(target, this);
		} else {
			return target;
		}
	}

	@Override
	public void setProperties(Properties properties) {
		//
	}
}
~~~

~~~java
public final class DynamicDataSourceHolder {

	// 使用ThreadLocal记录当前线程的数据源key
    private static final ThreadLocal<DynamicDataSourceGlobal> holder = new ThreadLocal<DynamicDataSourceGlobal>();

    private DynamicDataSourceHolder() {
        // ......
    }

    /**
     * 设置数据源
     * 
     * @param dataSource
     */
    public static void putDataSource(DynamicDataSourceGlobal dataSource){
        holder.set(dataSource);
    }

    /**
     * 获取数据源
     * @return
     */
    public static DynamicDataSourceGlobal getDataSource(){
        return holder.get();
    }

    /**
     * 清理数据源
     */
    public static void clearDataSource() {
        holder.remove();
    }

}

public enum DynamicDataSourceGlobal {
    READ, WRITE;
}
~~~

### 2.6.2.AbstractRountingDataSource

~~~java
/**
 * 获取数据源，用于动态切换数据源
 * <p>
 * <pre>
 * 实现Spring提供的AbstractRoutingDataSource，只需要实现determineCurrentLookupKey方法即可，
 * 由于DynamicDataSource是单例的，线程不安全的，所以采用ThreadLocal保证线程安全，由DynamicDataSourceHolder完成。
 * </pre>
 *
 */
public class DynamicDataSource extends AbstractRoutingDataSource {
    // 写数据源 3306
    private Object writeDataSource;
    // 读数据源 3307
    private Object readDataSource;

    @Override
    public void afterPropertiesSet() {
        if (this.writeDataSource == null) {
            throw new IllegalArgumentException("Property 'writeDataSource' is required");
        }
        // 覆盖AbstractRoutingDataSource中的默认数据源为 write
        setDefaultTargetDataSource(writeDataSource);
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DynamicDataSourceGlobal.WRITE.name(), writeDataSource);
        if (readDataSource != null) {
            // 如果读数据源不为空就压入map
            targetDataSources.put(DynamicDataSourceGlobal.READ.name(), readDataSource);
        }
        // 设置目标数据源为map中压入的数据源
        setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    /**
     * 获取与数据源相关的key 此key是Map<String,DataSource> resolvedDataSources 中与数据源绑定的key值
     * 在通过determineTargetDataSource获取目标数据源时使用
     */
    @Override
    protected Object determineCurrentLookupKey() {
        // 使用DynamicDataSourceHolder保证线程安全，并且得到当前线程中的数据源key
        DynamicDataSourceGlobal dynamicDataSourceGlobal = DynamicDataSourceHolder.getDataSource();

        if (dynamicDataSourceGlobal == null || dynamicDataSourceGlobal == DynamicDataSourceGlobal.WRITE) {
            return DynamicDataSourceGlobal.WRITE.name();
        }

        return DynamicDataSourceGlobal.READ.name();
    }

    // get and set method

    public void setWriteDataSource(Object writeDataSource) {
        this.writeDataSource = writeDataSource;
    }

    public Object getWriteDataSource() {
        return writeDataSource;
    }

    public Object getReadDataSource() {
        return readDataSource;
    }

    public void setReadDataSource(Object readDataSource) {
        this.readDataSource = readDataSource;
    }
}
~~~

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context" 
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
       http://www.springframework.org/schema/tx
       http://www.springframework.org/schema/tx/spring-tx-4.1.xsd
       http://www.springframework.org/schema/context 
	   http://www.springframework.org/schema/context/spring-context-4.1.xsd">

	<context:component-scan base-package="com.study" />

	<bean id="abstractDataSource" abstract="true" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
		<property name="driverClassName" value="com.mysql.jdbc.Driver" />
		<!-- 配置初始化大小、最小、最大 -->
		<property name="initialSize" value="1" />
		<property name="minIdle" value="10" />
		<property name="maxActive" value="10" />
		<!-- 配置获取连接等待超时的时间 -->
		<property name="maxWait" value="60000" />
	</bean>
	
	<!-- 写库 -->
	<bean id="dataSourceWrite" parent="abstractDataSource">
		<!-- 基本属性 url、user、password -->
		<property name="url" value="jdbc:mysql://mysql.study.com:3306/mybatis?serverTimezone=UTC&amp;characterEncoding=utf-8&amp;autoReconnect=true&amp;allowMultiQueries=true" />
		<property name="username" value="root" />
		<property name="password" value="root" />
	</bean>
	
	<!-- 读库 -->
	<bean id="dataSourceRead" parent="abstractDataSource">
		<property name="url" value="jdbc:mysql://mysql.study.com:3307/mybatis?serverTimezone=UTC&amp;characterEncoding=utf-8&amp;autoReconnect=true&amp;allowMultiQueries=true" />
		<property name="username" value="root" />
		<property name="password" value="root" />
	</bean>

	<!-- 动态数据源 -->
	<bean id="dataSource" class="com.study.rwdb.DynamicDataSource">
		<property name="writeDataSource" ref="dataSourceWrite"></property>
		<property name="readDataSource" ref="dataSourceRead"></property>
	</bean>

	<!-- 配置sqlSessionFactory -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<!-- 实例化sqlSessionFactory时需要使用上述配置好的数据源以及SQL映射文件 -->
		<property name="dataSource" ref="dataSource" />
		<!-- 所有配置的mapper文件 -->
		<property name="mapperLocations" value="classpath:com/study/mapper/*.xml" />
		<!-- 所有的实体 -->
		<property name="typeAliasesPackage" value="com.study.entity" />
		<property name="plugins">
			<array>
				<!-- 分页插件 -->
				<bean class="com.github.pagehelper.PageInterceptor">
					<!-- 这里的几个配置主要演示如何使用，如果不理解，一定要去掉下面的配置 -->
					<property name="properties">
						<value>
							helperDialect=mysql
							reasonable=true
							supportMethodsArguments=true
							params=count=countSql
							autoRuntimeDialect=true
						</value>
					</property>
				</bean>

				<!-- 读写分离插件 -->
				<bean class="com.study.rwdb.DynamicPlugin" />
			</array>
		</property>
	</bean>

	<!-- 配置扫描器 -->
	<!--<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">-->
	<bean class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
		<!-- 扫描包以及它的子包下的所有映射接口类 -->
		<property name="basePackage" value="com.study.mapper" />
		<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
	</bean>

</beans>
~~~

# 3.MyBatis缓存

## 3.1.MyBatis缓存结构

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622212435491.png" alt="image-20210622212435491" style="zoom:50%;" />

* 一级缓存时SqlSession级别的缓存.在操作数据库时需要构造SqlSession对象,在对象中有一个数据结构HashMap用于存储缓存数据.不同的SqlSession之间的缓存数据区域(HashMap)是互不影响的;
* 二级缓存时Mapper级别的缓存,多个SqlSession去操作同一个Mapper的SQL语句,多个SqlSession可以共用二级缓存,二级缓存是跨SqlSession的;

### 3.1.1.MyBatis一级缓存

![image-20210622213946693](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622213946693.png)

MyBatis在开启一个数据库会话时,会创建一个新的SqlSession对象,SqlSession对象中会有一个新的Executor对象中持有一个PerpetualCache对象,当会话结束时,SqlSession对象及其内部的Executor对象还有PerpetualCache对象会一并释放掉;

1. 如果SqlSession调用了`close()`方法,会释放掉一级缓存PerpetualCache对象,一级缓存将不可用;
2. 如果Sqlsession调用了`clearCache()`方法,会清空PerpetualCache对象中的数据,但是该对象仍可用;
3. SqlSession中执行任何一个update操作(`update()`,`delete()`,`insert()`),都会清空缓存;

两次相同的查询条件,如下的条件需完全一样:

1. 传入的statementId完全一样(namespaceId+sqlId);
2. 查询时要求的结果集中的结果范围;
3. 这次查询所产生的最终要传递给java.sql.PreparedStatement的SQL语句字符串;
4. 传递给java.sql.Statement要设置的参数值;

如下是BaseExecutor缓存的源码:

![image-20210622214955089](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622214955089.png)

### 3.1.2.MyBatis二级缓存

![image-20210622215119652](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622215119652.png)

#### 3.1.2.1.MyBatis二级缓存使用

1. 在mybatis-config.xml中开启二级缓存

![image-20210622223750312](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622223750312.png)

2. 在映射文件中开启二级缓存

![image-20210622223940265](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622223940265.png)

3. MyBatis要求返回的POJO必须可序列化,也就是现实Serializable接口;

#### 3.1.2.2.二级缓存整合Redis

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622224746541.png" alt="image-20210622224746541" style="zoom:50%;" />

![image-20210622225117999](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622225117999.png)

# 4.MyBatis类型转换器

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622230030102.png" alt="image-20210622230030102" style="zoom:50%;" />

MyBatis中整个类型处理器的实现架构路上,TypeHandler接口定义了类型处理器,而TypeReference抽象类定义了一个类型引用,用于引用一个泛型类型,BaseTypeHandler是类型处理器的基础,是所有类型处理器的公共模块;

## 4.1.自定义加解密的类型转换器

1. 编写TypeHandler

~~~java
public class FechinTypeHandle implements TypeHandler {

    //private static String KEY = "123456";

    /**
     * 通过preparedStatement对象设置参数，将T类型的数据存入数据库。
     *
     * @param ps
     * @param i
     * @param parameter
     * @param jdbcType
     * @throws SQLException
     */
    @Override
    public void setParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType) throws SQLException {
        try {
            String encrypt = EncryptUtil.encode(((String) parameter).getBytes());
            ps.setString(i, encrypt);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 通过列名或者下标来获取结果数据，也可以通过CallableStatement获取数据。
    @Override
    public Object getResult(ResultSet rs, String columnName) throws SQLException {
        String result = rs.getString(columnName);
        if (result != null && result != "") {
            try {
                return EncryptUtil.decode(result.getBytes());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return result;
    }

    @Override
    public Object getResult(ResultSet rs, int columnIndex) throws SQLException {
        String result = rs.getString(columnIndex);
        if (result != null && result != "") {
            try {
                return EncryptUtil.decode(result.getBytes());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return result;
    }

    @Override
    public Object getResult(CallableStatement cs, int columnIndex) throws SQLException {
        String result = cs.getString(columnIndex);
        if (result != null && result != "") {
            try {
                return EncryptUtil.decode(result.getBytes());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return result;
    }
}
~~~

2. 在mybatis-config.xml中注册type-handler

![image-20210622235534025](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622235534025.png)

3. 在mapper文件中使用

![image-20210622235807437](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210622235807437.png)

![image-20210623000053610](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210623000053610.png)

# 5.MyBatis问题汇总

## 5.1.MyBatis常用的Executor

![image-20210623094629963](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210623094629963.png)

BaseExecutor属于抽象类,常用的Executor包括:

* SimpleExecutor:每次请求完都会关闭statement
* BatchExecutor:批量执行;
* ReuseExecutor:用map缓存statement,实现重用;

## 5.2.MyBatis延迟加载原理

MyBatis支持association关联对象和collection关联集合对象的延迟加载,在配置文件中,可以配置是否启用延迟加载`LazyLoadingEnabled=true|false`;原理是,使用cglib创建目标对象的代理对象,当调用目标方法时候,进入拦截器方法,比如调用`a.getB().getName()`,拦截器`invoke()`方法发现`a.getB()`是null值,那么就会单独发送事先保存好的查询关联B对象的SQL,把B查询上来,然后调用`a.setB(b)`,于是a的对象b属性就有值了.

## 5.3.`#{}`和`${}`区别

1. `#{}`是占位符,预编译处理,`${}`是拼接符,字符串替换,没有预编译处理;
2. `#{}`传入参数是以字符串传入,会将SQL中的`#{}`替换为`?`,PreparedStatement的set方法来赋值;
3. `#{}`可以防止SQL注入,`${}`不能防止;
4. `#{}`的变量替换是在DBMS中,`${}`的变量替换在DBMS外;

## 5.4.MyBatis分页原理

1. MyBatis使用RowBounds对象进行分页,它是针对ResultSet结果集执行的内存分页,而非物理分页;
2. 可以使用分页插件来完成物理分页;



