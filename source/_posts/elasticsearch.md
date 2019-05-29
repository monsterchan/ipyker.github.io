layout: post
title: Elasticsearch7.0详解
author: Pyker
categories: elastic
tags:
  - elasticsearch
date: 2019-05-20 10:45:30
---

# 什么是Elasticsearch
Elasticsearch是一个高度可扩展的开源全文搜索和分析引擎。 它允许您快速，近乎实时地存储，搜索和分析大量数据。 它通常用作底层引擎/技术，本身扩展性很好，可以扩展到上百台服务器，处理PB级别的数据。 为具有复杂搜索特性和需求的应用程序提供支持。Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。

# Elasticsearch能干什么
* 您运行在线网上商店，允许您的客户搜索您销售的产品。 在这种情况下，您可以使用Elasticsearch存储整个产品目录和库存，并为它们提供搜索和自动填充建议。

* 您希望收集日志或交易数据，并且希望分析和挖掘此数据以查找趋势，统计信息，摘要或异常。 在这种情况下，您可以使用Logstash（Elasticsearch / Logstash / Kibana堆栈的一部分）来收集，聚合和解析数据，然后让Logstash将此数据提供给Elasticsearch。 一旦数据在Elasticsearch中，您就可以运行搜索和聚合来挖掘您感兴趣的任何信息。

* 您运行价格提醒平台，允许对价格明确的客户指定一条规则，例如“我有兴趣购买特定的电子产品，如果某个供应商的电子产品价格在下个月内跌破X美元，我希望得到通知”。在这种情况下，您可以抓取供应商价格，将其推入Elasticsearch并使用其反向搜索（Percolator）功能来匹配价格变动与客户查询，并最终在发现匹配后将警报推送给客户。

* 您有分析/商业智能需求，并希望快速调查，分析，可视化并询问有关大量数据的特定问题（想想数百万或数十亿条记录）。 在这种情况下，您可以使用Elasticsearch存储数据，然后使用Kibana（Elasticsearch / Logstash / Kibana堆栈的一部分）构建自定义仪表板，该仪表板可以可视化对您来说重要的数据。 此外，您可以使用Elasticsearch聚合功能针对您的数据执行复杂的商业智能查询。

# Elasticsearch核心概念
## Near-realtime(NRT)： 近实时
* Elasticsearch是一个近实时搜索平台。 这意味着从索引文档到可搜索文档的时间有一点延迟（通常是一秒）。

## Cluster：集群
* cluster是一个或多个节点（服务器）的集合，它们共同保存您的整个数据，并提供跨所有节点的联合索引和搜索功能。 群集由唯一名称标识，默认情况下为“elasticsearch”。 这个标识很重要，节点会根据这个标识加入到一个集群。

* 确保不要在不同的环境中使用相同的群集名称，这会导致节点加入错误的集群。例如，您可以将logging-dev，logging-stage和logging-prod用于开发，测试和生产集群。
{% note warning %}请注意，如果一个集群只有一个节点也是完全可行的，此外你也可以建立多个独立的集群，每个集群有自己的唯一标识。{% endnote %}

## Node：节点
* 节点是作为群集一部分的单个服务器，用于存储集群数据并为集群提供`索引`和`搜索`能力。 与集群一样，每个节点也有一个唯一名称标识，在节点启动的时候会默认分配给它一个随机通用唯一标识(UUID)。 你也可以自定义节点标识，以代替默认标识。在集群的管理中，可以使用节点标识来对应网络中的服务器名称。

* 节点可以根据配置的集群标识加入指定的集群。一般来说，每个节点在启动之后会默认加入一个叫elasticsearch的集群。假如在一个节点可以相互发现对方的网络环境中，启动多个节点，他们会自动加入到一个标识为elasticsearch的集群。

* 在一个独立的集群中，你可以建立任意个数的节点。此外，如果当前网络环境中没有其他的elasticsearch节点在运行，那么单独启动一个节点会形成一个单节点的集群，其集群标识为elasticsearch

## Index：索引
索引是具有某些类似特征的`文档（Document）`集合。 例如，可以为客户数据建立索引，为产品目录建立另一个索引，为订单数据建立另一个索引。索引由名称标识(必须全部为小写)，当对其中的文档执行索引、搜索、更新和删除操作时，需要引用索引名称。

## ~~Type：类型~~
{% label danger@warning :在6.0.0后弃用 %}

* 它曾经是索引的逻辑类别/分区，允许您在同一索引中存储不同类型的文档，例如，一种用户类型，另一种用于博客帖子。 不再可能在索引中创建多个类型。在Elasticsearch 7.0.0或更高版本中创建的索引不再接受_default_映射。 在6.x中创建的索引将继续像以前一样在Elasticsearch 6.x中运行。 在7.0中的API中不推荐使用type，对索引创建，放置映射，获取映射，放置模板，获取模板和获取字段映射API进行重大更改。详见[Removal of mapping types](https://www.elastic.co/guide/en/elasticsearch/reference/current/removal-of-types.html)

## Document：文档
* Document是可以被索引的基本信息单元。 例如，您可以为单个客户提供文档，为单个产品提供另一个文档，为单个订单提供另一个文档。 该文档以JSON格式表示。在`索引（index）`或者`类型（type）`中可以存储任意多的文档。尽管文档实际存储在索引中，但是使用中必须把文档编入或者分配到一个索引的类型中。

## Shard：分片
* 一个索引index可能会存储大量的数据，大到超过一个节点的存储能力。例如：一个占用1T磁盘存储空间的索引，对单节点来说可能会太大，或者造成查询请求太慢。

* 为了解决这个问题，Elasticsearch提供了将索引细分为多个称为`分片(shard)`的功能。 创建索引时，只需定义所需的分片数即可。 每个分片本身都是一个功能齐全且独立的“索引”，每个分片可以放到不同的服务器上。

* 当你查询的索引分布在多个分片上时，Elasticsearch会把查询发送给每个相关的分片，并将结果组合在一起，而应用程序并不知道分片的存在。即：这个过程对用户来说是透明的。

## Replicas：副本
* 为提高查询吞吐量或实现高可用性，可以使用分片副本。 副本是一个分片的精确复制，每个分片可以有零个或多个副本。Elasticsearch会中可以有许多相同的分片，其中之一被选择更改索引操作，这种特殊的分片称为主分片。 当主分片丢失时，如：该分片所在的数据不可用时，集群将副本提升为新的主分片。

## Full-text-search：全文检索
全文检索就是对一篇文章进行索引，可以根据关键字搜索，类似于mysql里的like语句。 全文索引就是把内容根据词的意义进行分词，然后分别创建索引，例如“你们的激情是因为什么事情来的” 可能会被分词成：“你们“，”激情“，“什么事情“，”来“ 等token，这样当你搜索“你们” 或者 “激情” 都会把这句搜出来。

# Elasticsearch的安装
由于[上篇文章](https://www.ipyker.com/2019/03/15/install-efk.html)已经对elasticsearch7.0.1版本进行了安装，用户可以参考该安装方式，进行多节点集群安装部署。

# Restful API接口能干什么
Elasticsearch提供了一个非常全面和强大的REST API，可以使用它与集群进行交互。使用该API可以做一些事情如下:
* 检查群集，节点和索引运行状况，状态和统计信息
* 管理您的群集，节点和索引数据和元数据
* 对索引执行CRUD（创建，读取，更新和删除）和搜索操作
* 执行高级搜索操作，例如分页，排序，过滤，脚本编写，聚合等等

下面我们将通过使用`Restful API接口`风格来使用Elasticsearch。

# 集群信息检查
通过上面对elasticsearch的了解后，我们现在可以从检查集群基本运行状况开始，elasticsearch支持`curl`、`HTTP\REST`以及`Kibana开发控制台`上来操作elasticsearch。我们这里使用`curl`方式进行操作,并且使用[Elastic stack之EFK安装](https://www.ipyker.com/2019/03/15/install-efk.html)的elasticsearch集群服务器。

## 集群健康
```bash
# 通过响应我们可以观察到集群 efk-cluster除于green状态，节点总数2个，节点存储2个，分片数38，主分片19
$ curl -XGET "http://192.168.20.211:9200/_cat/health?v"
epoch      timestamp cluster        status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1559096592 02:23:12  efk-cluster green           2         2     38  19    0    0        0             0                  -                100.0%

# 也可以这样操作获取集群信息状态
$ curl -XGET http://192.168.20.211:9200/_cluster/health?pretty
{
  "cluster_name" : "efk-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 2,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 19,
  "active_shards" : 38,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
>* {% label success@green %} — 一切都很好（集群功能齐全）
* {% label warning@yellow %} — 所有数据都可用，但尚未分配一些副本（群集功能齐全）
* {% label danger@red %} — 某些数据由于某种原因不可用（群集部分功能）

## 集群详细信息
```bash
# 这将获得集群和节点的名称以及UUID，及一些其他属性
$ curl -XGET http://192.168.20.211:9200/_cluster/state/nodes?pretty
{
  "cluster_name" : "efk-cluster",
  "cluster_uuid" : "93hPHHs9SGu6EqtOJCpQbA",
  "nodes" : {
    "eKK8iLHjSx2pW9yCfIwacA" : {
      "name" : "efk-node2",
      "ephemeral_id" : "Goc8GevQThakZ7y3GY4oLA",
      "transport_address" : "192.168.20.212:9300",
      "attributes" : {
        "ml.machine_memory" : "8353046528",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true"
      }
    },
    "fK9VZWuJTWKBhFlXgHLkRA" : {
      "name" : "efk-node1",
      "ephemeral_id" : "483CBL_zRwyqKI3QUTwyUg",
      "transport_address" : "192.168.20.211:9300",
      "attributes" : {
        "ml.machine_memory" : "8353046528",
        "xpack.installed" : "true",
        "ml.max_open_jobs" : "20"
      }
    }
  }
}
```
## 集群master状态
```bash
# 和查看集群信息类似，这里只查看集群master信息
$ curl -XGET http://192.168.20.211:9200/_cluster/state/master_node?pretty
{
  "cluster_name" : "efk-cluster",
  "cluster_uuid" : "93hPHHs9SGu6EqtOJCpQbA",
  "master_node" : "fK9VZWuJTWKBhFlXgHLkRA"
}

#也可以这样获取mster id ip信息
$ curl -XGET http://192.168.20.211:9200/_cat/master?v
id                     host           ip             node
fK9VZWuJTWKBhFlXgHLkRA 192.168.20.211 192.168.20.211 efk-node1
```
## 集群节点状态
```bash
# 可以查询到集群的节点数，堆、运存、cpu、平均负载等信息，其中efk-node1为* 表示为ES主节点
$ curl -XGET http://192.168.20.211:9200/_cat/nodes?v
ip             heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.20.212           57          56   0    0.07    0.09     0.07 mdi       -      efk-node2
192.168.20.211           50          56   0    0.05    0.07     0.05 mdi       *      efk-node1
```

## 查看集群索引
```bash
# 这里没有粘贴出.开头的索引，只列出自建的alading索引，这样方便看出索引健康、状态、主/副本数等信息
$ curl -XGET http://192.168.20.211:9200/_cat/indices?v
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   alading                         JEG96RCIRkOgzC-JQudF5Q   1   1          1            0        7kb          3.5kb
```

# Elasticsearch的CRUD操作
通过上面的介绍，相信大家已经知道如何和elasticsearch数据通信了。下面我们将简单介绍如何在ES中进行增删改查操作。而Elasticsearch的Restful API对应的增删改查为：`GET`、`POST`、`PUT`、`DELETE`、`HEAD`，他们的含义分别为：
* {% label success@GET %}：获取请求对象的当前状态。 
* {% label success@POST %}：改变对象的当前状态。 
* {% label success@PUT %}：创建一个对象。 
* {% label success@DELETE %}：删除对象。 
* {% label success@HEAD %}：请求获取对象的基础信息。 

如果Elasticsearch的CRUD和mysql的概念关系对比的话，可以理解为下表：

| Mysql | Elasticsearch |
| :-----: | :-------------: |
| Database | Index |
| Table | Type |
| Row | Document |
| Column | Field |
| Schema | Mapping |
| Index | Everything is indexed | 
| SQL | Query DSL |
| Select * from table ... | GET http://... |
| Update table Set ... | PUT http://... |

## 创建索引
```bash
# 例如我们创建一个customer的索引
$ curl -XPUT http://192.168.20.211:9200/customer?pretty
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "customer"
}
```
>`pretty`为以美观的json格式输出。

此时我们查看当前已建立的索引可以发现customer
```bash
# alading为我们之前建立的，customer为我们刚刚建立的
$ url -XGET http://192.168.20.211:9200/_cat/indices?v
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   alading                         JEG96RCIRkOgzC-JQudF5Q   1   1          1            0        7kb          3.5kb
green  open   customer                        aayvk0MRTP2OMaBOcSF0DQ   1   1          0            0       566b           283b
```

## 添加索引文档
```bash
# 我们在customer索引中建立一个id为1的文档
$ curl -H "Content-Type: application/json" -XPUT http://192.168.20.211:9200/customer/_doc/1?pretty -d '{"name":"pyker"}'
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}
```

## 查询索引文档
```bash
# 这里查询上一步骤建立的索引为customer，类型为_doc，id为1的文档
$ curl -XGET http://192.168.20.211:9200/customer/_doc/1?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "pyker"
  }
}
```
>`_version`每一次操作都会加1，`found`为true表示找到了该文档，`_source`返回我们从索引中检出的完整JSON文档。

## 更改索引文档内容
```bash
# 执行方式和前面一样，指定要操作的索引、类型、文档id，然后修改需要修改的字段名和值。
$ curl -H "Content-Type: application/json" -XPOST http://192.168.20.211:9200/customer/_doc/1?pretty -d '{"name":"pyker zhang"}'
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}
```

```bash
# 更新后我们在查询，发现name字段值已经被更改了，并且_version的值也加1了。
$ curl -XGET http://192.168.20.211:9200/customer/_doc/1?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "_seq_no" : 2,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "pyker zhang"
  }
}
```

## 删除索引文档
按照上面的步骤,我们在建立一个id为2的文档
```bash
$ curl -H "Content-Type: application/json" -XPUT http://192.168.20.211:9200/customer/_doc/2?pretty -d '{"name":"jack"}'
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}
```
```bash
$ curl -XGET http://192.168.20.211:9200/customer/_doc/2?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "_seq_no" : 1,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "jack"
  }
}
```
那么此时如果我们要删除这个文档进行如下操作
```bash
# 可以发现_version为第二次操作，result结果为deleted，表示该文档已经被删除
$ curl -XDELETE http://192.168.20.211:9200/customer/_doc/2?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 2,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}
```
此时你在去customer中查找id为2的文档就会显示没找到，如下
```bash
# found为false表示该文档没有找到
$ curl -XGET http://192.168.20.211:9200/customer/_doc/2?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "2",
  "found" : false
}
```

## ES批处理操作
通过上面的操作都是基于单个文档进行的，Elasticsearch也提供了使用 `_bulk API` 批量执行上述任何操作的功能。这个功能是非常重要的，因为它提供了一个非常有效的机制来尽可能快地进行多个操作，并且尽可能减少网络的往返行程。简单举个例子，下面会在一个 bulk操作中索引两个文档：
```bash
# 通过_bulk我们同时创建了id为3和4 name为John Doe和Jane Doe的两个文档
$ curl -H "Content-Type: application/json" -XPOST http://192.168.20.211:9200/customer/_doc/_bulk?pretty -d '
{"index":{"_id":"3"}}
{"name": "John Doe" }
{"index":{"_id":"4"}}
{"name": "Jane Doe" }
'
{
  "took" : 4,
  "errors" : false,
  "items" : [
    {
      "index" : {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "3",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 6,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "4",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 7,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}
```
下面语句批处理执行增加id为2的数据然后执行删除id为3的数据，并且更新id为4的数据
```bash
# 可以清楚的看到各_id的result值对应着我们的增加、删除、更新需求。
$ curl -H "Content-Type: application/json" -XPOST http://192.168.20.211:9200/customer/_doc/_bulk?pretty -d '
{"index":{"_id":"2"}}
{"name": "jack" }
{"delete":{"_id":"3"}}
{"update": {"_id":"4"}}
{"doc":{"name":"John Doe becomes Jane Doe"}} 
'
{
  "took" : 7,
  "errors" : false,
  "items" : [
    {
      "index" : {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "2",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 12,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "delete" : {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "3",
        "_version" : 2,
        "result" : "deleted",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 13,
        "_primary_term" : 1,
        "status" : 200
      }
    },
    {
      "update" : {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "4",
        "_version" : 2,
        "result" : "updated",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 14,
        "_primary_term" : 1,
        "status" : 200
      }
    }
  ]
}
```
通过上面的操作，现在我们来验证一下结果
```bash
# _id为2 name为jack的文档我们创建成功了
$ curl -XGET http://192.168.20.211:9200/customer/_doc/2?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "2",
  "_version" : 1,
  "_seq_no" : 12,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "jack"
  }
}

# _id为3的文档我们进行了删除操作，所以found为false了
$ curl -XGET http://192.168.20.211:9200/customer/_doc/3?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "3",
  "found" : false
}

# _id为4的文档name值已经被我们更新成John Doe becomes Jane Doe
$ curl -XGET http://192.168.20.211:9200/customer/_doc/4?pretty
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "4",
  "_version" : 2,
  "_seq_no" : 14,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "John Doe becomes Jane Doe"
  }
}
```
>值得注意点是`_Bulk API` 不会因其中一个操作失败而失败。 如果单个操作因任何原因失败，它将继续处理其后的其余操作。 `_bulk API`返回时，它将为每个操作提供一个状态（按照发送的顺序），以便您可以检查特定操作是否失败。

## 列出索引中所有文档
```bash
# 通过索引名/_search 可以查看当前索引下所有类型的文档
$ curl -XGET "http://192.168.20.211:9200/customer/_search?pretty"
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 7,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "name" : "pyker zhang"
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "en",
        "_score" : 1.0,
        "_source" : {
          "language" : "english"
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "cn",
        "_score" : 1.0,
        "_source" : {
          "language" : "chinese"
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "6",
        "_score" : 1.0,
        "_source" : {
          "name" : "lily"
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 1.0,
        "_source" : {
          "name" : "lucy"
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "name" : "jack"
        }
      },
      {
        "_index" : "customer",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "name" : "John Doe becomes Jane Doe"
        }
      }
    ]
  }
}
```
>以上就是ES的增删改查的基本操作，如果仔细学习了以上命令，应该会发现 elasticsearch 访问数据所使用的模式，概括如下：
**`curl -X<REST Verb> <Node>:<Port>/<Index>/<Type>/<ID>`**

# Search API (查询API)
运行搜索有两种基本方法:一种是通过`REST请求URI`发送搜索参数，另一种是通过`REST请求体`发送搜索参数。请求体方法允许您更富表现力，还可以以更可读的JSON格式定义搜索。我们将尝试请求URI方法的一个示例，但是在本教程的其余部分中，我们将只使用请求体方法。
<div class="note primary"><p>为了方便操作，我们加载示例数据集,可以从[这里](https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/src/test/resources/accounts.json)下载示例数据集(accounts.json)。</p></div>

将其解压缩到当前目录，并将其加载到集群中，如下所示:
```bash
$ curl -H "Content-Type: application/json" -XPOST "http://192.168.20.211:9200/bank/_bulk?pretty&refresh" --data-binary "@accounts.json"

# 加载完后查看索引清楚的看到bank索引有1000条数据文档
$ curl -XGET "http://192.168.20.211:9200/_cat/indices?v"
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   bank                            J5R8uBBoQ4ie-x3eNf8LpQ   1   1       1000            0    828.7kb        414.3kb
```
### 通过REST请求URI发送搜索参数
```bash
$ curl -X GET "http://192.168.20.211:9200/bank/_search?q=*&sort=account_number:asc&pretty"
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "0",
        "_score" : null,
        "_source" : {
          "account_number" : 0,
          "balance" : 16623,
          "firstname" : "Bradshaw",
          "lastname" : "Mckenzie",
          "age" : 29,
          "gender" : "F",
          "address" : "244 Columbus Place",
          "employer" : "Euron",
          "email" : "bradshawmckenzie@euron.com",
          "city" : "Hobucken",
          "state" : "CO"
        },
        "sort" : [
          0
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "account_number" : 1,
          "balance" : 39225,
          "firstname" : "Amber",
          "lastname" : "Duke",
          "age" : 32,
          "gender" : "M",
          "address" : "880 Holmes Lane",
          "employer" : "Pyrami",
          "email" : "amberduke@pyrami.com",
          "city" : "Brogan",
          "state" : "IL"
        },
        "sort" : [
          1
        ]
      },
      ...
      ...
```
示例返回所有bank中的索引数据。其中 `q=*`  表示匹配索引中所有的数据, 按照`account_number`值进行升序排序。
<div class="note info"><p>注意：如果siez不指定，则默认搜索返回10条文档数据。</p></div>
* `took`: Elasticsearch执行搜索的时间（以毫秒为单位）
* `time_out`：告诉我们搜索是否超时
* `_shards`：告诉我们搜索了多少个分片，以及搜索成功/失败分片的计数
* `hits`：搜索结果
* `hits.total`： 包含与搜索条件匹配的文档总数的信息的对象，默认值为10000，可以显示通过[track_total_hits](https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-request-track-total-hits.html)设置为true或整数
* `hits.hits`：实际的搜索结果数组（默认为前10个文档）
* `hits.sort`：按哪个字段排序的结果（如果按分数排序则丢失）
* `hits._score` and `max_score`：暂时忽略这些字段

### 通过REST请求体发送搜索参数

```bash
$ curl -XGET "http://192.168.20.211:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d '
{"query": {"match_all": {}}, "from": 370, "size": 7, "sort": [{
"account_number": {"order": "asc"}}]}'
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "370",
        "_score" : null,
        "_source" : {
          "account_number" : 370,
          "balance" : 28499,
          "firstname" : "Oneill",
          "lastname" : "Carney",
          "age" : 25,
          "gender" : "F",
          "address" : "773 Adelphi Street",
          "employer" : "Bedder",
          "email" : "oneillcarney@bedder.com",
          "city" : "Yorklyn",
          "state" : "FL"
        },
        "sort" : [
          370
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "371",
        "_score" : null,
        "_source" : {
          "account_number" : 371,
          "balance" : 19751,
          "firstname" : "Barker",
          "lastname" : "Allen",
          "age" : 32,
          "gender" : "F",
          "address" : "295 Wallabout Street",
          "employer" : "Nexgene",
          "email" : "barkerallen@nexgene.com",
          "city" : "Nanafalia",
          "state" : "NE"
        },
        "sort" : [
          371
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "372",
        "_score" : null,
        "_source" : {
          "account_number" : 372,
          "balance" : 28566,
          "firstname" : "Alba",
          "lastname" : "Forbes",
          "age" : 24,
          "gender" : "M",
          "address" : "814 Meserole Avenue",
          "employer" : "Isostream",
          "email" : "albaforbes@isostream.com",
          "city" : "Clarence",
          "state" : "OR"
        },
        "sort" : [
          372
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "373",
        "_score" : null,
        "_source" : {
          "account_number" : 373,
          "balance" : 9671,
          "firstname" : "Simpson",
          "lastname" : "Carpenter",
          "age" : 21,
          "gender" : "M",
          "address" : "837 Horace Court",
          "employer" : "Snips",
          "email" : "simpsoncarpenter@snips.com",
          "city" : "Tolu",
          "state" : "MA"
        },
        "sort" : [
          373
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "374",
        "_score" : null,
        "_source" : {
          "account_number" : 374,
          "balance" : 19521,
          "firstname" : "Blanchard",
          "lastname" : "Stein",
          "age" : 30,
          "gender" : "M",
          "address" : "313 Bartlett Street",
          "employer" : "Cujo",
          "email" : "blanchardstein@cujo.com",
          "city" : "Cascades",
          "state" : "OR"
        },
        "sort" : [
          374
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "375",
        "_score" : null,
        "_source" : {
          "account_number" : 375,
          "balance" : 23860,
          "firstname" : "Phoebe",
          "lastname" : "Patton",
          "age" : 25,
          "gender" : "M",
          "address" : "564 Hale Avenue",
          "employer" : "Xoggle",
          "email" : "phoebepatton@xoggle.com",
          "city" : "Brule",
          "state" : "NM"
        },
        "sort" : [
          375
        ]
      },
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "376",
        "_score" : null,
        "_source" : {
          "account_number" : 376,
          "balance" : 44407,
          "firstname" : "Mcmillan",
          "lastname" : "Dunn",
          "age" : 21,
          "gender" : "F",
          "address" : "771 Dorchester Road",
          "employer" : "Eargo",
          "email" : "mcmillandunn@eargo.com",
          "city" : "Yogaville",
          "state" : "RI"
        },
        "sort" : [
          376
        ]
      }
    ]
  }
}
```
命令中`query`是REST请求体的查询语句，`match_all`为匹配所有文档，`from`为从第370个文档开始， `size`为查询文档的个数，`sort`为按照account_number的值进行升序排序。

在例如以下请求将查询bank索引 `state`为NE的文档且只显示`account_number`, `balance`, `state`三个字段信息，显示结果以`account_number`字段进行升序排序。
```bash
$ curl -XGET "http://192.168.20.211:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "query": {"match": {
    "state": "NE"
  }}
  , "_source": ["account_number", "balance","state"]
  , "sort": [
    {
      "account_number": {
        "order": "asc"
      }
    }
  ]
}'
```
<div class="note success"><p>更多如`bool`、`must`、`must_not`、`shouldSearch`、`filter`、`range`、`aggs` 的Search API文档请参考[这里](https://www.elastic.co/guide/en/elasticsearch/reference/7.0/getting-started-search.html)</p></div>

### 查询单个字段匹配

>为了方便演示，下面的操作都是在kibana 开发工具上操作的
```bash
# 查询匹配account_number值为200的文档
GET /bank/_search
{
 "query": {"match": {
   "account_number": "200"
 }} 
}
```

### 包含短句匹配
```bash
# 匹配address字段包含值为mill lane整体的文档
GET /bank/_search
{
  "query": {"match_phrase": {
    "address": "mill lane"
  }}
}
```
### 与关系匹配
```bash
# 匹配address字段值中有mill也有lane的文档
GET /bank/_search 
{
  "query": {"bool": {"must": [
    {"match": {
      "address": "mill"
    }},
    {"match": {
      "address": "lane"
    }}
  ]}}
}
```

### 或关系匹配
```bash
# 匹配address字段值中有mill或者有lane的文档
GET /bank/_search
{
  "query": {"bool": {"should": [
    {"match": {
      "address": "mill"
    }},{
      "match": {
        "address": "lane"
      }
    }
  ]}
  }
}
```

### 非关系匹配
```bash
# 匹配address字段值中不能有mill和lane的文档
GET /bank/_search
{
  "query": {"bool": {"must_not": [
    {"match": {
      "address": "mill"
    }},
    {"match": {
      "address": "lane"
    }}
  ]}}
}
```

### 与和非匹配
```bash
# 匹配age字段值为40，但是state不能是ID的文档
GET /bank/_search
{
  "query": {"bool": {"must": [
    {"match": {
      "age": "40"
    }}
  ],
    "must_not": [
      {"match": {
        "state": "ID"
      }}
    ]
  }
  }
}
```

### 范围匹配
```bash
# 全文匹配balance字段值大于10000小于20000的文档
GET /bank/_search
{
  "query": {"bool": {"must": [
    {"match_all": {}}
  ],
    "filter": {"range": {
      "balance": {
        "gte": 10000,
        "lte": 20000
      }
    }}
  }}
}
```

### 聚合匹配
```bash
# 统计state对应的值的合计数量，size=0表示不显示搜索匹配的结果只显示聚合结果。
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
    "terms": {
      "field": "state.keyword"
     }
    }
  }
}
```

### 嵌套聚合匹配
```bash
# 统计state对应值的合计数量以及对应state文档中balance字段平均值
GET /bank/_search
{
  "size": 0
  , "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
      , "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```