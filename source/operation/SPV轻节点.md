# SPV轻节点 部署和使用文档
## 概述
### SPV轻节点概述
SPV轻节点在`spv`和`light`两种模式下，支持独立部署和作为组件集成的方式使用：
- 独立部署，单独一个进程。在`spv`模式下，作为验证节点，通过同步区块头和部分其他数据，可对外提供交易存在性及有效性证明服务；在`light`模式下，作为轻节点，可同步区块及同组织内的交易。
- 作为组件集成进其他项目，与其他项目在一个进程中。在`spv`模式下，调用启动以获取业务链的数据，可提供交易存在性及有效性证明功能；在`light`模式下，可同步和查询区块和同组织内的交易数据，并支持用户注册回调函数，在提交区块后将被执行。

了解SPV轻节点设计方案，请点击如下链接：  
[SPV轻节点设计文档](../tech/SPV轻节点.md)

## SPV轻节点独立部署流程
### 源码下载
- 下载`spv`源码到本地
```bash
$ git clone -b v1.2.3 https://git.chainmaker.org.cn/chainmaker/spv.git
```

### 生成物料
- 进入`spv/scripts`目录，执行`prepare.sh`脚本，将编译生成spv二进制文件，并生成spv所需要的配置文件，存于`spv/build/release`路径中。
```bash
# 进入脚本目录
$ cd spv/scripts

# 查看目录结构
$ tree 
.
├── local              # 该文件下的脚本，在生成物料时，将被拷贝到spv/build/release/bin目录下
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
../build/release
├── bin
│   ├── spv      // spv二进制文件
│   ├── start.sh // 启动脚本
│   └── stop.sh  // 停止脚本
└── config
    ├── chainmaker
    │   ├── chainmaker_sdk.yml  // chainmaker sdk 配置文件
    │   └── crypto-config       // chainmaker 证书文件
    ├── fabric
    │   ├── crypto-config      // fabric sdk 配置文件
    │   └── fabric_sdk.yml     // fabric 证书文件
    ├── spv.yml
    └── tls
        ├── ca.pem       // tls ca证书
        ├── server.key   // tls 私钥
        └── server.pem   // tls ca证书
               
```

### 修改配置文件
- 从SPV轻节点节点所要链接的远端ChainMaker链或者Fabric链，拷贝节点证书配置文件`crypto-config`，更新SPV轻节点项目`spv/build/release/config/chainmaker或fabric`路径下的`crypto-config`文件。
- 修改`spv/build/release/config`路径下的SPV配置文件`spv.yml`

> 在chains中配置SPV轻节点信息，只支持`ChainMaker_Light`,`ChainMaker_SPV`和`Fabric_SPV`三种模式，可支持多链。需配置的远端链信息包括链ID、同步链最新区块高度时间间隔、并发请求区块的数量、链SDK配置文件路径。  
> 在grpc中配置SPV提供交易存在性和有效性服务的grpc地址/端口以及tls相关信息。 
> 在web中配置查询区块信息、交易信息、以及SPV轻节点同步的区块高度等服务的web地址/端口以及tls相关信息。
> 在storage中配置SPV的存储模块，目前多链共用同一存储模块。  
> 在log中配置SPV的log信息，目前多链共用日志模块，多链以chainId区分。  
> **注意：SPV配置文件中的路径请使用绝对路径。**

```yaml
# 链配置
chains:
  # 类型，当前仅支持（ChainMaker_Light，ChainMaker_SPV，Fabric_SPV）三种类型
  - chain_type: "ChainMaker_Light"
    # 链ID
    chain_id: "chain1"
    # 同步链中节点区块最新高度信息的时间间隔，单位：毫秒
    sync_interval: 10000
    # 并发请求区块的数量
    concurrent_nums: 100
    # sdk配置文件路径
    sdk_config_path: "/release_path/config/chainmaker/chainmaker_sdk.yml"

  # 类型，当前仅支持（ChainMaker_Light，ChainMaker_SPV，Fabric_SPV）三种类型
  - chain_type: "ChainMaker_SPV"
    # 链ID
    chain_id: "chain1"
    # 同步链中节点区块最新高度信息的时间间隔，单位：毫秒
    sync_interval: 10000
    # 并发请求区块的数量
    concurrent_nums: 100
    # sdk配置文件路径
    sdk_config_path: "/release_path/config/chainmaker/chainmaker_sdk.yml"

  # 类型，当前仅支持（ChainMaker_Light，ChainMaker_SPV，Fabric_SPV）三种类型
  - chain_type: "Fabric_SPV"
    # 链ID
    chain_id: "mychannel"
    # 同步链中节点区块最新高度信息的时间间隔，单位：毫秒
    sync_interval: 10000
    # 并发请求区块的数量
    concurrent_nums: 100
    # sdk配置文件路径
    sdk_config_path: "/release_path/config/fabric/fabric_sdk.yml"
    # fabric特有的配置项，其他类型的链不需要配置
    fabric_extra_config:
      # 节点列表
      peers:
        - peer: "peer0.org1.example.com"
        - peer: "peer1.org1.example.com"

# grpc配置
grpc:
  # grpc监听网卡地址
  address: 127.0.0.1
  # grpc监听端口
  port: 12345
  # 是否开启tls验证
  enable_tls: false
  security:
    # 是否开启CA验证
    ca_auth: false
    # ca文件
    ca_file:
      - "/release_path/config/tls/ca.pem"
    # tls证书文件
    cert_file: "/release_path/config/tls/server.pem"
    # tls私钥文件
    key_file: "/release_path/config/tls/server.key"

# web配置
web:
  # web服务监听网卡地址，http或https由${enable_tls}参数判断，无需配置
  address: 127.0.0.1
  # web监听端口
  port: 12346
  # 是否开启tls验证
  enable_tls: false
  security:
    # 是否开启CA验证
    ca_auth: false
    # ca文件
    ca_file:
      - "/release_path/config/tls/ca.pem"
    # tls证书文件
    cert_file: "/release_path/config/tls/server.pem"
    # tls私钥文件
    key_file: "/release_path/config/tls/server.key"

# 存储配置
storage:
  # 存储类型，当前仅支持leveldb类型
  provider: "leveldb"
  # leveldb的详细配置
  leveldb:
    # leveldb的存储路径
    store_path: "/release_path/data/spv_db"
    # leveldb写入Buffer大小，单位：MB
    write_buffer_size: 32
    # leveldb布隆过滤器的bit长度
    bloom_filter_bits: 10

# 日志配置，用于配置日志的打印
log:
  # 日志打印级别
  log_level: "INFO"
  # 日志文件路径
  file_path: "/release_path/log/spv.log"
  # 日志最长保存时间，单位：天
  max_age: 365
  # 日志滚动时间，单位：小时
  rotation_time: 1
  # 是否展示日志到终端，仅限于调试使用
  log_in_console: false
  # 是否打印颜色日志
  show_color: true
```

- 修改`spv/build/release/config/chainmaker或fabric`路径下各链的SDK配置`chainmaker_sdk.yml或fabric_sdk.yml`

> 在chain_client中配置用户证书信息。  
> 在nodes中配置节点信息。  
> 在rpc_client中传输最大数据尺寸。  
> **注意：SDK配置文件中的路径请使用绝对路径。**

```yaml
chain_client:
   # 链ID
   chain_id: "chain1"
   # 组织ID
   org_id: "wx-org1.chainmaker.org"
   # 客户端用户私钥路径（如果是ChainMaker_SPV类型，此处请配置为Client私钥，如果是ChainMaker_Light类型，此处请配置为Light私钥，下面另外三项配置同理）
   user_key_file_path: "/release_path/config/chainmaker/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key"
   # 客户端用户证书路径
   user_crt_file_path: "/release_path/config/chainmaker/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt"
   # 客户端用户交易签名私钥路径
   user_sign_key_file_path: "/release_path/config/chainmaker/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key"
   # 客户端用户交易签名证书路径
   user_sign_crt_file_path: "/release_path/config/chainmaker/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt"

   nodes:
      - # 节点地址，格式为：IP:端口，端口是ChainMaker中的RPC端口
         node_addr: "127.0.0.1:12301"
         # 节点连接数
         conn_cnt: 10
         # RPC连接是否启用双向TLS认证
         enable_tls: true
         # 信任证书池路径
         trust_root_paths:
            - "/release_path/config/chainmaker/crypto-config/wx-org1.chainmaker.org/ca"
            - "/release_path/config/chainmaker/crypto-config/wx-org2.chainmaker.org/ca"
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
            - "/release_path/config/chainmaker/crypto-config/wx-org1.chainmaker.org/ca"
            - "/release_path/config/chainmaker/crypto-config/wx-org2.chainmaker.org/ca"
         # TLS hostname
         tls_host_name: "chainmaker.org"
   archive:
      # 数据归档链外存储相关配置
      type: "mysql"
      dest: "root:123456:localhost:3306"
      secret_key: xxx

   rpc_client:
      # grpc客户端最大接受容量(MB)
      max_receive_message_size: 32
```

> 在client中配置用户证书信息。  
> 在channels中配置通道信息。
> 在organizations中配置组织信息。
> 在peers中配置peer信息。
> **注意：SDK配置文件中的路径请使用绝对路径。**

```yaml
# 版本
version: 1.0.0
# client配置
client:
   # 客户端默认使用的组织
   organization: Org1
   logging:
      # sdk日志级别
      level: info
   tlsCerts:
      systemCertPool: false
      client:
         # 用户TLS私钥路径
         key:
            path: /release_path/config/fabric/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/tls/client.key
         # 用户TLS证书路径
         cert:
            path: /release_path/config/fabric/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/tls/client.crt
# 通道信息
channels:
   # 通道名
   mychannel:
      # peer节点列表
      peers:
         # peer节点名
         peer0.org1.example.com:
            endorsingPeer: true
            chaincodeQuery: true
            ledgerQuery: true
            eventSource: true
         peer1.org1.example.com:
            endorsingPeer: true
            chaincodeQuery: true
            ledgerQuery: true
            eventSource: true
# 组织信息
organizations:
   # 组织名
   org1:
      # 组织mspId
      mspid: Org1MSP
      # 该组织下的节点列表
      peers:
         - peer0.org1.example.com
         - peer1.org1.example.com
      # 组织用户
      users:
         # 用户名，固定为user
         user:
            # 用户私钥路径
            key:
               path: /release_path/config/fabric/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/keystore/8cc7e53b2b2f3095b985139f667b260988afa3dad2c0ff24cb9e45fb93d77970_sk
            # 用户证书路径
            cert:
               path: /release_path/config/fabric/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/signcerts/User1@org1.example.com-cert.pem
# 节点信息
peers:
   # peer节点名
   peer0.org1.example.com:
      # peer节点url
      url: grpcs://localhost:7051
      grpcOptions:
         # peer节点TLS中的SNI，使用节点名
         ssl-target-name-override: peer0.org1.example.com
         keep-alive-time: 0s
         keep-alive-timeout: 20s
         keep-alive-permit: false
         fail-fast: false
         allow-insecure: false
      # TLS CA证书路径
      tlsCACerts:
         path: /release_path/config/fabric/crypto-config/peerOrganizations/org1.example.com/tlsca/tlsca.org1.example.com-cert.pem
   # peer节点名
   peer1.org1.example.com:
      # peer节点url
      url: grpcs://localhost:8051
      grpcOptions:
         # peer节点TLS中的SNI，使用节点名
         ssl-target-name-override: peer1.org1.example.com
         keep-alive-time: 0s
         keep-alive-timeout: 20s
         keep-alive-permit: false
         fail-fast: false
         allow-insecure: false
      # TLS CA证书路径
      tlsCACerts:
         path: /release_path/config/fabric/crypto-config/peerOrganizations/org1.example.com/tlsca/tlsca.org1.example.com-cert.pem
entityMatchers:
   peer:
      - pattern: (\w*)peer0.org1.example.com(\w*)
        mappedHost: peer0.org1.example.com
      - pattern: (\w*)peer1.org1.example.com(\w*)
        mappedHost: peer1.org1.example.com
```

### 启动SPV轻节点

- 在`spv/scripts`目录，运行 `start.sh` 脚本，将会调用`spv/build/release/bin`目录中的`start.sh`脚本，启动SPV轻节点。
```bash
$ ./start.sh
```

- 查看进程是否存在
```bash
$ ps -ef|grep spv | grep -v grep
501 82533     1   0 12:27AM ttys011    0:00.23 ./spv start -c ../config/spv.yml
```

- 查看端口是否监听
```bash
$ lsof -i:12345
COMMAND   PID      USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
spv     82533 liukemeng   14u  IPv4 0x321a94eae97e5edf      0t0  TCP localhost:12345 (LISTEN)

$ lsof -i:12346
COMMAND   PID      USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
spv     85673 liukemeng   14u  IPv4 0x425a94eae97e5edf      0t0  TCP localhost:12346 (LISTEN)
```

- 查看日志
```bash
$ tail -f ../build/release/log/spv.log
2021-06-23 00:28:02.318 [INFO]  [SPV]   server/spv_server.go:88 ==== Start SPV Server! ====
2021-06-23 00:28:02.340 [INFO]  [StateManager]  manager/state_manager.go:145    [ChainId:chain1] ---- start chain listening and state manager! ----
2021-06-23 00:28:02.342 [INFO]  [StateManager]  manager/state_manager.go:176    [ChainId:chain1] subscribe block success!
2021-06-23 00:28:02.345 [INFO]  [Rpc]   rpcserver/rpc_server.go:65      GRPC Server Listen on 127.0.0.1:12345
2021-06-23 00:28:02.345 [INFO]  [Web]   webserver/web_server.go:85      Web Server Listen on HTTP 127.0.0.1:12346
2021-06-23 00:28:12.414 [INFO]  [BlockManager]  manager/block_manager.go:167    [ChainId:chain1] spv has synced to the highest block! current local height:0, remote max height:0
```

### 停止SPV轻节点
- 在`spv/scripts`目录，运行 `stop.sh` 脚本，将会调用`spv/build/release/bin`目录中的`stop.sh`脚本，停止SPV轻节点。
```bash
$ ./stop.sh
```

### 停止SPV轻节点并清除data和log
- 在`spv/scripts`目录，运行 `stop.sh` 脚本，并添加`clean`命令，将会调用`spv/build/release/bin`目录中的`stop.sh`脚本，停止SPV轻节点，并清除`spv/build/release/data`中的所有数据。
```bash
$ ./stop.sh clean
```

## SPV模式独立部署时，Client端通过grpc验证交易有效性示例
```go
package usecase

import (
	"context"
	"log"

	"chainmaker.org/chainmaker/spv/v2/pb/api"
	"google.golang.org/grpc"
)

func useCase() {
	// 1.构造Client
	conn, err := grpc.Dial("127.0.0.1:12308", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
		return
	}
	client := api.NewRpcProverClient(conn)

	// 2.构造交易验证信息
	request := &api.TxValidationRequest{
		ChainId: "chainId", // 链Id
		BlockHeight: 1,     // 交易所在区块高度
		//Index: -1,        // 此版本未验证该字段，不需要填写
		TxKey: "TxId",      // 交易Id
		ContractData: &api.ContractData{
			Name: "contractName",  // 合约名
			Method: "method",              // 方法名
			Version: "version",            // 合约版本
			Params: []*api.KVPair{
				{Key: "argName1", Value: []byte("argValue1")},  // Key是所调用合约方法的参数名，Value是参数值
				{Key: "argName2", Value: []byte("argValue2")},
				{Key: "argName3", Value: []byte("argValue3")},
			},
			Extra: nil,    // 预留扩展字段
		},
		Timeout: 5000,     // 验证超时时间 
		Extra: nil,        // 预留扩展字段
	}

	// 3.验证交易有效性
	response, err := client.ValidTransaction(context.Background(), request)
	if err != nil {
		log.Fatal(err)
	}

	if int32(response.Code) != 0 {
		log.Fatal(err)
	}

	// 4.用户其他逻辑

}
```

## SPV模式独立部署时，Client端通过web验证交易有效性示例
```go
package usecase

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"

	"chainmaker.org/chainmaker/spv/v2/pb/api"
)

func useCase() {
	// 1.构造交易验证信息
	request := &api.TxValidationRequest{
		ChainId: "chainId", // 链Id
		BlockHeight: 1,     // 交易所在区块高度
		//Index: -1,        // 此版本未验证该字段，不需要填写
		TxKey: "TxId",      // 交易Id
		ContractData: &api.ContractData{
			Name: "contractName",  // 合约名
			Method: "method",              // 方法名
			Version: "version",            // 合约版本
			Params: []*api.KVPair{
				{Key: "argName1", Value: []byte("argValue1")},  // Key是所调用合约方法的参数名，Value是参数值
				{Key: "argName2", Value: []byte("argValue2")},
				{Key: "argName3", Value: []byte("argValue3")},
			},
			Extra: nil,    // 预留扩展字段
		},
		Timeout: 5000,     // 验证超时时间 
		Extra: nil,        // 预留扩展字段
	}

	// 2.验证交易有效性
	bz, err := json.Marshal(request)
	if err != nil {
		log.Fatal(err)
		return
	}
	resp, err := http.Post("http://localhost:12346/ValidTransaction", "application/json", bytes.NewBuffer(bz))
	if err != nil {
		log.Fatal(err)
		return
	}
	data, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		log.Fatal(err)
		return
	}
	fmt.Println(string(data))

	// 3.用户其他逻辑

}
```

## SPV模式作为组件集成进其他项目时，进行交易有效性验证示例
### 创建并启动组件
```go
package usecase

import (
	"log"

	"chainmaker.org/chainmaker/spv/v2/server"
	"go.uber.org/zap"
)

func useCase() {
	var (
		ymlFile = "/release_path/config/spv.yml"
		logger     = &zap.SugaredLogger{}
	)

	// 1.创建 spv server
	spvServer, err := server.NewSPVServer(ymlFile, logger)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 2.启动 spv server
	err = spvServer.Start()
	if err != nil {
		log.Fatal(err)
		return
	}
}
``` 
### 进行交易有效性验证
```go
package usecase

import (
	"log"
	
	"chainmaker.org/chainmaker/spv/v2/pb/api"
	"chainmaker.org/chainmaker/spv/v2/server"
	"go.uber.org/zap"
)

func useCase() {
	// 1.构造交易验证信息
	request := &api.TxValidationRequest{
		ChainId: "chainId", // 链Id
		BlockHeight: 1,     // 交易所在区块高度
		//Index: -1,        // 此版本未验证该字段，不需要填写
		TxKey: "TxId",      // 交易Id
		ContractData: &api.ContractData{
			Name: "contractName",  // 合约名
			Method: "method",              // 方法名
			Version: "version",            // 合约版本
			Params: []*api.KVPair{
				{Key: "argName1", Value: []byte("argValue1")},  // Key是所调用合约方法的参数名，Value是参数值
				{Key: "argName2", Value: []byte("argValue2")},
				{Key: "argName3", Value: []byte("argValue3")},
			},
			Extra: nil,    // 预留扩展字段
		},
		Timeout: 5000,     // 验证超时时间 
		Extra: nil,        // 预留扩展字段
	}

	// 2.验证交易有效性
	err := spvServer.ValidTransaction(request)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 3.用户其他逻辑
}
```

## SPV模式或Light模式独立部署时，可对外提供如下查询方法
```go
package webserver
import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"

	"chainmaker.org/chainmaker/spv/v2/pb/api"
	"chainmaker.org/chainmaker/spv/v2/webserver"
)

func useCase() {
	// 1.根据chainId和区块height获取区块，其中fromRemote为true表明从远端链获取，fromRemote为false表明从本地获取（只包含区块头）
	wc := &webserver.WithChainIdAndHeight{
		ChainId:    "chain1",
		Height:     1,
		FromRemote: false,
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetBlockByHeight", bz)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 2.根据chainId和区块hash获取区块，其中fromRemote为true表明从远端链获取，fromRemote为false表明从本地获取（只包含区块头）
	wc := &webserver.WithChainIdAndHash{
		ChainId:    "chain1",
		Hash:       "eb7c3f3e0a3791266457f62c9fbf14eae29048754ca0206e62115e18f48c0807",
		FromRemote: false,
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetBlockByHash", bz)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 3.根据chainId和txKey获取获取交易所在区块，其中fromRemote为true表明从远端链获取，fromRemote为false表明从本地获取（只包含区块头）
	wc := &webserver.WithChainIdAndTxKey{
		ChainId:    "chain1",
		TxKey:      "a7861fd190f44d8ca12f31cb6d97929c6332a6172e7341c081dce08734d6b59c",
		FromRemote: false,
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetBlockByTxKey", bz)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 4.根据chainId获取最近被提交的区块，其中fromRemote为true表明从远端链获取，fromRemote为false表明从本地获取（只包含区块头）
	wc := &webserver.WithChainId{
		ChainId:    "chain1",
		FromRemote: false,
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetLastBlock", bz)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 5.根据chainId获取同步的区块高度
	wc := &webserver.WithChainId{
		ChainId: "chain1",
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetCurrentBlockHeight", bz)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 6.根据chainId和txKey获取light节点同组织内的某一交易，其中fromRemote为true表明从远端链获取，fromRemote为false表明从本地获取
	wc := &webserver.WithChainIdAndTxKey{
		ChainId:    "chain1",
		TxKey:      "a7861fd190f44d8ca12f31cb6d97929c6332a6172e7341c081dce08734d6b59c",
		FromRemote: false,
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetTransactionByTxKey", bz)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 7.根据chainId获取light节点同步的同组织交易总数
	wc := &webserver.WithChainId{
		ChainId: "chain1",
	}
	bz, err := json.Marshal(wc)
	if err != nil {
		log.Fatal(err)
		return
	}
	err = sendPostRequest("http://localhost:12346/GetBlockTotalNum", bz)
	if err != nil {
		log.Fatal(err)
		return
	}
}

func sendPostRequest(url string, jsonBz []byte) error {
	resp, err := http.Post(url, "application/json", bytes.NewBuffer(jsonBz))
	if err != nil {
		return err
	}
	fmt.Println(resp)
	data, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return err
	}
	fmt.Println(string(data))
	return nil
}

```

## Light模式作为组件集成进其他项目时，可注册回调函数
```go
package usecase

import (
	"fmt"
	"log"

	"chainmaker.org/chainmaker/spv/v2/common"
	"chainmaker.org/chainmaker/spv/v2/server"
	"go.uber.org/zap"
)

func useCase() {
	var (
		ymlFile = "/release_path/config/spv.yml"
		logger  = &zap.SugaredLogger{}
	)

	// 1.创建 spv server
	spvServer, err := server.NewSPVServer(ymlFile, logger)
	if err != nil {
		log.Fatal(err)
		return
	}

	// 2.注册回调，当区块在light提交至数据库时被执行
	err = spvServer.RegisterCallBack("chain1", func(block common.Blocker) {
		fmt.Printf("block height: %d", block.GetHeight())
	})

	// 3.启动 spv server
	err = spvServer.Start()
	if err != nil {
		log.Fatal(err)
		return
	}
}
```