# HIBE 教程

本文分为两部分：

1. 方案接口、工具介绍：对 ChainMaker·长安链的 HIBE SDK 接口、cmc 工具和 common 库提供的算法的介绍。
2. 用例：介绍了如何使用 ChainMaker·长安链 HIBE 特性，从 HIBE 系统初始化、编写智能合约，到使用 SDK（本文使用 go 的 SDK） 进行数据的加密上链、获取链上数据解密。



## 接口、工具介绍



### 生成工具 CMC 子命令——hibe 介绍

#### 简介

CMC 中的 hibe 命令是管理、使用HIBE身份层级加密的辅助工具，用于初始化一个公司的系统层级，生成`params`、`MasterKey`、根据参数生成成员的`privateKey`，并保存到本地文件，由上司去分发。

使用命令`./cmc -hibe -h` 查看命令该子命令详细提示：

```sh
./cmc hibe -h
ChainMaker hibe command

Usage:
  cmc hibe [command]

Available Commands:
  genPrvKey   generates a key for an Id using the master key
  getParams   getParams storage path
  init        setup generates the system parameters

Flags:
  -h, --help   help for hibe

Use "cmc hibe [command] --help" for more information about a command.
```

#### init简介

`init`用于初始化一个系统层级结构。

查看命令详情`./ cmc hibe init -h`：

```sh
./cmc hibe init -h
setup generates the system parameters

Usage:
  cmc hibe init [flags]

Flags:
  -h, --help           help for init
  -l, --level string   the parameter "l" is the maxi depth that the hierarchy will support.
  -o, --orgId string   the result storage name, please enter your orgId
  -s, --spath string   the result storage path, include org's params、MasterKey
```

`init`参数详解：

- ` -h, --help`：输出命令提示信息
- `-l, --level`：构造的系统层级支持的最大深度，接受一个字符串，内部转成int，请输入合法的数字字符串
- `-o, --orgId`：构造的组织或公司的 `orgId`
- `-s, --spath`：初始化之后返回的`params`、`masterKey` 存储路径



#### getParams简介

`getParams`用于获取初始化后`Params`文件的具体存储路径并打印`Params`，要求输入组织名和设置的根目录。

`./cmc hibe getParams -h`查看命令详情：

```sh
./cmc hibe getParams -h
getParams storage path

Usage:
  cmc hibe getParams [flags]

Flags:
  -h, --help           help for getParams
  -o, --orgId string   the result storage name, please enter your orgId
  -p, --path string    the init path
```

`getParams`参数详解：

- `-h, --help`：查看命令详细信息
- `-o, --orgId`：组织名
- `-p, --path`：初始化时的根路径



#### genPrivKey简介

`genPrivKey`为下属生成`privateKey`，存储在本地文件，由生成者颁发私钥文件给下属。

该命令由两种方式，基于`parentKey`和`masterKey`，由`flag`控制。

查看该命令详细提示：

```sh
./cmc hibe genPrvKey -h
generates a key for an Id using the master key

Usage:
  cmc hibe genPrvKey [flags]

Flags:
  -m, --fromMaster int   generate prvKey from masterKey or privateKey, 1 from master, 0 from parent, m default is 0
  -h, --help             help for genPrvKey
  -i, --id string        get the private key of the ID, Must be formatted in the sample format with" / ", for example: id org1/ou1/Alice
  -k, --kpath string     the masterKey Or parentKey file path
  -o, --orgId string     the result storage name, please enter your orgId
  -p, --ppath string     the hibe params file's path
  -s, --spath string     the result storage file path, and the file name is the id
```

`genPrivKey`参数详解：

- `-h, --help`：查看命令详细信息
- `-m, --fromMaster`：传入`1`和`0`，`1`标识根据`MasterKey`生成私钥，`-k, --kpath`中就要提供`MasterKey`的存储路径 ，`0`标识根据`ParentKey`生成私钥，`-k`中提供上级私钥的存储路径，默认值为 `0`，根据`ParentKey`生成私钥
- `-i, --id`：当前生成私钥的`Id`
- `-o, --orgId`: 指定组织`orgId`，会根据`orgId`分文件夹保存生成的私钥，方便管理
- `-k, --kpath`：`MasterKey`或者`ParentKey`的存储路径
- `-p, --ppath`：`params` 存储路径
- `-s, --spath`：生成的私钥存储的根路径（私钥会在根路径中根据组织`Id`创建指定文件夹，几种管理每个公司的私钥，私钥文件以*ID.privatekey*命名）



### `common`提供的加密和解密方法

#### 加密，并封装信息的方法

**参数介绍：**

- `plaintext`：待加密明文信息
- `receiverIds`: 消息接收者对应的加密参数，需和`paramsList` 一一对应
- `paramsList`: 与`receiverIds`对应的`hibe`系统参数
- `keyType`: 对明文进行对称加密的方法，请传入 `common` 项目中 `crypto` 包提供的方法，目前提供`AES`和`SM4`两种方法，key长度已在内部指定统一使用 `128bit` 长度类型的

```go
func EncryptHibeMsg(plaintext []byte, receiverIds []string, paramsList []*hibe.Params,
                    symKeyType crypto.KeyType) (map[string]string, error)
```

#### 解密，返回原始信息的字节数组的方法

**参数介绍：**

- `localId`: 本地身份分层加密 `id`
- `hibeParams`: 本地身份分层加密系统参数
- `privKey`: 本地身份分层加密私钥
- `hibeMsgMap`: 身份分层加密交易的链上获取信息
- `keyType`: 对加密信息进行对称解密的方法，请和加密时使用的方法保持一致，请传入 `common` 中 `crypto` 包提供的方法，目前提供`AES`和`SM4`两种方法，`key`长度已在内部指定统一使用`128bit`长度类型的

```go
func DecryptHibeMsg(localId string, hibeParams *hibe.Params, prvKey *hibe.PrivateKey,
                    hibeMsgMap map[string]string, symKeyType crypto.KeyType) ([]byte, error)
```



### SDK

身份分层加密类接口

> 注意：身份分层加密模块 `Id` 使用 `/` 作为分隔符，例如： `Org1/Ou1/Member1`
> 身份 `Id` 中禁用符号 `#` ，`/`在文件名中无法使用，我们采用 `#` 作为保存私钥的文件名 `Id`分隔符，替换掉 `/`，使用 `#` 做分隔符在使用cmc
> 工具的时候会正常上下级匹配不到的状况。

#### 创建生成身份分层参数初始化交易 payload

**参数说明**

- `contractName`：合约名
- `orgId`：组织`Id`
- `hibeParamsFilePath`：`hibe.Params`存储文件位置

```go
CreateHibeInitParamsTransactionPayloadParams(contractName, orgId string, hibeParamsFilePath string) (map[string]string, error)
```

#### 生成身份分层加密交易 payload params，加密参数已知

**参数说明**

- `plaintext`: 待加密交易消息明文
- `receiverIds`: 消息接收者对应的加密参数，需和`paramsList` 一一对应
- `paramsBytesList`: 消息接收者对应的加密参数，需和 `receiverIds` 一一对应
- `txId`: 交易 Id 作为链上存储 hibeMsg 的 Key, 如果不提供存储的信息可能被覆盖
- `keyType`: 对明文进行对称加密的方法，请传入 `common` 项目中 `crypto` 包提供的方法，目前提供`AES`和`SM4`f两种方法

```go
CreateHibeTxPayloadParamsWithHibeParams(plaintext []byte, receiverIds []string, paramsBytesList [][]byte, txId string, keyType crypto.KeyType) (map[string]string, error)
```

#### 生成身份分层加密交易 payload params，hibe.Params根据 `receiverOrgIds` 链上查询得出

**参数说明**

- `contractName`: 合约名
- `queryParamsMethod`: 查询链上`hibe.Params`合约方法名
- `plaintext`: 交易信息
- `receiverIds`: 消息接收者对应的加密参数，需和`paramsList` 一一对应
- `paramsList`: 消息接收者对应的加密参数，需和 `receiverIds` 一一对应
- `receiverOrgIds`: 链上查询 hibe Params 的 Key 列表，需要和 `receiverIds` 一一对应
- `txId`: 交易 Id 作为链上存储 hibeMsg 的 Key, 如果不提供存储的信息可能被覆盖
- `keyType`: 对明文进行对称加密的方法，请传入 `common` 项目中 `crypto` 包提供的方法，目前提供`AES`和`SM4`两种方法
- `timeout`：（内部查询 `hibe.Params` 的）超时时间，单位：s，若传入-1，将使用默认超时时间：10s

```go
CreateHibeTxPayloadParamsWithoutHibeParams(contractName, queryParamsMethod string, plaintext []byte, receiverIds []string, receiverOrgIds []string, txId string, keyType crypto.KeyType, timeout int64) (map[string]string, error)
```

#### 查询某一组织的加密公共参数

**参数说明**

- `contractName`：合约名
- `method`：查询方法
- `orgId`: 参与方 `id`
- `timeout`: 超时时间，单位：`s`，若传入`-1`，将使用默认超时时间：`10s`


```go
QueryHibeParamsWithOrgId(contractName, method, orgId string, timeout int64) ([]byte, error)
```

##### 已知交易id，私钥，解密链上hibe交易密文信息

**参数说明**

- `localId`: 本地身份分层加密 `id`
- `hibeParams`: hibeParams 序列化后的byte数组
- `hibePrvKey`: hibe私钥序列化后的byte数组
- `txId`: 身份分层加密交易 `id`
- `keyType`: 对加密信息进行对称解密的方法，请和加密时使用的方法保持一致，请传入 `common` 中 `crypto` 包提供的方法，目前提供`AES`和`SM4`两种方法


```go
DecryptHibeTxByTxId(localId string, hibeParams []byte, hibePrvKey []byte, txId string, keyType crypto.KeyType) ([]byte, error)
```



## 用例

### 使用 cmc hibe 进行初始化工作

#### 初始化组织分层架构

打印文件存储信息，文件存储在指定目录，并根据组织名进行管理：

```sh
./cmc hibe init \
-o wx-org1.chainmaker.org \
-l 5 \
-s ./hibe-data
[wx-org1.chainmaker.org params] storage file path: hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.params
[wx-org1.chainmaker.org masterKey] storage file path: hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.masterKey
```

这就是我们的`params`和`masterKey`，我们可以基于这两个系统参数去生成密钥

此时生成的目录结构如下：

```
├── hibe-data
│   └── wx-org1.chainmaker.org
│       ├── wx-org1.chainmaker.org.masterKey
│       └── wx-org1.chainmaker.org.params
```

#### 根据根路径和Orgid查看生成的 hibeParams 具体存储路径

```
./cmc hibe getParams -o wx-org1.chainmaker.org -p ./hibe-data
[wx-org1.chainmaker.org params] file path: hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.params
[wx-org1.chainmaker.org Params] : &{G:bn256.G2((47178902283841676690169313198230611764434450586385483963780608219448256132608, 
...
55153595869182115492300723746636577085717766168705040101215917962024994307961)] Pairing:<nil>}
```

#### 根据MasterKey生成私钥

```
./cmc hibe genPrvKey \
-m 1 \
-i wx-topL \
-k ./hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.masterKey \
-p ./hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.params \
-o wx-org1.chainmaker.org \
-s ./hibe-data
[wx-topL] privateKey storage file path: ./hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL.privateKey
```

打印密钥存储路径，集中管理在 *./hibe-data/wx-org1.chainmaker.org/privateKeys* ：下

```
├── hibe-data
    └── wx-org1.chainmaker.org
        ├── privateKeys
        │   └── wx-topL.privateKey
        ├── wx-org1.chainmaker.org.masterKey
        └── wx-org1.chainmaker.org.params
```

`masterKey`是要妥善保管的，下层私钥建议使用第二种方法，基于parentKey进行生成，分发。

例如：我们基于上面生成的私钥为直接下属生成私钥

```
./cmc hibe genPrvKey \
-m 0 \
-i wx-topL/secondL \
-k ./hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL.privateKey \
-p ./hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.params \
-o wx-org1.chainmaker.org \
-s ./hibe-data
[wx-topL secondL] privateKey storage file path: ./hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL#secondL.privateKey
```

也可以为自己下属的其他层级员工生成私钥：

```
./cmc hibe genPrvKey \
-m 0 \
-i wx-topL/secondL/thirdL \
-k ./hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL.privateKey \
-p ./hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.params \
-o wx-org1.chainmaker.org \
-s ./hibe-data
[wx-topL secondL thirdL] privateKey storage file path: ./hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL#secondL#thirdL.privateKey
```

注意，所有的私钥都平铺在指定目录（这里是 *./hibe-data/wx-org1.chainmaker.org/privateKeys* ）下进行管理，并且会以 `_` 替换 `/` 之后的*ID.privatekey*命名，此时的文件夹结构及子密钥名称如下：

```go
./hibe-data/
└── wx-org1.chainmaker.org
    ├── privateKeys
    │   ├── wx-topL.privateKey
    │   ├── wx-topL#secondL.privateKey
    │   └── wx-topL#secondL#thirdL.privateKey
    ├── wx-org1.chainmaker.org.masterKey
    └── wx-org1.chainmaker.org.params
```



#### 私钥文件分发到下属，各自保管，我们在这里把自己的私钥放在在SDK如下路径：

```
├── testdata
    └── hibe-data
        └── wx-org1.chainmaker.org
            ├── privateKeys
            │   ├── wx-topL.privateKey
            │   ├── wx-topL#secondL.privateKey
            │   └── wx-topL#secondL#thirdL.privateKey
            ├── wx-org1.chainmaker.org.masterKey
            └── wx-org1.chainmaker.org.params

```



### 编写智能合约

#### 使用 go 编写测试的智能合约

智能合约的编写要提供针对`hibe.Params`和`hibe.Msg`存储、查询的方法，建议按照如下格式编写，或内部形成约定，指定好查询两个信息的id(标识)，即存储的Key：

> ParamsId: 建议使用 OrgId
>
> HibeMsgId: 使用 TxId

> 注意：编写的智能合约存储的数据的时候字段名称要和sdk中保持一致：
>
> ```
> 	// HibeMsgKey as a payload parameter
> 	HibeMsgKey = "hibe_msg"
> 
> 	// HibeMsgIdKey Key as a hibeMsgMap parameter
> 	HibeMsgIdKey = "tx_id"
> 
> 	// HibeParamsKey The value of the key (org_id) is the unique identifier of a HIBE params
> 	HibeParamsKey = "org_id"
> 
> 	// HibeParamsValueKey The value of the key (params) is the Hibe's params
> 	HibeParamsValueKey = "params
> ```
>
> 

首先我们根据前面的智能合约部分提示，编写一个合约：

```go
package main

//export init_contract
func initContract() {

}

//export upgrade
func upgrade() {

}

//export save_hibe_params
func save_params() {
	params := Args()
	itemMap := make(map[string]string, 0)
	itemMap = EasyCodecItemToParamsMap(params)
	itemBytes := EasyMarshal(params)
	PutState("hibe_params", itemMap["org_id"], string(itemBytes))
}

//export find_params_by_org_id
func findParamsByOrgId() {
	org_id, _ := Arg("org_id")
	if result, resultCode := GetStateByte("hibe_params", org_id); resultCode != SUCCESS {
		ErrorResult("错误提示 ， orgId:" + org_id)
	} else {
		LogMessage("get val:" + string(result))
		SuccessResultByte(result)
	}
}

//export save_hibe_msg
func saveHibeMsg() {
	parameters := Args()

	stones := make(map[string]string)
	for _, elem := range parameters {
		stones[elem.Key] = elem.Value.(string)
	}

	items := ParamsMapToEasyCodecItem(stones)
	itemsBytes := EasyMarshal(items)
	logMessage(string(itemsBytes))

	PutState("hibe", stones["tx_id"], string(itemsBytes))

}

func main() {

}
```

#### 编译字节码文件放到项目中

使用`tinygo build -no-debug -opt=s -o contract-hibe-1.0.0.wasm -target wasm`生成合约字节码文件，放入到SDK下，本例放到如下路径（使用的时候要在 SDK 中指定配置位置）：

```
├── testdata
│   ├── counter-go-demo
│   │   ├── contract-hibe-1.0.0.wasm
```

### 使用SDK和密钥文件对信息加密上链并从链上获取和加密

#### HIBE相关消息Key规定

下面三个方法名是测试的示例合约提供的方法，分别是 `hibeParams`和`hibemsg`的存储和查询方法。

下面是合约存储数据规定的一些Key，

- `HIBE_BIZID_KEY` ：整个hibe消息的合约查询Key
- `HIBE_MSG_KEY`：hibe消息体的Key
- `HIBE_MSG_CIPHER_TEXT_KEY`：消息体里被加密信息的Key
- `HIBE_PARAMS_KEY`：params消息的合约查询Key
- `HIBE_PARAMS_VALUE_KEY`：params 值的 Key

```go
// hibe msg's Keys
const (
	// HibeMsgKey as a payload parameter
	HibeMsgKey = "hibe_msg"

	// HibeMsgIdKey Key as a hibeMsgMap parameter
	HibeMsgIdKey = "tx_id"

	// HibeMsgCipherTextKey Key as a hibeMsgMap parameter
	// The value of the key (CT) is the hibe_msg's message (ciphertext)
	HibeMsgCipherTextKey = "CT"

	// HibeParamsKey The value of the key (org_id) is the unique identifier of a HIBE params
	HibeParamsKey = "org_id"

	// HibeParamsValueKey The value of the key (params) is the Hibe's params
	HibeParamsValueKey = "params"

```

测试合约的方法名：

```go
// test contract functionName
const (
	// save Hibe Message
	saveHibeMsg = "save_hibe_msg"

	// save params
	saveHibeParams = "save_hibe_params"

	// find params by ogrId
	findParamsByOrgId = "find_params_by_org_id"
)
```



#### 前置数据

提前准备了一些数据来做测试，包括超时时间、本地 `hibeParams` 存储路径，测试消息、`hibe_msg`信息标识以及测试的三组`ID`和对应的私钥文件。（一般使用时本地只存放自己的私钥文件和本公司的 `hibeParams` 文件）。

```go
// test data
const (
	hibeContractByteCodePath = "./testdata/counter-go-demo/contract-hibe-1.0.0.wasm"

	//
	hibeContractName = "contract-hibe-1"

	// 本地 hibe params 文件路径
	localHibeParamsFilePath = "./testdata/hibe-data/wx-org1.chainmaker.org/wx-org1.chainmaker.org.params"

	// 测试源消息
	msg = "这是一条HIBE测试存证 ✔✔✔"

	// hibe_msg 的消息 Id
	bizId2 = "1234567890123452"

	// Id 和 对应私钥文件路径 这里测试3组
	localTopLevelId                 = "wx-topL"
	localTopLevelHibePrvKeyFilePath = "./testdata/hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL.privateKey"

	localSecondLevelId                 = "wx-topL/secondL"
	localSecondLevelHibePrvKeyFilePath = "./testdata/hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL#secondL.privateKey"

	localThirdLevelId                 = "wx-topL/secondL/thirdL"
	localThirdLevelHibePrvKeyFilePath = "./testdata/hibe-data/wx-org1.chainmaker.org/privateKeys/wx-topL#secondL#thirdL.privateKey"
)
```

#### 初始化客户端，进行测试

```go
func TestHibeContractCounterGo(t *testing.T) {

	txId = GetRandTxId()
	client, err := createClientWithConfig()
	require.Nil(t, err)

	admin1, err := createAdmin(orgId1)
	require.Nil(t, err)
	admin2, err := createAdmin(orgId2)
	require.Nil(t, err)
	admin3, err := createAdmin(orgId3)
	require.Nil(t, err)
	admin4, err := createAdmin(orgId4)
	require.Nil(t, err)

	fmt.Println("====================== 创建合约（异步）======================")
	testUserHibeContractCounterGoCreate(t, client, admin1, admin2, admin3, admin4, false)
	time.Sleep(5 * time.Second)

	fmt.Println("====================== 调用合约 params 上链 （异步）======================")
	testUserHibeContractParamsGoInvoke(t, client, saveHibeParams, false)
	time.Sleep(5 * time.Second)

	fmt.Println("====================== 执行合约 params 查询接口 ======================")
	testUserHibeContractParamsGoQuery(t, client, findParamsByOrgId, nil)

	fmt.Println("====================== 调用合约 加密数据上链（异步）======================")
	testUserHibeContractMsgGoInvoke(t, client, saveHibeMsg, false)
	time.Sleep(5 * time.Second)

	fmt.Println("====================== 执行合约 加密数据查询接口 ======================")
	testUserHibeContractMsgGoQuery(t, client)
}
```

#### 创建合约

```go
// 创建Hibe合约
func testUserHibeContractCounterGoCreate(t *testing.T, client *ChainClient,
	admin1, admin2, admin3, admin4 *ChainClient, withSyncResult bool) {

	resp, err := createUserHibeContract(client, admin1, admin2, admin3, admin4,
		hibeContractName, version, hibeContractByteCodePath, common.RuntimeType_GASM, []*common.KeyValuePair{}, withSyncResult)
	require.Nil(t, err)

	t.Logf("CREATE contract-hibe-1 contract resp: %+v\n", resp)

```

创建合约要指定合约名，合约版本，合约字节码文件位置，和运行时环境等，具体流程如下：

```go
func createUserHibeContract(client *ChainClient, admin1, admin2, admin3, admin4 *ChainClient,
	contractName, version, byteCodePath string, runtime common.RuntimeType, kvs []*common.KeyValuePair, withSyncResult bool) (*common.TxResponse, error) {

	payloadBytes, err := client.CreateContractCreatePayload(contractName, version, byteCodePath, runtime, kvs)
	if err != nil {
		return nil, err
	}

	// 各组织Admin权限用户签名
	signedPayloadBytes1, err := admin1.SignContractManagePayload(payloadBytes)
	if err != nil {
		return nil, err
	}

	signedPayloadBytes2, err := admin2.SignContractManagePayload(payloadBytes)
	if err != nil {
		return nil, err
	}

	signedPayloadBytes3, err := admin3.SignContractManagePayload(payloadBytes)
	if err != nil {
		return nil, err
	}

	signedPayloadBytes4, err := admin4.SignContractManagePayload(payloadBytes)
	if err != nil {
		return nil, err
	}

	// 收集并合并签名
	mergeSignedPayloadBytes, err := client.MergeContractManageSignedPayload([][]byte{signedPayloadBytes1,
		signedPayloadBytes2, signedPayloadBytes3, signedPayloadBytes4})
	if err != nil {
		return nil, err
	}

	// 发送创建合约请求
	resp, err := client.SendContractManageRequest(mergeSignedPayloadBytes, createContractTimeout, withSyncResult)
	if err != nil {
		return nil, err
	}

	err = checkProposalRequestResp(resp, true)
	if err != nil {
		return nil, err
	}

	return resp, nil
}
```



#### 本地 params 上链

```go
// 调用Hibe合约
// params 上链
func testUserHibeContractParamsGoInvoke(t *testing.T, client *ChainClient,
	method string, withSyncResult bool) {
	err := invokeUserHibeContractParams(t, client, hibeContractName, method, "", withSyncResult)
	require.Nil(t, err)
}
```

上链要指定合约名，存储上链的合约方法，本地`hibeParams`文件路径等，先构造存储 `hibeParams`的`payloadParams`，然后调用`InvokeContract`接口上链：

```go
func invokeUserHibeContractParams(t *testing.T, client *ChainClient, contractName, method, txId string, withSyncResult bool) error {
	localParams, err := ReadHibeParamsWithFilePath(localHibeParamsFilePath)
	if err != nil {
		return err
	}
	payloadParams, err := client.CreateHibeInitParamsTxPayloadParams(orgId1, localParams)
	resp, err := client.InvokeContract(contractName, method, txId, payloadParams, -1, withSyncResult)
	if err != nil {
		return err
	}

	if resp.Code != common.TxStatusCode_SUCCESS {
		return fmt.Errorf("invoke contract failed, [code:%d]/[msg:%s]\n", resp.Code, resp.Message)
	}

	if !withSyncResult {
		t.Logf("invoke contract success, resp: [code:%d]/[msg:%s]/[txId:%s]\n", resp.Code, resp.Message, resp.ContractResult.Result)
	} else {
		t.Logf("invoke contract success, resp: [code:%d]/[msg:%s]/[contractResult:%s]\n", resp.Code, resp.Message, resp.ContractResult)
	}

	return nil
}
```



#### 根据组织Id链上查询其公司Params

```go
// params 查询
func testUserHibeContractParamsGoQuery(t *testing.T, client *ChainClient,
	method string, params map[string]string) {
	hibeParams, err := client.QueryHibeParamsWithOrgId(hibeContractName, findParamsByOrgId, orgId1, -1)
	require.Nil(t, err)
	t.Logf("QUERY %s contract resp -> hibeParams:%s\n", hibeContractName, hibeParams)
}
```



#### hibe加密信息上链

```go
// 加密数据上链
func testUserHibeContractMsgGoInvoke(t *testing.T, client *ChainClient,
	method string, withSyncResult bool) {
	err := invokeUserHibeContractMsg(t, client, hibeContractName, method, txId, withSyncResult)
	require.Nil(t, err)
}
```

加密信息上链需要指定合约名、合约方法、要加密上链的明文信息，可解密者的ID及其公司Params、对文本进行对称加密的算法（目前支持AES、SM4(key目前不支持传递，统一内部设定为128bit)），具体使用方法如下：

```go
func invokeUserHibeContractMsg(t *testing.T, client *ChainClient, contractName, method, txId string, withSyncResult bool) error {
	receiverId := make([]string, 3)
	receiverId[0] = localSecondLevelId
	receiverId[1] = localThirdLevelId
	receiverId[2] = localTopLevelId

	// fetch orgId []string from receiverId []string
	org := make([]string, len(receiverId))
	org[0] = "wx-org1.chainmaker.org"
	org[1] = "wx-org1.chainmaker.org"
	org[2] = "wx-org1.chainmaker.org"

	// query params
	paramsBytesList := make([][]byte, 0)
	for _, id := range org {
		hibeParamsBytes, err := client.QueryHibeParamsWithOrgId(hibeContractName, findParamsByOrgId, id, -1)
		if err != nil {
			//t.Logf("QUERY hibe-contract-go-1 contract resp: %+v\n", hibeParams)
			return fmt.Errorf("client.QueryHibeParamsWithOrgId(hibeContractName, id, -1) failed, err: %v\n", err)
		}

		if len(hibeParamsBytes) == 0 {
			return fmt.Errorf("no souch params of %s's org, please check it", id)
		}

		paramsBytesList = append(paramsBytesList, hibeParamsBytes)
	}

	keyType := crypto.AES
	//keyType := crypto.SM4
	params, err := client.CreateHibeTxPayloadParamsWithHibeParams([]byte(msg), receiverId, paramsBytesList, txId, keyType)
	if err != nil {
		return err
	}

	resp, err := client.InvokeContract(contractName, method, txId, params, -1, withSyncResult)
	if err != nil {
		return err
	}

	if resp.Code != common.TxStatusCode_SUCCESS {
		return fmt.Errorf("invoke contract failed, [code:%d]/[msg:%s]\n", resp.Code, resp.Message)
	}

	if !withSyncResult {
		t.Logf("invoke contract success, resp: [code:%d]/[msg:%s]/[txId:%s]\n", resp.Code, resp.Message, resp.ContractResult.Result)
	} else {
		t.Logf("invoke contract success, resp: [code:%d]/[msg:%s]/[contractResult:%s]\n", resp.Code, resp.Message, resp.ContractResult)
	}

	return nil
}
```

#### 获取链上的 `hibeMsg` 并解密：

解密时需要注意，解密的时候指定的算法要和加密时候的算法相同！

```go
// 获取加密数据
func testUserHibeContractMsgGoQuery(t *testing.T, client *ChainClient) {
	keyType := crypto.AES
	//keyType := crypto.SM4

	localParams, err := ReadHibeParamsWithFilePath(localHibeParamsFilePath)
	require.Nil(t, err)

	topHibePrvKey, err := ReadHibePrvKeysWithFilePath(localTopLevelHibePrvKeyFilePath)
	require.Nil(t, err)

	msgBytes1, err := client.DecryptHibeTxByTxId(localTopLevelId, localParams, topHibePrvKey, txId, keyType)
	require.Nil(t, err)
	t.Logf("QUERY hibe-contract-go-1 contract resp DecryptHibeTxByBizId [Decrypt Msg By TopLevel privateKey] message: %s\n", string(msgBytes1))

	msgBytes2, err := client.DecryptHibeTxByTxId(localTopLevelId, localParams, topHibePrvKey, txId, keyType)
	require.Nil(t, err)
	t.Logf("QUERY hibe-contract-go-1 contract resp DecryptHibeTxByBizId [Decrypt Msg By SecondLevel privateKey] message: %s\n", string(msgBytes2))

	msgBytes3, err := client.DecryptHibeTxByTxId(localTopLevelId, localParams, topHibePrvKey, txId, keyType)
	require.Nil(t, err)
	t.Logf("QUERY hibe-contract-go-1 contract resp DecryptHibeTxByBizId [Decrypt Msg By ThirdLevel privateKey] message: %s\n", string(msgBytes3))

}
```

上面是指定AES算法，下面指定SM4算法测试：

加密处指定SM4：

```go
func invokeUserHibeContractMsg(client *ChainClient, contractName, method, txId string, withSyncResult bool) error {
	// ...
	
	//keyType := crypto.AES
	keyType := crypto.SM4
	// ...
}
```

解密处指定SM4算法：

```go
// 获取加密数据
func testUserHibeContractMsgGoQuery(t *testing.T, client *ChainClient) {
	//keyType := crypto.AES
	keyType := crypto.SM4
	//...
}
```

#### 执行测试：

```
====================== 创建合约（异步）======================
2021-04-23 22:16:27.226	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:31	[SDK] create [ContractCreate] to be signed payload
2021-04-23 22:16:27.226	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:60	[SDK] create [ContractManage] to be signed payload
2021-04-23 22:16:27.230	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"OK" 
    sdk_hibe_test.go:95: CREATE contract-hibe-1 contract resp: message:"OK" contract_result:<result:"da36c5ae4d29439bbf589b088347e3843540853b50f94f3e9c3d074c60d9b043" message:"OK" > 
====================== 调用合约 params 上链 （异步）======================
�`�[*�Eꑃ�r��B....�]�1y]]
2021-04-23 22:16:32.232	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"OK" 
    sdk_hibe_test.go:224: invoke contract success, resp: [code:0]/[msg:OK]/[txId:0bfe933152664c918ba07dd5007bfb4dd7de5631497d4857a8dbcbc14d6dd260]
====================== 执行合约 params 查询接口 ======================
2021-04-23 22:16:37.233	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:240	[SDK] begin to QUERY contract, [contractName:contract-hibe-1]/[method:find_params_by_org_id]/[txId:2e018185e4c6437f94ea0d49fb9677759d6593a756da4b14bf921e9c8fd634e2]/[params:map[org_id:wx-org1.chainmaker.org]]
2021-04-23 22:16:37.245	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:...10\224]\2331\177y" gas_used:11114830 > 
�`�[*�Eꑃ�r��B�`.....)Q�<ϙ=!�6WT��
====================== 调用合约 加密数据上链（异步）======================
2021-04-23 22:16:37.246	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:240	[SDK] begin to QUERY contract, [contractName:contract-hibe-1]/[method:find_params_by_org_id]/[txId:ca07d3ee21a54bf2ae3679af9719f02221877cd04da34516a5ce12e091fd0e0a]/[params:map[org_id:wx-org1.chainmaker.org]]
2021-04-23 22:16:37.259	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\013\000\000\0...\224]\2331\177y" gas_used:11114830 > 
2021-04-23 22:16:37.260	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:240	[SDK] begin to QUERY contract, [contractName:contract-hibe-1]/[method:find_params_by_org_id]/[txId:519b7ff922124f60b8dd8dcbebb5286586b8f0b03139460fb0559528da8be996]/[params:map[org_id:wx-org1.chainmaker.org]]
2021-04-23 22:16:37.271	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\013\000\000\000\001\...331\177y" gas_used:11114830 > 
2021-04-23 22:16:37.272	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:240	[SDK] begin to QUERY contract, [contractName:contract-hibe-1]/[method:find_params_by_org_id]/[txId:0b20ddd3c7474325a75077179913f44439b4fa71ec4c483aa5013b575f2af444]/[params:map[org_id:wx-org1.chainmaker.org]]
2021-04-23 22:16:37.283	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\013\000\000\000\001\000...\305[\317\014@\306G\202V\224y\357\324:N\023\231\375\225\351'G\365\035\225\246\335\357\272J\265n;=\276\010\224]\2331\177y" gas_used:11114830 > 
2021-04-23 22:16:37.294	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_user_contract.go:197	[SDK] begin to INVOKE contract, [contractName:contract-hibe-1]/[method:save_hibe_msg]/[txId:d2f51965405f4f04bd1a622904114c1cb4171c98f5534c828e29d560a7b9a130]/[params:map[hibe_msg:{"CT":"igAoeoyhpAjaCZOe9dzeaEO6cIYB4hBRcv/SMuP/REwZLbAyomMe46lbIof4S7K5bJhAs10wA+x2roo6xZYjDg==","wx-topL/secondL/thirdL":"SZoLPK/fu/3awMYZ8k/e9aTP9AJyXKYEvF4gNsXNVshdnN5G5tNHWKk93PYhCtIut2suZ+YBNHydNrshf5VewEOH0wZ9rPFr0iEY/voAYilif7gFLSdaFjy/yknBG894W5rVYdYsipHV5+35kMsOKh70jvr5eBwFEVTYZ5e91d6OxJ8cGwGIWNBezLLnDXuHNXtXhOD2+3+56ZEceuIow33Qo+yAaxUWE09XvOvmZkhHW+PrVYwqKyleWcUmgsOjR9X4iwZI4NUEzBUK6MQVVV9I6fkZHfxea9I0EpWt+HtXAyIiYGc2Flz/PjvMAmx/3h6XtliYSJWgek60FtNcpUKZFanr3905v/3+cAMnY3SkKXE/3UqG3eY1rq1TYT8AK6GIdKeaO70Lhjzzu5o0eMsF3i3zAT+GFhtLy6cfEe0FZIAqSiXJAVasfxoiYOyPSubChCDKGY4wCUe2zKTsT0RirQB0JjB9TGwJFTl+ZGfT3NesvDW39QMkuij4sh9EQ5IEfHOyU2ulnCXKPRbW8FMiG2t6EEdEKyv/JLEPmd1RYxm4A1WwmHZ2B8sE0MdADIk5nfVX3A7tKbo06t+OJYNEJezrNkhqDVSXGdJ/ydLAn8VBevmSHjeNDgEIobjOc1ZMg3VpoQ40eviQwblsLep/pyTEuXw1pveuoD0Rz5c16GjiV5a9KHh5ak+HMxjZ9nX8J6zRIzRCG4ltwrqOtDNd0qktef/EhXrIb5pEe8qcbHkEUK4yRYnpP61e1x59"} tx_id:d2f51965405f4f04bd1a622904114c1cb4171c98f5534c828e29d560a7b9a130]]
2021-04-23 22:16:37.295	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"OK" 
    sdk_hibe_test.go:281: invoke contract success, resp: [code:0]/[msg:OK]/[txId:d2f51965405f4f04bd1a622904114c1cb4171c98f5534c828e29d560a7b9a130]
====================== 执行合约 加密数据查询接口 ======================
2021-04-23 22:16:42.295	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_system_contract.go:24	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:d2f51965405f4f04bd1a622904114c1cb4171c98f5534c828e29d560a7b9a130]
2021-04-23 22:16:42.298	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\n\376\t... message:"OK" > 
    sdk_hibe_test.go:146: QUERY hibe-contract-go-1 contract resp DecryptHibeTxByBizId [Decrypt Msg By TopLevel privateKey] message: 这是一条HIBE测试存证 ✔✔✔
2021-04-23 22:16:42.334	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_system_contract.go:24	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:d2f51965405f4f04bd1a622904114c1cb4171c98f5534c828e29d560a7b9a130]
2021-04-23 22:16:42.335	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\n\376\t\n\21... message:"OK" > 
    sdk_hibe_test.go:150: QUERY hibe-contract-go-1 contract resp DecryptHibeTxByBizId [Decrypt Msg By SecondLevel privateKey] message: 这是一条HIBE测试存证 ✔✔✔
2021-04-23 22:16:42.352	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_system_contract.go:24	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:d2f51965405f4f04bd1a622904114c1cb4171c98f5534c828e29d560a7b9a130]
2021-04-23 22:16:42.353	[DEBUG]	[SDK]	chainmaker-sdk-go/sdk_client.go:351	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\n\376\t\n\214\001\n\00... message:"OK" > 
    sdk_hibe_test.go:154: QUERY hibe-contract-go-1 contract resp DecryptHibeTxByBizId [Decrypt Msg By ThirdLevel privateKey] message: 这是一条HIBE测试存证 ✔✔✔
--- PASS: TestHibeContractCounterGo (15.18s)
PASS
```

