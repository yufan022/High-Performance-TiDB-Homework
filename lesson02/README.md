# HP - Lesson 2 -【yufan022】

# TiDB性能测试报告
## 测试机配置及环境
> 配置参考 [https://docs.pingcap.com/zh/tidb/stable/hardware-and-software-requirements](https://docs.pingcap.com/zh/tidb/stable/hardware-and-software-requirements)

虚拟机全部采用阿里云ECS 华北3 张家口C区，TiKV节点为NVMe本地SSD磁盘。
系统为阿里云镜像 CentOS 8.2 64位，默认配置未进行OS调优。

| 组件 | 阿里云 ECS | CPU | 内存 | 磁盘 | 内网带宽 | 价格 |
| --- | --- | --- | --- | --- | --- | --- |
| TiUP&测试机 | ecs.c6.2xlarge | 8C | 16G | ESSD PL1 云盘 100G 6800 IOPS | 2.5 Gbps |
| TiDB | ecs.c6.2xlarge * 3 | 8C | 16G | ESSD PL1 云盘 200G 11800 IOPS | 2.5 Gbps |
| PD | ecs.c6.xlarge | 4C | 8G | ESSD PL1 云盘 200G 11800 IOPS | 1.5 Gbps |
| TiKV | ecs.i2g.2xlarge * 3 | 8C | 32G | 高效云盘 100G 2600 IOPS 本地NVMe SSD 894G 147510 IOPS | 2 Gbps |
| Monitor | ecs.c6.xlarge | 4C | 8G | ESSD PL0 云盘 200G 6800 IOPS | 1.5 Gbps |

- 172.29.35.242 tiup&测试机
- 172.29.35.243 monitor
- 172.29.35.244 pd
- 172.29.35.245 tidb1
- 172.29.35.246 tikv1
- 172.29.35.248 tikv2
- 172.29.35.247 tikv3
- 172.29.35.249 tidb2
- 172.29.35.250 tidb3
### TiUP配置文件
```yaml
global:
  user: tidb
  ssh_port: 22
  deploy_dir: /tidb-deploy
  data_dir: /tidb-data
  os: linux
  arch: amd64
monitored:
  node_exporter_port: 9100
  blackbox_exporter_port: 9115
  deploy_dir: /tidb-deploy/monitor-9100
  data_dir: /tidb-data/monitor-9100
  log_dir: /tidb-deploy/monitor-9100/log
server_configs:
  tidb:
    binlog.enable: false
    binlog.ignore-error: false
    log.slow-threshold: 300
  tikv:
    readpool.coprocessor.use-unified-pool: true
    readpool.storage.use-unified-pool: false
    server.grpc-concurrency: 3
    storage.block-cache.capacity: 16G
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64
  tiflash: {}
  tiflash-learner: {}
  pump: {}
  drainer: {}
  cdc: {}
tidb_servers:
- host: 172.29.35.245
  ssh_port: 22
  port: 4000
  status_port: 10080
  deploy_dir: /tidb-deploy/tidb-4000
  config:
    log.level: error
  arch: amd64
  os: linux
- host: 172.29.35.249
  ssh_port: 22
  port: 4000
  status_port: 10080
  deploy_dir: /tidb-deploy/tidb-4000
  config:
    log.level: error
  arch: amd64
  os: linux
- host: 172.29.35.250
  ssh_port: 22
  port: 4000
  status_port: 10080
  deploy_dir: /tidb-deploy/tidb-4000
  config:
    log.level: error
  arch: amd64
  os: linux
tikv_servers:
- host: 172.29.35.246
  ssh_port: 22
  port: 20160
  status_port: 20180
  deploy_dir: /data/tidb-deploy/tikv-20160
  data_dir: /data/tidb-data/tikv-20160
  log_dir: /data/tidb-deploy/tikv-20160/log
  config:
    log.level: error
  arch: amd64
  os: linux
- host: 172.29.35.248
  ssh_port: 22
  port: 20160
  status_port: 20180
  deploy_dir: /data/tidb-deploy/tikv-20160
  data_dir: /data/tidb-data/tikv-20160
  log_dir: /data/tidb-deploy/tikv-20160/log
  config:
    log.level: error
  arch: amd64
  os: linux
- host: 172.29.35.247
  ssh_port: 22
  port: 20160
  status_port: 20180
  deploy_dir: /data/tidb-deploy/tikv-20160
  data_dir: /data/tidb-data/tikv-20160
  log_dir: /data/tidb-deploy/tikv-20160/log
  config:
    log.level: error
  arch: amd64
  os: linux
tiflash_servers: []
pd_servers:
- host: 172.29.35.244
  ssh_port: 22
  name: pd-172.29.35.244-2379
  client_port: 2379
  peer_port: 2380
  deploy_dir: /tidb-deploy/pd-2379
  data_dir: /tidb-data/pd-2379
  arch: amd64
  os: linux
monitoring_servers:
- host: 172.29.35.243
  ssh_port: 22
  port: 9090
  deploy_dir: /tidb-deploy/prometheus-9090
  data_dir: /tidb-data/prometheus-9090
  arch: amd64
  os: linux
grafana_servers:
- host: 172.29.35.243
  ssh_port: 22
  port: 3000
  deploy_dir: /tidb-deploy/grafana-3000
  arch: amd64
  os: linux
alertmanager_servers:
- host: 172.29.35.243
  ssh_port: 22
  web_port: 9093
  cluster_port: 9094
  deploy_dir: /tidb-deploy/alertmanager-9093
  data_dir: /tidb-data/alertmanager-9093
  arch: amd64
  os: linux
```
## 测试命令
#### sysbench
插入数据
```shell
sysbench --config-file=config oltp_point_select --tables=32 --table-size=10000000 prepare
```
测试点查询 1 TiDB 1 PD 1 TiKV
```shell
nohup sysbench --config-file=config oltp_point_select --threads=64 --tables=32 --table-size=10000000 --time=120 run > optimize_t64.out 2>&1
nohup sysbench --config-file=config oltp_update_index --tables=32 --table-size=10000000 --time=120 --threads=2048 run > optimize_update_t2048.out 2>&1 &
```


#### go-ycsb
准备数据
```shell
./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=10000000 -p mysql.host=172.29.35.245 -p mysql.port=4000 --threads 256
```
执行命令
```shell
./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=10000000 -p mysql.host=172.29.35.245 -p mysql.port=4000 --threads 256
```
#### go-tpc-c
准备数据
```shell
./bin/go-tpc tpcc -H '172.29.35.245' -P 4000 -D tpcc --warehouses 1000 prepare
```
执行命令
```shell
./bin/go-tpc tpcc -H 172.29.35.245 -P 4000 -D tpcc --warehouses 1000 run --time 1m --threads 128
```
#### go-tpc-h
准备数据
```shell
./bin/go-tpc tpch prepare -H 172.29.35.245 -P 4000 -D tpch --sf 5 --analyze
```
运行测试
```shell
./bin/go-tpc tpch run -H 172.29.35.245 -P 4000 -D tpch --sf 5
```
## 测试结果
### sysbench
#### point select
|  | 64T | 128T | 256T | 512T | 1024T | 2048T |
| --- | --- | --- | --- | --- | --- | --- |
| QPS | 132044.02 | 202130.69 | 230774.06 | 241733.78 | 248247.31 | 249194.12 |
| P95 latency | 0.63 | 1.04 | 2.26 | 4.57 | 9.22 | 18.95 |

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image.png?raw=true)


#### index update
|  | 64T | 128T | 256T | 512T | 1024T | 2048T |
| --- | --- | --- | --- | --- | --- | --- |
| TPS | 8566.14 | 11323.34 | 12863.93 | 15395.4 | 17272.33 | 18639.39 |
| P95 latency | 9.73 | 16.12 | 33.12 | 55.82 | 108.68 | 207.82 |

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-2.png?raw=true)
### go-ycsb
```shell
***************** properties *****************
"requestdistribution"="uniform"
"threadcount"="256"
"insertproportion"="0"
"recordcount"="1000"
"workload"="core"
"updateproportion"="0.5"
"mysql.host"="172.29.35.245"
"mysql.port"="4000"
"readproportion"="0.5"
"readallfields"="true"
"dotransactions"="true"
"operationcount"="10000000"
"scanproportion"="0"
**********************************************
READ   - Takes(s): 10.0, Count: 92444, OPS: 9248.6, Avg(us): 4529, Min(us): 899, Max(us): 39585, 99th(us): 16000, 99.9th(us): 24000, 99.99th(us): 31000
UPDATE - Takes(s): 10.0, Count: 91600, OPS: 9186.8, Avg(us): 23087, Min(us): 5113, Max(us): 457875, 99th(us): 207000, 99.9th(us): 253000, 99.99th(us): 399000
...
Run finished, takes 8m59.035740975s
READ   - Takes(s): 539.0, Count: 5000073, OPS: 9276.0, Avg(us): 4313, Min(us): 841, Max(us): 449406, 99th(us): 16000, 99.9th(us): 26000, 99.99th(us): 52000
UPDATE - Takes(s): 539.0, Count: 4999799, OPS: 9276.0, Avg(us): 23019, Min(us): 2667, Max(us): 951521, 99th(us): 212000, 99.9th(us): 323000, 99.99th(us): 451000
```
### go-tpc-c
|  | 64T | 128T | 256T | 512T |
| --- | --- | --- | --- | --- |
| tpmC | 25480.2 | 28463.5 | 30581.9 | 30623 |

### ![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-3.png?raw=true)
### go-tpc-h
```shell
Got signal [terminated] to exit.
Finished
[Summary] Q1: 7.62s
[Summary] Q10: 3.27s
[Summary] Q11: 1.46s
[Summary] Q12: 3.51s
[Summary] Q13: 3.37s
[Summary] Q14: 3.15s
[Summary] Q15: 7.02s
[Summary] Q16: 1.59s
[Summary] Q17: 9.70s
[Summary] Q18: 16.20s
[Summary] Q19: 4.31s
[Summary] Q2: 1.70s
[Summary] Q20: 2.97s
[Summary] Q21: 6.71s
[Summary] Q22: 1.75s
[Summary] Q3: 5.69s
[Summary] Q4: 2.34s
[Summary] Q5: 6.10s
[Summary] Q6: 2.95s
[Summary] Q7: 4.01s
[Summary] Q8: 3.96s
[Summary] Q9: 11.22s
```
## 监控截图
### sysbench 
> sysbench --config-file=config oltp_point_select --threads=1024 --tables=32 --table-size=10000000 --time=120 run

- TiDB Query Summary 中的 qps 与 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-4.png?raw=true)

- TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-5.png?raw=true)

- TiKV Details 面板中 grpc 的 qps 以及 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-6.png?raw=true)
> sysbench --config-file=config oltp_update_index --tables=32 --table-size=10000000 --time=120000 --threads=1024 run

- TiDB Query Summary 中的 qps 与 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-7.png?raw=true)

- TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-8.png?raw=true)

- TiKV Details 面板中 grpc 的 qps 以及 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-9.png?raw=true)
### go-ycsb
> ./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=10000000 -p mysql.host=172.29.35.245 -p mysql.port=4000 --threads 256

- TiDB Query Summary 中的 qps 与 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-10.png?raw=true)

- TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-11.png?raw=true)

- TiKV Details 面板中 grpc 的 qps 以及 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-12.png?raw=true)
### go-tpc-c
> ./go-tpc tpcc -H '172.29.35.245' -P 4000 -D tpcc --warehouses 1000 run

- TiDB Query Summary 中的 qps 与 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-13.png?raw=true)

- TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-14.png?raw=true)

- TiKV Details 面板中 grpc 的 qps 以及 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-15.png?raw=true)
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-16.png?raw=true)
### go-tpc-h
> ./bin/go-tpc tpch run -H 172.29.35.245 -P 4000 -D tpch --sf 5

- TiDB Query Summary 中的 qps 与 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-17.png?raw=true)

- TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-18.png?raw=true)

- TiKV Details 面板中 grpc 的 qps 以及 duration

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-19.png?raw=true)
## 分析
前期**TiDB使用单节点**发现**TiDB CPU负载被打满，TiKV负载不高。**
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-20.png?raw=true)
判断TiDB对计算资源需求较高，为TiDB进行扩容操作，3TiDB 1PD 3TiKV。
并且调整TiDB集群配置：
```yaml
server_configs:
  tidb:
    binlog.enable: false
    binlog.ignore-error: false
    log.slow-threshold: 300
  tikv:
    readpool.coprocessor.use-unified-pool: true
    readpool.storage.use-unified-pool: false
    server.grpc-concurrency: 3
    storage.block-cache.capacity: 16G
  pd:
    schedule.leader-schedule-limit: 4
    schedule.region-schedule-limit: 2048
    schedule.replica-schedule-limit: 64
## 以及TiDB TiKV log.level: error
```


### 压测机
扩容后使用sysbench点查询发现64线程、128线程、256线程测试QPS/TPS提升不大，均在18000左右。判断**压测机器性能不足**，导致压力不够，升配为ecs.c6.2xlarge。


### sysbench
重新测试后在**点查询场景**下发现在512线程达到24000TPS后，再增加线程对TPS提升不大，通过监控判断**TiDB达到瓶颈** CPU负载吃满，TiKV CPU负载在60-70之间 iowait不高 NVMe磁盘未达到瓶颈，因为**TiDB负载较高导致TPS无法提升、TiKV未到瓶颈**。
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-21.png?raw=true)
在sysbench **索引update**场景下，**TiDB负载较低，TiKV CPU负载很高**先达到瓶颈，但观察磁盘IO不大。
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-22.png?raw=true)
结合火焰图来看，rocksdb __write函数与grpc __sendmsg开销较大。


### go-ycsb
go-ycsb测试中未找到支持多mysql host方式。通过相关Issue [https://github.com/pingcap/go-ycsb/issues/131](https://github.com/pingcap/go-ycsb/issues/131) 未找到后续解决方案（需自建loadbalance？），所以只测试了workloada 1TiDB 1PD 3TiKV情况下负载。瓶颈主要在**TiKV IO**（只使用单TiDB节点测试），IO利用率较高，update慢sql较多。
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-23.png?raw=true)
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-24.png?raw=true)
推测where条件查询磁盘并写入压力较大
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-25.png?raw=true)
#### 
### go-tpc-c
同样TiDB未做LoadBalance，未发现测试工具支持配置多host，测试过程单节点TiDB很快达到瓶颈 90+，TiKV节点CPU在50-70之间。分析性能发现runtime、gc耗时较长，优化执行计划耗时较长。若继续测试整体性能需增加TiDB节点，同时增加loadBalance使tpcc请求分发到不同TiDB节点上。
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-26.png?raw=true)
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-27.png?raw=true)
### go-tpc-h
由于未部署TiFlash，结果没有参考意义。这里描述一下分析过程。
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-28.png?raw=true)
首先发现集群整体负载并不高。
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson02/img/image-29.png?raw=true)
通过TiDB查看SQL执行时间发现TPC-H OLAP测试SQL较为复杂，Coprocessor等待时间较长，TiKV根据请求算子对数据进行过滤聚合，TiDB再对数据二次聚合，总体符合判断。
