# Tikv安装部署
## 概述
TiDB 是 PingCAP 公司自主设计、研发的开源分布式关系型数据库，是一款同时支持在线事务处理与在线分析处理 (Hybrid Transactional and Analytical Processing, HTAP) 的融合型分布式数据库产品，具备水平扩容或者缩容、金融级高可用、实时 HTAP、云原生的分布式数据库、兼容 MySQL 5.7 协议和 MySQL 生态等重要特性。目标是为用户提供一站式 OLTP (Online Transactional Processing)、OLAP (Online Analytical Processing)、HTAP 解决方案。TiDB 适合高可用、强一致要求较高、数据规模较大等各种应用场景。长安链支持在使用Tikv作为存储。

### Tiup安装
建议使用tiup安装tidb cluster,与长安链中的tikv pdprovider交互,服务器配置最好高于或等于CPU: 8 core, 内存: 32GB

- 在线部署 TiUP 组件，执行如下命令安装 TiUP 工具：


`curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh`


- 重新声明全局环境变量：

`source .bash_profile`
- 确认 TiUP 工具是否安装：

`which tiup`
- 安装 TiUP cluster 组件

`tiup cluster`
- 如果已经安装，则更新 TiUP cluster 组件至最新版本：

`tiup update --self && tiup update cluster`
- 预期输出 “Update successfully!” 字样。
- 验证当前 TiUP cluster 版本信息。执行如下命令查看 TiUP cluster 组件版本：

`tiup --binary cluster`

- 可以参照 [tidb官方部署文档](https://docs.pingcap.com/zh/tidb/dev/production-deployment-using-tiup#%E7%AC%AC-2-%E6%AD%A5%E5%9C%A8%E4%B8%AD%E6%8E%A7%E6%9C%BA%E4%B8%8A%E5%AE%89%E8%A3%85-tiup-%E7%BB%84%E4%BB%B6)

### topo.yaml示例

``` yaml
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  user: "root"
  ssh_port: 22
  deploy_dir: "/yourpath/data/tidb-deploy"
  data_dir: "/yourpath/data/tidb-data"

server_configs:
  tikv:
    raftstore.sync-log: true
    storage.reserve-space: "0"
    storage.block-cache.capacity: "4G"
    server.grpc-concurrency: 48
    server.grpc-concurrent-stream: 4096
    server.grpc-stream-initial-window-size: "32M"
    storage.scheduler-concurrency: 1048576
    storage.scheduler-worker-pool-size: 32
    rocksdb.titan.enabled: true
    rocksdb.defaultcf.write-buffer-size: "512MB"
    rocksdb.defaultcf.max-write-buffer-number: 32
    rocksdb.max-background-jobs: 32
    rocksdb.defaultcf.block-cache-size: "16GB"
    rocksdb.defaultcf.compression-per-level: [
        'zstd',
        'zstd',
        'lz4',
        'lz4',
        'lz4',
        'lz4',
        'lz4',
    ]

  pd:
    replication.location-labels: ["host"]
    replication.max-replicas: 1

pd_servers:
  - host: 127.0.0.1

tikv_servers:
  - host: 127.0.0.1
    port: 20160
    status_port: 20180
    deploy_dir: "/yourpath/data/deploy/tikv1"
    data_dir: "/yourpath/data/data/tikv1"
    log_dir: "/yourpath/data/log/tikv1"
    config:
      server.labels: { host: "logic-host-1" }

  - host: 127.0.0.1
    port: 20161
    status_port: 20181
    deploy_dir: "/yourpath/data/deploy/tikv2"
    data_dir: "/yourpath/data/data/tikv2"
    log_dir: "/yourpath/data/log/tikv2"
    config:
      server.labels: { host: "logic-host-2" }

  - host: 127.0.0.1
    port: 20162
    status_port: 20182
    deploy_dir: "/yourpath/data/deploy/tikv3"
    data_dir: "/yourpath/data/data/tikv3"
    log_dir: "/yourpath/data/log/tikv3"
    config:
      server.labels: { host: "logic-host-3" }

monitoring_servers:
  - host: 127.0.0.1

grafana_servers:
  - host: 127.0.0.1

alertmanager_servers:
  - host: 127.0.0.1

```

### 配置ssh连接上限
- 添加如下配置到/etc/ssh/sshd_config
  MaxSessions 100
  MaxStartups 50:30:100
- 重启centos上 sshd: systemctl restart sshd.service
```
vim /etc/ssh/sshd_config
```

### tiup操作tidb cluster
```
tiup cluster deploy chainmaker-tidb v5.1.1 topo.yaml --user root -p # 部署 tidb cluster, 输入密码为服务器登陆密码
tiup cluster list # 查看 tidb cluster
tiup cluster start chainmaker-tidb # 启动 tidb cluster
tiup cluster display chainmaker-tidb # 查看 tidb cluster 状态
tiup cluster stop chainmaker-tidb # 停止 tidb cluster
tiup cluster clean chainmaker-tidb --all --ignore-role prometheus --ignore-role grafana # 清理 tidb cluster数据，并保留监控数据
tiup cluster destroy chainmaker-tidb # 销毁 tidb cluster
```

### 配置chainmaker storage模块
```
  blockdb_config:
    provider: tikvdb
    tikvdb_config:
      endpoints: "127.0.0.1:2379" # tikv pd server url，支持多个url， 如: "192.168.1.2:2379,192.168.1.3:2379"
      max_batch_count: 128 # 每次kv batch最大大小 默认128
      grpc_connection_count: 16 # chainmaker连接tikv的连接数， 默认4
      grpc_keep_alive_time: 10 # 保持连接的连接数， 默认10
      grpc_keep_alive_timeout: 3 # 保持连接的超时时间 默认3
      write_batch_size: 128 # 每次提交tikv批次最大大小，默认128
  statedb_config:
    provider: tikvdb
    tikvdb_config:
      endpoints: "127.0.0.1:2379"
      max_batch_count: 128
      grpc_connection_count: 16
      grpc_keep_alive_time: 10
      grpc_keep_alive_timeout: 3
      write_batch_size: 128
  disable_historydb: true
  historydb_config:
    provider: tikvdb
    tikvdb_config:
      endpoints: "127.0.0.1:2379"
      max_batch_count: 128
      grpc_connection_count: 16
      grpc_keep_alive_time: 10
      grpc_keep_alive_timeout: 3
      write_batch_size: 128
  resultdb_config:
    provider: tikvdb
    tikvdb_config:
      endpoints: "127.0.0.1:2379"
      max_batch_count: 128
      grpc_connection_count: 16
      grpc_keep_alive_time: 10
      grpc_keep_alive_timeout: 3
      write_batch_size: 128

```

### tikv状态监控
- 可以登陆 http://127.0.0.1:3000 监控tikv状态
