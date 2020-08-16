# High Performance TiDB Lesson 01 HomeWork - 源码编译及改写



<a name="SwlZf"></a>
## 下载代码
```shell
# TiKV
https://github.com/tikv/tikv.git
# pd
https://github.com/pingcap/pd.git
# TiDB
https://github.com/pingcap/tidb.git
```


<a name="hvmDA"></a>
## 修改代码
在TiDB源码：session/session.go文件 ExecuteStmt方法 s.PrepareTxnCtx(ctx)代码前添加：
```go
logutil.BgLogger().Info("hello transaction");

// 这里如果只想打印用户提交的sql事务日志可使用以下代码
if !s.sessionVars.InRestrictedSQL {
    logutil.BgLogger().Info("hello transaction");
}
```
![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson01/img/image1.png?raw=true)<br />

<a name="qb0oJ"></a>
## 编译代码
<a name="JEY5c"></a>
#### TiDB
> 参考文档 [https://github.com/pingcap/community/blob/master/contributors/README.md](https://github.com/pingcap/community/blob/master/contributors/README.md)

```shell
cd tidb
make
# ./bin/tidb-server 为编译后二进制包
```
<a name="G8CO0"></a>
#### TiKV
> 参考文档 [https://github.com/tikv/tikv/blob/master/CONTRIBUTING.md](https://github.com/tikv/tikv/blob/master/CONTRIBUTING.md)

在源码目录执行
```shell
cd tikv
make build
# ./target/debug/tikv-server 为编译后二进制包
```
<a name="fyeFt"></a>
#### PD
> 参考文档 [https://github.com/pingcap/pd](https://github.com/pingcap/pd)

在源码目录执行
```shell
cd pd
make
# ./bin/pd-server 为编译后二进制包
```


<a name="k2KEi"></a>
## 启动项目
<a name="idFH0"></a>
#### PD
```shell
./pd-server
```
<a name="t8K4p"></a>
#### TiKV
```shell
# 此处tikv-server拷贝了3个不同目录运行
./tikv-server -A 127.0.0.1:9000  -s ../db0 --pd-endpoints 127.0.0.1:2379
./tikv-server -A 127.0.0.1:9001  -s ../db1 --pd-endpoints 127.0.0.1:2379
./tikv-server -A 127.0.0.1:9002  -s ../db2 --pd-endpoints 127.0.0.1:2379
```
<a name="b4TbS"></a>
#### TiDB
```shell
./tidb-server -store tikv -path '127.0.0.1:2379/pd?cluster=1' -P 3306 -L info
```


<a name="CITOg"></a>
## 结果
使用mysql客户端连接，执行sql
```shell
mysql -uroot -h 127.0.0.1 -P 3306
select 1;
```
查看TiDB控制台日志
> [2020/08/16 17:03:57.050 +08:00] [INFO] [session.go:1124] ["hello transaction"]  
此处如果只打印用户sql事务，可判断代码增加判断s.sessionVars.InRestrictedSQL

![image.png](https://github.com/yufan022/High-Performance-TiDB-Homework/blob/master/lesson01/img/image0.png?raw=true)

