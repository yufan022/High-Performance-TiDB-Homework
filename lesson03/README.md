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
运行workload，发现有查询 INFORMATION_SCHEMA.CLUSTER_SLOW_QUERY 的慢SQL<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437932/1598781095176-2bae376f-d6b9-4251-88af-94b204d4839c.png#align=left&display=inline&height=603&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1206&originWidth=1976&size=201248&status=done&style=none&width=988)<br />运行命令
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
单独查看该sql graph，flame graph<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437932/1598843640617-2e51478d-e79c-43ce-9ffd-58b903b63bb4.png#align=left&display=inline&height=846&margin=%5Bobject%20Object%5D&name=image.png&originHeight=846&originWidth=1162&size=174638&status=done&style=none&width=1162)<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437932/1598843655766-b04f2170-9827-4c40-9b09-5e1b644019cb.png#align=left&display=inline&height=756&margin=%5Bobject%20Object%5D&name=image.png&originHeight=756&originWidth=1309&size=127832&status=done&style=none&width=1309)<br />发现parseSlowLog函数较慢，getOneLine、makeSlice、split耗时较多<br />
<br />同时查看整体TiKV节点IO<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437932/1598863398081-3e45a120-793a-40ae-8b69-39b8bee30f08.png#align=left&display=inline&height=248&margin=%5Bobject%20Object%5D&name=image.png&originHeight=248&originWidth=857&size=41217&status=done&style=none&width=857)<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437932/1598863339932-1d86ceac-df59-4f26-abdb-6a038a430e6d.png#align=left&display=inline&height=550&margin=%5Bobject%20Object%5D&name=image.png&originHeight=550&originWidth=1234&size=110976&status=done&style=none&width=1234)<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437932/1598863354739-4cdbfd04-2d73-4b1d-8419-4ca0b77346c3.png#align=left&display=inline&height=548&margin=%5Bobject%20Object%5D&name=image.png&originHeight=548&originWidth=1236&size=138439&status=done&style=none&width=1236)<br />压测期间TiKV磁盘IO压力较大（本身磁盘性能较差）导致TiKV节点CPU压力不高。<br />

<a name="6J88q"></a>
# 建议优化

1. 对于TiDB查询慢sql，是否有优化空间，对于split makeSlice较为耗时函数能发优化？
1. TiKV节点磁盘是否可根据磁盘性能动态调整特性，例如根据磁盘性能动态调整参数，例如更多batch、buffer？


