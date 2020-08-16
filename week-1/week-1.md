## 明确任务
根据 Homework 的描述，是要部署三个 TiKV，一个TiDB，通过 pd 进行调度，于是通过 Google 搜索 `tikv build`，找到了[这篇文章](https://pingcap.com/blog/building-running-and-benchmarking-tikv-and-tidb)，根据文中的描述
> Assuming you want to run TiDB, you will need TiDB, Placement Driver (PD), and TiKV. Clone each repo and run make in the root directory of each cloned repo.

嗯，看起来非常 straightforward，于是下面的步骤就围绕着这三大组件的编译和部署展开了。

## PD

首先是 PD，因为根据[build 文档中的 Running 部分](https://pingcap.com/blog/building-running-and-benchmarking-tikv-and-tidb/#running)说的，需要先运行PD。

1. 通过 `git clone https://github.com/pingcap/pd.git` 下载源码。
2. 发现了根目录下的`Makefile`，并根据文档提示直接`make`，在bin目录下发现了所需的产物。
3. 根据[文档](https://pingcap.com/blog/building-running-and-benchmarking-tikv-and-tidb/#running)所说，通过`./bin/pd-server -client-urls http://127.0.0.1:2379 -data-dir ./data`启动pd，从标准输出中发现了以下日志，说明启动成功

```txt
[2020/08/16 23:37:46.571 +08:00] [INFO] [util.go:40] ["Welcome to Placement Driver (PD)"]
[2020/08/16 23:37:46.572 +08:00] [INFO] [util.go:41] [PD] [release-version=v4.0.0-rc.2-140-g865fbd82]
[2020/08/16 23:37:46.572 +08:00] [INFO] [util.go:42] [PD] [edition=Community]
[2020/08/16 23:37:46.572 +08:00] [INFO] [util.go:43] [PD] [git-hash=865fbd82a028aecfb875a20b932fee3ba4b8c73c]
[2020/08/16 23:37:46.572 +08:00] [INFO] [util.go:44] [PD] [git-branch=master]
[2020/08/16 23:37:46.572 +08:00] [INFO] [util.go:45] [PD] [utc-build-time="2020-08-16 02:36:24"]
[2020/08/16 23:37:46.572 +08:00] [INFO] [metricutil.go:81] ["disable Prometheus push client"]
```

## TiKV

1. 通过 `git clone https://github.com/tikv/tikv.git` 下载源码。
2. 根据上面说的，观察到项目中有 Makefile，于是直接make，经过极其漫长的等待终于完成
![tikv](tikv.png)
3. 根据 [build 文档中的 Running 部分](https://pingcap.com/blog/building-running-and-benchmarking-tikv-and-tidb/#running)给的例子  

```txt
$TIKV_DIR/target/release/tikv-server --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20160" \
    --status-addr="127.0.0.1:20181" \
    --data-dir=tikv1 \
    --log-file=tikv1.log &
$TIKV_DIR/target/release/tikv-server --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20161" \
    --status-addr="127.0.0.1:20182" \
    --data-dir=tikv2 \
    --log-file=tikv2.log &
$TIKV_DIR/target/release/tikv-server --pd-endpoints="127.0.0.1:2379" \
    --addr="127.0.0.1:20162" \
    --status-addr="127.0.0.1:20183" \
    --data-dir=tikv3 \
    --log-file=tikv3.log &  
```

尝试直接启动，观察到根目录下多了`tikv1，tikv2和tikv3`三个文件夹以及三个对应的`.log`文件，点开`tikv1.log`发现
> [2020/08/17 00:33:35.939 +08:00] [INFO] [raft.rs:985] ["became leader at term 6"] [term=6] [raft_id=3] [region_id=2]

应该是选主成功了；通过`bin/pd-ctl store  -u http://127.0.0.1:2379`检查集群状态，得到以下结果

```json
{
  "count": 3,
  "stores": [
    {
      "store": {
        "id": 1,
        "address": "127.0.0.1:20160",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20181",
        "git_hash": "ae7a6ecee6e3367da016df0293a9ffe9cc2b5705",
        "start_timestamp": 1597595615,
        "deploy_path": "/Users/bytedance/pingcap/tikv/target/release",
        "last_heartbeat": 1597595725969484000,
        "state_name": "Up"
      },
      "status": {
        "capacity": "233.5GiB",
        "available": "141GiB",
        "used_size": "28.8MiB",
        "leader_count": 1,
        "leader_weight": 1,
        "leader_score": 1,
        "leader_size": 1,
        "region_count": 1,
        "region_weight": 1,
        "region_score": 1,
        "region_size": 1,
        "start_ts": "2020-08-17T00:33:35+08:00",
        "last_heartbeat_ts": "2020-08-17T00:35:25.969484+08:00",
        "uptime": "1m50.969484s"
      }
    },
    {
      "store": {
        "id": 4,
        "address": "127.0.0.1:20161",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20182",
        "git_hash": "ae7a6ecee6e3367da016df0293a9ffe9cc2b5705",
        "start_timestamp": 1597595689,
        "deploy_path": "/Users/bytedance/pingcap/tikv/target/release",
        "last_heartbeat": 1597595729203904000,
        "state_name": "Up"
      },
      "status": {
        "capacity": "233.5GiB",
        "available": "141GiB",
        "used_size": "28.8MiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 1,
        "region_weight": 1,
        "region_score": 1,
        "region_size": 1,
        "start_ts": "2020-08-17T00:34:49+08:00",
        "last_heartbeat_ts": "2020-08-17T00:35:29.203904+08:00",
        "uptime": "40.203904s"
      }
    },
    {
      "store": {
        "id": 6,
        "address": "127.0.0.1:20162",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20183",
        "git_hash": "ae7a6ecee6e3367da016df0293a9ffe9cc2b5705",
        "start_timestamp": 1597595717,
        "deploy_path": "/Users/bytedance/pingcap/tikv/target/release",
        "last_heartbeat": 1597595727829462000,
        "state_name": "Up"
      },
      "status": {
        "capacity": "233.5GiB",
        "available": "141GiB",
        "used_size": "28.8MiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 1,
        "region_weight": 1,
        "region_score": 1,
        "region_size": 1,
        "start_ts": "2020-08-17T00:35:17+08:00",
        "last_heartbeat_ts": "2020-08-17T00:35:27.829462+08:00",
        "uptime": "10.829462s"
      }
    }
  ]
}
```

## TiDB

与前面两个项目类似，首先是`git clone`，在此就不赘述了。另外，需要在事务开启时打出那条日志，那么应该在事务begin之后打出；为此，我在项目中搜索了`Transaction`关键字，发现在`kv/kv.go`中有一个interface名为`Transaction`，它的实现是`tikvStore`，因此我决定把代码写在`tikvStore`中的`Begin`函数中，如下所示：

```go
func (s *tikvStore) Begin() (kv.Transaction, error) {
    txn, err := newTiKVTxn(s)
    if err != nil {
        return nil, errors.Trace(err)
    }
    logutil.BgLogger().Info("hello transaction")
    return txn, nil
}
```

同样根据[文档](https://pingcap.com/blog/building-running-and-benchmarking-tikv-and-tidb/#running)所介绍的，通过`bin/tidb-server --store=tikv --path="127.0.0.1:2379" --log-file=tidb.log`启动tidb-server，观察到`tidb.log`中出现了下面的日志
> [2020/08/17 00:45:22.788 +08:00] [INFO] [server.go:235] ["server is running MySQL protocol"] [addr=0.0.0.0:4000]

这说明tidb-server应该启动成功了，但同时，我发现在我尚未对数据库进行事务操作的情况下，`tidb.log`中就出现了多条`"hello transaction"`日志，我不知道这是否符合预期。抱着试试看的心态，我手贱地尝试把这里的调用栈完整地打出来，

```go
func (s *tikvStore) Begin() (kv.Transaction, error) {
    txn, err := newTiKVTxn(s)
    if err != nil {
        return nil, errors.Trace(err)
    }
    for i := 0; ; i++ { // Skip the expected number of frames
        _, file, line, ok := runtime.Caller(i)
        if !ok {
            break
        }
        info := fmt.Sprintf("called from line %d of file %s", line, file)
        logutil.BgLogger().Info(info)
    }
    logutil.BgLogger().Info("hello transaction")
    return txn, nil
}
```

重新编译并运行之后，在log中发现了，以下内容每隔一秒钟就会出现一次：

```txt
[2020/08/17 01:17:17.084 +08:00] [INFO] [kv.go:293] ["called from line 288 of file /Users/bytedance/go/src/pingcap/tidb/store/tikv/kv.go"]
[2020/08/17 01:17:17.084 +08:00] [INFO] [kv.go:293] ["called from line 36 of file /Users/bytedance/go/src/pingcap/tidb/kv/txn.go"]
[2020/08/17 01:17:17.084 +08:00] [INFO] [kv.go:293] ["called from line 433 of file /Users/bytedance/go/src/pingcap/tidb/ddl/ddl_worker.go"]
[2020/08/17 01:17:17.084 +08:00] [INFO] [kv.go:293] ["called from line 154 of file /Users/bytedance/go/src/pingcap/tidb/ddl/ddl_worker.go"]
[2020/08/17 01:17:17.084 +08:00] [INFO] [kv.go:293] ["called from line 1357 of file /usr/local/go/src/runtime/asm_amd64.s"]
[2020/08/17 01:17:17.084 +08:00] [INFO] [kv.go:295] ["hello transaction"]
```

追踪代码`line 154 of file /Users/bytedance/go/src/pingcap/tidb/ddl/ddl_worker.go`可知，`worker`的`start`函数中，有一个死循环，每过`2 * lease`就会做一次更新操作，由于我未对`lease`进行设置，所以这里采取的默认值是一秒，与我在log文件中观察到的现象是吻合的，因此我的改动应该是生效了。

```go
    // We use 4 * lease time to check owner's timeout, so here, we will update owner's status
    // every 2 * lease time. If lease is 0, we will use default 1s.
    // But we use etcd to speed up, normally it takes less than 1s now, so we use 1s as the max value.
    checkTime := chooseLeaseTime(2*d.lease, 1*time.Second)

    ticker := time.NewTicker(checkTime)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            logutil.Logger(w.logCtx).Debug("[ddl] wait to check DDL status again", zap.Duration("interval", checkTime))
        case <-w.ddlJobCh:
        case <-w.ctx.Done():
            return
        }

        err := w.handleDDLJobQueue(d)
        if err != nil {
            logutil.Logger(w.logCtx).Error("[ddl] handle DDL job failed", zap.Error(err))
        }
    }
```
