# ElasticStack-分布式搜索引擎ElasticSearch02-倒排索引,Java客户端

# 1.全文检索

## 1.1.倒排索引

倒排索引源于实际应用中需要根据属性的值来查找记录.这种索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址.由于不是由记录来确定属性值.而是由属性值来确定记录的位置,因而称为倒排索引(inverted index).带有倒排索引的文件我们称为倒排索引文件,简称倒排文件(inverted file).

正排索引:

![image-20200312190707167](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200312190707167.png)

转化成倒排索引:

![image-20200312190743406](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200312190743406.png)

说明:

* "单词ID"一栏记录了每个单词的单词编号;
* 第二栏是对应的单词;
* 第三栏即每个单词对应的倒排列表;
* 比如单词"谷歌",其单词编号为1,倒排列表为(1,2,3,4,5),说明文档集中每个文档都包含了这个单词.

而事实上,索引系统还可以记录除此之外的更多信息,在单词对应的倒排列表中不仅记录了文档编号,还记载了单词频率信息(TF),即这个单词在某个文档中的出现次数,之所以要记录这个信息,是因为词频信息在搜索结果排序时,计算查询和文档相似度是很重要的一个计算因子,所以将其记录在倒排列表中,以方便后续排序时进行分值计算.

![image-20200312193153744](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200312193153744.png)

倒排索引还可以记载更多的信息,除了记录文档编号和单词频率信息外,额外记载了两类信息,即每个单词对应的"文档频率信息",以及在倒排列表中记录单词某个文档出现的位置信息.

![image-20200312193634724](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200312193634724.png)

## 1.2.全文搜索

全文搜索两个最重要的方面是:

* **相关性(Relevance)**它是评价查询与其结果见的相关程度,并根据这种相关程度对结果排名的一种能力,这种计算方式可以是TF/DF方法,地理位置邻近,模糊相似,或其他的某些算法.
* **分析(Analysis)**它是将文本块转换为有区别的,规范化的token的一个过程,目的是为了创建倒排索引以及查询倒排索引.

### 1.2.1.构造数据

~~~shell
PUT http://172.16.124.131:9200/fechin01
#请求体
{
	"settings":{
		"index":{
			"number_of_shards":"1",
			"number_of_replicas":"0"
		}
	},
	"mappings":{
		"person":{
			"properties":{
				"name":{
					"type":"text"
				},
				"age":{
					"type":"integer"
				},
				"mail":{
					"type":"keyword"
				},
				"hobby":{
					"type":"text",
					"analyzer":"ik_max_word"
				}
			}
		}
	}
}
~~~

之后插入数据:

~~~shell
POST http://172.16.124.131:9200/fechin01/_bulk
#请求体
{"index":{"_index":"fechin","_type":"person"}}
{"name":"张三","age":20,"mail":"111@qq.com","hobby":"羽毛球,乒乓球,足球"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"李四","age":21,"mail":"222@qq.com","hobby":"羽毛球,乒乓球,足球,篮球"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"王五","age":22,"mail":"333@qq.com","hobby":"羽毛球,篮球,游泳,听音乐"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"赵六","age":23,"mail":"444@qq.com","hobby":"跑步,游泳,篮球"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"孙七","age":24,"mail":"555@qq.com","hobby":"听音乐,看电影,羽毛球"}
~~~

### 1.2.2.单词搜索

~~~shell
POST http://172.16.124.131:9200/fechin01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":"音乐"
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
~~~

过程说明:

1. **检查字段类型**:爱好hobby字段是一个text类型(指定了IK分词器),这意味着查询字符串本身也应该被分词;
2. **分析查询字符串**:将查询的字符串"音乐"传入IK分词器中,输出的结果是单个项音乐,因为只有一个单词项,所以match查询执行的是单个底层term查询;
3. **查找匹配文档**:用term查询在倒排索引中查找"音乐",然后获取一组包含该项的文档,本例的结果是文档:3,5;
4. **为每个文档评分**:用term查询计算每个文档相关度评分_score,这是种将词频(term frequency,即词"音乐"在相关文档的hobby字段中出现的频率)和反向文档频率(inverse document frequency,即词"音乐"在所有文档的hobby字段中出现的频率),以及字段的长度(即字段越短相关度越高)相结合的计算方式.

### 1.2.3.多词搜索

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":"音乐 篮球"
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 22,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 1.3192271,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 1.3192271,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,<em>篮球</em>,游泳,听<em>音乐</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EanHznAByqluhSOWWW9g",
                "_score": 0.81652206,
                "_source": {
                    "name": "孙七",
                    "age": 24,
                    "mail": "555@qq.com",
                    "hobby": "听音乐,看电影,羽毛球"
                },
                "highlight": {
                    "hobby": [
                        "听<em>音乐</em>,看电影,羽毛球"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EKnHznAByqluhSOWWW9g",
                "_score": 0.6987338,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "跑步,游泳,篮球"
                },
                "highlight": {
                    "hobby": [
                        "跑步,游泳,<em>篮球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.50270504,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,乒乓球,足球,<em>篮球</em>"
                    ]
                }
            }
        ]
    }
}
~~~

由上述结果可以得到查出来的是既包含"音乐"也包含"篮球"的用户,结果返回的是"或"的关系.在ElasticSearch中,可以指定词之间的逻辑关系.

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":{
				"query":"音乐 篮球",
				"operator":"and"
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 40,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1.3192271,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 1.3192271,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,<em>篮球</em>,游泳,听<em>音乐</em>"
                    ]
                }
            }
        ]
    }
}
~~~

我们可以看到结果是符合预期的.前面我们测试了"OR"和"AND"搜索,这是两个极端,在实际场景中,并不会选取这两个极端,我们只需要符合一定的相似度就可以查询到数据,在ElasticSearch中也支持这样的查询,通过minimum_should_match来指定匹配度,如70%.

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":{
				"query":"音乐 篮球",
				"minimum_should_match":"80%"
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
~~~

相似度多少合适,需要在实际的需求中进行反复测试,才可得到合理的值

### 1.2.4.组合搜索

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"bool":{
			"must":{
				"match":{
					"hobby":"篮球"
				}
			},
			"must_not":{
				"match":{
					"hobby":"音乐"
				}
			},
			"should":{
				"match":{
					"hobby":"游泳"
				}
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 22,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 1.8336569,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EKnHznAByqluhSOWWW9g",
                "_score": 1.8336569,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "跑步,游泳,篮球"
                },
                "highlight": {
                    "hobby": [
                        "跑步,<em>游泳</em>,<em>篮球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.50270504,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,乒乓球,足球,<em>篮球</em>"
                    ]
                }
            }
        ]
    }
}
~~~

上面搜索的结果的意思是:搜索结果中必须包含篮球,不能包含音乐,如果包含游泳,那么相似度会更高.

> 评分的计算规则:
>
> * bool查询会为每个文档计算相关度评分\_score,再讲所有匹配的must和should语句的分数\_score求和,最后除以must和should语句的总数;
> * must_not语句不会影响评分,它的作用只是将不相关的文档排除;

默认情况下,should中的内容不是必须匹配的,如果查询语句中没有must,那么就会至少匹配其中一个,当然,也可以通过`minimum_should_match`参数进行控制,该值可以是数字也可以是百分比.

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"bool":{
			"should":[
				{
					"match":{
						"hobby":"游泳"
					}
				},
				{
					"match":{
						"hobby":"篮球"
					}
				},
				{
					"match":{
						"hobby":"音乐"
					}
				}
			],
			"minimum_should_match":2
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 25,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 2.135749,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 2.135749,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,<em>篮球</em>,<em>游泳</em>,听<em>音乐</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EKnHznAByqluhSOWWW9g",
                "_score": 1.8336569,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "跑步,游泳,篮球"
                },
                "highlight": {
                    "hobby": [
                        "跑步,<em>游泳</em>,<em>篮球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EanHznAByqluhSOWWW9g",
                "_score": 0.81652206,
                "_source": {
                    "name": "孙七",
                    "age": 24,
                    "mail": "555@qq.com",
                    "hobby": "听音乐,看电影,羽毛球"
                },
                "highlight": {
                    "hobby": [
                        "听<em>音乐</em>,看电影,羽毛球"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.50270504,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,乒乓球,足球,<em>篮球</em>"
                    ]
                }
            }
        ]
    }
}
~~~

`minimum_should_match`值为2,意思是should中的三个词,至少要满足2个.

### 1.2.5.权重

有些时候,我们可能需要对某些词增加权重来影响该条数数据的得分,如下:

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"bool":{
			"must":{
				"match":{
					"hobby":{
						"query":"游泳篮球",
						"operator":"and"
					}
				}
			},
			"should":[
				{
					"match":{
						"hobby":{
							"query":"音乐",
							"boost":10
						}
					}
				},
				{
					"match":{
						"hobby":{
							"query":"跑步",
							"boost":2
						}
					}
				}
				
			]
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
~~~

在添加完权重之后,我们发现`_socre`会和不加权重完全不一样.

### 1.2.6.短语匹配

在ElasticSearch中,短语匹配意味着不仅仅是词要匹配,并且词的顺序也要一致,如下:

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match_phrase":{
			"hobby":{
				"query":"羽毛球篮球"
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 61,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1.307641,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 1.307641,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "<em>羽毛</em><em>球</em>,<em>篮球</em>,游泳,听音乐"
                    ]
                }
            }
        ]
    }
}
~~~

如果觉得这样太过于苛刻,可以增加`slop`参数,允许跳过N个词进行匹配

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match_phrase":{
			"hobby":{
				"query":"羽毛球足球",
				"slop":3
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 9,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 0.6476142,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DanHznAByqluhSOWWW9g",
                "_score": 0.6476142,
                "_source": {
                    "name": "张三",
                    "age": 20,
                    "mail": "111@qq.com",
                    "hobby": "羽毛球,乒乓球,足球"
                },
                "highlight": {
                    "hobby": [
                        "<em>羽毛</em><em>球</em>,乒乓球,<em>足球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.594337,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "<em>羽毛</em><em>球</em>,乒乓球,<em>足球</em>,篮球"
                    ]
                }
            }
        ]
    }
}
~~~

# 2.Java客户端

在ElasticSearch中,为Java提供了2种客户端,一种是Rest风格的客户端.另一种是Java API的客户端.https://www.elastic.co/guide/en/elasticsearch/client/index.html

![image-20200314173131710](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20200314173131710.png)

## 2.1.REST客户端

ElasticSearch提供了2种REST客户端,一种是低级客户端,一种是高级客户端

* Java Low Level REST Client :官方提供的低级客户端.该客户端通过http来连接ElasticSearch集群.用户在使用该客户端需要将请求数据手动拼接成ElasticSearch所需JSON格式进行发送,收到响应同时也需要将返回的JSON数据手动封装成对象.虽然麻烦,不过客户端兼容所有的ElasticSearch版本;
* Java High Level Rest Client:官方提供的高级客户端.该客户端基于低级客户端实现,它提供了很多边界的API来解决低级客户端需要手动转换数据格式的问题.

## 2.2.构造函数

~~~shell
POST http://172.16.124.131:9200/haoke/house/_bulk
#请求体
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1001","title":"整租 · 南丹大楼 1居室 7500","price":"7500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1002","title":"陆家嘴板块，精装设计一室一厅，可拎包入住诚意租。","price":"8500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1003","title":"整租 · 健安坊 1居室 4050","price":"7500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1004","title":"整租 · 中凯城市之光+视野开阔+景色秀丽+拎包入住","price":"6500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1005","title":"整租 · 南京西路品质小区 21213三轨交汇 配套齐* 拎包入 住","price":"6000"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1006","title":"祥康里 简约风格 *南户型 拎包入住 看房随时","price":"7000"}
~~~

## 2.3.REST低级客户端

### 2.3.1.POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.0.RELEASE</version>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>demo-elasticsearch-low-level</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- java编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
~~~

### 2.3.2.编写测试用例

~~~java
package org.fechin.elasticsearch.test;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.http.HttpHost;
import org.apache.http.util.EntityUtils;
import org.elasticsearch.client.*;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

/**
 * @Author:朱国庆
 * @Date：2020/3/14 19:00
 * @Desription: haoke-manage
 * @Version: 1.0
 */
public class TestElasticSearchRest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private RestClient restClient;

    @Before
    public void init(){
        RestClientBuilder restClientBuilder = RestClient.builder(
                new HttpHost("172.16.124.131", 9200, "http"),
                new HttpHost("172.16.124.131", 9201, "http"),
                new HttpHost("172.16.124.131", 9202, "http"));

        restClientBuilder.setFailureListener(new RestClient.FailureListener() {
            @Override
            public void onFailure(Node node){
                System.out.println("出错了 -> " + node);
            }
        });

        this.restClient = restClientBuilder.build();
    }

    @After
    public void after() throws IOException {
        restClient.close();
    }

    /**
     * 查询集群的状态
     * @throws IOException
     */
    @Test
    public void testInfo() throws IOException {
        Request request = new Request("GET", "/_cluster/state");
        request.addParameter("pretty", "true");
        Response response = this.restClient.performRequest(request);

        System.out.println("请求完成 -> "+response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));

    }

    /**
     * 新增数据
     * @throws Exception
     */
    @Test
    public void testSave() throws Exception{
        Request request = new Request("POST", "/haoke/house");
        request.addParameter("pretty", "true");

        // 构造数据
        Map<String, Object> data = new HashMap<>();
        data.put("id", 2001);
        data.put("title", "南京西路 一室一厅");
        data.put("price", 3500);

        String json = MAPPER.writeValueAsString(data);
        request.setJsonEntity(json);

        Response response = this.restClient.performRequest(request);
        System.out.println("请求完成 -> "+response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));

    }

    // 根据id查询数据
    @Test
    public void testQueryData() throws IOException {
        Request request = new Request("GET", "/haoke/house/E4vC2HABnqcVXysGZA_M");
        request.addParameter("pretty", "true");

        Response response = this.restClient.performRequest(request);

        System.out.println(response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    // 搜索数据
    @Test
    public void testSearchData() throws IOException {
        Request request = new Request("POST", "/haoke/house/_search");
        String searchJson = "{\"query\": {\"match\": {\"title\": \"拎包入住\"}}}";
        request.setJsonEntity(searchJson);
        request.addParameter("pretty","true");
        Response response = this.restClient.performRequest(request);
        System.out.println(response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));
    }
}
~~~

## 2.4.REST高级客户端

### 2.4.1.POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>


    <groupId>org.example</groupId>
    <artifactId>demo-elasticsearch-high-level</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- java编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
~~~

### 2.4.2.编写测试用例

~~~java
package org.fechin.elasticsearch.test;

import org.apache.http.HttpHost;
import org.elasticsearch.action.ActionListener;
import org.elasticsearch.action.delete.DeleteRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.Strings;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.fetch.subphase.FetchSourceContext;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * @Author:朱国庆
 * @Date：2020/3/14 19:37
 * @Desription: haoke-manage
 * @Version: 1.0
 */
public class TestElasticSearchRest {

    private RestHighLevelClient restHighLevelClient;

    @Before
    public void init() {
        RestClientBuilder restClientBuilder = RestClient.builder(
                new HttpHost("172.16.124.131", 9200, "http"),
                new HttpHost("172.16.124.131", 9201, "http"),
                new HttpHost("172.16.124.131", 9202, "http"));
        this.restHighLevelClient = new RestHighLevelClient(restClientBuilder);
    }

    @After
    public void close() throws Exception {
        this.restHighLevelClient.close();
    }

    @Test
    public void testSave() throws Exception {

        Map<String, Object> data = new HashMap<>();
        data.put("id", 2002);
        data.put("title", "南京东路 二室一厅");
        data.put("price", 4000);

        IndexRequest indexRequest = new IndexRequest("haoke", "house").source(data);

        IndexResponse response = this.restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);

        System.out.println("id -> " + response.getId());
        System.out.println("version -> " + response.getVersion());
        System.out.println("result -> " + response.getResult());
    }

    @Test
    public void testCreateAsync() throws Exception {

        Map<String, Object> data = new HashMap<>();
        data.put("id", "2004");
        data.put("title", "南京东路2 最新房源 二室一厅");
        data.put("price", "5600");

        IndexRequest indexRequest = new IndexRequest("haoke", "house").source(data);

        this.restHighLevelClient.indexAsync(indexRequest, RequestOptions.DEFAULT, new
                ActionListener<IndexResponse>() {
                    @Override
                    public void onResponse(IndexResponse indexResponse) {
                        System.out.println("id->" + indexResponse.getId());
                        System.out.println("index->" + indexResponse.getIndex());
                        System.out.println("type->" + indexResponse.getType());
                        System.out.println("version->" + indexResponse.getVersion());
                        System.out.println("result->" + indexResponse.getResult());
                        System.out.println("shardInfo->" + indexResponse.getShardInfo());
                    }

                    @Override
                    public void onFailure(Exception e) {
                        System.out.println(e);
                    }
                });

        System.out.println("ok");
        Thread.sleep(20000);
    }

    @Test
    public void testQuery() throws Exception {
        GetRequest getRequest = new GetRequest("haoke", "house",
                "Fosz2XABnqcVXysGIQ8l");

        // 指定返回的字段
        String[] includes = new String[]{"title", "id"};
        String[] excludes = Strings.EMPTY_ARRAY;
        FetchSourceContext fetchSourceContext =
                new FetchSourceContext(true, includes, excludes);

        getRequest.fetchSourceContext(fetchSourceContext);

        GetResponse response = this.restHighLevelClient.get(getRequest,
                RequestOptions.DEFAULT);

        System.out.println("数据 -> " + response.getSource());
    }

    /**
     * 判断是否存在
     *
     * @throws Exception
     */
    @Test
    public void testExists() throws Exception {
        GetRequest getRequest = new GetRequest("haoke", "house",
                "Fosz2XABnqcVXysGIQ8l");

        // 不返回的字段
        getRequest.fetchSourceContext(new FetchSourceContext(false));
        boolean exists = this.restHighLevelClient.exists(getRequest, RequestOptions.DEFAULT);
        System.out.println("exists -> " + exists);
    }

    /**
     * 删除数据
     *
     * @throws Exception
     */
    @Test
    public void testDelete() throws Exception {
        DeleteRequest deleteRequest = new DeleteRequest("haoke", "house",
                "Fosz2XABnqcVXysGIQ8l");

        DeleteResponse response = this.restHighLevelClient.delete(deleteRequest,
                RequestOptions.DEFAULT);

        System.out.println(response.status());// OK or NOT_FOUND
    }

    /**
     * 更新数据
     *
     * @throws Exception
     */
    @Test
    public void testUpdate() throws Exception {
        UpdateRequest updateRequest = new UpdateRequest("haoke", "house",
                "EYuH2HABnqcVXysGrQ9D");

        Map<String, Object> data = new HashMap<>();
        data.put("title", "南京西路2 一室一厅2");
        data.put("price", "4000");
        updateRequest.doc(data);

        UpdateResponse response = this.restHighLevelClient.update(updateRequest,
                RequestOptions.DEFAULT);
        System.out.println("version -> " + response.getVersion());
    }

    /**
     * 测试搜索
     *
     * @throws Exception
     */
    @Test
    public void testSearch() throws Exception {
        SearchRequest searchRequest = new SearchRequest("haoke");
        searchRequest.types("house");

        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchQuery("title", "拎包入住"));
        sourceBuilder.from(0);
        sourceBuilder.size(5);
        sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
        searchRequest.source(sourceBuilder);

        SearchResponse search = this.restHighLevelClient.search(searchRequest,
                RequestOptions.DEFAULT);

        System.out.println("搜索到 " + search.getHits().totalHits + " 条数据.");

        SearchHits hits = search.getHits();
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
~~~

# 3.Spring Data ElasticSearch

https://spring.io/projects/spring-data-elasticsearch

## 3.1.POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.0.RELEASE</version>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>demo-elasticsearch-springboot</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
        </dependency>

        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- java编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
~~~

## 3.2.application.properties

~~~properties
spring.application.name = demo-elasticsearch-springboot

spring.data.elasticsearch.cluster-name=es-fechin-cluster
spring.data.elasticsearch.cluster-nodes=172.16.124.131:9300,172.16.124.131:9301,172.16.124.131:9302
~~~

## 3.3.启动类(略)

## 3.4.实体类

~~~java
package org.fechin.elasticsearch.pojo;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Document(indexName = "fechin", type = "user", createIndex = false)
public class User {

    @Id
    private Long id;
    @Field(store = true)
    private String name;
    @Field
    private Integer age;
    @Field(store = true)
    private String hobby;
}
~~~

## 4.5.测试类

~~~java
package org.fechin.elasticsearch.test;

import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.index.query.QueryBuilders;
import org.fechin.elasticsearch.pojo.User;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.aggregation.AggregatedPage;
import org.springframework.data.elasticsearch.core.query.*;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.ArrayList;
import java.util.List;

/**
 * @Author:朱国庆
 * @Date：2020/3/14 21:37
 * @Desription: haoke-manage
 * @Version: 1.0
 */

@RunWith(SpringRunner.class)
@SpringBootTest
public class TestSpringBootElasticSearch {
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;

    @Test
    public void save() {
        User user = new User();
        user.setId(1001L);
        user.setName("张三");
        user.setAge(20);
        user.setHobby("足球、篮球、听音乐");

        IndexQuery indexQuery = new IndexQueryBuilder().withObject(user).build();

        String index = this.elasticsearchTemplate.index(indexQuery);
        System.out.println(index);
    }

    @Test
    public void testBulk() {
        List list = new ArrayList();
        for (int i = 0; i < 5000; i++) {
            User user = new User();
            user.setId(1001L + i);
            user.setAge(i % 50 + 10);
            user.setName("张三" + i);
            user.setHobby("足球、篮球、听音乐");
            IndexQuery indexQuery = new IndexQueryBuilder().withObject(user).build();
            list.add(indexQuery);
        }
        Long start = System.currentTimeMillis();
        this.elasticsearchTemplate.bulkIndex(list);
        System.out.println("用时：" + (System.currentTimeMillis() - start));
    }

    /**
     * 局部更新，全部更新使用index覆盖即可
     */
    @Test
    public void testUpdate() {
        IndexRequest indexRequest = new IndexRequest();
        indexRequest.source("age", "30");

        UpdateQuery updateQuery = new UpdateQueryBuilder()
                .withId("1001")
                .withClass(User.class)
                .withIndexRequest(indexRequest).build();
        this.elasticsearchTemplate.update(updateQuery);
    }

    @Test
    public void testDelete() {
        this.elasticsearchTemplate.delete(User.class, "1001");
    }

    @Test
    public void testSearch() {
        PageRequest pageRequest = PageRequest.of(1, 10); //设置分页参数

        SearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.matchQuery("name", "张三")) // match查询
                .withPageable(pageRequest)
                .build();

        AggregatedPage<User> users =
                this.elasticsearchTemplate.queryForPage(searchQuery, User.class);

        System.out.println("总页数：" + users.getTotalPages()); //获取总页数

        for (User user : users.getContent()) { // 获取搜索到的数据
            System.out.println(user);
        }
    }
}

~~~



