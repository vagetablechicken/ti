
# 测试环境
## 机器
|类别|名称|
|-|-|
|OS |	Linux (CentOS 7.3.1611)|
|CPU |24 vCPUs, Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz|
|RAM |128GB|
|DISK |	SSD_480G*1|
|NIC| 10Gb Ethernet|

## 测试版本
TiDB 版本 v4.0.4

## 集群拓扑
机器|部署实例
|-|-|
host-monitor|	prometheus+grafana+alertmgr
host-1|	1* pd 1*tikv
host-2|	1* pd 1*tikv
host-3|	1* pd 1*tikv
host-4| 1*tidb
host-5| 1*tidb
host-6| 1*tidb

## 集群配置
配置上除了host的修改，只改变了data_dir路径。因为通常log放在hdd盘，data放ssd盘，所以dir配置，如下所示：
```
  deploy_dir: "<prefix>/ti/tidb-deploy"
  data_dir: "<ssd_prefix>/ti/tidb-data"
```
## cluster display
```
tidb Cluster: benchmark-test
tidb Version: v4.0.4
ID                  Role          Host          Ports        OS/Arch       Status  Data Dir                                        Deploy Dir
--                  ----          ----          -----        -------       ------  --------                                        ----------
host-monitor:9093   alertmanager  host-monitor  9093/9094    linux/x86_64  -       <ssd_prefix>/ti/tidb-data/alertmanager-9093     <prefix>/ti/tidb-deploy/alertmanager-9093
host-monitor:3000   grafana       host-monitor  3000         linux/x86_64  -       -                                               <prefix>/ti/tidb-deploy/grafana-3000
host-3:2379         pd            host-3        2379/2380    linux/x86_64  Up|UI   <ssd_prefix>/ti/tidb-data/pd-2379               <prefix>/ti/tidb-deploy/pd-2379
host-1:2379         pd            host-1        2379/2380    linux/x86_64  Up      <ssd_prefix>/ti/tidb-data/pd-2379               <prefix>/ti/tidb-deploy/pd-2379
host-2:2379         pd            host-2        2379/2380    linux/x86_64  Up|L    <ssd_prefix>/ti/tidb-data/pd-2379               <prefix>/ti/tidb-deploy/pd-2379
host-monitor:9092   prometheus    host-monitor  9092         linux/x86_64  -       <ssd_prefix>/ti/tidb-data/prometheus-9092       <prefix>/ti/tidb-deploy/prometheus-9092
host-4:4000         tidb          host-4        4000/10080   linux/x86_64  Up      -                                               <prefix>/ti/tidb-deploy/tidb-4000
host-5:4000         tidb          host-5        4000/10080   linux/x86_64  Up      -                                               <prefix>/ti/tidb-deploy/tidb-4000
host-6:4000         tidb          host-6        4000/10080   linux/x86_64  Up      -                                               <prefix>/ti/tidb-deploy/tidb-4000
host-3:20160        tikv          host-3        20160/20180  linux/x86_64  Up      <ssd_prefix>/ti/tidb-data/tikv-20160            <prefix>/ti/tidb-deploy/tikv-20160
host-1:20160        tikv          host-1        20160/20180  linux/x86_64  Up      <ssd_prefix>/ti/tidb-data/tikv-20160            <prefix>/ti/tidb-deploy/tikv-20160
host-2:20160        tikv          host-2        20160/20180  linux/x86_64  Up      <ssd_prefix>/ti/tidb-data/tikv-20160            <prefix>/ti/tidb-deploy/tikv-20160
```

# 1. sysbench 测试
## prepare
create database
```
mysql> create database sbtest;
Query OK, 0 rows affected (0.07 sec)
```
sysbench 编译

配置文件sysbench.config:
```
mysql-host=host-4
mysql-port=4000
mysql-user=root

mysql-db=sbtest
time=600
threads=16
report-interval=10
db-driver=mysql
```
乐观事务下导入数据。
```
$ sysbench --config-file=sysbench.config oltp_point_select --tables=32 --table-size=10000000 prepare
```
因config只连接1个tidb，所以是tidb单点导入。

## run
由于rocksdb有block cache，如果数据在cache中，读性能提升会很明显，而且实际场景中读热点是比较常见的。
所以可以开启sysbench的warmup功能。
```
--warmup-time=N                 execute events for this many seconds with statistics disabled before the actual benchmark run with statistics enabled [0]
```
warmup时间也不设太长了，所以取总测试时间的10%。
### oltp_point_select
这是第一次读，所以补充一次预热，之后两个测试就不用预热了。
```
sysbench --config-file=sysbench.config oltp_point_select --threads=128 --tables=32 --table-size=5000000 --warmup-time=60 run
```
oltp_point_select模式，即'1 query per event'。

#### 输出结果
```
SQL statistics:
    queries performed:
        read:                            44937642
        write:                           0
        other:                           0
        total:                           44937642
    transactions:                        44937642 (74890.66 per sec.)
    queries:                             44937642 (74890.66 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

Throughput:
    events/s (eps):                      74890.6600
    time elapsed:                        600.0446s
    total number of events:              44937642

Latency (ms):
         min:                                    1.38
         avg:                                    1.71
         max:                                  204.07
         95th percentile:                        2.39
         sum:                             76780260.69

Threads fairness:
    events (avg/stddev):           351074.0625/10794.87
    execution time (avg/stddev):   599.8458/0.01
```
#### 监控截图 

![Query Summary qps](/img/sb1-query-qps.png)
![Query Summary duration](/img/sb1-query-dura.png)

![](/img/sb1-kv-cpu.png)
![](/img/sb1-kv-qps.png)

上图（qps图）中三条高线为kv_batch_get_command

![](/img/sb1-grpc-qps.png)
![](/img/sb1-grpc-dura.png)

#### 补充
重复测试，结果比较稳定。eps 74k/s左右。
test | eps
-|-
test1|74890.6600
test2|74898.9605
test3|74016.7372

### oltp_update_index
```
sysbench --config-file=sysbench.config oltp_update_index --threads=128 --tables=32 --table-size=5000000 run
```
这一模式只做update index。
#### 输出结果
```
SQL statistics:
    queries performed:
        read:                            0
        write:                           7121958
        other:                           83179
        total:                           7205137
    transactions:                        7205137 (12008.24 per sec.)
    queries:                             7205137 (12008.24 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

Throughput:
    events/s (eps):                      12008.2373
    time elapsed:                        600.0162s
    total number of events:              7205137

Latency (ms):
         min:                                    1.58
         avg:                                   10.66
         max:                                 1400.68
         95th percentile:                       15.27
         sum:                             76796935.97

Threads fairness:
    events (avg/stddev):           56290.1328/321.41
    execution time (avg/stddev):   599.9761/0.00
```

#### 监控截图

![](/img/sb2-query-dura.png)
![](/img/sb2-query-qps.png)

![](/img/sb2-kv-cpu.png)

|![](/img/sb2-kv-qps-1.png)|![](/img/sb2-kv-qps-2.png) |
|:-|:-|

![](/img/sb2-grpc.png)

#### 补充
重复测试，Latency基本一致，eps 12k左右。
|test|eps|
|-|-|
|test1|12008.2373|
|test2|11610.2859|

### oltp_read_only
```
sysbench --config-file=sysbench.config oltp_read_only --threads=128 --tables=32 --table-size=5000000 run
```
这一模式与oltp_point_select的区别在于，它是'point_selects defaults to 10'。也就是一次event有10次query。

#### 输出结果
```
SQL statistics:
    queries performed:
        read:                            17764292
        write:                           0
        other:                           2537756
        total:                           20302048
    transactions:                        1268878 (2114.59 per sec.)
    queries:                             20302048 (33833.37 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

Throughput:
    events/s (eps):                      2114.5854
    time elapsed:                        600.0600s
    total number of events:              1268878

Latency (ms):
         min:                                   31.13
         avg:                                   60.53
         max:                                  269.68
         95th percentile:                       77.19
         sum:                             76800449.96

Threads fairness:
    events (avg/stddev):           9913.1094/148.83
    execution time (avg/stddev):   600.0035/0.02
```

#### 监控截图
![](/img/sb3-query-dura.png)
![](/img/sb3-query-qps.png)
![](/img/sb3-kv-cpu.png)

| ![](/img/sb3-kv-qps1.png) | ![](/img/sb3-kv-qps2.png) |
|:-|:-|

![](/img/sb3-grpc.png)

#### 补充
eps 2K左右。
|test|eps|
|-|-|
test1|2114.5854
test2|2099.7004


# 2. go-ycsb
基于workloada，测试demo只增大了count数量，其他配置保持原样，run时读（更新）写比例1比1。

## load
```
./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=10000000 -p mysql.host=host-4 -p mysql.port=4000 --threads 128
```
```
Run finished, takes 21m58.91559064s
INSERT - Takes(s): 1318.9, Count: 9999999, OPS: 7582.1, Avg(us): 16613, Min(us): 3198, Max(us): 5996910, 99th(us): 179000, 99.9th(us): 769000, 99.99th(us): 1787000
```
load期间kv regionMiss较多。

## run
```
./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=10000000 -p mysql.host=host-4 -p mysql.port=4000 --threads xxx
```

### test1-thread 128
```
Run finished, takes 11m49.262264808s
READ   - Takes(s): 709.3, Count: 5000195, OPS: 7049.9, Avg(us): 3066, Min(us): 1766, Max(us): 1185667, 99th(us): 10000, 99.9th(us): 29000, 99.99th(us): 329000
UPDATE - Takes(s): 709.2, Count: 4999805, OPS: 7049.5, Avg(us): 14734, Min(us): 3115, Max(us): 7855672, 99th(us): 215000, 99.9th(us): 810000, 99.99th(us): 1668000
```
通过监控，发现128线程对tidb的CPU压力不是很大。因此尝试将线程数调大到256。
### test2-thread 256
```
Run finished, takes 9m9.353035199s
READ   - Takes(s): 549.3, Count: 5000934, OPS: 9103.6, Avg(us): 4883, Min(us): 1783, Max(us): 2359658, 99th(us): 19000, 99.9th(us): 152000, 99.99th(us): 654000
UPDATE - Takes(s): 549.3, Count: 4998915, OPS: 9099.9, Avg(us): 22889, Min(us): 3118, Max(us): 10903323, 99th(us): 250000, 99.9th(us): 1415000, 99.99th(us): 3743000
UPDATE_ERROR - Takes(s): 461.2, Count: 23, OPS: 0.0, Avg(us): 5312763, Min(us): 2868613, Max(us): 8087501, 99th(us): 8088000, 99.9th(us): 8088000, 99.99th(us): 8088000
```
线程数提高后，出现`UPDATE_ERROR`。于是再测试两次，以排除偶然错误。后两次测试仍然有`UPDATE_ERROR`。
阅读go-ycsb源码，看到mysql db的Update()是有maxBadConnRetries，但connect没问题的话，应该是没有重试的。所以`UPATE_ERROR`应该是正常现象。


### 监控截图
thread 256的情况下，监控截图如下。
![](/img/ycsb-query-dura.png)
![](/img/ycsb-query-qps.png)

![](/img/ycsb-kv-cpu.png)

|![](/img/ycsb-kv-qps1.png)|![](/img/ycsb-kv-qps2.png)|
|:-|:-|

![](/img/ycsb-grpc.png)

# 3. go-tpc
## tpcc
### prepare
```
./bin/go-tpc tpcc -H host-4 -P 4000 -D tpcc --warehouses 1000 prepare
```
导入挺久的，后来才发现
[import-to-tidb](https://github.com/pingcap/go-tpc/blob/master/docs/import-to-tidb.md#load-data-directly)，可以使用`-T`多线程来加速导入。
### run
```
./bin/go-tpc tpcc -H host-4 -P 4000 -D tpcc --warehouses 1000 run
```
没有指定`--count`，而是跑3h以上时间。总计count为338740。有少量的Failed Query（监控得到），但输出报告中没有，可能经重试完成了。

```
Finished
[Summary] DELIVERY - Takes(s): 12059.9, Count: 13561, TPM: 67.5, Sum(ms): 1312534, Avg(ms): 96, 90th(ms): 112, 99th(ms): 160, 99.9th(ms): 512
[Summary] NEW_ORDER - Takes(s): 12062.8, Count: 152248, TPM: 757.3, Sum(ms): 6741825, Avg(ms): 44, 90th(ms): 64, 99th(ms): 80, 99.9th(ms): 160
[Summary] ORDER_STATUS - Takes(s): 12062.2, Count: 13818, TPM: 68.7, Sum(ms): 209751, Avg(ms): 15, 90th(ms): 20, 99th(ms): 32, 99.9th(ms): 96
[Summary] PAYMENT - Takes(s): 12062.6, Count: 145576, TPM: 724.1, Sum(ms): 3210577, Avg(ms): 22, 90th(ms): 24, 99th(ms): 32, 99.9th(ms): 80
[Summary] STOCK_LEVEL - Takes(s): 12061.5, Count: 13537, TPM: 67.3, Sum(ms): 414955, Avg(ms): 30, 90th(ms): 64, 99th(ms): 112, 99.9th(ms): 512
tpmC: 757.3
```

#### 监控截图
![](/img/tpcc-query-dura.png)
![](/img/tpcc-query-qps.png)
![](/img/tpcc-kv-cpu.png)

|![](/img/tpcc-kv-qps1.png)|![](/img/tpcc-kv-qps2.png)|
|:-|:-|

![](/img/tpcc-grpc.png)
 
# 分析
考虑整体的测试报告，从报告与监控中，可以看到以下几点。

纯读的表现都非常好，不管是point select还是read only。

但是在读写混合场景中，例如ycsb中workloada的read&update各占一半时，读P99增高比较厉害，从ycsb的summary query p99和grpc p99监控图中可以看出。

再观察多次ycsb测试期间的监控，发现KV Cmd Duration较高，也就是TiDB 发送请求给 TiKV 到收到回复的延迟变大。
![](/img/kv-cmd-dura-p99.png)
再看gRPC的监控，
![](/img/total-grpc.png)
Coprocessor P99 duration比较高，pessimistic_lock P99也是。所以推测，这两个地方可能存在性能问题。

而storage监控显示，高于1s的异步写入也有一些。根据[storage](https://docs.pingcap.com/zh/tidb/stable/grafana-tikv-dashboard#storage)指标的解释说明，集群这一表现也不太理想。所以，推测此处也可能有性能问题。
![](/img/storage.png)

另外读写时，txnLock和resolve指标都升高较明显。
![](/img/kv-resolve-txnlock.png)
检查db的事务锁：
```
MySQL [(none)]> select @@GLOBAL.tidb_txn_mode;
+------------------------+
| @@GLOBAL.tidb_txn_mode |
+------------------------+
| pessimistic            |
+------------------------+
1 row in set (0.01 sec)
```
是悲观的，所以txnLock应该是[KeyIsLocked错误](https://docs.pingcap.com/zh/tidb/stable/troubleshoot-lock-conflicts#keyislocked-%E9%94%99%E8%AF%AF)，也就是锁冲突。

测试中，选取一些sql语句来看，会发现有些慢查询的prewrite耗时占比较高，例如，58s sql，50s prewrite，8s commit。当然这个commit耗时也很高了。因此，猜测prewrite和commit操作的性能可能也存在问题。

## 推测性能瓶颈点
coprocessor, pessimistic_lock, prewrite, commit

P.S. 锁冲突通常应该是业务层的问题。