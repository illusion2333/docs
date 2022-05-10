
# FAQ

## 部署编译相关

### unknown revision v2.0.0

> go: chainmaker.org/chainmaker/pb-go@v2.0.0+incompatible: reading chainmaker.org/chainmker/pb-go/go.mod at revision v2.0.0: unknown revision v2.0.0

1、可能chainmaker.org域名可能记录为其他主机地址了，删除 `~/.ssh/known_hosts文件` 重试即可。
2、可能代理无法访问chainmaker.org，请切换代理尝试`go env -w GOPROXY=https://goproxy.cn,direct`

### misssing go.sum entry

到对应目录执行`go mod download` 或者 `go mod tidy`

### not found GLIBC_2.18

缺少glibc的库，在linux下可进入chainmaker-go/scripts/3rd目录安装glibc-2.18.tar.gz依赖

```sh
# 注：此操作为安装替换GCC版本，请慎重操作。一旦出错系统将不可用。
cd scripts/3rd
sh install.sh
```

### restart.sh 权限不足

> ./restart.sh: Permission denied

给脚本添加可执行权限

```sh
cd chainmaker-go/script/bin
chmod +x *.sh
```

### syscall/js.valueGet not exported

>执行gasm合约时报错：resolve imports: syscall/js.valueGet not exported in module env

tinygo不支持fmt等函数，tinygo支持的包参考：https://tinygo.org/lang-support/stdlib/

### runtime type error | byte code validation failed

> 发送交易成功，但链打印错误信息：contract invoke failed, runtime type error, expect rust:[2], but got 4。同时根据该交易id查询到交易错误信息。
> failed to create vm runtime, contract: contract_test, [contract_test], byte code validation failed

执行交易时异步的（查询类交易除外），返回的状态为链成功接收到交易的状态。执行合约是，runtimeType选择错误，需要根据自己的合约语言选择对应的runtimeType。
byte code validation failed：可能原因：1、运行类型错误；2、wasm文件损坏；3、wasm文件非官网指定渠道编译

| 语言     | 类型                      |
| :------- | ------------------------- |
| 系统合约 | RuntimeType_NATIVE = 1    |
| rust     | RuntimeType_WASMER = 2    |
| c++      | RuntimeType_WXVM = 3      |
| tinygo   | RuntimeType_GASM = 4      |
| solidity | RuntimeType_EVM = 5       |
| golang   | RuntimeType_DOCKER_GO = 6 |

### 返回成功，但实际执行失败

>使用sdk、cmc执行安装、调用合约时，SDK 返回message为ok，但链和交易显示执行失败

交易的执行是异步的。SDK返回的成功信息指的是链成功接收到该交易。
获取查看交易实际结果的方式：

- 根据txId查询该交易，解析出结果。
- 使用SDK是选择同步发送交易，等待执行结果。

### 出块标记是什么

>进入log目录，查看日志文件 筛选  `put block` 即可
> `cat system.log|grep "ERROR\|put block"` 
>其中一行解释如下：
>2021-11-04 15:55:06.351 [INFO] [Storage] @chain1 blockstore_impl.go:363 chain[chain1]: put block[1] (txs:1 bytes:15946), time used (mashal:0, log:1, blockdb:0, statedb:0, historydb:0, resultdb:0, contractdb:0, batchChan:0, total:1)
>时间 [日志级别] [模块] @链名称 文件名.go:行数 链chain[链名称]:put block[区块高度](txs:交易个数 bytes:区块大小), 使用时间毫秒(mashal:0, log:1, blockdb:0, statedb:0, historydb:0, resultdb:0, contractdb:0, batchChan:0, total:1)

### 组网成功标记是什么

组网成功后，即可发送交易。此时接收到的交易将进入到交易池当中，并且会广播给网络的每一个节点（共识、同步节点、轻节点），随后等待共识成功选举leader开始打包区块。
启动成功日志： `init blockchain[chain1] success` 
多节点时，组网成日志： `all necessary peers connected`

### 如何查看合约内的日志

1、修改log.yml配置文件将vm的级别调整为debug
2、重启该节点
3、发起交易，即可查看到以后的日志
如下：

```sh
vim chainmaker-v2.2.0-wx-org1.chainmaker.org/config/wx-org1.chainmaker.org/log.yml

log:
  system: # 链日志配置
    log_level_default: INFO       # 默认日志级别
    log_levels:
      core: INFO                  # 查看commit block落快信息关键字，需将core改为info级别及以下
      net: INFO
      vm: DEBUG                   # 合约中的日志，需将vm改为debug级别
      storage: INFO               # sql模式查看sql语句，需将storage改为debug级别
```

### 节点的node_id如何生成

可使用cmc工具可获取nodeid: ./cmc cert nid -h，是对证书的公钥进行SHA2_256，再base58编码后形成nodeid




## P2P网络相关（包含libp2p,liquid）

## 链账户身份与权限相关（证书问题、public、多签投票问题）

### PermissionedWithCert、PermissionedWithKey、Public 模式的区别是什么？

长安链的用户标识体系分为以下两大类：

1. 基于数字证书的用户标识体系——PermissionedWithCert 即证书模式，为长安链的默认用户标识体系。通常适用于权限控制要求较高的应用场景。
2. 基于公钥的用户标识体系：
   - PermissionedWithKey，该模式下进行管理操作时需要权限验证，通常适用于权限控制要求较高的应用场景。
   - Public 该模式下进行管理操作时不需要权限验证，通常适用于权限控制要求不高的应用场景。

### 一条链是否支持同时使用PermissionedWithCert、PermissionedWithKey、Public等多个模式？

暂不支持，某条链只能选择其中一种模式。

### 长安链的组织证书是用来干嘛的？

长安链的组织证书即是配置trust_root里面的证书，用来验证交易发起者或链参与者是否为该链的联盟成员。trust_root中可以配置组织根证书或组织中间证书。建议使用组织中间证书，以免根证书遗失或不慎泄露造成的不便。

### 长安链的节点证书是用来干嘛的？

长安链的节点证书分为两类。一类是tls证书，一类是sign证书。tls证书用于跟客户端建立tls链接以及节点间通信。sign证书用于签名验签等，通常在共识投票过程中使用。上述证书均需通过`CA证书` 签发获得。
通过建链脚本生成的节点证书为consensus和common两套，均包括上述tls和sign证书。其中，配置使用的是consensus，而common作为预留。

### 长安链的用户证书是用来干嘛的？

长安链的用户证书从角色上分为admin、client和light三类。

- admin角色证书，通常称为管理员证书，该证书拥有对系统合约（管理类）调用交易的签名投票权限。链配置中的权限配置项，默认为admin角色。
- client角色证书，通常称为普通用户证书，该证书拥有用户合约（普通类）调用交易和查询类交易的操作权限。
- light角色证书，通
  上述每种角色的用户从用途上分为tls和sign两种，的tls证书和sign证书的主要作用是
- 用户tls证书主要用户跟节点建立tls链接。
- 用户sign证书主要用户签名验签。

### 如何申请证书？

证书包括

- CA 证书
- 节点证书
  -  共识节点的TLS 证书和sign证书  
  -  同步节点的TLS证书和sign证书
  -  轻节点的TLS证书和sign证书
- 用户证书
  - admin 证书的TLS证书和 sign 证书
  - client 证书的TLS证书和 sign 证书

以上证书均可以通过 [chainmaker-crypogen](https://docs.chainmaker.org.cn/dev/%E8%AF%81%E4%B9%A6%E7%94%9F%E6%88%90%E5%B7%A5%E5%85%B7.html) 或者[自建 CA 证书服务生成](https://docs.chainmaker.org.cn/operation/CA%E8%AF%81%E4%B9%A6%E6%9C%8D%E5%8A%A1.html)


### 不同组织间有没有共同的ca，证书的组织和org_id有什么联系？

不同组织间的CA证书可以使用同一个。但是不建议这样做，建议是一个组织一个CA证书。
证书的组织字段和trust_roots的org_id字段，无强制联系。

```
# 各组织不同CA配置
trust_roots:
  - org_id: "wx-org1.chainmaker.org"
    root: "ca1.crt"
  - org_id: "wx-org2.chainmaker.org"
    root: "ca2.crt"
  - org_id: "wx-org3.chainmaker.org"
    root: "ca3.crt"
    
# 各组织相同CA配置
trust_roots:
  - org_id: "wx-org1.chainmaker.org"
    root: "ca1.crt"
  - org_id: "wx-org2.chainmaker.org"
    root: "ca1.crt"
  - org_id: "wx-org3.chainmaker.org"
    root: "ca1.crt"
```

### 一个组织的证书申请数量是否有有上限？

理论上没有上限

### 组织间的数据能否实现哪些数据可以公开给对方，哪些数据不能公开给对方

上链数据均共享。可以根据场景需要，采用混合加密、分层身份加密、同态加密、零知识证明等方式保护数据隐私。

### 是否支持外部证书，外部证书和长安链证书在使用上有什么差异点？

支持外部证书。目前长安链的用户、节点的签名证书（即业务证书）支持使用BJCA、CFCA等国家认可的第三方CA颁发的外部证书。因TLS证书仅作用于通信层，故无需使用外部证书。[外部证书参考](https://docs.chainmaker.org.cn/operation/%E5%A4%96%E9%83%A8%E8%AF%81%E4%B9%A6%E5%85%BC%E5%AE%B9%E9%85%8D%E7%BD%AE%E6%89%8B%E5%86%8C.html)
外部证书和长安链证书的差别是，外部证书需要在链的外部进行签名，然后链拿到签名，和外部证书的私钥进行匹配。需要和外部进行通信，长安链的证书可以不经过外部调用，自己内部可以生成证书。

### 证书的有效期为多久？

用x509将证书解析出来。
例如：openssl x509 -in ca-sign.crt -noout -text

``` 
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 464217 (0x71559)
        Signature Algorithm: 1.2.156.10197.1.501
        Issuer: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = root-cert, CN = ca.wx-org1.chainmaker.org
        Validity
            Not Before: Feb 11 03:08:18 2022 GMT
            Not After : Feb  9 03:08:18 2032 GMT
        Subject: C = CN, ST = Beijing, L = Beijing, O = wx-org1.chainmaker.org, OU = root-cert, CN = ca.wx-org1.chainmaker.org
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:b3:8d:35:b6:3c:d1:2e:1e:be:ee:b4:d5:19:e1:
                    fc:f4:f3:99:d0:51:1e:29:e4:c5:c1:b7:36:cf:75:
                    6a:63:66:a2:83:a3:0e:9a:a6:ea:35:8c:85:87:11:
                    68:d7:52:8f:a3:08:b1:c5:81:36:d1:e0:49:99:32:
                    37:b4:71:d3:cd
                ASN1 OID: SM2
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier: 
                AE:F3:05:93:10:85:6E:EC:94:47:0E:42:54:35:46:1A:7E:03:D8:57:63:31:E7:D9:84:42:7C:CF:7C:93:8B:9B
            X509v3 Subject Alternative Name: 
                DNS:chainmaker.org, DNS:localhost, DNS:ca.wx-org1.chainmaker.org, IP Address:127.0.0.1
    Signature Algorithm: 1.2.156.10197.1.501
         30:45:02:21:00:b6:1c:57:ae:10:93:61:a2:1e:a3:4d:08:6a:
         f3:4e:b0:e6:99:ee:f8:22:de:7f:4b:0d:dc:1b:4b:b9:63:88:
         2e:02:20:1e:ba:ac:2a:e0:14:fc:43:77:cc:92:ff:6f:d4:8b:
         3a:3f:02:19:0e:b3:af:07:da:bf:68:3e:fe:6c:a3:33:66

```

如下。位于两个时间段之内的都是在有效期内。有效期可以在生成的时候指定

``` 
Validity
            Not Before: Feb 11 03:08:18 2022 GMT
            Not After : Feb  9 03:08:18 2032 GMT
```

### 证书支持哪些管理，如何管理证书？

证书的管理支持：查询证书，添加证书，查询证书，生成crl列表，冻结证书，解除冻结证书，吊销证书。可以参考 go 的 sdk 来进行证书管理操作，[见文档](https://docs.chainmaker.org.cn/dev/chainmaker-go-sdk.html#id60) 长安链提供了也cmc工具管理证书：cmc目前支持，生成crl列表，冻结证书，解除冻结证书，吊销证书等功能。
cmc命令参考如下
1.生成crl(Certificate Revocation List)列表

```
$ ./cmc cert crl -C ./ca.crt -K ca.key --crl-path=./client1.crl --crt-path=../sdk/testdata/crypto-config/wx-org2.chainmaker.org/user/client1/client1.tls.crt
```

2.冻结证书

```
$ ./cmc client certmanage freeze \
--sdk-conf-path=./testdata/sdk_config.yml \
--cert-crl-path=./client1.crl \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key
```

3.解除冻结证书

```
$ ./cmc client certmanage unfreeze \
--sdk-conf-path=./testdata/sdk_config.yml \
--cert-crl-path=./client1.crl \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key
```

4.吊销证书

```
$ ./cmc client certmanage revoke \
--sdk-conf-path=./testdata/sdk_config.yml \
--cert-crl-path=./client1.crl \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key
```

[通过cmc命令工具进行管理证书](https://docs.chainmaker.org.cn/dev/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7.html#certManage)

### TLS证书是否一定需求，还是可以按需选择？

起链的时候可以选择是否要开启TLS，如果开启则需要，如果不开启则不需要。
chainmaker.yml  **rpc** 支持 net 不支持

### TLS怎么开启和关闭？

参考 chainmaker.yml 的配置rpc的 tls 配置

```
# RPC TLS settings
  tls:
    # TLS mode, can be disable, oneway, twoway.
    mode:           disable

    # RPC TLS private key file path
    priv_key_file:  ../config/wx-org1.chainmaker.org/certs/node/consensus1/consensus1.tls.key

    # RPC TLS public key file path
    cert_file:      ../config/wx-org1.chainmaker.org/certs/node/consensus1/consensus1.tls.crt
```

tls 关闭时，mode的值为 disable。
tls 开启时，mode的值可以是 oneway 或 twoway。

- oneway 模式即单向认证，客户端保存着rpc的tls证书并信任该证书即可使用。
- twoway 模式即双向认证，用户企业应用。双向认证模式先决条件是有两个或两个以上的证书，一个是rpc的tls证书，另一个或多个是客户端证书。长安链上保存着客户端的证书并信任该证书，客户端保存着长安链的证书并信任该证书。这样，在证书验证成功时可以完成链的相关操作。

### 长安链是否支持国密TLS,如果支持，再哪里可以启用？

支持国密。可以在生成tls证书的时候选用国密方式，tls证书接入到长安链上，长安链会根据tls的加密方式匹配到国密的加密方式，无需用户多余的操作。
长安链支持用 chainmaker-cryptogen 工具生成tls证书
需要修改配置 pk_algo: sm2。配置文件参考

```yaml
crypto_config:
  - domain: chainmaker.org
    host_name: wx-org
    count: 4                # 如果为1，直接使用host_name，否则添加递增编号
		pk_algo: sm2
    ski_hash: sha256
    ## pkcs11配置
```

[参考文档](https://docs.chainmaker.org.cn/operation/%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E4%B8%80%E8%A7%88.html?highlight=cryptogen#chainmaker-cryptogen)

### 链权限是否支持动态配置，如何配置？

链权限是基于系统合约去修改的，可以动态配置，只要符合多签的要求，配置块落块，新配置就会生效。

### 如何把一个组织踢出网络？

删除组织根证书。即删除trustroot证书。cmc命令参考

```
./cmc client chainconfig trustroot remove \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--trust-root-org-id=wx-org5.chainmaker.org \
--trust-root-path=./testdata/crypto-config/wx-org5.chainmaker.org/ca/ca.crt
```

### 组织被踢出网络后，对组织下的节点和用户会有什么影响？

该组织下的节点不能入网参与共识，用户不能发起交易。

### 如何把共识节点降级为同步节点？

长安链把共识节点降级为同步节点，只需要在链上将该共识节点的nodeId删除。
删除共识节点nodeid的cmc命令参考

```
./cmc client chainconfig consensusnodeid remove \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org1.chainmaker.org
```

### 如何把一个共识节点踢出网络？

#### BFT类(TBFT、HotStuff)共识、RAFT共识

- 1.[使用cmc删除共识节点](https://docs.chainmaker.org.cn/dev/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7.html#chainConfig.delConsensusNodeId)
- 2.停止节点程序
  `$ kill -15 <节点程序pid>`

#### DPoS共识

- 1.[查询网络验证人节点的最少抵押数量要求](https://docs.chainmaker.org.cn/dev/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7.html#chainConfig.dposMinSelfDelegation)
- 2.[查询验证人数据](https://docs.chainmaker.org.cn/dev/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7.html#chainConfig.dposValidatorInfo)
- 3.[解除共识节点的抵押](https://docs.chainmaker.org.cn/dev/%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7.html#chainConfig.dposUndelegate)
- 4.停止节点程序
  `$ kill -15 <节点程序pid>`

### 如何把一个同步节点升级为共识节点？

1.停止当前的同步节点

```
$ kill -15 <节点程序pid>
```

2.用ca证书签发一套共识节点的证书
可以用长按链提供的正式生成工具[chainmaker-cryptogen](https://docs.chainmaker.org.cn/dev/%E8%AF%81%E4%B9%A6%E7%94%9F%E6%88%90%E5%B7%A5%E5%85%B7.html)
3.添加共识节点Id
在该节点基础上增加一个 nodeid命令如下

```
./cmc client chainconfig consensusnodeid add \
--sdk-conf-path=./testdata/sdk_config.yml \
--org-id=wx-org1.chainmaker.org \
--user-tlscrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.crt \
--user-tlskey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.tls.key \
--user-signcrt-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.crt \
--user-signkey-file-path=./testdata/crypto-config/wx-org1.chainmaker.org/user/client1/client1.sign.key \
--admin-crt-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.crt,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.crt \
--admin-key-file-paths=./testdata/crypto-config/wx-org1.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org2.chainmaker.org/user/admin1/admin1.tls.key,./testdata/crypto-config/wx-org3.chainmaker.org/user/admin1/admin1.tls.key \
--node-id=QmcQHCuAXaFkbcsPUj7e37hXXfZ9DdN7bozseo5oX4qiC4 \
--node-org-id=wx-org5.chainmaker.org
```

## 核心交易引擎相关（交易池、DAG）

## 共识相关

## 智能合约相关

### 智能合约支持什么语言？

智能合约目前支持Go、Solidity、RUST、C++、TinyGo

### 合约开发建议采用哪种开发语言？建议使用的开发工具是什么？

建议开发语言：rust，合约内可以引用大多数外部依赖（如：含随机数的不可用）。 建议开发工具：vscode，+ 插件：rust-analyzer

### 智能合约中PutState有两个参数是什么意思？

```
func PutState(key string, field string, value string) ResultCode
```

实际存储到leveldb的key为：contractName + “#” + key + field
长度限制： key:64、 field:64、 value:1M
且key、field符合正则 `^[a-zA-Z0-9._-]+$` ，只能为英文字母、数字、点、横杠、下划线
两个参数的原因：一个逻辑性的命名空间概念，key作为namespace，一般为有规律的值。

### 合约代码是否需要每个组织节点都部署？

合约代码部署也是一个交易。发送给某个节点后，该节点会把交易广播到自己的网络中。其他节点也就有了这个交易了。交易上链需要各个节点达成共识，其他共识节点也会执行该交易。

## 存储相关

## SDK相关

### build失败，错误：找不到符号 import sun.security.ec.CurveDB;

CurveDB是用来实现国密tls的依赖包。jdk 在1.8的不同版本中该包位置有变动，较低版本为sun.security.ec.CurveDB，
较高版本为sun.security.util.CurveDB，请自行替换。maven官方仓库为较低版本sun.security.ec.CurveDB。

### 发送交易sendTxRequest失败：connect to peer error 

与节点建立连接失败，可能由以下原因导致：

    1. 链节点端口未开放
    2. 链节点端口配置错误
    3. tls证书和私钥配置错误，及sdk_config.yml中的user_key_file_path和user_crt_file_path配置，需和链用户保持一致
    4. 根证书路径trust_root_paths配置错误，该配置值为证书路径
    5. 链开启tls，而sdk关闭tls，或者相反。总之原则就是同时开启或者同时关闭

### 发送交易异常：tx verify failed, verify tx authentation failed, authentication error: authentication failed, [refine endorsements failed, all endorsers have failed verification]

交易发送者用户身份错误，具体表现为签名证书配置错误，sdk_config.yml中的user_sign_key_file_path和user_sign_crt_file_path配置，需和链用户保持一致

### 创建合约异常（多签异常）：authentication fail: not enough participants support this action: 3 valid endorsements required, 2 valid endorsements received

创建合约需线下多签，多签用户不足，或多签用户证书配置错误，java sdk test中的多签用户为TestBase中的adminUser1、adminUser2、adminUser3，需为这个三个用户配置正确的证书和私钥

###  依赖sdk jar包须知

应用项目中，使用sdk jar包可通过maven仓库直接依赖，需注意一下几点：

    1. 需将sdk代码/lib目录中提供的netty-tcnative-openssl-static-2.0.39.Final.jar包引入到项目中
    2. 需将sdk的配置文件放在应用项目中
    3. windows系统需将动态库文件 libcrypto-1_1-x64.dll 和 libssl-1_1-x64.dll 放到项目中的resources/win32-x86-64目录中，可参考skd代码。本机需安装openssl。

## 长安链CMC工具

## 长安链管理台

### 订阅链失败

如提示"订阅链失败，与节点连接失败，请检查节点端口是否开放"，需检查以下几点:

    1. 链节点端口是否开放
    
    2. 所订阅的链的节点部署的ip和端口是否和生成节点证书时填写的一致;导入外部证书时，填写的ip和端口是否与已启动的链一致。
    
    3. 链是否为容器启动，容器启动的链不能用容器启动的管理台订阅，因为网络是不通的。
    
    4. 外部导入的tls证书是否正确
    
    5. 申请和导入节点证书时，ip地址不能填127.0.0.1

如提示"订阅链失败，证书错误，请检查用户证书是否正确"，需检查以下几点:

    1. 一般导入外部链容易出现，请检查是否全部证书导入正常，是否和链使用的证书为同一套。

如提示"订阅链失败，tls握手失败，请检查证书是否正确"，需检查以下几点:

    1. 一般导入外部链容易出现，请检查TLS证书是否正确导入，是否是订阅组织下的用户。

如提示"订阅链失败，chainId错误，请检查chainId是否正确"，需检查以下几点:

    1. 请检查订阅的输入chainid是否和链一致。

### 管理平台区块高度落后于链

一般由管理平台订阅中断导致，请重新订阅。订阅中断可能由以下原因引起：

    1. 链节点关闭或重启
    2. 链节点所在机器磁盘空间不足
    3. 链节点出现异常，检查日志
    4. 导入已经部署的2.1.0版本链，并使用docker-go合约，会导致订阅失败

###  管理平台上链交易失败

一般由合约问题和合约参数问题导致，检查合约和参数是否正常，检查链是否正常运行

### 管理平台上链成功后，交易id不存在或不能跳转

管理平台调用合约后，会同步查询该交易txID，确认是否成功出块，如果查不到会导致该问题，可能由以下原因导致

    1. 链异常，检查链日志和panic日志
    2. 链返回数据大，导致超时

### 管理平台部署合约为什么要投票

部署合约需按照对应策略进行线下多签，才能正常部署到链上。
  目前策略分别有MAJORITY、ANY、ALL，分别对应超半数组织投票，任意组织投票和所有组织投票。可通过sdk或者cmc工具修改多签策略

### 投票失败

检查投票组织下是否有用户，如果没有请申请用户证书，如果是导入的组织，请导入该组织下用户证书

### 投票完成后，合约部署失败

合约部署失败可能由以下几个原因引起：

    1. 链异常，检查链日志和panic日志
    2. 链返回数据大，导致超时
    3. 合约或者传参有问题

### 管理平台的浏览器合约执行结果乱码

合约执行结果是由合约内容控制，需自己手写合约。系统合约执行是乱码，比如部署用户合约，实际是调用了系统合约，该交易执行结果为乱码

### 怎么修改节点ip和端口

目前还不支持修改

## 长安链浏览器

### 订阅链失败

需检查以下几点:
    

    1. 链节点端口是否开放
    2. 所订阅的链的节点部署的ip和端口是否与已启动的链一致。
    3. 链是否为容器启动，容器启动的链不能用容器启动的浏览器订阅，因为网络是不通的。
    4. tls证书是否正确
    5. 节点ip地址不能填127.0.0.1

## 长安链合约IDE

## 长安链web签名插件

## 跨链相关

## 轻节点相关

## 隐私计算相关

### 长安链在隐私计算方面的能力

长安链目前支持同态加密、零知识证明、层级加密等算法，并基于隐私合约方案在长安链上原生支持基于TEE的硬件可信计算环境方案，后续还会逐渐丰富扩展。

## 密码学相关

## 环境依赖

## 其他

### 长安链的性能表现
TPS能达到10万级，并获得了信通院可信区块链联盟测试报告。

### 长安链TPS的测试方法
长安链TPS目前已通过权威外部测评，获得了信通院可信区块链联盟测试报告。基于社区版本，我们的测试软硬件配置信息如下：32核CPU，64G内存，SSD硬盘，万兆网络，在TBFT共识、国密算法、1K存证数据下进行测试。