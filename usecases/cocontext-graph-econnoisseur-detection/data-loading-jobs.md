# 数据加载

之前的案例中，介绍了 TigerGraph 从文件加载以及 HTTP 导入数据的方法，本案例会再介绍另一种数据导入方式，即从 Kafka 中导入。

从 Kafka 导入边，最大的好处是可以很大程度提升数据的时效性，如果原始日志也是从 Kafka 中拉取，配合上一节介绍的 Pipeline，即可以以接近实时的方式，完成边的创建工作。

{% hint style="info" %}
注意: 这里说的从 Kafka 导入边，指的是在前面所说的 ETL Pipeline 中，将最后产出的边，存放在 Kafka 的一个 Topic 中，TigerGraph 会自动从该 Topic 中将新生成的边同步到数据库里。
{% endhint %}

### 创建 Data Loading Job

{% code title="co\_context\_edges\_loader.gsql" %}
```sql
/*
sample edge in json format:

{
  "src_node": "u1", 
  "tgt_node": "u2", 
  "edge_type": "co_ip", 
  "edge_attrs": {
      "context", "1.1.1.1",
      "create_time": 1583024431, 
      "time_diff": 30
  }
 }
 
 co_ip edge schema:
 
 UNDIRECTED EDGE co_ip(FROM Account, TO Account, ip STRING, create_time DATETIME, time_diff INT)
 
*/
DROP JOB load_co_context_edges_json
CREATE LOADING JOB load_co_context_edges_json FOR GRAPH MyGraph {
    DEFINE FILENAME jsonfile;
    LOAD jsonfile
        TO EDGE co_ip VALUES (
            $"src_node", 
            $"tgt_node", 
            $"edge_attrs":"context", 
            REDUCE(max($"edge_attrs":"create_time")),
            REDUCE(min($"edge_attrs": "time_diff"))
        ) WHERE $"edge_type" == "co_ip"
    USING JSON_FILE="true";
}
```
{% endcode %}

```bash
$ gsql -g MyGraph co_context_edges_loader.gsql
```

`co_ip` 边第三个字段需要填写的是 `ip` 信息，他来自于 JSON 数据中的 `edge_attrs` 下的 `context` 字段，可以通过冒号来访问下一层级的字段。比如这里，通过 `$"edge_attrs":"context"` 来获得 `ip` 信息。

在 TigerGraph 中，两个节点之间，相同类型的边，只能有一条。比如 `u1` 和 `u2` 之间，只能有一条 `co_ip` 边，如果数据中出现了多条，那么后面的数据将覆盖之前的数据。对于 `co_ip` 边，两个账号非常有可能在后续的时间再次在同一个 IP 下共现的，这里我们希望将 `create_time` 设置成他们最后一次共现的时间，将 `time_diff` 设置成他们曾今时间最接近的时候的时间差。这里我们就需要用到 **Reducer Functions**。

`REDUCE(max($"edge_attrs":"create_time"))` 用来保留最大的 `create_time`

`REDUCE(min($"edge_attrs": "time_diff"))` 用来保留最小的 `time_diff`

除了`min` 和 `max` 外，还有很多 Reducer Functions，譬如可以用 `add` 来统计这条边出现过来几次，等等。详细可以在 TigerGraph 官方文档中搜索 Reducer Functions。

之前有说过，除了 `co_ip` 边外，后续可能还有其他的 `co_context` 边，即数据源中除了 `co_ip` 外还有其他类型的边，因此我们需要增加一个过滤条件:

 `WHERE $"edge_type" == "co_ip"`

### 创建 Kafka Data Source

首先，需要写一个配置文件，用来定义 Kafka 集群的地址，消费组等信息。假设集群地址为 `192.168.5.90:9092` ，给 TigerGraph 定义的消费组 id 为 `tigergraph` :

{% code title="kafka.config" %}
```javascript
{
    "broker": "192.168.5.60:9092",
    "kafka_config": {
        "group.id": "tigergraph"
    }
}
```
{% endcode %}

将该配置文件放置在 TigerGraph 所在服务器的 `/home/tigergraph` 目录下，进入 GSQL 客户端:

```bash
GSQL-Dev > CREATE DATA_SOURCE KAFKA k1 = "/home/tigergraph/kafka.config" FOR GRAPH MyGraph
```

上述命令定义里一个名为 `k1` 的 DATA\_SOURCE

### 配置 Topic 信息

除了 DATA\_SOURCE 外，我们还需要配置之后要消费的 Topic

在 TigerGraph 所在服务器的 `/home/tigergraph` 目录下，创建一个关于 `co-context` 边的 topics 配置。假设我们在 ETL Pipeline 的最后，将生成的边存放到了名为 `co-context-edges` 的 topic 中:

{% code title="kafka\_co\_context\_edges\_topic\_partition.conf" %}
```javascript
{
  "topic": "co-context-edges",
  "partition_list": [
    {
      "start_offset": -1,
      "partition": 0
    }
  ]
}
```
{% endcode %}

上面 `start_offset = -1` 表示，从这个队列中最新的数据开始消费，`start_offset = -2` 表示，从这个队列中第一条消息开始消费。

### 执行 Loading Job

进入 GSQL 客户端

```bash
GSQL-Dev > USE GRAPH MyGraph
Using graph 'MyGraph'
GSQL-Dev > RUN LOADING JOB load_co_context_edges_json USING jsonfile="$k1:/home/tigergraph/kafka_co_context_edges_topic_partition.conf"Try to list topic metadata from Kafka broker '192.168.5.60:9092', timeout: 3 sec ...
[Tip: Use "CTRL + C" to stop displaying the loading status update, then use "SHOW LOADING STATUS jobid" to track the loading progress again]
[Tip: Manage loading jobs with "ABORT/RESUME LOADING JOB jobid"]
Starting the following job, i.e.
  JobName: load_co_context_edges_json, jobid: MyGraph.load_co_context_edges_json.kafka.k1.1.1585207057408
  Loading log: '/home/tigergraph/tigergraph/logs/restpp/restpp_kafka_loader_logs/MyGraph/MyGraph.load_co_context_edges_json.kafka.k1.1.1585207057408.log'

Job "MyGraph.load_co_context_edges_json.kafka.k1.1.1585207057408" loading status
[RUNNING] m1 ( Total: 1 )
  +---------------------------------------------------------------------------------+
  |   TOPIC PARTITION |   LOADED MESSAGES |   AVG SPEED |   DURATION |   LOADED SIZE|
  |co-context-edges:0 |            149600 |     51 kl/s |     2.90 s |       2.94 MB|
  +---------------------------------------------------------------------------------+
Job "MyGraph.load_co_context_edges_json.kafka.k1.1.1585205329349" loading status
```

Job 后面跟着的是 `jobid` ，可以看到从 job 开始起，一共消费了 149600 条消息，平均每秒消费 5 万条消息，这速度还是挺惊人的。

后续可以通过这个 `jobid` 查看最新的运行情况。

```bash
$ gsql -g MyGraph "SHOW LOADING STATUS MyGraph.load_co_context_edges_json.kafka.k1.1.1585205329349"
Connecting to 192.168.5.60
If there is any relative path, it is relative to tigergraph/dev/gdk/gsql
Job "MyGraph.load_co_context_edges_json.kafka.k1.1.1585207057408" loading status
[RUNNING] m1 ( Total: 1 )
  +---------------------------------------------------------------------------------+
  |   TOPIC PARTITION |   LOADED MESSAGES |   AVG SPEED |   DURATION |   LOADED SIZE|
  |co-context-edges:0 |            149600 |     51 kl/s |     2.90 s |       2.94 MB|
  +---------------------------------------------------------------------------------+
```

如果忘记了 `jobid` 也没有关系，直接用 `SHOW LOADING STATUS ALL` 。

