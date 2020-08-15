# 题目描述
本地下载编译TIDB, TIKV, PD，并部署1DB, 1PD, 3KV。TIDB要求改写为，启动事务时，打印"hello transaction"的日志。

# 过程
## 1. 改写TIDB
TIDB启动事务，即`BEGIN;`等命令会触发NewTxn。
找到源代码的对应部分，并加上"hello transaction"的日志。
```
func (s *session) NewTxn(ctx context.Context) error {
	//...

	txn, err := s.store.Begin()
	if err != nil {
		return err
	}

    // here
	logutil.Logger(ctx).Info("hello transaction")
	txn.SetVars(s.sessionVars.KVVars)
	if s.GetSessionVars().GetReplicaRead().IsFollowerRead() {
		txn.SetOption(kv.ReplicaRead, kv.ReplicaReadFollower)
	}
```
选择此处打印日志，是因为此处才真正开启一个事务，此行之前都可能因错误而提前返回。

tidb默认打info及以上的日志，日志文件默认位于`~/.tiup/data/xxx/tidb-0/tidb.log`。所以"hello transaction"日志级别设置为info级即可。

## 2. 编译
tidb与pd make即可。

tikv make build。

## 3. 部署运行

由于要使用自己编译的包，以及三个组件配置较多，我选择使用tiup playground进行部署。配置上不用特别修改，但需要指定binpath为自己编译的包，指定1db，1pd，3kv。可关掉tiflash。

部署运行步骤：
```
$ tiup playground --db.binpath ~/Code/tidb/bin/tidb-server --kv 3 --kv.binpath ~/Code/tikv/target/debug/tikv-server --pd.binpath ~/Code/pd/bin/pd-server --tiflash 0
Starting component `playground`: /Users/ark/.tiup/components/playground/v1.0.9/tiup-playground --db.binpath /Users/ark/Code/tidb/bin/tidb-server --kv 3 --kv.binpath /Users/ark/Code/tikv/target/debug/tikv-server --pd.binpath /Users/ark/Code/pd/bin/pd-server --tiflash 0
Use the latest stable version: v4.0.4

    Specify version manually:   tiup playground <version>
    The stable version:         tiup playground v4.0.0
    The nightly version:        tiup playground nightly

Playground Bootstrapping...
Start pd instance...
Start tikv instance...
Start tikv instance...
Start tikv instance...
Start tidb instance...
.........
Waiting for tikv 127.0.0.1:20160 ready
Waiting for tikv 127.0.0.1:20161 ready
Waiting for tikv 127.0.0.1:20162 ready
CLUSTER START SUCCESSFULLY, Enjoy it ^-^
To connect TiDB: mysql --host 127.0.0.1 --port 4000 -u root
To view the dashboard: http://127.0.0.1:2379/dashboard
To view the Prometheus: http://127.0.0.1:9090
To view the Grafana: http://127.0.0.1:3000
```

通过mysql连接tidb，使用`BEGIN;`或`START TRANSCTION;`启动事务，会打印出日志。

日志示例如下：
```
[2020/08/15 14:11:26.425 +08:00] [INFO] [session.go:1424] ["hello transaction"]
```

# 补充
在作业过程中，我还尝试用了tiup cluster部署，但没有发现生成自己mirror的简单方法，导致只能tiup cluster部署官方镜像bin，再patch三个编译包。tiup cluster这个组件还需要进一步摸索。