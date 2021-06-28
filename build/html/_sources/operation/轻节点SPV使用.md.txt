# SPV 部署和使用
## 概述
SPV支持两种使用方式：
- 独立部署，单独一个进程，通过获取业务链的数据，可对外提供交易存在性及有效性证明服务。
- 作为组件集成进其他项目，与其他项目在一个进程中，调用启动以获取业务链的数据，可提供交易存在性及有效性证明功能。

了解SPV设计方案,请点击如下链接：  
[SPV设计文档](../tech/SPV.md)

## SPV独立部署流程
### 源码下载
从[长安链官网](https://www.chainmaker.org/)下载源码：[https://git.code.tencent.com/ChainMaker/chainmaker-spv.git](https://git.code.tencent.com/ChainMaker/chainmaker-spv.git)

> 当前为私有仓库，需要先进行账号注册

- 下载`chainmaker-spv`源码到本地
```bash
$ git clone --recursive https://git.code.tencent.com/ChainMaker/chainmaker-spv.git
```

### 生成物料
- 进入`chainmaker-spv/scripts`目录，执行`prepare.sh`脚本,将编译生成spv二进制文件，并生成spv所需要的配置文件，存于`chainmaker-spv/build/release`路径中。 
```bash
# 进入脚本目录
$ cd chainmaker-spv/scripts

# 查看目录结构
$ tree 
.
├── local              # 该文件下的脚本，在生成物料时，将被拷贝到chainmaker-spv/build/release/bin目录下
│   ├── start.sh
│   └── stop.sh
├── prepare.sh        # 编译生成spv二进制并生成配置文件模板的脚本
├── start.sh          # 启动SPV的脚本
└── stop.sh           # 停止SPV的脚本    

# 编译生成spv二进制文件并生成配置文件模板
$ ./prepare.sh

# 查看生成好的二进制文件和配置文件
$ tree -L 3 ../build/release
../build/release
├── bin
│   ├── spv       # 二进制文件
│   ├── start.sh  # 启动脚本
│   └── stop.sh   # 停止脚本
└── config                                 
    ├── chainmaker_sdk_config_chain1.yml    # 链ID为chain1的SDK配置文件
    ├── chainmaker_sdk_config_chain2.yml    # 链ID为chain2的SDK配置文件
    ├── crypto-config                       # ChainMaker-go-sdk所需证书配置文件
    │   ├── wx-org1.chainmaker.org
    │   ├── wx-org2.chainmaker.org
    │   ├── wx-org3.chainmaker.org
    │   └── wx-org4.chainmaker.org
    └── spv_config.yml                      # SPV配置文件                   
```

### 修改配置文件
- 从SPV节点所要链接的远端ChainMaker链，拷贝节点证书配置文件`crypto-config`，更新SPV项目`chainmaker-spv/build/release/config`路径下的`crypto-config`文件。

> 目前SPV只支持作为ChainMaker的轻节点，通常`crypto-config`在`chainmaker-go/build`路径下。

- 修改`chainmaker-spv/build/release/config`路径下的SPV配置文件`spv_config.yml`

> 在chain中配置SPV需要连接的远端链信息，可支持多链。需配置的远端链信息包括链ID、同步链最新区块高度时间间隔、链SDK配置文件路径。  
> 在grpc中配置SPV提供交易存在性和有效性服务的grpc地址/端口。  
> 在storage中配置SPV的存储模块，目前多链共用同一存储模块。  
> 在log中配置SPV的log信息，目前多链共用日志模块，多链以chainID区分。  
> **注意：SPV配置文件中的路径是基于bin文件夹下spv二进制文件的相对路径，也可以使用绝对路径。**

```yaml
# 链配置
chain:
   # 链ID
 - chain_id: "chain1"
   # 同步配置，同步链中节点区块最新高度信息的时间间隔，单位：毫秒
   sync_chainInfo_interval: 10000
   # sdk配置文件路径
   sdk_config_path: "../config/chainmaker_sdk_config_chain1.yml"
   # 链ID
 - chain_id: "chain2"
   # 同步配置，同步链中节点区块最新高度信息的时间间隔，单位：毫秒
   sync_chainInfo_interval: 10000
   # sdk配置文件路径
   sdk_config_path: "../config/chainmaker_sdk_config_chain2.yml"

# grpc配置
grpc:
  # grpc监听网卡地址
  address: 127.0.0.1
  # grpc监听端口
  port: 12308

# 存储配置，用于配置当前SPV对区块头和交易哈希的存储记录
storage:
  # 存储采用的类型,当前仅支持leveldb类型
  provider: "leveldb"
  # 存储采用leveldb的情况下，对应leveldb的详细配置
  leveldb:
    # leveldb的存储路径
    store_path: "../data/spv_db"
    # leveldb的写入Buffer大小，单位：M
    write_buffer_size: 4
    # leveldb的布隆过滤器的bit长度
    bloom_filter_bits: 10

# 日志配置，用于配置日志的打印
log:
  system:
    # 日志打印级别
    log_level: "INFO"
    # 日志文件路径
    file_path: "../log/spv.log"
    # 日志最长保存时间，单位：天
    max_age: 365
    # 日志滚动时间，单位：小时
    rotation_time: 1
    # 是否展示日志到终端，仅限于调试使用
    log_in_console: false
    # 是否打印颜色日志
    show_color: true
```

- 修改`chainmaker-spv/build/release/config`路径下各链的SDK配置`chainmaker_sdk_config_chainx.yml`

> 在chain_client中配置证书信息。  
> 在nodes中配置连接节点信息。**注意：修改节点的地址/端口**。  
> **注意：SDK配置文件中的路径是基于bin文件夹下spv二进制文件的相对路径。**

```yaml
chain_client:
  # 链ID
  chain_id: "chain1"
  # 组织ID
  org_id: "wx-org1.chainmaker.org"
  # 客户端用户私钥路径
  user_key_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key"
  # 客户端用户证书路径
  user_crt_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt"
  # 客户端用户交易签名私钥路径(若未设置，将使用user_key_file_path)
  user_sign_key_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key"
  # 客户端用户交易签名证书路径(若未设置，将使用user_crt_file_path)
  user_sign_crt_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt"

  nodes:
    - # 节点地址，格式为：IP:端口:连接数
      node_addr: "127.0.0.1:12301"
      # 节点连接数
      conn_cnt: 10
      # RPC连接是否启用双向TLS认证
      enable_tls: true
      # 信任证书池路径
      trust_root_paths:
        - "../config/crypto-config/wx-org1.chainmaker.org/ca"
        - "../config/crypto-config/wx-org2.chainmaker.org/ca"
      # TLS hostname
      tls_host_name: "chainmaker.org"
    - # 节点地址，格式为：IP:端口:连接数
      node_addr: "127.0.0.1:12302"
      # 节点连接数
      conn_cnt: 10
      # RPC连接是否启用双向TLS认证
      enable_tls: true
      # 信任证书池路径
      trust_root_paths:
        - "../config/crypto-config/wx-org1.chainmaker.org/ca"
        - "../config/crypto-config/wx-org2.chainmaker.org/ca"
      # TLS hostname
      tls_host_name: "chainmaker.org"
```
### 启动SPV轻节点

- 在`chainmaker-spv/scripts`目录，运行 `start.sh` 脚本，将会调用`../build/release/bin`目录中的`start.sh`脚本，启动SPV轻节点。
```bash
$ ./start.sh
```

- 查看进程是否存在
```bash
$ ps -ef|grep spv | grep -v grep
501 82533     1   0 12:27AM ttys011    0:00.23 ./spv start -c ../config/spv_config.yml
```

- 查看端口是否监听
```bash
$ lsof -i:12308
COMMAND   PID      USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
spv     82533 liukemeng   14u  IPv4 0x321a94eae97e5edf      0t0  TCP localhost:12308 (LISTEN)
```

- 查看系统日志
```bash
# 查看启动日志
$ cat ../build/release/bin/system.log 
2021-06-23 00:28:02.343 [INFO]  [Cli]   cmd/cli_start.go:42     Start SPV Server!

# 查看SPV日志
$ tail -f ../build/release/log/spv.log
2021-06-23 00:28:02.313 [INFO]  [SDK]   chainmaker-sdk/sdk_cert_manage.go:89    [SDK] begin to query cert, [contract:SYSTEM_CONTRACT_CERT_MANAGE]/[method:CERTS_QUERY]
2021-06-23 00:28:02.340 [INFO]  [StateManager]  manager/state_manager.go:133    [ChainId:chain1] Start state manager!
2021-06-23 00:28:02.342 [INFO]  [StateManager]  manager/state_manager.go:162    [ChainId:chain1] subscribe block successfully!
2021-06-23 00:28:02.345 [INFO]  [Rpc]   rpcserver/rpc_server.go:54      GRPC Server Listen on 127.0.0.1:12308
2021-06-23 00:28:12.414 [INFO]  [BlockManager]  manager/block_manager.go:130    [ChainId:chain1] spv has synced to the highest block! current local height:1, remote max height:1
```

### 停止SPV轻节点
- 在`chainmaker-spv/scripts`目录，运行 `stop.sh` 脚本，将会调用`../build/release/bin`目录中的`stop.sh`脚本，停止SPV轻节点。
```bash
$ ./stop.sh
```

### 停止SPV轻节点并清除data 
- 在`chainmaker-spv/scripts`目录，运行 `stop.sh` 脚本，并添加`clean`命令，将会调用`../build/release/bin`目录中的`stop.sh`脚本，停止SPV轻节点，并清除`../build/release/data`中的所有数据。
```bash
$ ./stop.sh clean
```

## Client端验证交易有效性示例

```go
import (
	"chainmaker.org/chainmaker-spv/pb/api"
	"context"
	"fmt"
	"google.golang.org/grpc"
)

func useCase() {
	// 1.构造Client
	conn, err := grpc.Dial("127.0.0.1:12308", grpc.WithInsecure())
	if err != nil {
		fmt.Println("connection failed!")
		return
	}

	client := api.NewRpcProverClient(conn)

	// 2.构造交易验证信息
	txInfo := &api.TxVerifyInfo{}

	// 3.验证交易有效性
	_, err = client.ValidTransaction(context.Background(), txInfo)
	if err != nil {
		fmt.Println("invalid transaction!")
	}
	
	// 4.用户其他逻辑
	
}
```

## SPV作为组件集成进其他项目
### 创建并启动组件
```go
import (
	"chainmaker.org/chainmaker-spv/pb/api"
	"chainmaker.org/chainmaker-spv/server"
	"fmt"
	"go.uber.org/zap"
	"time"
)

func useCase() {
	var (
		ymlFile = "../config/spv_config.yml"
		log     = &zap.SugaredLogger{}
	)

	// 1. 创建 spv server
	spvServer, err := server.NewSpvServer(ymlFile, log)
	if err != nil {
		fmt.Println("new spv server failed!")
		return
	}

	// 2. 启动 spv server
	err = spvServer.Start()
	if err != nil {
		fmt.Println("start spv server failed!")
		return
	}
}
``` 
### 进行交易有效性验证
```go
func useCase() {
	// 1. 构造交易验证信息
	txInfo := &api.TxVerifyInfo{}
	timeout := 20000*time.Millisecond
	
	// 2. 交易存在性和有效性验证
	err = spvServer.ValidTransaction(txInfo, timeout)
	if err != nil {
		fmt.Println("invalid transaction!")
		return
	}
}
```