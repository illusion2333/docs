# 3. 命令行工具

   

  ## 3.1. 简介

  cmc`(ChainMaker Client)`是ChainMaker提供的命令行工具，用于和ChainMaker链进行交互以及生成证书或者密钥等功能。cmc基于go语言编写，通过使用ChainMaker的go语言sdk（使用grpc协议）达到和ChainMaker链进行交互的目的。<br>
  cmc的详细日志请查看`./sdk.log`

  ## 3.2. 身份模式

  长安链在2.1版本以后支持不同身份模式，详情见[身份权限管理](../tech/身份权限管理.md)的身份模式部分。本篇文章中主要是介绍Public模式，其它两种模式的cmc使用文档如下：

  * Public
  * [PermissionedWithCert](./命令行工具.md)
  * [PermissionedWithKey](./命令行工具pwk.md)

  ## 3.3. 编译&配置

  cmc工具的编译&运行方式如下：

  > 创建工作目录 $WORKDIR 比如 ~/chainmaker
  > 启动测试链 [在工作目录下 使用脚本搭建](../tutorial/通过命令行工具启动链.html#runUseScripts)
```sh
  # 编译cmc
  $ cd $WORKDIR/chainmaker-go/tools/cmc
  $ go build
  # 配置测试数据
  $ cp -rf $WORKDIR/chainmaker-go/build/crypto-config $WORKDIR/chainmaker-go/tools/cmc/testdata/ # 使用chainmaker-cryptogen生成的测试链的密钥
  # 查看help
  $ cd $WORKDIR/chainmaker-go/tools/cmc
  $ ./cmc --help
```

  ## 3.4. 自定义配置

  cmc 依赖 sdk-go 配置文件。
  编译&配置 步骤使用的是 [SDK配置模版](https://git.chainmaker.org.cn/chainmaker/sdk-go/-/blob/master/testdata/sdk_config.yml)
  可通过修改 ~/chainmaker/chainmaker-go/tools/cmc/testdata/sdk_config_pk.yml 实现自定义配置。
  比如 `user-signkey-file-path` 参数可设置为普通用户或admin用户的私钥路径。设置后cmc将会以对应用户身份与链建立连接。
  其他详细配置项请参看 ~/chainmaker/chainmaker-go/tools/cmc/testdata/sdk_config_pk.yml 中的注解。

  ## 3.5. 功能

  TBFT共识场景下的cmc提供功能如下: 

  - [私钥管理](#keyManage)：私钥生成功能
  - [交易功能](#sendRequest)：主要包括链管理、用户合约发布、升级、吊销、冻结、调用、查询等功能
  - [查询链上数据](#queryOnChainData)：查询链上block和transaction
  - [链配置](#chainConfig)：查询及更新链配置
  - [归档&恢复功能](#archive)：将链上数据转移到独立存储上，归档后的数据具备可查询、可恢复到链上的特性
  - [gas管理](#gasManagement)：gas管理类命令，包括设置 gas admin、充值 gas 等功能
  - [地址转换](#address)：转换成指定类型的地址。支持转换成至信链等类型的地址

  [DPOS共识场景下的cmc命令](#dpos)

  ### 3.5.1. 示例（TBFT共识）

<span id="keyManage"></span>

  #### 3.5.1.1. 私钥管理

  生成私钥， 目前支持的算法有SM2、ECC_P256，未来将支持更多算法。
  **参数说明**：

  ```sh
  $ ./cmc key gen -h 
  Private key generate
  Usage:
    cmc key gen [flags]

  Flags:
    -a, --algo string specify key generate algorithm
    -h, --help 		help for gen
    -n, --name string specify storage name
    -p, --path string specify storage path
  ```

  **示例：**
  `$ ./cmc key gen -a ECC_P256 -n ca.key -p ./`

<span id="sendRequest"></span>

  #### 3.5.1.2. 交易功能

  ##### 3.5.1.2.1. 用户合约

  cmc的交易功能用来发送交易和链进行交互，主要参数说明如下：

  ```sh
    sdk配置文件flag
    --sdk-conf-path：指定cmc使用sdk的配置文件路径

    如果想覆盖sdk配置文件中的配置，则使用以下两个flag且都必填；如不传，则默认使用sdk配置文件中的配置参数
    --chain-id: 指定链Id, 会覆盖sdk配置文件读取的配置
    --user-signkey-file-path: 指定发送交易的用户sign私钥路径, 会覆盖sdk配置文件读取的配置
      
    其他flags
    --byte-code-path：指定合约的wasm文件路径
    --contract-name：指定合约名称
    --method：指定调用的合约方法名称
    --runtime-type：指定合约执行虚拟机环境，包含：GASM、EVM、WASMER、WXVM、NATIVE、DOCKER VM
    --version：指定合约的版本号，在发布和升级合约时使用
    --sync-result：指定是否同步等待交易执行结果，默认为false，如果设置为true，在发送完交易后会主动查询交易执行结果
    --params：指定发布合约或调用合约时的参数信息
    --concurrency：指定调用合约并发的go routine，用于压力测试
    --total-count-per-goroutine：指定单个go routine发送的交易数量，用于压力测试，和--concurrency配合使用
    --block-height：指定区块高度
    --tx-id：指定交易Id
    --with-rw-set：指定获取区块时是否附带读写集，默认是false
    --abi-file-path：调用evm合约时需要指定被调用合约的abi文件路径，如：--abi-file-path=./testdata/balance-evm-demo/ledger_balance.abi
  ```

   

  - 创建wasm合约

    ```sh
    $ ./cmc client contract user create \
    --contract-name=fact \
    --runtime-type=WASMER \
    --byte-code-path=./testdata/claim-wasm-demo/rust-fact-2.0.0.wasm \
    --version=1.0 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --sync-result=true \
    --params="{}"
    ```

    > 如下返回表示成功：
    > response: message:”OK” contract_result:<result:”\n\004fact\022\0031.0\030\002<\n\026wx-org1.chainmaker.org\020\001\032 F]\334,\005O\200\272\353\213\274\375nT\026%K\r\314\362\361\253X\3562\377\216\250kh\031” message:”OK” > tx_id:”991a1c00369e4b76853dadf410182bcdfc86062f8cf1478f93482ba9000191d7”
    > 注：智能合约编写参见：[智能合约开发](../dev/智能合约.html)

  - 创建evm合约

    ```sh
    $ ./cmc client contract user create \
    --contract-name=balance001 \
    --runtime-type=EVM \
    --byte-code-path=./testdata/balance-evm-demo/ledger_balance.bin \
    --version=1.0 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --sync-result=true
    ```

    > 如下返回表示成功：
    > EVM contract name in hex: 532c238cec7071ce8655aba07e50f9fb16f72ca1 response: message:”OK” contract_result:<result:”\n(532c238cec7071ce8655aba07e50f9fb16f72ca1\022\0031.0\030\005<\n\026wx-org1.chainmaker.org\020\001\032 F]\334,\005O\200\272\353\213\274\375nT\026%K\r\314\362\361\253X\3562\377\216\250kh\031” message:”OK” > tx_id:”e2af1241ff464d47b869a69ce8a615df50da57d3faff4754ad6e45b9f914b938”
    > 注：智能合约编写参见：[智能合约开发](../dev/智能合约.html)

  - 调用wasm合约

    ```sh
    $ ./cmc client contract user invoke \
    --contract-name=fact \
    --method=save \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --params="{\"file_name\":\"name007\",\"file_hash\":\"ab3456df5799b87c77e7f88\",\"time\":\"6543234\"}" \
    --sync-result=true
    ```

    > 如下返回表示成功：
    > INVOKE contract resp, [code:0]/[msg:OK]/[contractResult:gas_used:12964572 contract_event:<topic:”topic_vx” tx_id:”7c9e98befbb64cec916765d760d4def5aa26f8bac78d419c9018b8d220e7f041” contract_name:”fact” contract_version:”1.0” event_data:”ab3456df5799b87c77e7f88” event_data:”” event_data:”6543234” > ]/[txId:7c9e98befbb64cec916765d760d4def5aa26f8bac78d419c9018b8d220e7f041]

  - 调用evm合约

    evm的 –params 是一个数组json格式。如下updateBalance有两个形参第一个是uint256类型，第二个是address类型。

    10000对应第一个形参uint256的具体值，0xa166c92f4c8118905ad984919dc683a7bdb295c1对应第二个形参address的具体值。

    ```sh
    $ ./cmc client contract user invoke \
    --contract-name=balance001 \
    --method=updateBalance \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --params="[{\"uint256\": \"10000\"},{\"address\": \"0xa166c92f4c8118905ad984919dc683a7bdb295c1\"}]" \
    --sync-result=true \
    --abi-file-path=./testdata/balance-evm-demo/ledger_balance.abi
    ```

    > 如下返回表示成功：
    > EVM contract name in hex: 532c238cec7071ce8655aba07e50f9fb16f72ca1 INVOKE contract resp, [code:0]/[msg:OK]/[contractResult:result:”[]” gas_used:5888 ]/[txId:4f25f47518b14e6b92ce184dc6ed84f594341567050b4023ae1686a47e2e22ec]

  - 查询合约

    ```sh
    $ ./cmc client contract user get \
    --contract-name=fact \
    --method=find_by_file_hash \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --params="{\"file_hash\":\"ab3456df5799b87c77e7f88\"}"
    ```

    > 如下返回表示成功：
    > QUERY contract resp: message:”SUCCESS” contract_result:<result:”{“file_hash”:”ab3456df5799b87c77e7f88”,”file_name”:””,”time”:”6543234”}” gas_used:24354672 > tx_id:”25716b955ebd4a258c4bd6b6f682f1341dfe97e4bd18495c864992f1618a2003”

  - 升级合约

    ```sh
    $ ./cmc client contract user upgrade \
    --contract-name=fact \
    --runtime-type=WASMER \
    --byte-code-path=./testdata/claim-wasm-demo/rust-fact-2.0.0.wasm \
    --version=2.0 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --sync-result=true \
    --params="{}"
    ```

    > 如下返回表示成功：其中result结果为用户自定义，每个合约可能不一样，也可能没有。
    > upgrade user contract params:[] upgrade contract resp: message:”OK” contract_result:<result:”\n\004fact\022\0032.0\030\002<\n\026wx-org1.chainmaker.org\020\001\032 F]\334,\005O\200\272\353\213\274\375nT\026%K\r\314\362\361\253X\3562\377\216\250kh\031” message:”OK” > tx_id:”d89df9fcd87f4071972fdabdf3003a349250a94893fb43899eac4d68e7855d52”

  - 冻结合约

    ```sh
    $ ./cmc client contract user freeze \
    --contract-name=fact \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --sync-result=true
    ```

    > 如下返回表示成功：冻结后的合约再去执行查询、调用合约则会失败
    > freeze contract resp: message:”OK” contract_result:<result:”{“name”:”fact”,”version”:”3.0”,”runtime_type”:2,”status”:1,”creator”:{“org_id”:”wx-org1.chainmaker.org”,”member_type”:1,”member_info”:”Rl3cLAVPgLrri7z9blQWJUsNzPLxq1juKjL/jqhraBk=”}}” message:”OK” > tx_id:”09841775173548ad9a8a39e2987a4f5115d59d50dd3448e8b09a83624dee5367”

  - 解冻合约

    ```sh
    $ ./cmc client contract user unfreeze \
    --contract-name=fact \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --sync-result=true
    ```

    > 如下返回表示成功：解冻后的合约可正常使用
    > unfreeze contract resp: message:”OK” contract_result:<result:”{“name”:”fact”,”version”:”3.0”,”runtime_type”:2,”creator”:{“org_id”:”wx-org1.chainmaker.org”,”member_type”:1,”member_info”:”Rl3cLAVPgLrri7z9blQWJUsNzPLxq1juKjL/jqhraBk=”}}” message:”OK” > tx_id:”fccf024450c140dea999cc46ad24d381a679ce2142bd48b2a829abcd4f099866”

  - 吊销合约

    ```sh
    $ ./cmc client contract user revoke \
    --contract-name=fact \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --sync-result=true
    ```

    > 如下返回表示成功：吊销合约后，不可恢复，且不能对该合约执行任何操作，包括查询。
    > revoke contract resp: message:”OK” contract_result:<result:”{“name”:”fact”,”version”:”3.0”,”runtime_type”:2,”status”:2,”creator”:{“org_id”:”wx-org1.chainmaker.org”,”member_type”:1,”member_info”:”Rl3cLAVPgLrri7z9blQWJUsNzPLxq1juKjL/jqhraBk=”}}” message:”OK” > tx_id:”d971b57cf12c46ff8fe0d4f5897634c644fb802998f44360bb130f27ff54a10a”



<span id="queryOnChainData"></span>

  #### 3.5.1.3. 查询链上数据

  查询链上block和transaction 主要参数说明如下：

  ```sh
    --sdk-conf-path：指定cmc使用sdk的配置文件路径
    --chain-id：指定链Id
  ```

   

  - 根据区块高度查询链上未归档区块

    ```sh
    ./cmc query block-by-height [blockheight] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml
    ```

  - 根据区块hash查询链上未归档区块

    ```sh
    ./cmc query block-by-hash [blockhash] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml
    ```

  - 根据txid查询链上未归档区块

    ```sh
    ./cmc query block-by-txid [txid] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml
    ```

  - 根据txid查询链上未归档tx

    ```sh
    ./cmc query tx [txid] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml
    ```

   

<span id="chainConfig"></span>

  #### 3.5.1.4. 链配置

  查询及更新链配置 主要参数说明如下：

  ```sh
    sdk配置文件flag
    --sdk-conf-path：指定cmc使用sdk的配置文件路径

    admin签名者flags，此类flag的顺序及个数必须保持一致，且至少传入一个admin（当前只有管理员的添加删除需要用到管理员多签）
    --admin-key-file-paths: admin签名者的tls key文件的路径列表. 单签模式下只需要填写一个即可, 离线多签模式下多个需要用逗号分割
      
    如果想覆盖sdk配置文件中的配置，则使用以下两个flag且都必填；如不传，则默认使用sdk配置文件中的配置参数
    --chain-id: 指定链Id, 会覆盖sdk配置文件读取的配置
    --user-signkey-file-path: 指定发送交易的用户sign私钥路径, 会覆盖sdk配置文件读取的配置
      
    --block-interval: 出块时间 单位ms
    --tx-parameter-size: 交易参数最大限制 单位:MB
    --trust-root-org-id: 增加/删除/更新组织管理员公钥时指定的组织Id
    --trust-root-path: 增加/删除/更新组织管理员公钥时指定的管理员公钥文件目录
    --node-id: 增加/删除/更新共识节点Id时指定的节点Id
    --node-ids: 增加/更新共识节点Org时指定的节点Id列表
    --node-org-id: 增加/删除/更新共识节点Id,Org时指定节点的组织Id 
    --address-type: 地址类型 ChainMaker:0, 至信链:1
  ```

   

  - 查询链配置

    ```sh
    ./cmc client chainconfig query \
    --sdk-conf-path=./testdata/sdk_config_pk.yml
    ```

   

  - 更新出块时间

    ```sh
    ./cmc client chainconfig block updateblockinterval \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --block-interval 1000
    ```

  - 更新交易参数最大值限制

    ```sh
    ./cmc client chainconfig block updatetxparametersize \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --tx-parameter-size 10 
    ```

  - 添加共识节点

    ```sh
    ./cmc client chainconfig consensusnodeid add \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
    --node-org-id=public
    ```

  - 删除共识节点

    ```sh
    ./cmc client chainconfig consensusnodeid remove \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --node-id=QmXxeLkNTcvySPKMkv3FUqQgpVZ3t85KMo5E4cmcmrexrC \
    --node-org-id=public
    ```

  - 更新共识节点

    ```sh
    ./cmc client chainconfig consensusnodeid update \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --node-id=QmXxeLkNTcvySPKMkv3FUqQgpVZ3t85KMo5E4cmcmrexrC \
    --node-id-old=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
    --node-org-id=public
    ```

  - 更新共识节点Org

    ```sh
    ./cmc client chainconfig consensusnodeorg update \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --node-ids=QmbUjZPooufuTbRBxzVx546xyLpwibwsQXRPD6SwWfZMgx,QmP2GcRYFh2xQ2rh4CXsCN1d9CMxk9SktygXiwFKG487mo,QmSJGZxUSq5xXGZ68myuHcAqHsExioS8Bt87WZ3LsqRauC,QmcVEe4uVS2b6zNVKMzL57CdDhJtXey4wcktPdDv3JgyDg \
    --node-org-id=public
    ```

  - 更新管理员公钥

    ```sh
    ./cmc client chainconfig trustroot update \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --trust-root-org-id=public \
    --trust-root-path=./testdata/crypto-config/node1/admin/admin1/admin1.pem,./testdata/crypto-config/node1/admin/admin2/admin2.pem,./testdata/crypto-config/node1/admin/admin3/admin3.pem
    ```

  - 开启/关闭链的 gas 功能

    ```sh
    ./cmc client gas \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --gas-enable=true
    ```

  - 更新链的账户地址类型

    ```sh
    ./cmc client chainconfig alter-addr-type \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
    --address-type=1 \
    --sync-result=true
    ```

    

<span id="archive"></span>

  #### 3.5.1.5. 归档&恢复功能

  cmc的归档功能是指将链上数据转移到独立存储上，归档后的数据具备可查询、可恢复到链上的特性。
  为了保持数据一致性和防止误操作，cmc实现了分布式锁，同一时刻只允许一个cmc进程进行转储。
  cmc支持增量转储和恢复、断点中继转储和恢复，中途退出不影响数据一致性。

  > 注意：mysql需要设置好 sql_mode 以root用户执行 set global sql_mode = ‘ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION’;

  主要参数说明如下：

  ```sh
    --sdk-conf-path：指定cmc使用sdk的配置文件路径
    --chain-id：指定链Id
    --type：指定链下独立存储类型，如 --type=mysql 默认mysql，目前只支持mysql
    --dest：指定链下独立存储目标地址，mysql类型的格式如 --dest=user:password:localhost:port
    --target：指定转储目标区块高度，在达到这个高度后停止转储(包括这个块) --target=100 也可指定转存目标日期，转储在此日期之前的所有区块 --target="2021-06-01 15:01:41"
    --blocks：指定本次要转储的块数量，注意：对于target和blocks这两个参数，cmc会就近原则采用先符合条件的参数
    --start-block-height：指定链数据恢复时的起始区块高度，如设置为100，则从已转储并且未恢复的最大区块开始降序恢复链数据至第100区块
    --secret-key：指定密码，用于链数据转储和链数据恢复时数据一致性校验，转储和恢复时密码需要一致
  ```

   

  - 根据时间转储，将链上数据转移到独立存储上，需要权限：sdk配置文件中设置admin用户

    ```sh
    ./cmc archive dump --type=mysql \
    --dest=root:password:localhost:3306 \
    --target="2021-06-01 15:01:41" \
    --blocks=10000 \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --secret-key=mypassword
    ```

  - 根据区块高度转储，将链上数据转移到独立存储上，需要权限：sdk配置文件中设置admin用户

    ```sh
    ./cmc archive dump --type=mysql \
    --dest=root:password:localhost:3306 \
    --target=100 \
    --blocks=10000 \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --secret-key=mypassword
    ```

  - 恢复，将链下的链数据恢复到链上，需要权限：sdk配置文件中设置admin用户

    ```sh
    ./cmc archive restore --type=mysql \
    --dest=root:password:localhost:3306 \
    --start-block-height=0 \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --secret-key=mypassword
    ```

  - 根据区块高度查询链下已归档区块

    ```sh
    ./cmc archive query block-by-height [blockheight] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --type=mysql \
    --dest=root:password:localhost:3306
    ```

  - 根据区块hash查询链下已归档区块

    ```sh
    ./cmc archive query block-by-hash [blockhash] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --type=mysql \
    --dest=root:password:localhost:3306
    ```

  - 根据txid查询链下已归档区块

    ```sh
    ./cmc archive query block-by-txid [txid] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --type=mysql \
    --dest=root:password:localhost:3306
    ```

  - 根据txid查询链下已归档tx

    ```sh
    ./cmc archive query tx [txid] \
    --chain-id=chain1 \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --type=mysql \
    --dest=root:password:localhost:3306
    ```

<span id="gasManagement"></span>

  #### 3.5.1.6 gas管理

  gas管理类命令，包括设置 gas admin、充值 gas 等功能

  ```sh
    --sdk-conf-path：指定cmc使用sdk的配置文件路径
    --admin-key-file-paths：多签时，其他admin的私钥路径
  ```

  - 设置 gas admin

    ```sh
    ./cmc gas set-admin [gas账号地址。如果不传则默认设置sender为 gas admin] \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key
    ```

  - 查询 gas admin

    ```sh
    ./cmc gas get-admin --sdk-conf-path=./testdata/sdk_config_pk.yml
    ```

  - 充值 gas

    ```sh
    ./cmc gas recharge \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --address=ZX636256eae1cb42b7d2ab9bb1d202a94f8d93a555 \
    --amount=1000000
    ```

  - 查询 gas 余额

    ```sh
    ./cmc gas get-balance \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --address=ZX636256eae1cb42b7d2ab9bb1d202a94f8d93a555
    ```

  - 退还 gas

    ```sh
    ./cmc gas refund \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --address=ZX636256eae1cb42b7d2ab9bb1d202a94f8d93a555 \
    --amount=1
    ```

  - 冻结 gas 账户

    ```sh
    ./cmc gas frozen \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --address=ZX636256eae1cb42b7d2ab9bb1d202a94f8d93a555
    ```

  - 解冻 gas 账户

    ```sh
    ./cmc gas unfrozen \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --address=ZX636256eae1cb42b7d2ab9bb1d202a94f8d93a555
    ```

  - 查询 gas 账户的状态，true：账户可用，false：账户被冻结

    ```sh
    ./cmc gas account-status \
    --sdk-conf-path=./testdata/sdk_config_pk.yml \
    --address=ZX636256eae1cb42b7d2ab9bb1d202a94f8d93a555
    ```

<span id="address"></span>

  #### 3.5.1.7 地址转换

  转换成指定类型的地址。支持转换成至信链等类型的地址

  ```sh
    --address-type：指定要得到的地址的地址类型, 默认值 zxl 。所支持的地址类型如下：
    至信链: zxl
  ```

  - 公钥pem转换成指定类型的地址

    ```sh
    ./cmc address pk-to-addr ./testdata/crypto-config/node1/admin/admin1/admin1.pem \
    --address-type=zxl
    ```

  - hex转换成指定类型的地址

    ```sh
    ./cmc address hex-to-addr 3059301306072a8648ce3d020106082a811ccf5501822d034200044a4c24cf037b0c7a027e634b994a5fdbcd0faa718ce9053e3f75fcb9a865523a605aff92b5f99e728f51a924d4f18d5819c42f9b626bdf6eea911946efe7442d \
     --address-type=zxl
    ```

  - 证书pem转换成指定类型的地址

    ```sh
    ./cmc address cert-to-addr ./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.sign.crt \
     --address-type=zxl
    ```

 

<span id="dpos"></span>

### 3.5.2. 示例（DPOS共识）

#### 3.5.2.1. 私钥管理

生成私钥， 目前支持的算法有SM2、ECC_P256，未来将支持更多算法。 **参数说明**：

```
$ ./cmc key gen -h 
Private key generate
Usage:
  cmc key gen [flags]

Flags:
  -a, --algo string specify key generate algorithm
  -h, --help 		help for gen
  -n, --name string specify storage name
  -p, --path string specify storage path
```

**示例：** `$ ./cmc key gen -a ECC_P256 -n ca.key -p ./`

#### 3.5.2.2. 交易功能

##### 3.5.2.2.1. 用户合约

cmc的交易功能用来发送交易和链进行交互，主要参数说明如下：

```
  sdk配置文件flag
  --sdk-conf-path：指定cmc使用sdk的配置文件路径

  覆盖sdk配置flags，不传则使用sdk配置，如果想覆盖sdk的配置，则以下两个flag都必填
  --chain-id: 指定链Id, 会覆盖sdk配置文件读取的配置
  --user-signkey-file-path: 指定发送交易的用户sign私钥路径, 会覆盖sdk配置文件读取的配置

  其他flags
  --byte-code-path：指定合约的wasm文件路径
  --contract-name：指定合约名称
  --method：指定调用的合约方法名称
  --runtime-type：指定合约执行虚拟机环境，包含：GASM、EVM、WASMER、WXVM、NATIVE
  --version：指定合约的版本号，在发布和升级合约时使用
  --sync-result：指定是否同步等待交易执行结果，默认为false，如果设置为true，在发送完交易后会主动查询交易执行结果
  --params：指定发布合约或调用合约时的参数信息
  --concurrency：指定调用合约并发的go routine，用于压力测试
  --total-count-per-goroutine：指定单个go routine发送的交易数量，用于压力测试，和--concurrency配合使用
  --block-height：指定区块高度
  --tx-id：指定交易Id
  --with-rw-set：指定获取区块时是否附带读写集，默认是false
  --abi-file-path：调用evm合约时需要指定被调用合约的abi文件路径，如：--abi-file-path=./testdata/balance-evm-demo/ledger_balance.abi
```

- 创建wasm合约

  ```
  $ ./cmc client contract user create \
  --contract-name=fact \
  --runtime-type=WASMER \
  --byte-code-path=./testdata/claim-wasm-demo/rust-fact-2.0.0.wasm \
  --version=1.0 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --sync-result=true \
  --params="{}"
  ```

  > 如下返回表示成功： response: message:”OK” contract_result:<result:”\n\004fact\022\0031.0\030\002<\n\026wx-org1.chainmaker.org\020\001\032 F]\334,\005O\200\272\353\213\274\375nT\026%K\r\314\362\361\253X\3562\377\216\250kh\031” message:”OK” > tx_id:”991a1c00369e4b76853dadf410182bcdfc86062f8cf1478f93482ba9000191d7” 注：智能合约编写参见：[智能合约开发](https://docs.chainmaker.org.cn/dev/智能合约.html)

- 创建evm合约

  ```
  $ ./cmc client contract user create \
  --contract-name=balance001 \
  --runtime-type=EVM \
  --byte-code-path=./testdata/balance-evm-demo/ledger_balance.bin \
  --version=1.0 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --sync-result=true
  ```

  > 如下返回表示成功： EVM contract name in hex: 532c238cec7071ce8655aba07e50f9fb16f72ca1 response: message:”OK” contract_result:<result:”\n(532c238cec7071ce8655aba07e50f9fb16f72ca1\022\0031.0\030\005<\n\026wx-org1.chainmaker.org\020\001\032 F]\334,\005O\200\272\353\213\274\375nT\026%K\r\314\362\361\253X\3562\377\216\250kh\031” message:”OK” > tx_id:”e2af1241ff464d47b869a69ce8a615df50da57d3faff4754ad6e45b9f914b938” 注：智能合约编写参见：[智能合约开发](https://docs.chainmaker.org.cn/dev/智能合约.html)

- 调用wasm合约

  ```
  $ ./cmc client contract user invoke \
  --contract-name=fact \
  --method=save \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --params="{\"file_name\":\"name007\",\"file_hash\":\"ab3456df5799b87c77e7f88\",\"time\":\"6543234\"}" \
  --sync-result=true
  ```

  > 如下返回表示成功： INVOKE contract resp, [code:0]/[msg:OK]/[contractResult:gas_used:12964572 contract_event:<topic:”topic_vx” tx_id:”7c9e98befbb64cec916765d760d4def5aa26f8bac78d419c9018b8d220e7f041” contract_name:”fact” contract_version:”1.0” event_data:”ab3456df5799b87c77e7f88” event_data:”” event_data:”6543234” > ]/[txId:7c9e98befbb64cec916765d760d4def5aa26f8bac78d419c9018b8d220e7f041]

- 调用evm合约

  evm的 –params 是一个数组json格式。如下updateBalance有两个形参，第一个是uint256类型，第二个是address类型。

  10000对应第一个形参uint256的具体值，0xa166c92f4c8118905ad984919dc683a7bdb295c1对应第二个形参address的具体值。

  ```
  $ ./cmc client contract user invoke \
  --contract-name=balance001 \
  --method=updateBalance \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --params="[{\"uint256\": \"10000\"},{\"address\": \"0xa166c92f4c8118905ad984919dc683a7bdb295c1\"}]" \
  --sync-result=true \
  --abi-file-path=./testdata/balance-evm-demo/ledger_balance.abi
  ```

  > 如下返回表示成功： EVM contract name in hex: 532c238cec7071ce8655aba07e50f9fb16f72ca1 INVOKE contract resp, [code:0]/[msg:OK]/[contractResult:result:”[]” gas_used:5888 ]/[txId:4f25f47518b14e6b92ce184dc6ed84f594341567050b4023ae1686a47e2e22ec]

- 查询合约

  ```
  $ ./cmc client contract user get \
  --contract-name=fact \
  --method=find_by_file_hash \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --params="{\"file_hash\":\"ab3456df5799b87c77e7f88\"}"
  ```

  > 如下返回表示成功： QUERY contract resp: message:”SUCCESS” contract_result:<result:”{“file_hash”:”ab3456df5799b87c77e7f88”,”file_name”:””,”time”:”6543234”}” gas_used:24354672 > tx_id:”25716b955ebd4a258c4bd6b6f682f1341dfe97e4bd18495c864992f1618a2003”

- 升级合约

  ```
  $ ./cmc client contract user upgrade \
  --contract-name=fact \
  --runtime-type=WASMER \
  --byte-code-path=./testdata/claim-wasm-demo/rust-fact-2.0.0.wasm \
  --version=2.0 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
  --sync-result=true \
  --params="{}"
  ```

  > 如下返回表示成功：其中result结果为用户自定义，每个合约可能不一样，也可能没有。 upgrade user contract params:[] upgrade contract resp: message:”OK” contract_result:<result:”\n\004fact\022\0032.0\030\002<\n\026wx-org1.chainmaker.org\020\001\032 F]\334,\005O\200\272\353\213\274\375nT\026%K\r\314\362\361\253X\3562\377\216\250kh\031” message:”OK” > tx_id:”d89df9fcd87f4071972fdabdf3003a349250a94893fb43899eac4d68e7855d52”

- 冻结合约

  ```
  $ ./cmc client contract user freeze \
  --contract-name=fact \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
  --sync-result=true
  ```

  > 如下返回表示成功：冻结后的合约再去执行查询、调用合约则会失败 freeze contract resp: message:”OK” contract_result:<result:”{“name”:”fact”,”version”:”3.0”,”runtime_type”:2,”status”:1,”creator”:{“org_id”:”wx-org1.chainmaker.org”,”member_type”:1,”member_info”:”Rl3cLAVPgLrri7z9blQWJUsNzPLxq1juKjL/jqhraBk=”}}” message:”OK” > tx_id:”09841775173548ad9a8a39e2987a4f5115d59d50dd3448e8b09a83624dee5367”

- 解冻合约

  ```
  $ ./cmc client contract user unfreeze \
  --contract-name=fact \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
  --sync-result=true
  ```

  > 如下返回表示成功：解冻后的合约可正常使用 unfreeze contract resp: message:”OK” contract_result:<result:”{“name”:”fact”,”version”:”3.0”,”runtime_type”:2,”creator”:{“org_id”:”wx-org1.chainmaker.org”,”member_type”:1,”member_info”:”Rl3cLAVPgLrri7z9blQWJUsNzPLxq1juKjL/jqhraBk=”}}” message:”OK” > tx_id:”fccf024450c140dea999cc46ad24d381a679ce2142bd48b2a829abcd4f099866”

- 吊销合约

  ```
  $ ./cmc client contract user revoke \
  --contract-name=fact \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
  --sync-result=true
  ```

  > 如下返回表示成功：吊销合约后，不可恢复，且不能对该合约执行任何操作，包括查询。 revoke contract resp: message:”OK” contract_result:<result:”{“name”:”fact”,”version”:”3.0”,”runtime_type”:2,”status”:2,”creator”:{“org_id”:”wx-org1.chainmaker.org”,”member_type”:1,”member_info”:”Rl3cLAVPgLrri7z9blQWJUsNzPLxq1juKjL/jqhraBk=”}}” message:”OK” > tx_id:”d971b57cf12c46ff8fe0d4f5897634c644fb802998f44360bb130f27ff54a10a”

##### 3.5.2.2.2. 系统合约

###### 3.5.2.2.2.1. DPoS 计算地址

- 用户公钥计算出用户的地址

  ```
  $./cmc cert userAddr --pubkey-cert-path=./testdata/crypto-config/node1/user/client1/client1.pem --sdk-conf-path=./testdata/sdk_config_pk.yml
  ```

- 共识节点node-id计算

  ```
  ./cmc cert nid --node-pk-path=./testdata/crypto-config/node1/node1.pem
  ```

###### 3.5.2.2.2.2. DPoS-ERC20 系统合约

- 增发Token

  ```
  $ ./cmc client contract system mint \
  --amount=100000000 \
  --address=6CeSsjU5M62Ee3Gx9umUX6nXJoaBkWYufQdTZqEJM5di \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key \
  --sync-result=true
  ```

- 转账

  ```
  $ ./cmc client contract system transfer \
  --amount=100000000 \
  --address=6CeSsjU5M62Ee3Gx9umUX6nXJoaBkWYufQdTZqEJM5di \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询余额

  ```
  $ ./cmc client contract system balance-of \
  --address=6CeSsjU5M62Ee3Gx9umUX6nXJoaBkWYufQdTZqEJM5di \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询合约管理地址

  ```
  $ ./cmc client contract system owner \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询ERC20合约的精度

  ```
  $ ./cmc client contract system decimals \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询 Token 总供应量

  ```
  $ ./cmc client contract system total \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

###### 3.5.2.2.2.3. DPoS-Stake 系统合约

- 查询所有的候选人

  ```
  $ ./cmc client contract system all-candidates \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询指定验证人的信息

  ```
  $ ./cmc client contract system get-validator \
  --address=2W5SWDgx3kVSDmpRvHw351K4KaBx8QiUNBaTtGsFCWWH \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 抵押权益到验证人

  ```
  $ ./cmc client contract system delegate \
  --address=2W5SWDgx3kVSDmpRvHw351K4KaBx8QiUNBaTtGsFCWWH \
  --amount=2500000 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key \
  --sync-result=true
  ```

- 查询指定地址的抵押信息

  ```
  $ ./cmc client contract system get-delegations-by-address \
  --address=2W5SWDgx3kVSDmpRvHw351K4KaBx8QiUNBaTtGsFCWWH \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询指定验证人的抵押信息

  ```
  $ ./cmc client contract system get-user-delegation-by-validator \
  --delegator=ADZTrzVF9SuvQqmn9YTAJiwnLCnXonMTj6Bq1HRiwVnR \
  --validator=ADZTrzVF9SuvQqmn9YTAJiwnLCnXonMTj6Bq1HRiwVnR \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 从验证人解除抵押的权益

  ```
  $ ./cmc client contract system undelegate \
  --address=ADZTrzVF9SuvQqmn9YTAJiwnLCnXonMTj6Bq1HRiwVnR \
  --amount=100000000 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key \
  --sync-result=true
  ```

- 查询指定世代信息

  ```
  $ ./cmc client contract system read-epoch-by-id \
  --epoch-id=1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询当前世代信息

  ```
  $ ./cmc client contract system read-latest-epoch \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- Stake合约中设置验证人的NodeID

  ```
  $ ./cmc client contract system set-node-id \
  --node-id="QmWwNupMzs2GWyPXaUK3BvgvuZN74qxyz3rHaGioWDLX3D" \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key \
  --sync-result=true
  ```

- Stake合约中查询验证人的NodeID

  ```
  $ ./cmc client contract system get-node-id \
  --address=7E9czQBNz99iBfy4EDb7SUB9HxV4rQZjiXcnwBb3UFYk \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询验证人节点的最少自我抵押数量

  ```
  $ ./cmc client contract system min-self-delegation \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询世代中的验证人数

  ```
  $ ./cmc client contract system validator-number \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询世代中的区块数量

  ```
  $ ./cmc client contract system epoch-block-number \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询Stake合约的系统地址

  ```
  $ ./cmc client contract system system-address \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

- 查询收到解质押退款间隔的世代数

  ```
  $ ./cmc client contract system unbonding-epoch-number \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --chain-id=chain1 \
  --user-signkey-file-path=./testdata/crypto-config/node1/user/client1/client1.key
  ```

#### 3.5.2.3. 查询链上数据

查询链上block和transaction 主要参数说明如下：

```
  --sdk-conf-path：指定cmc使用sdk的配置文件路径
  --chain-id：指定链Id
```

- 根据区块高度查询链上未归档区块

  ```
  ./cmc query block-by-height [blockheight] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml
  ```

- 根据区块hash查询链上未归档区块

  ```
  ./cmc query block-by-hash [blockhash] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml
  ```

- 根据txid查询链上未归档区块

  ```
  ./cmc query block-by-txid [txid] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml
  ```

- 根据txid查询链上未归档tx

  ```
  ./cmc query tx [txid] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml
  ```

#### 3.5.2.4. 链配置

查询及更新链配置 主要参数说明如下：

```
  sdk配置文件flag
  --sdk-conf-path：指定cmc使用sdk的配置文件路径

  admin签名者flags，此类flag的顺序及个数必须保持一致，且至少传入一个admin（当前只有管理员的添加删除需要用到管理员多签）
  --admin-key-file-paths: admin签名者的tls key文件的路径列表. 单签模式下只需要填写一个即可, 离线多签模式下多个需要用逗号分割

  覆盖sdk配置flags，不传则使用sdk配置，如果想覆盖sdk的配置，则以下两个flag都必填
  --chain-id: 指定链Id, 会覆盖sdk配置文件读取的配置
  --user-signkey-file-path: 指定发送交易的用户sign私钥路径, 会覆盖sdk配置文件读取的配置

  --block-interval: 出块时间 单位ms
  --tx-parameter-size: 交易参数最大限制 单位:MB
  --trust-root-org-id: 增加/删除/更新组织管理员公钥时指定的组织Id
  --trust-root-path: 增加/删除/更新组织管理员公钥时指定的管理员公钥文件目录
  --node-id: 增加/删除/更新共识节点Id时指定的节点Id
  --node-ids: 增加/更新共识节点Org时指定的节点Id列表
  --node-org-id: 增加/删除/更新共识节点Id,Org时指定节点的组织Id 
```

- 查询链配置

  ```
  ./cmc client chainconfig query \
  --sdk-conf-path=./testdata/sdk_config_pk.yml
  ```

- 更新出块时间

  ```
  ./cmc client chainconfig block updateblockinterval \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
  --block-interval 1000
  ```

- 更新交易参数最大值限制

  ```
  ./cmc client chainconfig block updatetxparametersize \
  --sdk-conf-path=./testdata/sdk_config.yml \
  --admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
  --admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
  --tx-parameter-size 10 
  ```
  
- 更新管理员公钥(org-id=public)

  ```
  ./cmc client chainconfig trustroot update \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --user-signkey-file-path=./testdata/crypto-config/node1/admin/admin1/admin1.key \
  --admin-key-file-paths=./testdata/crypto-config/node1/admin/admin1/admin1.key,./testdata/crypto-config/node1/admin/admin2/admin2.key,./testdata/crypto-config/node1/admin/admin3/admin3.key \
  --trust-root-org-id=public \
  --trust-root-path=./testdata/crypto-config/node1/admin/admin1/admin1.pem,./testdata/crypto-config/node1/admin/admin2/admin2.pem,./testdata/crypto-config/node1/admin/admin3/admin3.pem
  ```

#### 3.5.2.5. 归档&恢复功能

cmc的归档功能是指将链上数据转移到独立存储上，归档后的数据具备可查询、可恢复到链上的特性。 为了保持数据一致性和防止误操作，cmc实现了分布式锁，同一时刻只允许一个cmc进程进行转储。 cmc支持增量转储和恢复、断点中继转储和恢复，中途退出不影响数据一致性。

> 注意：mysql需要设置好 sql_mode 以root用户执行 set global sql_mode = ‘ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION’;

主要参数说明如下：

```
  --sdk-conf-path：指定cmc使用sdk的配置文件路径
  --chain-id：指定链Id
  --type：指定链下独立存储类型，如 --type=mysql 默认mysql，目前只支持mysql
  --dest：指定链下独立存储目标地址，mysql类型的格式如 --dest=user:password:localhost:port
  --target：指定转储目标区块高度，在达到这个高度后停止转储(包括这个块) --target=100 也可指定转存目标日期，转储在此日期之前的所有区块 --target="2021-06-01 15:01:41"
  --blocks：指定本次要转储的块数量，注意：对于target和blocks这两个参数，cmc会就近原则采用先符合条件的参数
  --start-block-height：指定链数据恢复时的起始区块高度，如设置为100，则从已转储并且未恢复的最大区块开始降序恢复链数据至第100区块
  --secret-key：指定密码，用于链数据转储和链数据恢复时数据一致性校验，转储和恢复时密码需要一致
```

- 根据时间转储，将链上数据转移到独立存储上，需要权限：sdk配置文件中设置admin用户

  ```
  ./cmc archive dump --type=mysql \
  --dest=root:password:localhost:3306 \
  --target="2021-06-01 15:01:41" \
  --blocks=10000 \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --secret-key=mypassword
  ```

- 根据区块高度转储，将链上数据转移到独立存储上，需要权限：sdk配置文件中设置admin用户

  ```
  ./cmc archive dump --type=mysql \
  --dest=root:password:localhost:3306 \
  --target=100 \
  --blocks=10000 \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --secret-key=mypassword
  ```

- 恢复，将链下的链数据恢复到链上，需要权限：sdk配置文件中设置admin用户

  ```
  ./cmc archive restore --type=mysql \
  --dest=root:password:localhost:3306 \
  --start-block-height=0 \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --secret-key=mypassword
  ```

- 根据区块高度查询链下已归档区块

  ```
  ./cmc archive query block-by-height [blockheight] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --type=mysql \
  --dest=root:password:localhost:3306
  ```

- 根据区块hash查询链下已归档区块

  ```
  ./cmc archive query block-by-hash [blockhash] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --type=mysql \
  --dest=root:password:localhost:3306
  ```

- 根据txid查询链下已归档区块

  ```
  ./cmc archive query block-by-txid [txid] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --type=mysql \
  --dest=root:password:localhost:3306
  ```

- 根据txid查询链下已归档tx

  ```
  ./cmc archive query tx [txid] \
  --chain-id=chain1 \
  --sdk-conf-path=./testdata/sdk_config_pk.yml \
  --type=mysql \
  --dest=root:password:localhost:3306
  ```



