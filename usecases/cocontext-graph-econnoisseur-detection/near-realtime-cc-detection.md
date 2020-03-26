# 近实时团伙检测

### 为 CC 信息添加缓存

前面的案例中，介绍了有向无环图中的寻找 CC 的方法，即，先寻找祖先，再从祖先出发找到所有的后代。

但在本案例中，由于**共现关系是无向的**，在无向图中并没有所谓祖先的概念。在 TigerGraph 官方的 Github 仓库中，有一个在无向图上统计各个 CC 大小的算法，点击[此处](https://github.com/tigergraph/gsql-graph-algorithms/blob/master/algorithms/examples/Community/conn_comp.gsql)可以查看。此时你已经具备足够的 GSQL 基础，看懂这个代码不难。但是这种统计 CC 大小的方法是比较低效的，它只适用于一些很浅的图，不适合存在深度关系的图。

如之前所说，这个案例中的 Co-Context Graph，在工程化中做了简化，即，在同一个时间窗口内，账号之间并不是两两建边，而是只和前后出现的账号建边，这就会造成 Co-Context Graph 的深度会变得非常深，可以达到几百甚至几千层。

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



