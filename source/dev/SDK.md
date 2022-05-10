# SDK

<span id="section_sdk"></span>

## 概述

长安链`SDK`是业务模块与长安链交互的桥梁，支持双向`TLS`认证，提供安全可靠的加密通信信道。

长安链提供了多种语言的`SDK`，包括：`Go SDK`、`Java SDK`、`Python SDK`（实现中）、`Nodejs SDK`（实现中）方便开发者根据需要进行选用。

提供的`SDK`接口，覆盖合约管理、链配置管理、证书管理、多签收集、各类查询操作、事件订阅等场景，满足了不同的业务场景需要。

## 约定概念

- **`Node`（节点）**：代表一个链节点的基本信息，包括：节点地址、连接数、是否启用`TLS`认证等信息
- **`ChainClient`（链客户端）**：所有客户端对链节点的操作接口都来自`ChainClient`
- **压缩证书**：可以为`ChainClient`开启证书压缩功能，开启后可以减小交易包大小，提升处理性能

## 开发指南

### Go SDK

#### 环境依赖

**golang**

版本为1.16或以上

下载地址：https://golang.org/dl/

若已安装，请通过命令查看版本：

```bash
$ go version
go version go1.16 linux/amd64
```

#### 下载安装

```bash
$ git clone -b v2.2.0 https://git.chainmaker.org.cn/chainmaker/sdk-go.git
```

#### 示例代码

##### 创建节点

```go
// 创建节点
func createNode(nodeAddr string, connCnt int) *NodeConfig {
	node := NewNodeConfig(
		// 节点地址，格式：127.0.0.1:12301
		WithNodeAddr(nodeAddr),
		// 节点连接数
		WithNodeConnCnt(connCnt),
		// 节点是否启用TLS认证
		WithNodeUseTLS(true),
		// 根证书路径，支持多个
		WithNodeCAPaths(caPaths),
		// TLS Hostname
		WithNodeTLSHostName(tlsHostName),
	)

	return node
}
```

##### 以参数形式创建ChainClient

> 更多内容请参看：`sdk_client_test.go`
>
> 注：示例中证书采用路径方式去设置，也可以使用证书内容去设置，具体请参看`createClientWithCaCerts`方法

```go
// 创建ChainClient
func createClient() (*ChainClient, error) {
	if node1 == nil {
		// 创建节点1
		node1 = createNode(nodeAddr1, connCnt1)
	}

	if node2 == nil {
		// 创建节点2
		node2 = createNode(nodeAddr2, connCnt2)
	}

	chainClient, err := NewChainClient(
		// 设置归属组织
		WithChainClientOrgId(chainOrgId),
		// 设置链ID
		WithChainClientChainId(chainId),
		// 设置logger句柄，若不设置，将采用默认日志文件输出日志
		WithChainClientLogger(getDefaultLogger()),
		// 设置客户端用户私钥路径
		WithUserKeyFilePath(userKeyPath),
		// 设置客户端用户证书
		WithUserCrtFilePath(userCrtPath),
		// 添加节点1
		AddChainClientNodeConfig(node1),
		// 添加节点2
		AddChainClientNodeConfig(node2),
		)

	if err != nil {
		return nil, err
	}

	//启用证书压缩（开启证书压缩可以减小交易包大小，提升处理性能）
	err = chainClient.EnableCertHash()
	if err != nil {
		log.Fatal(err)
	}

	return chainClient, nil
}
```

##### 以配置文件形式创建ChainClient

> 注：参数形式和配置文件形式两个可以同时使用，同时配置时，以参数传入为准

```go
func createClientWithConfig() (*ChainClient, error) {

	chainClient, err := NewChainClient(
		WithConfPath("./testdata/sdk_config.yml"),
	)

	if err != nil {
		return nil, err
	}

	//启用证书压缩（开启证书压缩可以减小交易包大小，提升处理性能）
	err = chainClient.EnableCertHash()
	if err != nil {
		return nil, err
	}

	return chainClient, nil
}
```

##### 创建wasm合约

> `sdk_user_contract_claim_test.go`

```go
func testUserContractClaimCreate(t *testing.T, client *ChainClient,
	admin1, admin2, admin3, admin4 *ChainClient, withSyncResult bool, isIgnoreSameContract bool) {

	resp, err := createUserContract(client, admin1, admin2, admin3, admin4,
		claimContractName, claimVersion, claimByteCodePath, common.RuntimeType_WASMER, []*common.KeyValuePair{}, withSyncResult)
	if !isIgnoreSameContract {
		require.Nil(t, err)
	}

	fmt.Printf("CREATE claim contract resp: %+v\n", resp)
}

func createUserContract(client *ChainClient, admin1, admin2, admin3, admin4 *ChainClient,
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
```

##### 调用wasm合约

> `sdk_user_contract_claim_test.go`

```go
func testUserContractClaimInvoke(client *ChainClient,
	method string, withSyncResult bool) (string, error) {

	curTime := fmt.Sprintf("%d", CurrentTimeMillisSeconds())
	fileHash := uuid.GetUUID()
	params := map[string]string{
		"time":      curTime,
		"file_hash": fileHash,
		"file_name": fmt.Sprintf("file_%s", curTime),
	}

	err := invokeUserContract(client, claimContractName, method, "", params, withSyncResult)
	if err != nil {
		return "", err
	}

	return fileHash, nil
}

func invokeUserContract(client *ChainClient, contractName, method, txId string, params map[string]string, withSyncResult bool) error {

	resp, err := client.InvokeContract(contractName, method, txId, params, -1, withSyncResult)
	if err != nil {
		return err
	}

	if resp.Code != common.TxStatusCode_SUCCESS {
		return fmt.Errorf("invoke contract failed, [code:%d]/[msg:%s]\n", resp.Code, resp.Message)
	}

	if !withSyncResult {
		fmt.Printf("invoke contract success, resp: [code:%d]/[msg:%s]/[txId:%s]\n", resp.Code, resp.Message, resp.ContractResult.Result)
	} else {
		fmt.Printf("invoke contract success, resp: [code:%d]/[msg:%s]/[contractResult:%s]\n", resp.Code, resp.Message, resp.ContractResult)
	}

	return nil
}
```

##### 创建及调用evm合约

> `sdk-go/examples/user_contract_evm_balance/main.go`(https://git.chainmaker.org.cn/chainmaker/sdk-go/-/blob/master/examples/user_contract_evm_balance/main.go)

##### 更多示例和用法

> 更多示例和用法，请参看单元测试用例

| 功能     | 单测代码                      |
| -------- | ----------------------------- |
| 用户合约 | `sdk_user_contract_test.go`   |
| 系统合约 | `sdk_system_contract_test.go` |
| 链配置   | `sdk_chain_config_test.go`    |
| 证书管理 | `sdk_cert_manage_test.go`     |
| 消息订阅 | `sdk_subscribe_test.go`       |

#### 接口说明

请参看：[《chainmaker-go-sdk》](../dev/chainmaker-go-sdk.md)

###  Java SDK

#### 环境依赖

**java**

> jdk 1.8.0_202或以下

下载地址：https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html

若jdk版本较高，请在路径为`cryto/ChainmakerX509CryptoSuite.java`的代码中修改`import sun.security.ec.CurveDB`为`import sun.security.util.CurveDB`

若已安装，请通过命令查看版本：

```bash
$ java -version
java version "1.8.0_202"
```

#### 下载安装

```bash
$ git clone -b v2.2.0 https://git.chainmaker.org.cn/chainmaker/sdk-java.git
```

#### jar包依赖

需将`sdk`中依赖的`jar`包导入本地工程中，
同时，需将`sdk`中`lib`目录下的`netty-tcnative-openssl-static-2.0.39.Final.jar`包导入工程中，以便适配国密`tls`通信。

#### 应用demo

java sdk应用示例，请参考<a href="https://git.chainmaker.org.cn/chainmaker/sdk-java-demo/-/tree/v2.2.0"  target="_blank"> sdk-java-demo </a>

#### 示例代码

##### 以参数形式创建ChainClient

> 更多内容请参看：`TestBase`

```java
public void initWithNoConfig() throws SdkException {
    byte[][] tlsCaCerts = new byte[][]{FileUtils.getResourceFileBytes(ORG1_CERT_PATH)};
    
    SdkConfig sdkConfig = new SdkConfig();
    ChainClientConfig chainClientConfig = new ChainClientConfig();
    sdkConfig.setChainClient(chainClientConfig);

    RpcClientConfig rpcClientConfig = new RpcClientConfig();
    rpcClientConfig.setMaxReceiveMessageSize(MAX_MESSAGE_SIZE);

    ArchiveConfig archiveConfig = new ArchiveConfig();
    archiveConfig.setDest(DEST);
    archiveConfig.setType(TYPE);
    archiveConfig.setSecretKey(SECRET_KEY);

    NodeConfig nodeConfig = new NodeConfig();
    nodeConfig.setTrustRootBytes(tlsCaCerts);
    nodeConfig.setTlsHostName(TLS_HOST_NAME1);
    nodeConfig.setEnableTls(true);
    nodeConfig.setNodeAddr(NODE_GRPC_URL1);
    nodeConfig.setConnCnt(CONNECT_COUNT);

    NodeConfig[] nodeConfigs = new NodeConfig[]{nodeConfig};

    chainManager = ChainManager.getInstance();
    chainClient = chainManager.getChainClient(CHAIN_ID);

    chainClientConfig.setOrgId(ORG_ID1);
    chainClientConfig.setChainId(CHAIN_ID);
    chainClientConfig.setUserKeyBytes(FileUtils.getResourceFileBytes(CLIENT1_TLS_KEY_PATH));
    chainClientConfig.setUserCrtBytes(FileUtils.getResourceFileBytes(CLIENT1_TLS_CERT_PATH));
    chainClientConfig.setUserSignKeyBytes(FileUtils.getResourceFileBytes(CLIENT1_KEY_PATH));
    chainClientConfig.setUserSignCrtBytes(FileUtils.getResourceFileBytes(CLIENT1_CERT_PATH));
    chainClientConfig.setRpcClient(rpcClientConfig);
    chainClientConfig.setArchive(archiveConfig);
    chainClientConfig.setNodes(nodeConfigs);

    if (chainClient == null) {
        chainClient = chainManager.createChainClient(sdkConfig);
    }
}
```

##### 以配置文件形式创建ChainClient

> 更多内容请参看：`TestBase`

```java
public void init() throws IOException, SdkException {
    Yaml yaml = new Yaml();
    InputStream in = TestBase.class.getClassLoader().getResourceAsStream(SDK_CONFIG);

    SdkConfig sdkConfig;
    sdkConfig = yaml.loadAs(in, SdkConfig.class);
    assert in != null;
    in.close();

    for (NodeConfig nodeConfig : sdkConfig.getChain_client().getNodes()) {
        List<byte[]> tlsCaCertList = new ArrayList<>();
        for (String rootPath : nodeConfig.getTrustRootPaths()){
            List<String> filePathList = FileUtils.getFilesByPath(rootPath);
            for (String filePath : filePathList) {
                tlsCaCertList.add(FileUtils.getFileBytes(filePath));
            }
        }
        byte[][] tlsCaCerts = new byte[tlsCaCertList.size()][];
        tlsCaCertList.toArray(tlsCaCerts);
        nodeConfig.setTrustRootBytes(tlsCaCerts);
    }

    chainManager = ChainManager.getInstance();
    chainClient = chainManager.getChainClient(sdkConfig.getChain_client().getChainId());

    if (chainClient == null) {
        chainClient = chainManager.createChainClient(sdkConfig);
    }
}
```

##### 创建合约

> 更多内容请参看：`TestUserContract`

```java
public void testCreateContract() throws IOException, InterruptedException, ExecutionException, TimeoutException {
    ResultOuterClass.TxResponse responseInfo = null;
    try {
        byte[] byteCode = FileUtils.getResourceFileBytes(CONTRACT_FILE_PATH);
 
        // 1. create payload
        Request.Payload payload = chainClient.createContractCreatePayload(CONTRACT_NAME, "1", byteCode,
                ContractOuterClass.RuntimeType.WASMER, null);
 
        //2. create payloads with endorsement
        Request.EndorsementEntry[] endorsementEntries = SdkUtils.getEndorsers(payload, new User[]{adminUser1, adminUser2, adminUser3});
 
        // 3. send request
        responseInfo = chainClient.sendContractManageRequest(payload, endorsementEntries, rpcCallTimeout, syncResultTimeout);
    } catch (SdkException e) {
        e.printStackTrace();
        Assert.fail(e.getMessage());
    }
    Assert.assertNotNull(responseInfo);
}
```

##### 调用合约

> 更多内容请参看：`TestUserContract`

```java
public void testInvokeContract() throws Exception {
    ResultOuterClass.TxResponse responseInfo = null;
    try {
        responseInfo = chainClient.invokeContract(CONTRACT_NAME, INVOKE_CONTRACT_METHOD,
                null, null, rpcCallTimeout, syncResultTimeout);
    } catch (Exception e) {
        e.printStackTrace();
        Assert.fail(e.getMessage());
    }
    Assert.assertNotNull(responseInfo);
}
```

##### 更多示例和用法

> 更多示例和用法，请参看单元测试用例

| 功能     | 单测代码                |
| -------- | ----------------------- |
| 基础配置 | `TestBase`              |
| 用户合约 | `TestUserContract`      |
| 系统合约 | `TestSystemContract`    |
| 链配置   | `TestChainConfig`       |
| 证书管理 | `TestBaseCertManage`    |
| 消息订阅 | `TestSubscribe`         |
| 线上多签 | `TestContractMultisign` |
| 公钥身份 | `TestPubkeyManage`      |

#### 接口说明

请参看：[《chainmaker-java-sdk》](../dev/chainmaker-java-sdk.md)

###  Nodejs SDK

#### 环境依赖

**nodejs**

> nodejs 14.0.0+

下载地址：https://nodejs.org/dist/

若已安装，请通过命令查看版本：

```bash
$ node --version
v14.0.0
```

#### 下载安装

```bash
$ git clone -b v2.0.0 https://git.chainmaker.org.cn/chainmaker/sdk-nodejs.git
```

#### 示例代码

##### 创建节点

> 更多内容请参看：`sdkInit.js`

```javascript
// 创建节点
this.node = new Node(nodeConfigArray, timeout);
```

##### 参数形式创建ChainClient

> 更多内容请参看：`sdkInit.js`

```javascript
// 创建ChainClient
const ChainClient = new Sdk(chainID, orgID, userKeyPathFile, userCertPathFile, nodeConfigArray, 30000, archiveConfig);

```

##### 以配置文件形式创建ChainClient

> 更多内容请参看：`sdkinit.js`

```javascript
const ChainClient = new LoadFromYaml(path.join(__dirname, './sdk_config.yaml'));
```

##### 创建合约

> 更多内容请参看：`testUserContractMgr.js`

```javascript
  const testCreateUserContract = async (sdk, contractName, contractVersion, contractFilePath) => {
		const response = await sdk.userContractMgr.createUserContract({
			contractName,
			contractVersion,
			contractFilePath,
			runtimeType: utils.common.RuntimeType.GASM,
			params: {
				key1: 'value1',
				key2: 'value2',
			},
		});
		return response;
	};
```

##### 调用合约

> 更多内容请参看：`testUserContractMgr.js`

```javascript
  const testInvokeUserContract = async (sdk, contractName) => {
		const response = await sdk.callUserContract.invokeUserContract({
			contractName, method: 'save', params: {
				file_hash: '1234567890',
				file_name: 'test.txt',
			},
		});
		return response;
	};
```

##### 更多示例和用法

> 更多示例和用法，请参看单元测试用例

安装mocha：

```bash
$ npm install -g mocha
```

使用脚本搭建chainmaker运行环境（4组织4节点），将build文件中的cryptogen复制到当前项目的test/testFile文件中

运行测试命令：

```bash
$ npm test
```

| 功能     | 单测代码            |
| -------- | ------------------- |
| 基础配置 | `sdkInit.js`        |
| 用户合约 | `userContract.js`   |
| 系统合约 | `systemContract.js` |
| 链配置   | `chainConfig.js`    |
| 证书管理 | `cert.js`           |
| 消息订阅 | `subscribe.js`      |

#### 接口说明

请参看：[《chainmaker-nodejs-sdk》](../dev/chainmaker-nodejs-sdk.md)



<br><br>

