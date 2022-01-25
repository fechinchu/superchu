# ElasticStack-Beats,Kibana,Logstash

# 1.Beats简介

![image-20200317105501565](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317105501565.png)

![image-20200317105624651](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317105624651.png)

![image-20200317105700351](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317105700351.png)

## 1.1.Filebeat

![image-20200317105834435](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317105834435.png)

### 1.1.1架构

用于监控,收集服务器日志文件

![image-20200317105931870](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317105931870.png)

### 1.1.2.部署与运行

下载:https://www.elastic.co/cn/downloads/beats/filebeat

~~~shell
tar -xvf filebeat-6.5.4-linux-x86_64.tar.gz

#创建如下配置文件 haoke.yml
filebeat.inputs: 
- type: stdin
  enabled: true
setup.template.settings: 
  index.number_of_shards: 3
output.console: 
  pretty: true
  enable: true

#启动filebeat 
# -e:输出到标准输出，默认输出到syslog和logs下
# -c:指定配置文件
# -d:输出debug信息
#./filebeat -e -c haoke.yml -d "publish"
./filebeat -e -c haoke.yml

#输入hello进行测试
hello
~~~

### 1.1.3.读取文件

~~~shell
#配置读取文件项 haoke-log.yml
filebeat.inputs: 
- type: log
  enabled: true
  paths: 
    - /opt/elasticsearch/ElasticStack/logs/*.log
setup.template.settings: 
  index.number_of_shards: 3
output.console: 
  pretty: true
  enable: true

#启动filebeat
./filebeat -e -c haoke-log.yml

#/opt/elasticsearch/ElasticStack/logs/下创建test01.log文件,并输入如下内容
123
#观察filebeat输出
~~~

![image-20200316212645174](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316212645174.png)

### 1.1.4.自定义字段

~~~shell
filebeat.inputs: 
- type: log
  enabled: true
  paths: 
    - /opt/elasticsearch/ElasticStack/logs/*.log
  tags: ["web"] #添加自定义tag.便于后续的处理
  fields: #添加自定义字段
    from: haoke-im
  fields_under_root: true #为true为添加到根节点,false为添加到子节点
setup.template.settings: 
  index.number_of_shards: 3
output.console: 
  pretty: true
  enable: true
  
#启动filebeat
./filebeat -e -c haoke-log.yml

#/opt/elasticsearch/ElasticStack/logs/下创建test01.log文件,并输入如下内容
456
#观察filebeat输出
~~~

![image-20200316213544708](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316213544708.png)

### 1.1.5.输出到ElasticSearch

~~~shell
filebeat.inputs: 
- type: log
  enabled: true
  paths: 
    - /opt/elasticsearch/ElasticStack/logs/*.log
  tags: ["web"] #添加自定义tag.便于后续的处理
  fields: #添加自定义字段
    from: haoke-im
  fields_under_root: false #为true为添加到根节点,false为添加到子节点
setup.template.settings: 
  index.number_of_shards: 3
output.elasticsearch: #指定输出ES
  hosts: ["172.16.124.131:9200","172.16.124.131:9201","172.16.124.131:9202"]
~~~

![image-20200316214841334](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316214841334.png)

### 1.1.6.FileBeat工作原理

FilBeat由两个主要组件组成: prospector和harvester

* harvester:
  * 负责读取单个文件的内容
  * 如果文件在读取时被删除或者重命名,FileBeat将继续读取文件
* prospector
  * prospector负责管理harvester并找到所有要读取的文件来源
  * 如果输入类型为日志,则查找器将查找路径匹配的所有文件,并未每个文件启动一个harvester
  * FileBeat目前只支持两种prospector类型:log和stdin
* FileBeat如何保持文件的状态
  * Filebeat 保存每个文件的状态并经常将状态刷新到磁盘上的注册文件中.
  * 该状态用于记住harvester正在读取的最后偏移量，并确保发送所有日志行.
  * 如果输出(例如Elasticsearch或Logstash)无法访问,Filebeat会跟踪最后发送的行,并在输出再次可用时继续读取文件.
  * 在Filebeat运行时,每个prospector内存中也会保存的文件状态信息,当重新启动Filebeat时,将使用注册文件的数据来重建文件状态,Filebeat将每个harvester在从保存的最后偏移量继续读取.
  * 文件状态记录在`/(FilBeat目录)/data/registry`文件中.

### 1.1.7.Module

前面要想实现日志数据的读取以及处理都是自己手动配置的.其实,在FileBeat中,有大量的Module,可以简化我们的配置,直接就可以使用:

![image-20200316221321944](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316221321944.png)

我们可以通过`./filebeat modules enable redis`来启用redis module,同样我们也可以通过`./filebeat modules disable redis`来禁用redis module

![image-20200316221444145](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316221444145.png)

#### 1.1.7.1.redis module目录

![image-20200316222343073](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316222343073.png) 

#### 1.1.7.2.redis module配置

```shell
cd modules.d/
vim redis.yml

#如下是默认配置
- module: redis
  # Main logs
  log:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths: ["/var/log/redis/redis-server.log*"]

  # Slow logs, retrieved via the Redis API (SLOWLOG)
  slowlog:
    enabled: true

    # The Redis hosts to connect to.
    #var.hosts: ["localhost:6379"]

    # Optional, the password to use when connecting to Redis.
    #var.password:
```

#### 1.1.7.3.修改redis的docker容器

~~~shell
#Host模式
docker create --name redis-node01 --net host -v /opt/redis/data/node01:/data redis:5.0.2 --cluster-enabled yes --cluster-config-file nodes-node-01.conf --port 6379 --loglevel debug --logfile nodes-node01.log

docker create --name redis-node02 --net host -v /opt/redis/data/node02:/data redis:5.0.2 --cluster-enabled yes --cluster-config-file nodes-node-02.conf --port 6380 --loglevel debug --logfile nodes-node02.log

docker create --name redis-node03 --net host -v /opt/redis/data/node03:/data  redis:5.0.2 --cluster-enabled yes --cluster-config-file nodes-node-03.conf --port 6381 --loglevel debug --logfile nodes-node03.log

#启动容器
docker start redis-node01 redis-node02 redis-node03

#开始组建集群
#进入redis-node01进行操作
docker exec -it redis-node01 /bin/bash
#组建集群(172.16.124.131是主机的ip地址)
redis-cli --cluster create 172.16.124.131:6379 172.16.124.131:6380 172.16.124.131:6381 --cluster-replicas 0
~~~

loglevel日志等级分为:debug,verbose,notice,warning

* debug:大量信息,对开发测试有用;
* verbose:等于log4j的info,有很多信息;
* notice:一般信息;
* warning:只对关键信息有效

#### 1.1.7.4.修改redis.yml

#### 1.1.7.5.配置filebeat

~~~shell
vim haoke-redis.yml

#配置
filebeat.inputs: 
- type: log
  enabled: true
  paths: 
    - /opt/redis/data/node01/*.log
setup.template.settings: 
  index.number_of_shards: 3
output.console: 
  pretty: true
  enable: true
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
output.elasticsearch: #指定输出ES
  hosts: ["172.16.124.131:9200","172.16.124.131:9201","172.16.124.131:9202"]
  
#启动
./filebeat -e -c haoke-redis.yml --modules redis -d "publish"
~~~

![image-20200316231647689](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200316231647689.png)

可以看到日志数据已经导入进来了.

其他Modules用法参考官方文档:https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-modules.html

## 1.2.Metricbeat

![image-20200317110304101](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317110304101.png)

* 定期收集操作系统或引用服务的指标数据
* 存储到ElasticSearch中,进行实时分析;

### 1.2.1.Metricbeat组成

Metricbeat有2部分组成,一部分是Module,另一部分为Metricset

* Module:收集的对象,如:mysql,redis,操作系统等;
* Metricset:收集指标的集合,如:cpu,memory,network等;

### 1.2.2.部署与收集系统指标

~~~shell
vi metricbeat.yml

#修改metricbeat的配置文件如下:
metricbeat.config.modules: 
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings; 
  index.number_of_shards: 1
  index.codec: best_compression
setup.kibana: 
output.elasticsearch:
  hosts: ["172.16.124.131:9200","172.16.124.131:9201","172.16.124.131:9202"]
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~

#启动
./metricbeat -e
~~~

启动之后默认是采集的system中的数据

### 1.2.3.Module

我们这一次测试Redis的Module,修改`(metricbeat安装目录)/modules.d`目录下的redis.yml,激活redis的module`./metricbeat modules enable redis`

![image-20200317155206795](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317155206795.png)

启动`./metricbeat -e`

其余modules:https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-modules.html

# 2.Kibana

![image-20200317155744668](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317155744668.png)

## 2.1.非Docker安装

~~~shell
# 解压安装包 
tar -xvf kibana-6.5.4-linux-x86_64.tar.gz

#修改配置文件 
vim config/kibana.yml

server.host: "172.16.124.131" #对外暴露服务的地址 
elasticsearch.url: "http://172.16.124.131:9200" #配置ElasticSearch

#启动 
./bin/kibana

#通过浏览器进行访问 
http://172.16.124.131:5601/app/kibana
~~~

## 2.2.Docker安装

~~~shell
#创建配置文件
server.host: "172.16.124.131"
elasticsearch.url: "http://172.16.124.131:9200"

#拉取镜像
docker pull kibana:6.5.4

#创建容器
docker create --name kibana --net host -v /opt/elasticsearch/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml kibana:6.5.4

#启动容器
docker start kibana

#通过浏览器进行访问 
http://172.16.124.131:5601/app/kibana
~~~

## 2.3.Metricbeat仪表盘

~~~shell
#修改metricbeat配置
setup.kibana: 
  host: "172.16.124.131:5601"

#安装仪表盘到kibana
./metricbeat setup --dashboards
#启动
./metricbeat -e
~~~

![image-20200317171054866](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317171054866.png)

## 2.4.Filebeat仪表盘

~~~shell
#修改haoke-redis配置
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /opt/redis/data/node01/*.log
setup.template.settings:
  index.number_of_shards: 3
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
output.elasticsearch: #指定输出ES
  hosts: ["172.16.124.131:9200","172.16.124.131:9201","172.16.124.131:9202"]
setup.kibana: 
  host: "172.16.124.131:5601"
#安装仪表盘到kibana
./filebeat -c haoke-redis.yml setup
#启动
./filebeat -e -c haoke-redis.yml
~~~

# 3.Logstash

## 3.1.简介

![image-20200317171724514](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317171724514.png)

## 3.2.部署安装

~~~shell
#检查jdk环境,要求jdk1.8+
java -version

#解压安装包
tar -xvf logstash-6.5.4.tar.gz

#启动
bin/logstash -e 'input { stdin { } } output { stdout {} }'
~~~

![image-20200317173301158](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317173301158.png)

## 3.3.接收FileBeat输入的日志

### 3.3.1.安装Nginx

~~~shell
#很多软件包在yum里面没有的，解决的方法，就是使用epel源,也就是安装epel-release软件包。EPEL (Extra Packages for Enterprise Linux)是基于Fedora的一个项目，为“红帽系”的操作系统提供额外的软件包，适用于RHEL、CentOS等系统。可以在下面的网址上找到对应的系统版本，架构的软件包
yum install epel-release
yum install -y nginx

#/usr/sbin/nginx：主程序
#/etc/nginx：存放配置文件
#/usr/share/nginx：存放静态文件
#/var/log/nginx：存放日志

#nginx服务命令
systemctl start nginx
#查看日志
tail -f /var/log/nginx/access.log
~~~

### 3.3.2.配置FileBeat

~~~shell
vim haoke-nginx.yml

#配置
filebeat.inputs: 
- type: log
  enabled: true
  paths: 
    - /var/log/nginx/access.log
  tags: ["log"]
  fields: 
    from: nginx
  fields_under_root: false
output.logstash:
  hosts: ["172.16.124.131:5044"]

#启动
./filebeat -e -c haoke-nginx.yml
#现在启动会报错,因为Logstash还没有启动
~~~

### 3.3.3.配置Logstash

~~~shell
#在Logstash安装目录下vi haoke-logstash.conf

#配置文件
input {
  beats {
    port => 5044
  }
}

output {
  stdout { codec => rubydebug }
}

#启动 --config.test_and_exit 用于测试配置文件是否正确
bin/logstash -f haoke-logstash.conf --config.test_and_exit

#正式启动 --config.reload.automatic 热加载配置文件，修改配置文件后无需重新启动
bin/logstash -f haoke-logstash.conf --config.reload.automatic
~~~

### 3.3.4.测试

启动Filebeat:`./filebeat -e -c haoke-nginx.yml`

![image-20200317210419010](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317210419010.png)

控制台已经输出Nginx的日志.

### 3.3.5.配置filter

在前面的输出中,可以看出,虽然可以拿到日志信息,但是信息格式并不友好.比如说:不能直接拿到日志中的ip地址;

1. 第一步:自定义Nginx的日志格式

~~~shell
vim /etc/nginx/nginx.conf

#配置文件
log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
                    
#加载nginx
nginx -s reload
~~~

![image-20200317211655173](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317211655173.png)

2. 编写nginx-patterns文件(在Logstash安装目录下)

~~~shell
NGINX_ACCESS %{IPORHOST:remote_addr} - %{USERNAME:remote_user} \[%{HTTPDATE:time_local}\] \"%{DATA:request}\" %{INT:status} %{NUMBER:bytes_sent} \"%{DATA:http_referer}\" \"%{DATA:http_user_agent}\"
~~~

3. 修改haoke-logstash.conf文件

~~~shell
#配置文件
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    patterns_dir => "/opt/elasticsearch/ElasticStack/logstash-6.5.4/nginx-patterns"
    match => {"message" => "%{NGINX_ACCESS}"}
    remove_tag => [ "_grokparsefailure" ]
    add_tag => [ "nginx_access" ]
    }
}

output {
  stdout { codec => rubydebug }
}
~~~

4. 测试

启动`bin/logstash -f haoke-logstash.conf --config.reload.automatic`

![image-20200317220213839](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317220213839.png)

### 3.3.6.发送到ElasticSearch

~~~shell
vi haoke-logstash.conf

#配置文件
output {
  elasticsearch {
    hosts => ["172.16.124.131:9200","172.16.124.131:9201","172.16.124.131:9202"]
  }
}
~~~

![image-20200317220522797](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317220522797.png)

启动测试`bin/logstash -f haoke-logstash.conf --config.reload.automatic`

![image-20200317220858428](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317220858428.png)

在kibana中查看,也可以制作可视化

![image-20200317221734673](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200317221734673.png)

