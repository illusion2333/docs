# SDK

## 概述

长安链`SDK`是业务模块与长安链交互的桥梁，支持双向`TLS`认证，提供安全可靠的加密通信信道。

长安链提供了多种语言的`SDK`，包括：`Go SDK`、`Java SDK`、`Python SDK`（实现中）、`JS SDK`（实现中）方便开发者根据需要进行选用。

提供的`SDK`接口，覆盖合约管理、链配置管理、证书管理、多签收集、各类查询操作、事件订阅等场景，满足了不同的业务场景需要。

## 约定概念

- **`Node`（节点）**：代表一个链节点的基本信息，包括：节点地址、连接数、是否启用`TLS`认证等信息
- **`ChainClient`（链客户端）**：所有客户端对链节点的操作接口都来自`ChainClient`
- **压缩证书**：可以为`ChainClient`开启证书压缩功能，开启后可以减小交易包大小，提升处理性能

## 开发指南

### Go SDK

#### 环境依赖

**golang**

版本为1.15或以上

下载地址：https://golang.org/dl/

若已安装，请通过命令查看版本：

```bash
$ go version
go version go1.15 linux/amd64
```

#### 下载安装

```bash
$ git clone --recursive https://git.chainmaker.org.cn/chainmaker/chainmaker-sdk-go.git
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

##### 创建合约

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

##### 调用合约

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

> openjdk 1.8.151+

下载地址：https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html

若已安装，请通过命令查看版本：

```bash
$ java -version
java version "1.8.0_281"
```

#### 下载安装

```bash
$ git clone https://git.code.tencent.com/ChainMaker/chainmaker-sdk-java.git
```

#### 示例代码

##### 创建节点

> 更多内容请参看：`TestBase`

```java
// 创建节点
Node node = Node.builder()
                .clientKeyBytes(keyBytes)
                .clientCertBytes(certBytes)
                .tlsCertBytes(tlsCertBytes)
                .hostname(TLS_HOST_NAME1)
                .grpcUrl(NODE_GRPC_URL1)
                .sslProvider(OPENSSL_PROVIDER)
                .negotiationType(TLS_NEGOTIATION).build();
```

##### 创建ChainClient

> 更多内容请参看：`TestBase`

```java
// 创建ChainClient
chainManager = ChainManager.getInstance();
chainClient = chainManager.getChainClient(CHAIN_ID);
if (chainClient == null) {
  chainClient = chainManager.createChainClient(CHAIN_ID, ORG_ID, FileUtils.getResourceFileBytes(USER_KEY_PATH),
  FileUtils.getResourceFileBytes(USER_CERT_PATH), chainNode);
}

```

##### 创建合约

> 更多内容请参看：`TestUserContract`

```java
   public void testCreateContract() throws IOException, InterruptedException, ExecutionException, TimeoutException {
       byte[] byteCode = FileUtils.getResourceFileBytes(CONTRACT_FILE_PATH);
   
       // 1. create payload
       byte[] payload = chainClient.createPayloadOfContractCreation(CONTRACT_NAME,
               "1", Contract.RuntimeType.WASMER_RUST, null, byteCode);
   
       // 2. create payloads with endorsement
       byte[] payloadWithEndorsement1 = chainClient.signPayloadOfContractMgmt(payload);
   
       // 3. merge endorsements using payloadsWithEndorsement
       byte[][] payloadsWithEndorsement = new byte[1][];
       payloadsWithEndorsement[0] = payloadWithEndorsement1;
       byte[] payloadWithEndorsement = chainClient.mergeSignedPayloadsOfContractMgmt(payloadsWithEndorsement);
   
       // 4. create contract
       ResponseInfo responseInfo = chainClient.createContract(payloadWithEndorsement, 5000, 5000);
   }
```

##### 调用合约

> 更多内容请参看：`TestUserContract`

```java
   public void testInvokeContract() throws Exception {

       Map<String, String> params = new HashMap<>();
       params.put("time", System.currentTimeMillis()+"");
       params.put("file_hash", UUID.randomUUID().toString());
       params.put("file_name", UUID.randomUUID().toString()+System.currentTimeMillis());
       ResponseInfo responseInfo = chainClient.invokeContract(QUERY_CONTRACT_NAME, QUERY_CONTRACT_METHOD, params, 5000, 5000);
   }
```

##### 更多示例和用法

> 更多示例和用法，请参看单元测试用例

| 功能     | 单测代码                      |
| -------- | ----------------------------- |
| 基础配置 | `TestBase`   |
| 用户合约 | `TestUserContract`   |
| 系统合约 | `TestSystemContract` |
| 链配置   | `TestChainConfig`    |
| 证书管理 | `TestBaseCertManage`     |
| 消息订阅 | `TestSubscribe`       |

#### 接口说明

请参看：[《chainmaker-java-sdk》](../dev/chainmaker-java-sdk.md)