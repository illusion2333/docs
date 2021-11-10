# SPV轻节点部署和使用文档
## 概述
### SPV轻节点概述
SPV轻节点在`spv`和`light`两种模式下，支持独立部署和作为组件集成的方式使用：
- 独立部署，单独一个进程。在`spv`模式下，通过同步区块头和交易hash可对外提供交易存在性及有效性证明服务；在`light`模式下，可同步和查询区块和同组织内的交易数据。
- 作为组件集成进其他项目，与其他项目在一个进程中。在`spv`模式下，调用启动以获取业务链的数据，可提供交易存在性及有效性证明功能；在`light`模式下，可同步和查询区块和同组织内的交易数据，并支持用户注册回调函数，在提交区块后将被执行。

了解SPV轻节点设计方案，请点击如下链接：  
[SPV轻节点设计文档](../tech/SPV.md)

## SPV轻节点独立部署流程
### 源码下载
- 下载`chainmaker-spv`源码到本地
```bash
$ git clone --recursive https://git.chainmaker.org.cn/chainmaker/chainmaker-spv.git
```

### 生成物料
- 进入`chainmaker-spv/scripts`目录，执行`prepare.sh`脚本，将编译生成spv二进制文件，并生成spv所需要的配置文件，存于`chainmaker-spv/build/release`路径中。
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

# 查看生成的二进制文件和配置文件
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
- 从SPV轻节点节点所要链接的远端ChainMaker链，拷贝节点证书配置文件`crypto-config`，更新SPV轻节点项目`chainmaker-spv/build/release/config`路径下的`crypto-config`文件。

> 目前SPV轻节点只支持作为ChainMaker的轻节点，通常`crypto-config`在`chainmaker-go/build`路径下。

- 修改`chainmaker-spv/build/release/config`路径下的SPV配置文件`spv_config.yml`

> 在chains中配置SPV轻节点信息，只支持`chainmaker_light`,`chainmaker_spv`和`fabric_spv`三种模式，可支持多链。需配置的远端链信息包括链ID、同步链最新区块高度时间间隔、链SDK配置文件路径。  
> 在grpc中配置SPV提供交易存在性和有效性服务的grpc地址/端口。 
> 在web中配置查询区块信息、交易信息、以及SPV轻节点同步的区块高度等服务的web地址/端口。
> 在storage中配置SPV的存储模块，目前多链共用同一存储模块。  
> 在log中配置SPV的log信息，目前多链共用日志模块，多链以chainID区分。  
> **注意：SPV配置文件中的路径是基于bin文件夹下spv二进制文件的相对路径，也可以使用绝对路径。**

```yaml
# 链配置
chains:
  # 类型  仅支持（chainmaker_light，chainmaker_spv，fabric_spv）
  - chain_type: "chainmaker_light"
    # 链ID
    chain_id: "chain1"
    # 同步配置，同步链中节点区块最新高度信息的时间间隔，单位：毫秒
    sync_chainInfo_interval: 10000
    # sdk配置文件路径
    sdk_config_path: "../config/chainmaker_config/chainmaker_sdk_config_chain1.yml"

# - chain_type: "chainmaker_spv"
#   # 链ID
#   chain_id: "chain2"
#   # 同步配置，同步链中节点区块最新高度信息的时间间隔，单位：毫秒
#   sync_chainInfo_interval: 10000
#   # sdk配置文件路径
#   sdk_config_path: "../config/chainmaker_config/chainmaker_sdk_config_chain2.yml"

# - chain_type: "fabric_spv"
#   chain_id: "mychannel"
#   sync_chainInfo_interval: 10000
#   sdk_config_path: "../config/fabric_config/fabric_sdk_config_org1.yaml"
#   fabric_extra_config:    # fabric特有的配置项，其他类型的链不需要配置
#     user: "User1"    # 用户名
#     peers:           # 节点列表
#       - peer: "peer0.org1.example.com"
#       - peer: "peer1.org1.example.com"
#       - peer: "peer0.org2.example.com"
#       - peer: "peer1.org2.example.com"

# grpc配置
grpc:
  # grpc监听网卡地址
  address: 127.0.0.1
  # grpc监听端口
  port: 12308

# web配置
web:
  # web服务监听网卡地址
  address: 127.0.0.1
  # web监听端口
  port: 8080

# 存储配置，用于配置当前SPV对区块头和交易哈希的存储记录
storage:
  # 存储采用的类型，当前仅支持leveldb类型
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
> 在nodes中配置连接节点信息。**注意：修改节点的地址/端口，端口是ChainMaker的RPC端口**。  
> **注意：SDK配置文件中的路径是基于bin文件夹下spv二进制文件的相对路径，也可以使用绝对路径。私钥和证书配置请按照`spv`或`light`模式的不同选择对应的私钥和证书路径。**

```yaml
chain_client:
  # 链ID
  chain_id: "chain1"
  # 组织ID
  org_id: "wx-org1.chainmaker.org"
  # 客户端用户私钥路径（如果配置为chainmaker_spv，此处请配置为client私钥，如果配置为chainmaker_light，此处请配置为light私钥，下面另外三项配置同理）
  user_key_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key"
  # 客户端用户证书路径
  user_crt_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt"
  # 客户端用户交易签名私钥路径(若未设置，将使用user_key_file_path)
  user_sign_key_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key"
  # 客户端用户交易签名证书路径(若未设置，将使用user_crt_file_path)
  user_sign_crt_file_path: "../config/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt"

  nodes:
    - # 节点地址，格式为：IP:端口，端口是ChainMaker中的RPC端口
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
    - # 节点地址，格式为：IP:端口，端口是ChainMaker中的RPC端口
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

- 在`chainmaker-spv/scripts`目录，运行 `start.sh` 脚本，将会调用`chainmaker-spv/build/release/bin`目录中的`start.sh`脚本，启动SPV轻节点。
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

- 查看日志
```bash
$ tail -f ../build/release/log/spv.log
2021-06-23 00:28:02.313 [INFO]  [Sdk]   chainmaker-sdk/sdk_cert_manage.go:89    [SDK] begin to query cert, [contract:SYSTEM_CONTRACT_CERT_MANAGE]/[method:CERTS_QUERY]
2021-06-23 00:28:02.318 [INFO]  [SPV]   server/spv_server.go:88 Start SPV Server!
2021-06-23 00:28:02.340 [INFO]  [StateManager]  manager/state_manager.go:145    [ChainId:chain1] Start state manager!
2021-06-23 00:28:02.342 [INFO]  [StateManager]  manager/state_manager.go:176    [ChainId:chain1] subscribe block successfully!
2021-06-23 00:28:02.345 [INFO]  [Rpc]   rpcserver/rpc_server.go:65      GRPC Server Listen on 127.0.0.1:12308
2021-06-23 00:28:12.414 [INFO]  [BlockManager]  manager/block_manager.go:167    [ChainId:chain1] spv has synced to the highest block! current local height:1, remote max height:1
```

### 停止SPV轻节点
- 在`chainmaker-spv/scripts`目录，运行 `stop.sh` 脚本，将会调用`chainmaker-spv/build/release/bin`目录中的`stop.sh`脚本，停止SPV轻节点。
```bash
$ ./stop.sh
```

### 停止SPV轻节点并清除data和log
- 在`chainmaker-spv/scripts`目录，运行 `stop.sh` 脚本，并添加`clean`命令，将会调用`chainmaker-spv/build/release/bin`目录中的`stop.sh`脚本，停止SPV轻节点，并清除`chainmaker-spv/build/release/data`中的所有数据。
```bash
$ ./stop.sh clean
```

## SPV模式独立部署时，Client端验证交易有效性示例
```go
package usecase

import (
	"context"
	"log"

	"chainmaker.org/chainmaker-spv/pb/api"
	"google.golang.org/grpc"
)

func useCase() {
	// 1.构造Client
	conn, err := grpc.Dial("127.0.0.1:12308", grpc.WithInsecure())
	if err != nil {
		log.Println("connection failed!")
		return
	}
	client := api.NewRpcProverClient(conn)

	// 2.构造交易验证信息
	txInfo := &api.TxValidationInfo{
		ChainId: "chainId", // 链Id
		BlockHeight: 1,     // 交易所在区块高度
		//Index: -1,        // 此版本未验证该字段，不需要填写
		TxKey: "TxId",      // 交易Id
		ContractData: &api.ContractData{
			ContractName: "contractName",  // 合约名
			Method: "method",              // 方法名
			//Version: "version",          // 此版本未验证该字段，不需要填写
			Params: []*api.KVPair{
				{Key: "argName1", Value: "argValue1"},  // Key是所调用合约方法的参数名，Value是参数值
				{Key: "argName2", Value: "argValue2"},
				{Key: "argName3", Value: "argValue3"},
			},
			Extra: nil,    // 预留扩展字段
		},
		Extra: nil,        // 预留扩展字段
	}

	// 3.验证交易有效性
	response, err := client.ValidTransaction(context.Background(), txInfo)
	if err != nil {
		log.Println("invalid transaction!")
	}

	if int32(response.Code) != 0 {
		log.Println("invalid transaction!")
	}

	// 4.用户其他逻辑

}

```

## SPV模式作为组件集成进其他项目时，进行交易有效性验证示例
### 创建并启动组件
```go
package usecase

import (
	"log"

	"chainmaker.org/chainmaker-spv/server"
	"go.uber.org/zap"
)

func useCase() {
	var (
		ymlFile = "../config/spv_config.yml"
		logger     = &zap.SugaredLogger{}
	)

	// 1.创建 spv server
	spvServer, err := server.NewSPVServer(ymlFile, logger)
	if err != nil {
		log.Println("new spv server failed!")
		return
	}

	// 2.启动 spv server
	err = spvServer.Start()
	if err != nil {
		log.Println("start spv server failed!")
		return
	}
}
```
### 进行交易有效性验证
```go
func useCase() {
	// 1.构造交易验证信息
	txInfo := &api.TxValidationInfo{
		ChainId: "chainId", // 链Id
		BlockHeight: 1,     // 交易所在区块高度
		//Index: -1,        // 此版本未验证该字段，不需要填写
		TxKey: "TxId",      // 交易Id
		ContractData: &api.ContractData{
			ContractName: "contractName",  // 合约名
			Method: "method",              // 方法名
			//Version: "version",          // 此版本未验证该字段，不需要填写
			Params: []*api.KVPair{
				{Key: "argName1", Value: "argValue1"},  // Key是所调用合约方法的参数名，Value是参数值
				{Key: "argName2", Value: "argValue2"},
				{Key: "argName3", Value: "argValue3"},
			},
			Extra: nil,    // 预留扩展字段
		},
		Extra: nil,        // 预留扩展字段
	}

	// 2.设置验证超时时间（默认设置5000ms）
	timeout := 20000*time.Millisecond

	// 3.验证交易有效性
	err := spvServer.ValidTransaction(txInfo, timeout)
	if err != nil {
		log.Printf("invalid transaction! err:%s", err.Error())
		return
	}

	// 4.用户其他逻辑
}
```

## Light模式独立部署时，可对外提供如下查询方法
```go
package webserver
import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"

	"chainmaker.org/chainmaker-spv/pb/api"
)

func useCase() {
	// 1.根据chainId和区块height获取区块，其中fromRemote为true表明Light节点从远端链获取，fromRemote为false表明Light节从本地获取（只包含区块头）
	url1 := "http://localhost:8080/GetBlockByHeight?fromRemote=false&chainId=chain1&height=1"
	res1, err := http.Post(url1, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content1, err:= ioutil.ReadAll(res1.Body)
	defer res1.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content1)

	// 2.根据chainId和区块hash获取区块，其中fromRemote为true表明Light节点从远端链获取，fromRemote为false表明Light节从本地获取（只包含区块头）
	url2 := "http://localhost:8080/GetBlockByHash?fromRemote=false&chainId=chain1&blockHash=xxx"
	res2, err := http.Post(url2, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content2, err:= ioutil.ReadAll(res2.Body)
	defer res2.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content2)

	// 3.根据chainId和txId获取获取交易所在区块，其中fromRemote为true表明Light节点从远端链获取，fromRemote为false表明Light节从本地获取（只包含区块头）
	url3 := "http://localhost:8080/GetBlockByTxId?fromRemote=false&chainId=chain1&txId=xxx"
	res3, err := http.Post(url3, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content3, err:= ioutil.ReadAll(res3.Body)
	defer res3.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content3)

	// 4.根据chainId获取最近被提交的区块，其中fromRemote为true表明Light节点从远端链获取，fromRemote为false表明Light节从本地获取（只包含区块头）
	url4 := "http://localhost:8080/GetLastBlock?fromRemote=false&chainId=chain1"
	res4, err := http.Post(url4, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content4, err:= ioutil.ReadAll(res4.Body)
	defer res4.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content4)

	// 5.根据chainId和txId获取同组织内的某一交易，其中fromRemote为true表明Light节点从远端链获取，fromRemote为false表明Light节从本地获取
	url5 := "http://localhost:8080/GetTransactionByTxId?fromRemote=false&chainId=chain1&txId=xxx"
	res5, err := http.Post(url5, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content5, err:= ioutil.ReadAll(res5.Body)
	defer res5.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content5)

	// 6.根据chainId获取light节点同步的同组织交易总数
	url6 := "http://localhost:8080/GetTransactionTotalNum?chainId=chain1"
	res6, err := http.Post(url6, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content6, err:= ioutil.ReadAll(res6.Body)
	defer res6.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content6)

	// 7.根据chainId获取light节点同步的区块总数，最近被提交的区块高度+1
	url7 := "http://localhost:8080/GetBlockTotalNum?chainId=chain1"
	res7, err := http.Post(url7, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content7, err:= ioutil.ReadAll(res7.Body)
	defer res7.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content7)

	// 8.根据chainId获取light节点同步的区块高度
	url8 := "http://localhost:8080/GetCurrentBlockHeight?chainId=chain1"
	res8, err := http.Post(url8, "application/json", nil)
	if err != nil {
		log.Error(err)
		return
	}
	content8, err:= ioutil.ReadAll(res8.Body)
	defer res8.Body.Close()
	if err != nil {
		log.Error(err)
		return
	}
	fmt.Print(content8)

	// 9.根据交易验证信息api.TxValidationInfo验证同组织内交易有效性
	url9 := "http://localhost:8080/ValidTransaction"
	txInfo := &api.TxValidationInfo{
		ChainId: "chainId", // 链Id
		BlockHeight: 1,     // 交易所在区块高度
		//Index: -1,        // 此版本未验证该字段，不需要填写
		TxKey: "TxId",      // 交易Id
		ContractData: &api.ContractData{
			ContractName: "contractName",  // 合约名
			Method: "method",              // 方法名
			//Version: "version",          // 此版本未验证该字段，不需要填写
			Params: []*api.KVPair{
				{Key: "argName1", Value: "argValue1"},  // Key是所调用合约方法的参数名，Value是参数值
				{Key: "argName2", Value: "argValue2"},
				{Key: "argName3", Value: "argValue3"},
			},
			Extra: nil,    // 预留扩展字段
		},
		Extra: nil,        // 预留扩展字段
	}
	js, err := json.Marshal(txInfo)
	if err != nil {
		log.Error(err)
		return
	}
	body := bytes.NewBuffer(js)
	res9, err := http.Post(url9, "application/json", body)
	if err != nil {
		t.Log(err)
		return
	}
	content9, _ := ioutil.ReadAll(res9.Body)
	defer res9.Body.Close()
	fmt.Print(content9)
}

```

## Light模式作为组件集成进其他项目时，可注册回调函数
```go
package usecase

import (
	"fmt"
	"log"

	"chainmaker.org/chainmaker-spv/common"
	"chainmaker.org/chainmaker-spv/server"
	"go.uber.org/zap"
)

func useCase() {
	var (
		ymlFile = "../config/spv_config.yml"
		logger  = &zap.SugaredLogger{}
	)

	// 1.创建 spv server
	spvServer, err := server.NewSPVServer(ymlFile, logger)
	if err != nil {
		log.Println("new light server failed!")
		return
	}

	// 2.注册回调，当区块在light提交至数据库时被执行
	err = spvServer.RegisterCallBack("chain1", func(block common.Blocker) {
		fmt.Printf("block height: %d", block.GetHeight())
	})

	// 3.启动 spv server
	err = spvServer.Start()
	if err != nil {
		log.Println("start light server failed!")
		return
	}
}
```