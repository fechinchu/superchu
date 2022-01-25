# 17.MySQL集群解决方案(主从复制,PXC集群,MyCat,HAProxy)

# 1.MySQL数据库的集群方案

## 1.1.读写分离架构

### 1.1.1.说明

我们一般应用对数据库而言都是"读多写少",也就是说对数据库读取数据的压力比较大.有一个思路就是说采用数据库集群的方案:其中一个是主库,负责写入数据,称之为"写库".其他都是从库,负责读取数据,我们称之为"读库".

![image-20210725004843635](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725004843635.png)

* 数据库从之前的单节点变为多节点提供服务;
* 主节点数据,同步到从节点数据;
* 应用程序需要连接到2个数据库节点,并且在程序内部实现判断读写操作.

问题:

* 应用程序需要连接到多个节点,对应用程序而言开发变得复杂

  * **这个问题可以通过中间件解决**
  * 如果在程序内部实现,可使用Spring的AOP功能实现

  ![image-20210725004850902](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725004850902.png)

* 主从指间的同步,是异步完成,也就以为这是弱一致性
  * 可能会导致,数据写入主库后,应用程序读取从库获取不到数据,或者可能导致丢失数据,对于数据安全性要求比较高的应用是不合适的;
  * **该问题可以通过PXC集群解决**

## 1.2.中间件

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725004904039.png" alt="image-20210725004904039" style="zoom:50%;" />

通过上面的架构,可以看出,应用程序会连接到多个节点,使得应用程序的复杂度会提升,可以通过中间件方式解决,如下:

从架构中,可以看出:

* 应用程序只需要连接到中间件即可,无需连接多个数据库节点;
* 应用程序无需区分读取操作,对中间件直接进行读写操作即可;
* 在中间件中进行区分读写操作,读发送到从节点,写发送到主节点;

该架构也存在问题,中间件的性能称为了系统的瓶颈,那么架构可以改造成这样.

![image-20210725004912937](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725004912937.png)

这样的话,中间件的可靠性得到了保证,但是也带来新的问题,应用系统是需要连接到2个中间件,又为应用系统带来了复杂度.

## 1.3.负载均衡

为了解决以上问题,我们将继续优化架构,在应用程序和中间件之间增加proxy代理,由代理来完成负载的功能,应用程序只需要对接到proxy即可.
![image-20210725004946964](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725004946964.png)至此,主从复制架构的高可用架构才算是搭建完成.

## 1.4.PXC集群架构

在前面的架构中,都是基于MySQL主从的架构,那么在主从架构中,弱一致性问题依然没有解决,如果在需要强一致性的需求中,显然这种架构是不能应对的.比如:交易数据.

PXC提供了读写强一致的功能,可以保证数据在任何一个节点写入的同时可以同步到其他节点,也就意味着可以存其它的任何节点进行读取操作,无延迟.

![image-20210725004954327](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725004954327.png)

## 1.5.混合架构

在前面的PXC架构中,虽然可以实现了事务的强一致性,但是它是通过牺牲了性能换来的一致性,如果在某些业务场景下,如果没有强一致性的需求,那么使用PXC就不适合了.所以,在我们的系统架构中,需要将这两种方式综合起来,这样才是一个较为完善的架构.

![image-20210725005002029](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005002029.png)

# 2.搭建主从复制架构

## 2.1主从复制原理

![image-20210725005009993](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005009993.png)

MySQL主(master)从(slave)复制原理:

* master将数据改变记录到二进制日志(binary log)中,也就是我们配置文件my.conf中的log-bin指定的文件(这些记录叫做二进制日志事件,binary log events);
* slave将master的binary log events拷贝到它的中继日志(relay log);
* slave重做中继日志中的时间,将改变反应它自己的数据(数据重演)

主从配置需要注意的地方:

* 主DB server 和从DB server数据库的版本一致;
* 主DB server和从DB server数据库数据一致;
* 主DB server开启二进制日志,主DB server和从DB server 的server_id都必须唯一.

## 2.2.搭建主库

~~~shell
#创建配置文件vim my.cnf,没事不要改my.cnf的权限,这会导致mysql忽略该配置文件
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
log-bin=mysql-bin
server-id=1
skip-name-resolve
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

#创建容器
docker create --privileged=true  --name mysql-master01 -v /opt/mysql-cluster/master01/data:/var/lib/mysql -v /opt/mysql-cluster/master01/conf:/etc/mysql/conf.d -v /opt/mysql-cluster/master01/my.cnf:/etc/mysql/my.cnf -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql

#启动
docker start mysql-master01 && docker logs -f mysql-master01

#在Navicat中创建同步账户及授权
#create user 'fechinchu'@'%' identified by '320512';这一行会出现2061异常
#https://forums.mysql.com/read.php?26,663846,663880#msg-663880
CREATE USER 'fechinchu'@'%' IDENTIFIED WITH 'mysql_native_password' BY '320512';
grant replication slave on *.* to 'fechinchu'@'%';
flush privileges;

#出现 [Err] 1055 -Expression ...在my.cnf配置文件中设置
sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

#查看master状态,结果见下图
show master status;

#查看二进制日志相关的配置项
show global variables like 'binlog%';

#查看server相关的配置项
show global variables like 'server%';
~~~

master状态

![image-20210725005627785](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005627785.png)

## 2.3.搭建从库

~~~shell
#创建配置文件vim my.cnf
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
server-id=2
skip-name-resolve
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

#创建容器
docker create --privileged=true --name mysql-slave01 -v /opt/mysql-cluster/slave01/data:/var/lib/mysql -v /opt/mysql-cluster/slave01/conf:/etc/mysql/conf.d -v /opt/mysql-cluster/slave01/my.cnf:/etc/mysql/my.cnf -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root mysql

#启动
docker start mysql-slave01 && docker logs -f mysql-slave01

#在Navicat中设置master相关信息
CHANGE MASTER TO 
 master_host='172.16.124.131',
 master_user='fechinchu',
 master_password='320512',
 master_port=3306,
 master_log_file='mysql-bin.000003',
 master_log_pos=881
 
#启动同步
start slave;

#查看slave状态,结果见下图
show slave status;
~~~

![image-20210725005021433](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005021433.png)

搭建成功

## 2.4.主从复制模式

![image-20210725005030222](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005030222.png)

在查看二进制日志相关参数内容中,会发现默认的模式为ROW,其实在MySQL中提供了3种模式,基于SQL语句的复制(Statement-based replication,SBR),基于行的复制(row-based replication,RBR),混合复制模式(mixed-based replication,MBR).对应的,binlog的格式也有三种:STATEMENT,ROW,MIXED;

### 2.4.1.STATEMENT(SBR)

每一条会修改数据的SQL语句会记录到binlog中.

* 优点是并不需要记录每一条SQL语句和每一行的数据变化,减少了binlog的日志量,节约IO,提高性能;
* 缺点是在某些情况下会导致master-slave中的数据不一致(如sleep()函数,last_insert_id(),以及user-defined functions会出现问题);

### 2.4.2.ROW(RBR)

不记录每条SQL语句的上下文信息,仅记录哪条数据被修改了,修改成什么样了,而且不会出现某些特定情况下的存储过程,或funtion,或trigger的调用和触发无法被正确复制的问题.缺点就是会产生大量的日志,尤其是alter table会让日志暴涨;

### 2.4.3.MIXED(MBR)

以上两种模式的混合使用,一般的复制使用STATEMENT模式保存binlog,对于STATEMENT模式无法复制的操作使用ROW模式保存binlog,MySQL会根据执行的SQL语句选择日志保存方式;

建议使用MIXED

~~~shell
#主库的配置添加
binlog_format=MIXED

#重启主库后在navicat中查看相关配置
SHOW GLOBAL VARIABLES LIKE 'binlog%'; 
~~~

![image-20210725005047666](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005047666.png)

# 3.MyCat中间件

![image-20210725005056614](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005056614.png)

## 3.1.读写分离

| 主机           | 端口 | 容器名称       | 角色   |
| -------------- | ---- | -------------- | ------ |
| 172.16.124.131 | 3306 | mysql-master01 | master |
| 172.16.124.131 | 3307 | mysql-slave01  | slave  |

在mycat的安装目录下的conf/server.xml,**将该配置文件中的内容全部替换成以下内容**

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
    <system>
        <property name="nonePasswordLogin">0</property>
        <property name="useHandshakeV10">1</property>
        <property name="useSqlStat">0</property>
        <property name="useGlobleTableCheck">0</property>
        <property name="sequnceHandlerType">2</property>
        <property name="subqueryRelationshipCheck">false</property>
        <property name="processorBufferPoolType">0</property>
        <property name="handleDistributedTransactions">0</property>
        <property name="useOffHeapForMerge">1</property>
        <property name="memoryPageSize">64k</property>
        <property name="spillsFileBufferSize">1k</property>
        <property name="useStreamOutput">0</property>
        <property name="systemReserveMemorySize">384m</property>
        <property name="useZKSwitch">false</property>
    </system>
    <!--这里是设置的用户和虚拟逻辑库-->
		<user name="fechinchu" defaultAccount="true">
        <property name="password">320512</property>
        <property name="schemas">fechin</property>
    </user>
</mycat:server>
~~~

schema.xml,**将该配置文件中的内容全部替换成以下内容**

~~~xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd"> 
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!--配置数据表-->
    <schema name="fechin" checkSQLschema="false" sqlMaxLimit="100">
        <table name="tb_ad" dataNode="dn1" rule="mod-long" />
    </schema>
    <!--配置分片关系-->
    <dataNode name="dn1" dataHost="cluster1" database="fechin" /> 
    <!--配置连接信息-->
    <dataHost name="cluster1" maxCon="1000" minCon="10" balance="3" writeType="1" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="W1" url="172.16.124.131:3306" user="root" password="root">
            <readHost host="W1R1" url="172.16.124.131:3307" user="root" password="root" />
        </writeHost>
    </dataHost>
</mycat:schema>
~~~

> balance属性说明:负载均衡类型,目前的取值有四种:
>
> * balance="0",不开启读写分离机制,所有读操作都发送到当前可用的writeHost上.
> * balance="1",全部的readHost与stand by writeHost参与select语句的负载均衡,简单的说,当双主双从模式(M1->S1,M2->S2,并且M1与M2互为主备),正常情况下,M2,S1,S2都参与select语句的负载均衡.
> * balance="2",所有的读操作随机在writeHost,readHost上分发
> * balance="3",所有读请求随机分到对应的readHost执行,writeHost不负担读压力,注意balance=3只在1.4及其以后版本有.

rule.xml**修改内容**

~~~xml
<function name="mod-long" class="io.mycat.route.function.PartitionByMod"> 
    <property name="count">1</property>
</function>
~~~

启动

```shell
./mycat console
```

![image-20210725005108240](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005108240.png)

```shell
./startup_nowrap.sh
```

如果现实JAVA_HOME不存在,则执行如下内容

~~~shell
vi .bash_profile

#配置内容
JAVA_HOME=/usr/lib/jvm/java-1.8.0
export JAVA_HOME

source .bash_profile
~~~

![image-20210725005118044](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005118044.png)

执行成功,我们可以使用`jps`来停止MyCat

我们使用客户端连接MyCat

![image-20210725005134633](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005134633.png)

## 3.2.数据分片

### 3.2.1.架构

集群01

| 主机           | 端口 | 容器名称       | 角色   |
| -------------- | ---- | -------------- | ------ |
| 172.16.124.131 | 3306 | mysql-master01 | master |
| 172.16.124.131 | 3307 | mysql-slave01  | slave  |

集群02

| 主机           | 端口 | 容器名称       | 角色   |
| -------------- | ---- | -------------- | ------ |
| 172.16.124.131 | 3316 | mysql-master02 | master |
| 172.16.124.131 | 3317 | mysql-slave02  | slave  |

### 3.2.2.配置master

~~~shell
#创建配置文件vim my.cnf,没事不要改my.cnf的权限,这会导致mysql忽略该配置文件
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
log-bin=mysql-bin
server-id=1
skip-name-resolve
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

#创建容器
docker create --privileged=true  --name mysql-master02 -v /opt/mysql-cluster/master02/data:/var/lib/mysql -v /opt/mysql-cluster/master02/conf:/etc/mysql/conf.d -v /opt/mysql-cluster/master02/my.cnf:/etc/mysql/my.cnf -p 3316:3306 -e MYSQL_ROOT_PASSWORD=root mysql

#启动
docker start mysql-master02 && docker logs -f mysql-master02

#在Navicat中创建同步账户及授权
#create user 'fechinchu'@'%' identified by '320512';这一行会出现2061异常
#https://forums.mysql.com/read.php?26,663846,663880#msg-663880
CREATE USER 'fechinchu'@'%' IDENTIFIED WITH 'mysql_native_password' BY '320512';
grant replication slave on *.* to 'fechinchu'@'%';
flush privileges;

#出现 [Err] 1055 -Expression ...在my.cnf配置文件中设置
sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

#查看master状态,结果见下图
show master status;
~~~

![image-20210725005155432](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005155432.png)

### 3.2.3.配置slave

~~~shell
#创建配置文件vim my.cnf
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
server-id=2
skip-name-resolve
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

#创建容器
docker create --privileged=true --name mysql-slave02 -v /opt/mysql-cluster/slave02/data:/var/lib/mysql -v /opt/mysql-cluster/slave02/conf:/etc/mysql/conf.d -v /opt/mysql-cluster/slave02/my.cnf:/etc/mysql/my.cnf -p 3317:3306 -e MYSQL_ROOT_PASSWORD=root mysql

#启动
docker start mysql-slave02 && docker logs -f mysql-slave02

#在Navicat中设置master相关信息
CHANGE MASTER TO 
 master_host='172.16.124.131',
 master_user='fechinchu',
 master_password='320512',
 master_port=3316,
 master_log_file='mysql-bin.000003',
 master_log_pos=827
 
#启动同步
start slave;

#查看slave状态,结果见下图
show slave status;
~~~

### 3.2.4. 配置MyCat

schema.xml

~~~xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd"> 
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!--配置数据表-->
    <schema name="fechin" checkSQLschema="false" sqlMaxLimit="100">
        <table name="tb_ad" dataNode="dn1,dn2" rule="mod-long" />
    </schema>
    <!--配置分片关系-->
    <dataNode name="dn1" dataHost="cluster1" database="fechin" />
    <dataNode name="dn2" dataHost="cluster2" database="fechin" />
    <!--配置连接信息-->
    <dataHost name="cluster1" maxCon="1000" minCon="10" balance="3" writeType="1" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="W1" url="172.16.124.131:3306" user="root" password="root">
            <readHost host="W1R1" url="172.16.124.131:3307" user="root" password="root" />
        </writeHost>
    </dataHost>
    <dataHost name="cluster2" maxCon="1000" minCon="10" balance="3" writeType="1" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="W2" url="172.16.124.131:3316" user="root" password="root">
            <readHost host="W2R1" url="172.16.124.131:3317" user="root" password="root" />
        </writeHost>
    </dataHost>
</mycat:schema>
~~~

rule.xml

~~~xml
<function name="mod-long" class="io.mycat.route.function.PartitionByMod"> 
    <property name="count">2</property>
</function>
~~~

重新启动mycat进行测试

~~~shell
./startup_nowrap.sh && tail -f ../logs/mycat.log
~~~

![image-20210725005210900](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005210900.png)

![image-20210725005217996](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005217996.png)

![image-20210725005225024](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005225024.png)

## 3.3.MyCat集群

![image-20210725005237420](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005237420.png)

~~~shell
#我们可以复制mycat的目录

vim wrapper.conf
#设置jmx端口
wrapper.java.addition.7=-Dcom.sun.management.jmxremote.port=1985

vim server.xml
#设置服务端口以及管理端口
<property name="serverPort">8067</property>
<property name="managerPort">9067</property>

#重新启动服务
./startup_nowrap.sh
tail -f ../logs/mycat.log
~~~

# 4.负载均衡

在前面架构中,虽然对MyCat做了集群,保障了mycat的可靠性,但是,应用程序需要连接到多个mycat,显然不是很好,也就是说缺少负载均衡的组件,接下来我们来了解下HAProxy

## 4.1.haproxy简介

http://www.haproxy.org/

## 4.2.架构

![image-20210725005246948](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005246948.png)

## 4.3.部署安装

~~~shell
#拉取镜像
docker pull haproxy:1.9.3

docker create --name haproxy --net host -v /opt/haproxy:/usr/local/etc/haproxy haproxy:1.9.3
~~~

编写配置文件

~~~shell
#在/opt/haproxy目录下创建haproxy.cfg
#输入内容如下:

#创建文件
vim /opt/haproxy/haproxy.cfg
#输入如下内容 
global
    log       127.0.0.1 local2
    maxconn   4000
    daemon
    
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen admin_stats
    bind         0.0.0.0:4001
    mode         http
    stats uri    /dbs
    stats realm  Global\ statistics 
    stats auth   admin:admin123
    
listen proxy-mysql 
    bind         0.0.0.0:4002 
    mode         tcp
    balance      roundrobin
    option       tcplog
    server       mycat_1 172.16.124.131:8066 check port 8066 maxconn 2000
    server       mycat_2 172.16.124.131:8067 check port 8067 maxconn 2000
~~~

http://172.16.124.131:4001/dbs 

用户名:admin,密码:admin123

![image-20210725005255934](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005255934.png)

我们使用Navicat进行连接

![image-20210725005301901](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005301901.png)

# 5.PXC集群

## 5.1.简介

Percona XtraDB Cluster(PXC)是针对MySQL用户的高可用性和扩展性解决方案,基于Percona Server.其中包括了Write Set REPlication补丁,使用Galera 2.0库,这是一个针对事务性应用程序的同步多主机复制插件.

Percona Server是MySQL的改进版本,使用XtraDB存储引擎,在功能和性能上较MySQL有很显著的提升,如提升了高负载情况下的InnoDB的性能,为DBA提供了一些非常有用的性能诊断工具,另外有更多的参数和命令来控制服务器行为.

Percona XtraDB Cluster提供了:

* 同步复制,事务可以在所有节点上提交
* 多主机复制,可以写到任何节点;
* slave服务器上的并行应用事件.真正的"并行复制";
* 自动节点配置;
* 数据一致性,不再有未同步的服务器;

![image-20210725005311325](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005311325.png) 

## 5.2.架构

![image-20210725005318939](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005318939.png)

## 5.3.部署安装

我们部署安装三节点的PXC

| 节点   | 端口  | 容器名称   | 数据卷 |
| ------ | ----- | ---------- | ------ |
| node01 | 13306 | pxc-node01 | v1     |
| node02 | 13307 | pxc-node02 | v2     |
| node03 | 13308 | pxc_node03 | v3     |

~~~shell
#创建数据卷(数据卷的存储路径:/var/lib/docker/volumes)
docker volume create v1
docker volume create v2
docker volume create v3

#拉取镜像
docker pull percona/percona-xtradb-cluster:5.7

#重命名
docker tag percona/percona-xtradb-cluster:5.7 pxc

#创建网络
docker network create --subnet=172.30.0.0/24 pxc-network

#创建容器
#节点01
docker create -p 13306:3306 -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node01 --net=pxc-network --ip=172.30.0.2 pxc

#节点02(增加了CLUSTER_JOIN参数)
docker create -p 13307:3306 -v v2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node02 -e CLUSTER_JOIN=pxc-node01 --net=pxc-network --ip=172.30.0.3 pxc

#第三节点(增加了CLUSTER_JOIN参数)
docker create -p 13308:3306 -v v3:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node03 -e CLUSTER_JOIN=pxc-node01 --net=pxc-network --ip=172.30.0.4 pxc
#注意的是:先启动第一个节点,等MySQL客户端可以连接到服务后再启动其他节点
#查看集群节点,在Navicat,结果如下图
show status like 'wsrep_cluster%';
~~~

![image-20210725005333009](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005333009.png)

## 5.4.集群说明

* 尽可能的控制PXC集群的规模,节点越多,数据同步速度越慢;
* 所有PXC节点的硬件配置要一致,如果不一致,配置低的节点将拖慢数据同步的速度;
* PXC集群只支持InnoDB引擎,不支持其他的存储;

## 5.5.PXC集群方案与主从复制区别

* PXC集群方案所有节点都是可读可写的,Replication从节点不能写入,因为主从同步是单向的,无法从slave节点向master点同步;
* PXC同步机制是同步进行的,这也是它能保证数据强一致性的根本原因,Replication同步机制是异步进行的,它如果从节点停止同步,依然可以向主节点插入数据,正确返回,造成数据主从数据的不一致性;
* PXC是用牺牲性能保证数据的一致性,Replication在性能上是高于PXC的.所以两者用途也不一致.PXC是用于重要信息的存储,例如:订单,用户信息等.Replication用于一般信息的存储,能够容忍数据丢失,例如:购物车,用户行为日志等;

# 6.项目应用

MyCAT中间件,HAProxy负载均衡,PXC集群架构,在实际的项目中,往往不单单是一种架构,更多的使用的混合架构,下面我们将该项目采用混合架构进行完善数据库集群.

## 6.1.架构

![image-20210725005342130](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005342130.png)

说明:

* HAProxy作为负载均衡器;
* 部署了2个MyCAT节点作为数据库中间件;
* 部署了2个PXC集群节点,作为2个MyCat分片,每个PXC集群中有2个节点,作为数据的同步存储;
* 部署了1个主从复制集群;
* 房源数据保存到PXC分片中,其余数据保存到主从架构中;

## 6.2.部署PXC集群

集群1:

| 节点   | 端口  | 容器名称   | 数据卷   |
| ------ | ----- | ---------- | -------- |
| node01 | 13306 | pxc-node01 | haoke-v1 |
| node02 | 13307 | pxc-node02 | haoke-v2 |

集群2:

| 节点   | 端口  | 容器名称   | 数据卷   |
| ------ | ----- | ---------- | -------- |
| node03 | 13308 | pxc-node03 | haoke-v3 |
| node04 | 13309 | pxc-node04 | haoke-v4 |

~~~shell
#创建数据卷(数据卷的存储路径:/var/lib/docker/volumes)
docker volume create haoke-v1
docker volume create haoke-v2
docker volume create haoke-v3
docker volume create haoke-v4

#拉取镜像
docker pull percona/percona-xtradb-cluster:5.7

#重命名
docker tag percona/percona-xtradb-cluster:5.7 pxc

#创建网络
docker network create --subnet=172.30.0.0/24 pxc-network

#创建容器
#集群01节点01
docker create -p 13306:3306 -v haoke-v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node01 --net=pxc-network --ip=172.30.0.2 pxc

#集群01节点02(增加了CLUSTER_JOIN参数)
docker create -p 13307:3306 -v haoke-v2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node02 -e CLUSTER_JOIN=pxc-node01 --net=pxc-network --ip=172.30.0.3 pxc

#集群02节点01
docker create -p 13308:3306 -v haoke-v3:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node03 --net=pxc-network --ip=172.30.0.4 pxc

#集群02节点02(增加了CLUSTER_JOIN参数)
docker create -p 13309:3306 -v haoke-v4:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e CLUSTER_NAME=pxc --name=pxc-node04 -e CLUSTER_JOIN=pxc-node03 --net=pxc-network --ip=172.30.0.5 pxc

#注意的是:先启动第一个节点,等MySQL客户端可以连接到服务后再启动其他节点
#查看集群节点,在Navicat,结果如下图
show status like 'wsrep_cluster%';
~~~

## 6.3.部署主从复制集群

master:

~~~shell
#创建配置文件vim my.cnf,没事不要改my.cnf的权限,这会导致mysql忽略该配置文件
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
log-bin=mysql-bin
server-id=1
skip-name-resolve
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

#创建容器
docker create --privileged=true  --name mysql-master01 -v /opt/mysql-cluster/master01/data:/var/lib/mysql -v /opt/mysql-cluster/master01/conf:/etc/mysql/conf.d -v /opt/mysql-cluster/master01/my.cnf:/etc/mysql/my.cnf -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql

#启动
docker start mysql-master01 && docker logs -f mysql-master01

#在Navicat中创建同步账户及授权
#create user 'fechinchu'@'%' identified by '320512';这一行会出现2061异常
#https://forums.mysql.com/read.php?26,663846,663880#msg-663880
CREATE USER 'fechinchu'@'%' IDENTIFIED WITH 'mysql_native_password' BY '320512';
grant replication slave on *.* to 'fechinchu'@'%';
flush privileges;

#出现 [Err] 1055 -Expression ...在my.cnf配置文件中设置
sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'

#查看master状态,结果见下图
show master status;

#查看二进制日志相关的配置项
show global variables like 'binlog%';

#查看server相关的配置项
show global variables like 'server%';
~~~

salve:

~~~shell
#创建配置文件vim my.cnf
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
server-id=2
skip-name-resolve
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

#创建容器
docker create --privileged=true --name mysql-slave01 -v /opt/mysql-cluster/slave01/data:/var/lib/mysql -v /opt/mysql-cluster/slave01/conf:/etc/mysql/conf.d -v /opt/mysql-cluster/slave01/my.cnf:/etc/mysql/my.cnf -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root mysql

#启动
docker start mysql-slave01 && docker logs -f mysql-slave01

#在Navicat中设置master相关信息
CHANGE MASTER TO 
 master_host='172.16.124.131',
 master_user='fechinchu',
 master_password='320512',
 master_port=3306,
 master_log_file='mysql-bin.000003',
 master_log_pos=881
 
#启动同步
start slave;

#查看slave状态,结果见下图
show slave status;
~~~

## 6.4.部署MyCat

### 6.4.1.节点01

server.xml

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
    <system>
        <property name="nonePasswordLogin">0</property>
        <property name="useHandshakeV10">1</property>
        <property name="useSqlStat">0</property>
        <property name="useGlobleTableCheck">0</property>
        <property name="sequnceHandlerType">2</property>
        <property name="subqueryRelationshipCheck">false</property>
        <property name="processorBufferPoolType">0</property>
        <property name="handleDistributedTransactions">0</property>
        <property name="useOffHeapForMerge">1</property>
        <property name="memoryPageSize">64k</property>
        <property name="spillsFileBufferSize">1k</property>
        <property name="useStreamOutput">0</property>
        <property name="systemReserveMemorySize">384m</property>
        <property name="useZKSwitch">false</property>
    </system>
    <!--这里是设置的用户和虚拟逻辑库-->
		<user name="fechinchu" defaultAccount="true">
        <property name="password">320512</property>
        <property name="schemas">haoke</property>
    </user>
</mycat:server>
~~~

schema.xml

~~~shell
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd"> 
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!--配置数据表-->
    <schema name="haoke" checkSQLschema="false" sqlMaxLimit="100">
        <table name="tb_house_resources" dataNode="dn1,dn2" rule="mod-long" />
        <table name="tb_ad" dataNode="dn3"/>
    </schema>
    <!--配置分片关系-->
    <dataNode name="dn1" dataHost="cluster1" database="haoke" />
    <dataNode name="dn2" dataHost="cluster2" database="haoke" />
    <dataNode name="dn3" dataHost="cluster3" database="haoke" />
    
    <!--配置连接信息-->
    <dataHost name="cluster1" maxCon="1000" minCon="10" balance="2" writeType="1" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="W1" url="172.16.124.131:13306" user="root" password="root">
            <readHost host="W1R1" url="172.16.124.131:13307" user="root" password="root" />
        </writeHost>
    </dataHost>
    
    <dataHost name="cluster2" maxCon="1000" minCon="10" balance="2" writeType="1" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="W2" url="172.16.124.131:13308" user="root" password="root">
            <readHost host="W2R1" url="172.16.124.131:13309" user="root" password="root" />
        </writeHost>
    </dataHost>
    
    <dataHost name="cluster3" maxCon="1000" minCon="10" balance="3" writeType="1" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="W3" url="172.16.124.131:3306" user="root" password="root">
            <readHost host="W3R1" url="172.16.124.131:3307" user="root" password="root" />
        </writeHost>
    </dataHost>
</mycat:schema>
~~~

rule.xml

~~~xml
<function name="mod-long" class="io.mycat.route.function.PartitionByMod"> 
    <property name="count">2</property>
</function>
~~~

设置启动端口及启动

~~~shell
vim wrapper.conf
#设置jmx端口
wrapper.java.additional.7=-Dcom.sun.management.jmxremote.port=11985

vim server.xml
#设置服务端口以及管理端口
<property name="serverPort">18067</property> 
<property name="managerPort">19067</property>

./startup_nowrap.sh && tail -f ../logs/mycat.log
~~~

### 6.4.2.节点02

~~~shell
vim wrapper.conf
#设置jmx端口
wrapper.java.additional.7=-Dcom.sun.management.jmxremote.port=11986

vim server.xml
#设置服务端口以及管理端口
<property name="serverPort">18068</property> 
<property name="managerPort">19068</property>

./startup_nowrap.sh && tail -f ../logs/mycat.log
~~~

## 6.5.部署HAProxy

~~~shell
#在/opt/haproxy目录下创建haproxy.cfg
#输入内容如下:

#创建文件
vim /opt/haproxy/haproxy.cfg
#输入如下内容 
global
    log       127.0.0.1 local2
    maxconn   4000
    daemon
    
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen admin_stats
    bind         0.0.0.0:4001
    mode         http
    stats uri    /dbs
    stats realm  Global\ statistics 
    stats auth   admin:admin123
    
listen proxy-mysql 
    bind         0.0.0.0:4002 
    mode         tcp
    balance      roundrobin
    option       tcplog
    server       mycat_1 172.16.124.131:18067 check port 18067 maxconn 2000
    server       mycat_2 172.16.124.131:18068 check port 18068 maxconn 2000
~~~

启动容器

~~~shell
docker start haproxy && docker logs -f haproxy
~~~

http://172.16.124.131:4001/dbs

![image-20210725005359982](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210725005359982.png)