# ElasticStack-分布式搜索引擎ElasticSearch03-集群,原理,优化

# 1. ElasticSearch在项目中的使用

~~~java
			SearchRequest searchRequest = new SearchRequest(goodsIndexName);
      SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
      BoolQueryBuilder queryBuilder = null;

      if (tenantId != null) {

        queryBuilder =
            QueryBuilders.boolQuery()
                .filter(QueryBuilders.termQuery("tenantId", tenantId))
                .filter(QueryBuilders.termQuery("saleChannelCode", MallContext.getChannelCode()));
        if (Objects.nonNull(fromPrice)) {
          queryBuilder.filter(QueryBuilders.rangeQuery("convertedGtPrice").gte(fromPrice));
        }
        if (Objects.nonNull(toPrice)) {
          queryBuilder.filter(QueryBuilders.rangeQuery("convertedGtPrice").lte(toPrice));
        }
        if (saleOrgIdFlag == true) {
          queryBuilder.filter(QueryBuilders.termsQuery("orgId", saleOrgIdList));
        }
        if (StringUtils.isNotBlank(keyWord)) {
          queryBuilder.must(
              QueryBuilders.matchQuery("searchTag", param.getKeyWord())); // .boost(10);
        }
        if (CollectionUtils.isNotEmpty(categoryIdSet)) {
          queryBuilder.filter(QueryBuilders.termsQuery("categoryId", categoryIdSet));
        }
      } else if (companyId != null) {
        queryBuilder =
            QueryBuilders.boolQuery()
                .filter(QueryBuilders.termQuery("companyId", companyId))
                .filter(QueryBuilders.termQuery("saleChannelCode", MallContext.getChannelCode()));
        if (Objects.nonNull(fromPrice)) {
          queryBuilder.filter(QueryBuilders.rangeQuery("convertedGtPrice").gte(fromPrice));
        }
        if (Objects.nonNull(toPrice)) {
          queryBuilder.filter(QueryBuilders.rangeQuery("convertedGtPrice").lte(toPrice));
        }
        if (saleOrgIdFlag == true) {
          queryBuilder.filter(QueryBuilders.termsQuery("orgId", saleOrgIdList));
        }
        if (StringUtils.isNotBlank(keyWord)) {
          queryBuilder.must(
              QueryBuilders.matchQuery("searchTag", param.getKeyWord())); // .boost(10);
        }
        if (CollectionUtils.isNotEmpty(categoryIdSet)) {
          queryBuilder.filter(QueryBuilders.termsQuery("categoryId", categoryIdSet));
        }

      } else {
        queryBuilder =
            QueryBuilders.boolQuery()
                .filter(QueryBuilders.termQuery("orgId", orgId))
                .filter(QueryBuilders.termQuery("orgType", orgType))
                .filter(QueryBuilders.termQuery("saleChannelCode", MallContext.getChannelCode()));
        if (Objects.nonNull(fromPrice)) {
          queryBuilder.filter(QueryBuilders.rangeQuery("convertedGtPrice").gte(fromPrice));
        }
        if (Objects.nonNull(toPrice)) {
          queryBuilder.filter(QueryBuilders.rangeQuery("convertedGtPrice").lte(toPrice));
        }
        if (saleOrgIdFlag == true) {
          queryBuilder.filter(QueryBuilders.termsQuery("orgId", saleOrgIdList));
        }
        if (StringUtils.isNotBlank(keyWord)) {
          queryBuilder.must(
              QueryBuilders.matchQuery("searchTag", param.getKeyWord())); // .boost(10);
        }
        if (CollectionUtils.isNotEmpty(categoryIdSet)) {
          queryBuilder.filter(QueryBuilders.termsQuery("categoryId", categoryIdSet));
        }
      }

      // 参考ES官方文档-->按受欢迎度提升权重--ZGQ,目前weight排序权重为0.2F,最多为1.5F;
      FieldValueFactorFunctionBuilder weight =
          ScoreFunctionBuilders.fieldValueFactorFunction("weight").factor(0.2F);
      FunctionScoreQueryBuilder functionScoreQueryBuilder =
          QueryBuilders.functionScoreQuery(queryBuilder, weight)
              .boostMode(CombineFunction.SUM)
              .maxBoost(1.5F);

      searchSourceBuilder.query(functionScoreQueryBuilder);
      searchSourceBuilder.from(param.getCurrentPage() * DEFAULT_PAGE_SIZE);
      searchSourceBuilder.size(param.getPageSize());

      Integer saleNumWeight = param.getSaleNumWeight();
      Integer priceWeight = param.getPriceWeight();
      Integer appraiseNumWeight = param.getAppraiseNumWeight();
      Integer createTimeWeight = param.getCreateTimeWeight();
      buildSearchSourceBuilder(
          searchSourceBuilder,
          saleNumWeight,
          priceWeight,
          appraiseNumWeight,
          createTimeWeight,
          null);

      // 根据权重进行排序
      //            if(StringUtils.isEmpty(keyWord))
      //            searchSourceBuilder.sort("weight", SortOrder.DESC);

      searchRequest.source(searchSourceBuilder);
      SearchResponse searchResponse =
          searchClient.getClient().search(searchRequest, RequestOptions.DEFAULT);
      SearchHits searchHits = searchResponse.getHits();
      if (Objects.isNull(searchHits) || searchHits.getTotalHits().value == 0) {
        return pageResult;
      }
      SearchHit[] hits = searchHits.getHits();
      if (ArrayUtils.isEmpty(hits)) {
        return pageResult;
      }
      List<SearchOutputDTO> list = Lists.newArrayList();
      esSearchResultHandler.handleSearchResult(list, hits);
      return PageResult.build(list, searchHits.getTotalHits().value);

//排序的组合代码
private void buildSearchSourceBuilder(
      SearchSourceBuilder searchSourceBuilder,
      Integer saleNumWeight,
      Integer priceWeight,
      Integer appraiseNumWeight,
      Integer createTimeWeight,
      Integer radomWeight) {
    // ====排序代码====
    if (Objects.nonNull(saleNumWeight)) {
      if (1 == saleNumWeight) {
        searchSourceBuilder.sort("saleNum", SortOrder.ASC);
      }
      if (2 == saleNumWeight) {
        searchSourceBuilder.sort("saleNum", SortOrder.DESC);
      }
    }
    if (Objects.nonNull(priceWeight)) {
      if (1 == priceWeight) {
        // setConvertedGtPrice
        searchSourceBuilder.sort("convertedGtPrice", SortOrder.ASC);
      }
      if (2 == priceWeight) {
        searchSourceBuilder.sort("convertedGtPrice", SortOrder.DESC);
      }
    }
    if (Objects.nonNull(appraiseNumWeight)) {
      if (1 == appraiseNumWeight) {
        searchSourceBuilder.sort("appraiseNum", SortOrder.ASC);
      }
      if (2 == appraiseNumWeight) {
        searchSourceBuilder.sort("appraiseNum", SortOrder.DESC);
      }
    }
    if (Objects.nonNull(createTimeWeight)) {
      if (1 == createTimeWeight) {
        searchSourceBuilder.sort("createTime", SortOrder.ASC);
      }
      if (2 == createTimeWeight) {
        searchSourceBuilder.sort("createTime", SortOrder.DESC);
      }
    }

    if (Objects.nonNull(radomWeight)) {
      if (1 == radomWeight) {
        Script script = new Script("Math.random()");
        ScriptSortBuilder.ScriptSortType number =
            ScriptSortBuilder.ScriptSortType.fromString("number");
        ScriptSortBuilder scriptSortBuilder =
            SortBuilders.scriptSort(script, number).order(SortOrder.ASC);
        searchSourceBuilder.sort(scriptSortBuilder);
      }
    }
~~~

上述代码是我在SAAS千橙万店和悦购钟山项目中关于前台搜索的核心代码.可优化的点其实还有很多.比如用相应的策略设计模式,或者封装重复代码等.这里不多赘述;

![image-20210709140918180](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709140918180.png)

单独拿这段代码看,使用了bool查询,filter查询,match查询,range查询,term查询,terms查询,

![](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709141452493.png)

再拿这段代码看,用到了functionScore,一种自定义搜索评分改变查询结果.主要是用来项目中根据关键字搜索的评分与商品上架时候设置weight权重有一定冲突,我们需要在它们之间设置一个比重,然后综合两者重新设置评分.官网的文档连接:https://www.elastic.co/guide/cn/elasticsearch/guide/current/boosting-by-popularity.html;

# 2.搭建ElasticSearch集群

## 2.1.集群节点

ElasticSearch的集群是由多个节点组成的,通过cluster.name设置集群名称,并且用于区分其它的集群,每个节点通过node.name指定节点名称.

在ElasticSearch中,节点的类型主要有4种:

* master节点
  * 配置文件中`node.master`属性为true(默认为true),就有资格被选为master节点;
  * master节点用于控制整个集群的操作,比如创建或删除索引,管理其它非master节点.
* data节点
  * 配置文件中`node.data`属性为true(默认为true),就有资格被设置成data节点;
  * data节点主要用于执行数据相关的操作,比如文档的CRUD.
* 客户端节点
  * 配置文件中`node.master`属性和`node.data`属性均为false;
  * 该节点不能作为master节点,也不能作为data节点;
  * 可以作为客户端节点,用于响应客户的请求,把请求转发到其他节点
* 部落节点
  * 当一个节点配置`tribe.*`的时候,它是一个特殊的客户端,它可以连接多个集群,在所有连接的集群上执行搜索和其他操作;

## 2.2.使用docker搭建集群

~~~shell
#在/opt目录下创建elasticsearch的挂载目录/elasticsearch/node01 /elasticsearch/node02
#复制安装目录下的elasticsearch.yml、jvm.options文件，做如下修改:

#容器一的elasticsearch.yml配置
cluster.name: es-fechin-cluster
node.name: node01
network.host: 172.16.124.131 
http.port: 9200
discovery.zen.ping.unicast.hosts: ["172.16.124.131"]
discovery.zen.minimum_master_nodes: 1
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
#容器二的elasticsearch.yml配置
cluster.name: es-fechin-cluster
node.name: node02
network.host: 172.16.124.131 
http.port: 9201
discovery.zen.ping.unicast.hosts: ["172.16.124.131"]
discovery.zen.minimum_master_nodes: 1
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: false
node.data: true

#创建容器
docker create --name es-node01 --net host -v /opt/elasticsearch/node01/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /opt/elasticsearch/node01/jvm.options:/usr/share/elasticsearch/config/jvm.options -v /opt/elasticsearch/node01/data:/usr/share/elasticsearch/data elasticsearch:6.5.4

docker create --name es-node02 --net host -v /opt/elasticsearch/node02/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /opt/elasticsearch/node02/jvm.options:/usr/share/elasticsearch/config/jvm.options -v /opt/elasticsearch/node02/data:/usr/share/elasticsearch/data elasticsearch:6.5.4
#修改文件夹权限,在修改权限之前需要创建文件夹
chmod 777 /opt/elasticsearch/node01/data
chmod 777 /opt/elasticsearch/node02/data
#启动容器
docker start es-node01 && docker logs -f es-node01
docker start es-node02 && docker logs -f es-node02
~~~

在启动的过程中会遇到memory areas is too low的问题

![image-20210709143647571](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709143647571.png)

我们需要:

~~~shell
vi /etc/sysctl.conf 
#添加如下配置
vm.max_map_count=655360
#执行命令
sysctl -p
~~~

再次重启elasticsearch容器,完成.

下面我们进行测试:创建索引

![image-20210709143728690](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709143728690.png)

![image-20210709143738167](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709143738167.png)

查看集群状态:http://172.16.124.131:9200/_cluster/health

~~~json
{
    "cluster_name": "es-fechin-cluster",
    "status": "green",
    "timed_out": false,
    "number_of_nodes": 2,
    "number_of_data_nodes": 2,
    "active_primary_shards": 5,
    "active_shards": 10,
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 0,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 100
}
~~~

集群状态的三种颜色:

| 颜色   | 意义                                        |
| ------ | ------------------------------------------- |
| green  | 所有主要分片和复制分片都可用                |
| yellow | 所有主要的分片可用,但不是所有复制分片都可用 |
| red    | 不是所有的主要分片都可用                    |

## 2.3.分片和副本

为了将数据添加到ElasticSearch中,我们需要**索引(index)**----一个存储关联数据的地方.实际上,索引只是一个用来指向一个或多个**分片(shards)**的"逻辑命名空间(logical namespace)".

* 一个分片(shard)是一个最小级别的工作单元,它只是保存了索引中所有数据的一部分;
* 我们需要知道是分片就是一个Lucene实例,并且它本身就是一个完整的搜索引擎,应用程序不会和它直接通信;
* 分片可以是主分片(primary shard)或者是复制分片(replica shard);
* 索引中的每个文档都属于一个单独的主分片,所以主分片的数量决定了索引最多存储多少数据;
* 复制分片只是主分片的一个副本,它可以防止硬件故障导致的数据丢失,同时可以提供读请求,比如搜索或者从别的shard取回文档;
* 当索引创建完成的时候,主分片的数量就固定了,但是复制分片的数量可以随时调整;

## 2.4.故障转移

为了测试故障转移,需要再向集群中添加一个节点,并且将所有节点的node.master设置为true;

我们再创建索引,分片数为5,副本数为1,如下图所示

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709143759747.png" alt="image-20210709143759747" style="zoom: 67%;" />

### 2.4.1.测试

![image-20210709143815369](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709143815369.png)

当前集群状态为黄色,表示主节点可用,副本节点不完全可用;

![image-20210709144145496](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709144145496.png)

过一段时间,发现节点列表中看不到node01,副本节点分配到了node02和node03,集群状态恢复到了绿色.

![image-20210709143954856](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709143954856.png)

我们再将node01恢复,如上图:我们发现node01无法加入到集群中了,这其实就是集群中的的脑裂问题.

### 2.4.2.脑裂问题

![image-20210709144157844](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709144157844.png)

解决方案:

* 不能让节点很容易变成master,必须有多个节点确认后才可以;
* 设置`minimum_master_nodes`的大小为(N/2)+1,N为集群中节点数;

## 2.5.分布式文档

### 2.5.1.路由

![image-20210709144206166](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709144206166.png)

如图所示:当我们想一个集群保存文档时,文档该存储到哪个节点呢?是随机吗?是轮询吗?

实际上,在ElasticSearch中,会采用计算的方式来确定存储到哪个节点,计算公式如下:

> `shard = hash(routing)% number_of_primary_shards`

* routing值是一个任意的字符串,它默认是_id,但也可以自定义;
* 这个routing字符串通过哈希函数生成一个数字,然后除以主切片的数量得到一个余数(remainder),余数的范围永远是0到number_of_primary_shards - 1,这个数字就是特定文档所在的分片;
* 这也就是为什么创建了主分片后,不能修改的原因;

### 2.5.2.文档的写操作

![image-20210709144213418](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709144213418.png)

新建,索引,删除请求都是写(write)操作,它们必须在主分片上成功完成才能复制到相关的复制分片上.步骤如下:

1. 客户端给Node01发送新建,索引或删除请求;
2. 节点使用文档的`_id`确定文档属于主分片0,她转发请求到Node3,主分片0位于这个节点上.
3. Node3在主分片上执行请求,如果成功,它转发请求到响应的位于Node1和Node2的复制节点上.当所有的复制节点都报告成功,Node3报告成功到请求的节点,请求节点再报告给客户端.
4. 客户端接收到成功响应的时候,文档的修改已经被应用于主分片和所有复制分片,修改生效.

### 2.5.3.搜索单个文档

![image-20210709144220007](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709144220007.png)

1. 客户端给Node01发送get请求.
2. 节点使用文档的_id确定文档属于分片0,分片0对应的复制分片三个节点上都有,此时,转发请求到了Node02.
3. Node02返回文档给Node01然后返回给客户端

对于读请求,为了平衡负载,请求节点会为每个请求选择不同的分片,它会循环所有分片副本;

可能的情况是:一个索引的文档已经存在于主分片上确还没有来得及同步到复制分片上.这时复制分片会报告文档未找到,主分片会成功返回文档,一旦索引请求成功返回给用户,文档则在主分片和复制分片都是可用的.

### 2.5.4.全文搜索

全文搜索分为两个阶段:搜索(query)+取回(fetch)

![image-20210709144226862](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709144226862.png)

#### 2.5.4.1.搜索

1. 客户端发送一个search请求给Node3,Node3创建了一个长度为from+size的空优先级队;
2. Node3转发这个搜索请求到索引中每个分片的原本或副本,每个分片在本地执行这个查询并且将结果到一个大小为from+size的有序本地优先队列里去;
3. 每个分片返回document的ID和它优先队列里的所有documet的排序值给协调节点Node3.Node3把这些值合并到自己的优先队列里产生全局排列结果;

#### 2.5.4.2.取回

1. 协调节点辨别出哪个document需要取回,并且向相关分片发出GET请求;
2. 每个分片加载document并且根据需要丰富(enrich)它们,然后再将document返回协调节点.
3. 一旦所有的document都被取回,协调节点会将结果返回给客户端.

# 3.ElasticSearch分布式架构原理

![image-20200531110223944](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200531110223944.png)

# 4.ElasticSearch写入和查询的流程

![image-20210709160707578](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709160707578.png)

## 4.1.ES写数据过程

1. 客户端选择一个node发送请求过去,这个node就是coordinating node(协调节点);
2. coordinating node,对document 进行路由,将请求转发给对应的node(有primary shard);
3. 实际的node上的primary shard处理请求,然后将数据同步到replica node;
4. coordinating node,如果发现primary node和所有replica node都搞定之后,就返回响应结果给客户端;

## 4.2.ES读数据过程

1. 客户端发送请求到任意一个node,称为coordinate node;
2. coordinate node对document进行路由,将请求转发到对应的node,此时会使用round-robin随机轮询算法,在primary shard以及其所有replica中随机选择一个,让读请求负载均衡;
3. 接收请求的node返回document给coordinate node;
4. coordinate node返回document给客户端;

## 4.3.ES搜索数据过程

1. 客户端发送请求到一个coordinate node;
2. 协调节点将搜索请求转发到所有的shard对应的primary shard或replica shard也可以;
3. query phase:每个shard将自己的搜索结果(其实就是一些doc id),返回给协调节点,由协调节点进行数据的合并,排序,分页等操作,产出最终结果;
4. fetch phase:接着由协调节点,根据doc id去各个节点拉取实际的document数据,最终返回给客户端;

## 4.5.ES写数据的底层原理

![image-20210709164730198](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210709164730198.png)

> 先写入内存 buffer，在 buffer 里的时候数据是搜索不到的；同时将数据写入 translog 日志文件。
>
> 如果 buffer 快满了，或者到一定时间，就会将内存 buffer 数据 `refresh` 到一个新的 `segment file` 中，但是此时数据不是直接进入 `segment file` 磁盘文件，而是先进入 `os cache` 。这个过程就是 `refresh`。
>
> 每隔 1 秒钟，es 将 buffer 中的数据写入一个**新的** `segment file`，每秒钟会产生一个**新的磁盘文件** `segment file`，这个 `segment file` 中就存储最近 1 秒内 buffer 中写入的数据。
>
> 但是如果 buffer 里面此时没有数据，那当然不会执行 refresh 操作，如果 buffer 里面有数据，默认 1 秒钟执行一次 refresh 操作，刷入一个新的 segment file 中。
>
> 操作系统里面，磁盘文件其实都有一个东西，叫做 `os cache`，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 `os cache`，先进入操作系统级别的一个内存缓存中去。只要 `buffer` 中的数据被 refresh 操作刷入 `os cache`中，这个数据就可以被搜索到了。
>
> 为什么叫 es 是**准实时**的？ `NRT`，全称 `near real-time`。默认是每隔 1 秒 refresh 一次的，所以 es 是准实时的，因为写入的数据 1 秒之后才能被看到。可以通过 es 的 `restful api` 或者 `java api`，**手动**执行一次 refresh 操作，就是手动将 buffer 中的数据刷入 `os cache`中，让数据立马就可以被搜索到。只要数据被输入 `os cache` 中，buffer 就会被清空了，因为不需要保留 buffer 了，数据在 translog 里面已经持久化到磁盘去一份了。
>
> 重复上面的步骤，新的数据不断进入 buffer 和 translog，不断将 `buffer` 数据写入一个又一个新的 `segment file` 中去，每次 `refresh` 完 buffer 清空，translog 保留。随着这个过程推进，translog 会变得越来越大。当 translog 达到一定长度的时候，就会触发 `commit` 操作。
>
> commit 操作发生第一步，就是将 buffer 中现有数据 `refresh` 到 `os cache` 中去，清空 buffer。然后，将一个 `commit point`写入磁盘文件，里面标识着这个 `commit point` 对应的所有 `segment file`，同时强行将 `os cache` 中目前所有的数据都 `fsync` 到磁盘文件中去。最后**清空** 现有 translog 日志文件，重启一个 translog，此时 commit 操作完成。
>
> 这个 commit 操作叫做 `flush`。默认 30 分钟自动执行一次 `flush`，但如果 translog 过大，也会触发 `flush`。flush 操作就对应着 commit 的全过程，我们可以通过 es api，手动执行 flush 操作，手动将 os cache 中的数据 fsync 强刷到磁盘上去。
>
> translog 日志文件的作用是什么？你执行 commit 操作之前，数据要么是停留在 buffer 中，要么是停留在 os cache 中，无论是 buffer 还是 os cache 都是内存，一旦这台机器死了，内存中的数据就全丢了。所以需要将数据对应的操作写入一个专门的日志文件 `translog` 中，一旦此时机器宕机，再次重启的时候，es 会自动读取 translog 日志文件中的数据，恢复到内存 buffer 和 os cache 中去。
>
> translog 其实也是先写入 os cache 的，默认每隔 5 秒刷一次到磁盘中去，所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会**丢失** 5 秒钟的数据。但是这样性能比较好，最多丢 5 秒的数据。也可以将 translog 设置成每次写操作必须是直接 `fsync` 到磁盘，但是性能会差很多。
>
> 其实 es 第一是准实时的，数据写入 1 秒后可以搜索到；可能会丢失数据的。有 5 秒的数据，停留在 buffer、translog os cache、segment file os cache 中，而不在磁盘上，此时如果宕机，会导致 5 秒的**数据丢失**。
>
> 当segment file 多到一定程度的时候,es就会自动触发merge操作,将多个segment file给merge成一个segment file.
>
> **总结一下**，数据先写入内存 buffer，然后每隔 1s，将数据 refresh 到 os cache，到了 os cache 数据就能被搜索到（所以我们才说 es 从写入到能被搜索到，中间有 1s 的延迟）。每隔 5s，将数据写入 translog 文件（这样如果机器宕机，内存数据全没，最多会有 5s 的数据丢失），translog 大到一定程度，或者默认每隔 30mins，会触发 commit 操作，将缓冲区的数据都 flush 到 segment file 磁盘文件中。数据写入 segment file 之后，同时就建立好了倒排索引。

 # 5.ElasticSearch在数据量很大的情况下如何优化

## 5.1.FileSystem Cache

![image-20210711184029269](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210711184029269.png)

ES的搜索引擎严重依赖于底层的filesystem cache,如果给filesystem cache更多的内存,尽量让内存可以容纳所有的index segment file索引数据文件,那么搜索的时候基本都是走内存的,性能会非常高;

> 可以采用ElasticSearch+HBase的架构.HBase的特点是适用于海量数据的在线存储,就是对HBase可以写入海量数据,不要做复杂的搜索,就是做很简单的一些根据id或者范围进行查询的这个操作.例如:从ElasticSearch中去搜索,拿到的结果就可能就20个docId,然后根据docId到Hbase中查询. 
>
> 这种架构的话可以把需要进行搜索的字段放在ElasticSearch中,然后把详细数据放在HBase中.

## 5.2.数据预热

假如ES集群中每个机器写入的数据量还是超过了filesystem cache一倍,对于那些比较热的数据,做一个专门的缓存预热子系统,就是对热数据,每隔一段时间,就提前访问一下,让数据进入filesystem cache里面.

 ## 5.3.冷热分离

将冷数据写入一个索引中,然后热数据写入到另一个索引中,可以确保热数据在被预热后,尽量都让他们留在filesystem os cache中,别让冷数据给冲刷掉;

## 5.4.Document模型设计

不要考虑用es做一些它不好操作的事情。如果真的有那种操作，尽量在document模型设计的时候，写入的时候就完成。另外对于一些太复杂的操作，比如join，nested，parent-child搜索都要尽量避免，性能都很差的。

## 5.5.分页性能优化

假如你每页是10条数据，你现在要查询第100页，实际上是会把每个shard上存储的前1000条数据都查到一个协调节点上，如果你有个5个shard，那么就有5000条数据，接着协调节点对这5000条数据进行一些合并、处理，再获取到最终第100页的10条数据。

1. 默认不能深度分页/默认深度分页性能会不好.

> 在SAAS项目中,H5端的根据品类,关键字搜索查询的话商品的数量有限,所以不会深度分页,PC端的聚合搜索,品类搜索数量有限商品的数量也有限,也不会深度分页;

2. scroll api

~~~java
System.out.println("scroll 模式启动！");
begin = new Date();
SearchResponse scrollResponse = client.prepareSearch(INDEX)
    .setSearchType(SearchType.SCAN).setSize(10000).setScroll(TimeValue.timeValueMinutes(1)) 
    .execute().actionGet();  
count = scrollResponse.getHits().getTotalHits();//第一次不返回数据
for(int i=0,sum=0; sum<count; i++){
    scrollResponse = client.prepareSearchScroll(scrollResponse.getScrollId())  
        .setScroll(TimeValue.timeValueMinutes(8))  
	.execute().actionGet();
    sum += scrollResponse.getHits().hits().length;
    System.out.println("总量"+count+" 已经查到"+sum);
}
end = new Date();
System.out.println("耗时: "+(end.getTime()-begin.getTime()));
~~~

https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html;

> 在项目中H5端和PC端的全部商品搜索的话采用scroll api进行搜索

scroll会一次性生成所有数据的一个快照,然后每次翻页通过游标获取下一页;第一次做scroll搜索的话把scroll id给前端,之后的翻页带着scroll id去进行往下搜索;



