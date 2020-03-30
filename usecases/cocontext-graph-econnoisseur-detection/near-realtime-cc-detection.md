# 近实时团伙检测

### 为 CC 信息添加缓存

前面的案例中，介绍了有向无环图中的寻找 CC 的方法，即先寻找祖先，再从祖先出发找到所有的后代。

但在本案例中，由于**共现关系是无向的**，在无向图中并没有所谓祖先的概念。在 TigerGraph 官方的 Github 仓库中，有一个在无向图上统计各个 CC 大小的算法，点击[此处](https://github.com/tigergraph/gsql-graph-algorithms/blob/master/algorithms/examples/Community/conn_comp.gsql)可以查看。此时你已经具备足够的 GSQL 基础，看懂这个代码不难。

如之前所说，这个案例中的 Co-Context Graph，在工程化中做了简化，即在同一个时间窗口内，账号之间并不是两两建边，而是只和前后出现的账号建边，这就会造成 Co-Context Graph 的深度会变得非常深，可以达到几百甚至几千层。

通过之前的案例我们可以体会到，如果是从**一个节点**出发，去统计这个点所在 CC 的深度，TigerGraph 都可以在极短的时间内完成，不论深度多少。尽管如此，在真实的风控系统中，每秒的并发量会很大，可能同时会有几百个这样的查询，此时不论遍历有多块，都无法满足实时性要求。

此时解决问题的一个思路，就是利用缓存，具体来讲，我们可以将某一次 CC 的计算的结果存放在属于该 CC 的所有的节点下，这样后续需要查询的账号，如果它所在的 CC 在短时间内已经被计算过，则直接返回缓存结果即可，而不需要再次重新计算。

举例说明，有下面这个 CC:

```text
u1 - u2 - u3 - u4
```

在查询 `u1` 的时候，会做一次图遍历，返回 `cc_size=4`，此时可以将 `cc_size=4` 缓存到该 CC 下的所有节点中:

```text
u1(cc_size=4, cc_update_time=0) - 
u2(cc_size=4, cc_update_time=0) - 
u3(cc_size=4, cc_update_time=0) - 
u4(cc_size=4, cc_update_time=0)
```

下一次在查询 `u2` 的时候，将当前时间与 `cc_update_time` 做对比，如果很接近，则没有必要再做一次遍历，直接返回缓存结果 `cc_size=4` 即可。

### CC计算与查询分离

上面说的缓存思路，从理论上讲，是一个可行的办法，但在实际工程中，并不一定能达到我们要的效果。原因主要有如下几点:

1. 同一个 CC 中的账号，在时间上本来就具有很高程度的协同，也就是说，针对 `u1` `u2` `u3` `u4` 的查询，很有可能是在几乎相同的时间过来的。此时 `u1` 的查询还没有完全结束，`u2` `u3` `u4` 的查询就已经开始，缓存没有起到作用。
2. 在 CC 计算完成之后，需要将 CC 大小信息写入到该 CC 下的所有节点中，会拖慢响应时间。如果再遇到 1 中所述情况，则短期会面临更大的写入风暴。

另一种常见的思路，是将计算与查询分开。在后台常驻一个进程，用来负责循环更新全图所有节点的 CC 信息，查询则只负责返回缓存结果。

#### 1. 每次更新一个 batch

该 query 每次选取上次更新时间距离当前时间最远的一批节点，然后使用 FOREACH，每次从中挑选 1 个节点，更新这个节点以及该节点所在 CC 的其他节点的 `cc_size` 。详细可以查看如下代码以及注释。

{% code title="update\_cc\_size\_in\_batch.gsql" %}
```sql
CREATE QUERY update_cc_size_in_batch_v2(
  INT batch_size=100,
  DATETIME start_date_time,
  DATETIME end_date_time
) FOR GRAPH MyGraph {
  OrAccum<BOOL> @visited;
  MaxAccum<DATETIME> @cc_update_time;
  
  MapAccum<VERTEX, BOOL> @@vertex_update_status;
  SetAccum<VERTEX> @@batch_accounts;
  MinAccum<DATETIME> @@last_update_time;
  
  INT num_cc_updated = 0;
  INT num_vertices_updated = 0;
  
  all_accounts = {Account.*};
  /*
  按照 cc_update_time 排序，选取 cc 信息“最老”的一批节点
  */
  samples =
    SELECT t
    FROM all_accounts:t
    ORDER BY t.cc_update_time ASC
    LIMIT batch_size
  ;
  
  samples =
    SELECT t
    FROM samples:t
    POST-ACCUM
      @@vertex_update_status += (t -> FALSE),
      @@batch_accounts += t,
      @@last_update_time += t.cc_update_time
  ;
  
  /*
  每次从 1 个节点出发，找到这个节点所在 CC 的所有节点，将 cc_size
  信息更新到每个节点中
  */
  FOREACH account IN @@batch_accounts DO
    /*
    如果当前账号和前面循环中更新的任何一个账号属于同一个 cc，
    那么这个账号无须再更新
    */
    IF @@vertex_update_status.get(account) == TRUE THEN
      CONTINUE;
    END;
  
    seed = {account};
    comp_vs = seed;
    WHILE seed.size() > 0 DO
      seed =
        SELECT t
        FROM seed -(co_ip:e)-> Account:t
        WHERE
          (t.@visited == FALSE) AND
          (e.create_time BETWEEN start_date_time AND end_date_time)
        POST-ACCUM
          t.@visited = TRUE,
          @@vertex_update_status += (t -> TRUE)
        ;
      comp_vs = comp_vs UNION seed;
    END;
  
    UPDATE s FROM comp_vs:s
    SET s.cc_size = comp_vs.size(), 
        s.cc_update_time = now()
    ;
  
    num_cc_updated = num_cc_updated + 1;
    num_vertices_updated = num_vertices_updated + comp_vs.size();
  END;
  
  PRINT num_cc_updated,
        num_vertices_updated,
        @@last_update_time AS last_update_time,
        now() AS current_time
  ;
}
```
{% endcode %}

最后我们将打印出本轮更新涉及到的 CC 数量，节点数量，以及本轮更新之前，该批次节点中，"最早的 CC 更新时间"，以及 TigerGraph 系统当前时间。如果"最早的CC更新时间"与当前时间差异很小，说明数据库中所有节点的 `cc_size` 信息，都很新鲜，此时可以考虑降低该Query的调用频次，如果差异很大，则可以立马继续调用该 Query。

{% hint style="info" %}
注意到 query 中有两个参数，start\_date\_time 与 end\_date\_time，这两个参数用来筛选出在这个时间区间内创建的边，这样可以避免一个 CC 无限增长下去。
{% endhint %}

上述方法，每次取样一批节点，更新这些节点所处的 CC 中所有节点的 `cc_size` 信息，通过多次调用，以更新所有节点的 `cc_size`。由于每一次调用时取样的节点不同，更新所涉及的节点数量也不同，因此计算所需用时也不同。

这里我做了一个实验，以 `batch_size=1000`，将全图所有节点的 `cc_size` 做一次更新，每次调用时，统计本次调用更新了多少个节点，用时多少秒。

实验环境的机器配置为 Intel Core i7-7800X，6核12线程 CPU，使用 TigerGraph 开发版，单机部署。



![](data:image/png;base64,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)

可以发现，每次调用该 query 的用时，基本上和本次调用更新的节点数呈线性相关。平均而言每秒能更新 5000 个节点。

#### 2. 全图更新

参考 TigerGraph GitHub 上的算法例子，下面在提供一个并行化的，一个 Query 进行全图更新的方法

{% code title="update\_cc\_size\_whole\_graph.gsql" %}
```sql
CREATE QUERY update_cc_size_whole_graph(
  DATETIME start_date_time,
  DATETIME end_date_time
) FOR GRAPH MyGraph { 
  MinAccum<INT> @cc_id = 0;
  MinAccum<INT> @old_id = 0;
  OrAccum<BOOL> @active;
  MapAccum<INT, INT> @@comp_sizes;

  all_accounts = {Account.*};
  start =
    SELECT s
    FROM all_accounts:s
    POST-ACCUM 
      s.@cc_id = getvid(s),
      s.@old_id = getvid(s)
  ;
  
  WHILE start.size() > 0 DO
    start = 
      SELECT t
      FROM start:s -(co_ip:e)-> Account:t
      WHERE (e.create_time BETWEEN start_date_time AND end_date_time)
      ACCUM t.@cc_id += s.@cc_id
      POST-ACCUM
        CASE
          WHEN 
            t.@old_id != t.@cc_id
          THEN
            t.@old_id = t.@cc_id,
            t.@active = TRUE
          ELSE
            t.@active = FALSE
          END
      HAVING t.@active == TRUE
    ;
  END;
  
  all_accounts =
    SELECT s
    FROM all_accounts:s
    POST-ACCUM @@comp_sizes += (s.@cc_id -> 1)
  ;
  
  UPDATE s FROM all_accounts:s
    SET s.cc_size = @@comp_sizes.get(s.@cc_id),
        s.cc_update_time = now()
  ;
  
  PRINT
    @@comp_sizes.size() AS num_cc_updated,
    all_accounts.size() AS num_vertices_updated,
    now() AS current_time
  ;
}
```
{% endcode %}

这种方法从**全图所有的节点**同时出发，通知自己的邻居，自己的 `vid` 是多少，每个节点对比自己的 `vid` 和邻居的 `vid`，将较小的 `vid` 设置为自己的 `cc_id`，然后进行下一次迭代。每次迭代之后，只保留在本轮迭代中，更新过 `cc_id` 的节点，直到全图所有节点的 `cc_id` 都不再变化，则算法结束。

在相同的机器上，借助 TigerGraph 很好的并行性能，全图更新的算法效率更高，速度大约是 batch 更新方法的 3 倍。

### 查询节点信息

由于我们之前将查询与计算做了分离，计算的结果已经存放到了节点的属性中，因此我们可以直接使用 TigerGraph 的接口，查询一个节点的 `cc_size` 。

```bash
curl -X GET "http://localhost:9000/graph/MyGraph/vertices/Account/13600000000"
{
  "version": {
    "edition": "developer",
    "api": "v2",
    "schema": 0
  },
  "error": false,
  "message": "",
  "results": [
    {
      "v_id": "13600000000",
      "v_type": "Account",
      "attributes": {
        "cc_size": 20,
        "cc_update_time": "2020-03-01 00:00:00"
      }
    }
  ]
}
```

### 边的维护

为了避免 Graph 无限生长下去，我们可以考虑定期将创建时间比较久远的边删除，以保持整体的响应速度。

{% code title="delete\_edges.gsql" %}
```sql
CREATE QUERY delete_edges(
  STRING edge_type,
  DATETIME start_date_time,
  DATETIME end_date_time
) FOR GRAPH MyGraph {

  V = {ANY};
  DELETE e
  FROM V -(:e)-> ANY
  WHERE (e.type == edge_type) AND 
        (e.create_time BETWEEN start_date_time AND end_date_time)
  ;
}
```
{% endcode %}



