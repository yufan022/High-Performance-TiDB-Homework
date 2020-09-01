# HP - Lesson 3 -【yufan022】

<a name="FzdF8"></a>
# 环境拓扑
| 组件 | CPU | 内存 | 磁盘 |
| --- | --- | --- | --- |
| workload机 | 8C | 32G | ESSD PL0 云盘 200G |
| TiDB | 8C | 32G | ESSD PL0 云盘 100G |
| PD | 2C | 4G | ESSD PL0 云盘 100G |
| TiKV | 8C | 32G | ESSD PL0 云盘 200G |
| Monitor | 2C | 4G | ESSD PL0 云盘 500G |

<a name="VKXya"></a>
# 
<a name="8ppIf"></a>
# 执行workload
```shell
./gotpc tpcc -H '172.16.0.79' -P 4000 -D tpcc --warehouses 100 prepare
./gotpc tpcc -H '172.16.0.79' -P 4000 -D tpcc --warehouses 100 run --time 1m --threads 512
```
<a name="04GLl"></a>
# 
<a name="LBqvP"></a>
# 查看分析
运行workload，发现有查询 INFORMATION_SCHEMA.CLUSTER_SLOW_QUERY 的慢SQL<br />![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson03/img/image-30.png)<br />运行命令
```shell
curl http://127.0.0.1:10080/debug/zip?seconds=30 --output debug.zip

#执行sql
SELECT
  *,
  (unix_timestamp(Time) + 0E0) AS timestamp
FROM
  `INFORMATION_SCHEMA`.`CLUSTER_SLOW_QUERY`
WHERE
  (
    time BETWEEN from_unixtime(1598774400)
    AND from_unixtime(1598776200)
  )
  AND (DB IN ("tpcc"))
  AND (Plan_digest IN (""))
  AND (Digest = "678df037c4a37e0059fdcfd318aa40727dd327f5e316c68c28ca19fa59a0a22d")
ORDER BY
  Time DESC
LIMIT
  100;


go tool pprof -http=:8080 debug/profile
```
单独查看该sql graph，flame graph<br />![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson03/img/image-31.png)<br />![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson03/img/image-32.png)<br />发现parseSlowLog函数较慢，getOneLine、makeSlice、split耗时较多<br />
<br />同时查看整体TiKV节点IO<br />![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson03/img/image-33.png)<br />![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson03/img/image-34.png)<br />![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson03/img/image-35.png)<br />压测期间TiKV磁盘IO压力较大（本身磁盘性能较差）导致TiKV节点CPU压力不高。<br />

<a name="6J88q"></a>
# 建议优化

1. 对于TiDB查询慢sql，是否有优化空间，对于split makeSlice较为耗时函数能发优化？
1. TiKV节点磁盘是否可根据磁盘性能动态调整特性，例如根据磁盘性能动态调整参数，例如更多batch、buffer？


