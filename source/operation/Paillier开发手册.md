#  Paillier 开发手册

本文分为两部分：

1. 方案接口、工具、合约SDK介绍：对 ChainMaker 的 cmc 工具、合约 SDK 和 common 库提供的算法的介绍。
2. 用例：介绍了如何使用 ChainMaker Paillier 功能，从 Paillier 公私钥生成、编写智能合约到使用 SDK（本文使用 go 的 SDK） 进行数据的加密上链、链上同态运算和获取运算结果解密。

## 接口、工具、合约SDK介绍

`common`提供 paillier 半同态加密的基础能力。

`cmc`工具提供了生成公私钥、加解密的能力，方便用户快速体验同态加密功能。

`合约SDK`提供了链上同态运算的能力，目前支持同态运算的合约 SDK 有：Go、Rust。

### CMC 工具子命令 paillier 介绍

简介

CMC 中的 paillier 命令是用于辅助paillier算法的使用，生成公钥和私钥，并且根据参数保存到指定位置。

使用`./cmc paillier -h`获取使用帮助：

```shell
./cmc paillier -h                       
ChainMaker paillier command

Usage:
  cmc paillier [command]

Available Commands:
  genKey      Generate paillier's private, public Keys, and storage.

Flags:
  -h, --help   help for paillier

Use "cmc paillier [command] --help" for more information about a command.

```

genKey简介

`genKey`用于生成 paillier 算法的公私钥，并根据参数保存到指定位置，查看命令详情：

```shell
./cmc paillier genKey -h
generates paillier private public key

Usage:
  cmc paillier genKey [flags]

Flags:
  -h, --help          help for genKey
      --name string
      --path string   the result storage file path, and the file name is the id

```

genKey参数详解：

```
-h, --help：获取使用帮助
--name：用于保存公私钥的文件名，公钥和私钥文件名相同，后缀分别为`.prv`、`.pub`
--path：存储路径
```


### common 算法部分介绍

#### 公钥方法

PubKey 提供了序列化、反序列化、加密和同态运算方法。


**Encrypt**：加密一个big.Int类型的数得到对应的密文

Arguments:

- plaintext：用于加密的明文

return:

- Ct：加密得到的密文
- error：可能出现的错误

```go
Encrypt(plainText *big.Int) (*Ciphertext, error)
```



同态运算方法，同态运算一般在链上进行，所以一般不会使用到这几个方法，而是使用合约SDK提供的同态运算方法，这里不再详细描述。

```go
// 两个密文相加，返回结果密文
AddCiphertext(cipher1, cipher2 *Ciphertext) (*Ciphertext, error))

// 密文加明文，返回结果密文
AddPlaintext(cipher *Ciphertext, constant *big.Int) (*Ciphertext, error)

// 密文减密文，返回结果密文
SubCiphertext(cipher1, cipher2 *Ciphertext) (*Ciphertext, error)

// 密文减明文，返回结果密文
SubPlaintext(cipher *Ciphertext, constant *big.Int) (*Ciphertext, error)

// 密文乘以明文，返回结果密文
NumMul(cipher *Ciphertext, constant *big.Int)
```

#### 私钥方法

PrvKey 提供了包含 PubKey 所有的方法以及获取公钥、解密方法。



**Decrypt**：解密密文得到对应的明文

Arguments:

- ciphertext：用于解密的密文

return：

- *big.Int：解密得到的明文
- error：可能出现的错误

```go
Decrypt(ciphertext *Ciphertext) (*big.Int, error))
```



**GetPubKey**：根据私钥获取公钥

return：

- *PubKey：公钥
- error：可能出现的错误

```go
GetPubKey() (*PubKey, error)
```

#### GenKey

GenKey 用于初始化公私钥对

return：

- *PrvKey：生成的私钥（公钥可以从私钥中获取）
- error：可能出现的错误

```go
GenKey() (*PrvKey, error)
```

### 智能合约SDK

Go 和 Rust 合约SDK提供了进行同态运算的方法。

#### go

PaillierContext 接口提供了同态运算方法

**AddCiphertext**：使用公钥进行密文加密文的同态运算

Arguments:

- pubKey：公钥的字节数组
- ct1：密文字节数组
- ct1：密文字节数组

return：

- []byte结果密文字节数组
- ResultCode：函数执行状态码

~~~go
AddCiphertext(pubKey []byte, ct1 []byte, ct2 []byte) ([]byte, ResultCode)
~~~

**AddPlaintext**：使用公钥进行密文加明文的同态运算

Argumengs:

- pubKey：公钥字节数组
- ct：密文字节数组
- pt：明文，int64的字符串表示，超出int64链上执行将会报错

return：

- []byte：结果密文字节数组
- ResultCode：函数执行状态码

~~~go
AddPlaintext(pubKey, ct []byte, pt string) ([]byte, ResultCode)
~~~

**SubCiphertext**：使用公钥进行密文减密文的同态运算

Arguments:

- pubKey：公钥的字节数组
- ct1：密文字节数组
- ct1：密文字节数组

return：

- []byte：结果密文字节数组
- ResultCode：函数执行状态码


~~~go
SubCiphertext(pubKey, ct1, ct2 []byte) ([]byte, ResultCode)
~~~

**SubPlaintext**：使用公钥进行密文减明文的同态运算

Argumengs:

- pubKey：公钥字节数组
- ct：密文字节数组
- pt：明文，int64的字符串表示，超出int64链上执行将会报错

return：

- []byte：结果密文字节数组
- ResultCode：函数执行状态码


~~~go
SubPlaintext(pubKey, ct []byte, pt string) ([]byte, ResultCode)
~~~

**NumMul**：使用公钥进行密文乘明文的同态运算

Argumengs:

- pubKey：公钥字节数组
- ct：密文字节数组
- pt：明文，int64的字符串表示，超出int64链上执行将会报错

return：

- []byte：结果密文字节数组
- ResultCode：函数执行状态码


~~~go
NumMul(pubKey, ct []byte, pt string) ([]byte, ResultCode)
~~~

#### rust

rust合约与go合约相同，由trait `PaillierSimContext`提供了同态运算的方法：

**add_ciphertext**：使用公钥计算两个密文的和

Argument：

- pubKey：两个密文的公钥
- ciphertext1：密文
- ciphertext2：密文

return：

- return1: 计算结果
- return2: 函数执行状态码，0：success, 1: failed

```rust
fn add_ciphertext(
    &self,
    pubkey: Vec<u8>,
    ciphertext1: Vec<u8>,
    ciphertext2: Vec<u8>,
) -> Result<Vec<u8>, result_code>;
```

**add_plaintext**：使用公钥计算密文与明文的和

Argument：

- pubKey：两个密文的公钥
- ciphertext：密文
- plaintext：明文，i64的字符串表示，超出i64链上执行将会报错

return：

- return1: 计算结果
- return2: 函数执行状态码，0：success, 1: failed

```rust
fn add_plaintext(
    &self,
    pubkey: Vec<u8>,
    ciphertext: Vec<u8>,
    plaintext: &str,
) -> Result<Vec<u8>, result_code>;
```

**sub_ciphertext**：使用公钥计算密文减去密文

Argument：

- pubKey：两个密文的公钥
- ciphertext1：密文
- ciphertext2：密文

return：

- return1: 计算结果
- return2: 函数执行状态码，0：success, 1: failed

```rust
fn sub_ciphertext(
    &self,
    pubkey: Vec<u8>,
    ciphertext1: Vec<u8>,
    ciphertext2: Vec<u8>,
) -> Result<Vec<u8>, result_code>;
```

**sub_plaintext**：使用公钥计算密文减明文

Argument：

- pubKey：两个密文的公钥
- ciphertext：密文
- plaintext：明文，i64的字符串表示，超出i64链上执行将会报错

return：

- return1: 计算结果
- return2: 函数执行状态码，0：success, 1: failed

```rust
fn sub_plaintext(
     &self,
     pubkey: Vec<u8>,
     ciphertext: Vec<u8>,
     plaintext: &str,
) -> Result<Vec<u8>, result_code>;
```

**num_mul**：使用公钥计算密文乘明文

Argument：

- pubKey：两个密文的公钥
- ciphertext：密文
- plaintext：明文，i64的字符串表示，超出i64链上执行将会报错

return：

- return1: 计算结果
- return2: 函数执行状态码，0：success, 1: failed

```rust
fn num_mul(
    &self,
    pubkey: Vec<u8>,
    ciphertext: Vec<u8>,
    plaintext: &str,
) -> Result<Vec<u8>, result_code>;
```



## 用例

### 1. 使用cmc paillier生成并保存自己的公私钥

```sh
./cmc paillier genKey --name=test1 --path=./paillier-key
[paillier Private Key] storage file path: paillier-key/test1.prvKey
[paillier Public Key] storage file path: paillier-key/test1.pubKey
```

会在当前目录生成，paillier-key文件夹，来保存生成的公私钥文件，如下：

```shell
tree ./paillier-key
./paillier-key
├── test1.prvKey
└── test1.pubKey
```



### 2. 编写智能合约

go：

```go
package main

import (
	"encoding/base64"
	"strconv"
)

// 安装合约时会执行此方法，必须
//export init_contract
func initContract() {
	// 此处可写安装合约的初始化逻辑

}

// 升级合约时会执行此方法，必须
//export upgrade
func upgrade() {
	// 此处可写升级合约的逻辑

}

//export paillier_test_set
func paillier_test_set() {
	pubkeyBytes, _ := Arg("pubkey")
	handletype, _ := Arg("handletype")
	encodePara1, _ := Arg("para1")
	encodePara2, _ := Arg("para2")

	para1Bytes, _ := base64.StdEncoding.DecodeString(encodePara1)
	var result_code ResultCode
	var result_data []byte
	var result_data_str string
	test := NewPaillierContext()
	if handletype == "AddCiphertext" {
		para2Bytes, _ := base64.StdEncoding.DecodeString(encodePara2)
		result_data, result_code = test.AddCiphertext([]byte(pubkeyBytes), para1Bytes, para2Bytes)
	} else if handletype == "AddPlaintext" {
		result_data, result_code = test.AddPlaintext([]byte(pubkeyBytes), para1Bytes, encodePara2)
	} else if handletype == "SubCiphertext" {
		para2Bytes, _ := base64.StdEncoding.DecodeString(encodePara2)
		result_data, result_code = test.SubCiphertext([]byte(pubkeyBytes), para1Bytes, para2Bytes)
	} else if handletype == "SubPlaintext" {
		result_data, result_code = test.SubPlaintext([]byte(pubkeyBytes), para1Bytes, encodePara2)
	} else if handletype == "NumMul" {
		result_data, result_code = test.NumMul([]byte(pubkeyBytes), para1Bytes, encodePara2)
	} else {
		ErrorResult("finish paillier_test_set failure: error para: " + handletype)
	}
    
	if result_code != SUCCESS {
		ErrorResult("finish paillier_test_set failure: error result code: " + string(result_code))
	}

	result_data_str = base64.StdEncoding.EncodeToString(result_data)

	result := PutState("paillier_test", handletype, result_data_str)
	if result_code == 0 {
		SuccessResult("finish paillier_test_set success")
	} else {
		ErrorResult("finish paillier_test_set failure")
	}
}

//export paillier_test_get
func paillier_test_get() {
	handletype, _ := Arg("handletype")
	value, result := GetState("paillier_test", handletype)
	SuccessResult(value)
}

//export bulletproofs_test_set
func bulletproofs_test_set() {
	LogMessage("[bulletproofs] ========================================start")
	LogMessage("[bulletproofs] bulletproofs_test_set")
	handleType, _ := Arg("handletype")
	param1, _ := Arg("para1")
	param2, _ := Arg("para2")

	param1Bytes, _ := base64.StdEncoding.DecodeString(param1)
	var result_code ResultCode
	var result_data []byte
	var result_data_str string
	bulletproofsContext := NewBulletproofsContext()
	switch handleType {
	case BulletproofsOpTypePedersenAddNum:
		result_data, result_code = bulletproofsContext.PedersenAddNum(param1Bytes, param2)
	case BulletproofsOpTypePedersenAddCommitment:
		param2Bytes, _ := base64.StdEncoding.DecodeString(param2)
		result_data, result_code = bulletproofsContext.PedersenAddCommitment(param1Bytes, param2Bytes)
	case BulletproofsOpTypePedersenSubNum:
		result_data, result_code = bulletproofsContext.PedersenSubNum(param1Bytes, param2)
	case BulletproofsOpTypePedersenSubCommitment:
		param2Bytes, _ := base64.StdEncoding.DecodeString(param2)
		result_data, result_code = bulletproofsContext.PedersenSubCommitment(param1Bytes, param2Bytes)
	case BulletproofsOpTypePedersenMulNum:
		result_data, result_code = bulletproofsContext.PedersenMulNum(param1Bytes, param2)
	case BulletproofsVerify:
		param2Bytes, _ := base64.StdEncoding.DecodeString(param2)
		result_data, result_code = bulletproofsContext.Verify(param1Bytes, param2Bytes)
	default:
		ErrorResult("bulletproofs_test_set failed, error: " + handleType)
		result_code = 1
	}

	if result_code != SUCCESS {
		ErrorResult("bulletproofs_test_set failed, error: " + string(rune(result_code)))
	}

	result_data_str = base64.StdEncoding.EncodeToString(result_data)

	result := PutState("bulletproofs_test", handleType, result_data_str)
    
	if result_code == 0 {
		SuccessResult("bulletproofs_test_set success")
	} else {
		ErrorResult("bulletproofs_test_set failure")
	}
}

//export bulletproofs_test_get
func bulletproofs_test_get() {
	handletype, _ := Arg("handletype")
	value, result := GetState("bulletproofs_test", handletype)
    
	if handletype == "BulletproofsVerify" {
		decodeValue, err := base64.StdEncoding.DecodeString(value)
		if err != nil {
			ErrorResult("base64.StdEncoding.DecodeString(value) failed")
		}
		LogMessage(handletype)
		SuccessResult(string(decodeValue))
	} else {
		LogMessage(handletype)
		SuccessResult(value)
	}
}

func main() {

}
```

编译生成wasm文件：

```shell
tinygo build -no-debug -opt=s -o contract-paillier.wasm -target wasm
```



rust:

```rust
// 安装合约时会执行此方法，必须
#[no_mangle]
pub extern "C" fn init_contract() {
    // 安装时的业务逻辑，可为空
    sim_context::log("init_contract");
}

// 升级合约时会执行此方法，必须
#[no_mangle]
pub extern "C" fn upgrade() {
    // 升级时的业务逻辑，可为空
    sim_context::log("upgrade success");
}

#[no_mangle]
pub extern "C" fn paillier_test_set() {
    sim_context::log("[paillier] ========================================start");
    sim_context::log("[paillier] input func: paillier_test_set");

    let ctx = sim_context::get_sim_context();
    let pubkey = ctx.arg_default_blank("pubkey");

    let handletype = ctx.arg_default_blank("handletype");

    let para1 = ctx.arg_default_blank("para1");
    let decode_para1 = decode(para1.as_bytes()).unwrap();

    let para2 = ctx.arg_default_blank("para2");

    let test = ctx.get_paillier_sim_context();
    let r: Result<Vec<u8>, i32>;
    if handletype == "AddCiphertext" {
        let decode_para2 = decode(para2.as_bytes()).unwrap();
        r = test.add_ciphertext(pubkey.into_bytes(), decode_para1, decode_para2);
    } else if handletype == "AddPlaintext" {
        r = test.add_plaintext(pubkey.into_bytes(), decode_para1, &para2);
    } else if handletype == "SubCiphertext" {
        let decode_para2 = decode(para2.as_bytes()).unwrap();
        r = test.sub_ciphertext(pubkey.into_bytes(), decode_para1, decode_para2);
    } else if handletype == "SubPlaintext" {
        r = test.sub_plaintext(pubkey.into_bytes(), decode_para1, &para2);
    } else if handletype == "NumMul" {
        r = test.num_mul(pubkey.into_bytes(), decode_para1, &para2);
    } else {
        ctx.error(&format!(
            "finish paillier_test_set failure: error para: {}",
            handletype
        ));
        return;
    }
    if r.is_err() {
        ctx.error("finish paillier_test_set failure");
        return;
    }

    let data = r.unwrap();
    let data_u8 = data.as_slice();
    let data_str = encode(data_u8);

    let put_code = ctx.put_state("paillier_test", &handletype, data_str.as_bytes());
    ctx.ok("finish paillier_test_set success".as_bytes());
}

#[no_mangle]
pub extern "C" fn paillier_test_get() {
    let ctx = sim_context::get_sim_context();
    let handletype = ctx.arg_default_blank("handletype");
    let r = ctx.get_state("paillier_test", &handletype);
    if r.is_err() {
        sim_context::log("[zitao] paillier_test_get error");
        ctx.error("finish paillier_test_get failure");
        return;
    }
    let data = r.unwrap();

    let result = String::from_utf8(data);
    let result_str = result.unwrap();

    ctx.ok(result_str.as_bytes());
}
```

编译生成wasm字节码文件

```shell
make build
```



### 3. 使用SDK编写测试用例

> SDK并未提供 paillier 相关接口，开发者需要直接调用common库中的 *chainmaker.org/sdk-go/common/crypto/paillier*包。

总测试函数：

```go
const (
	sdkConfigOrg1Client1Path = "../sdk_configs/sdk_config_org1_client1.yml"

	createContractTimeout = 5
)

const (

	// go 合约
	paillierContractName = "pailliergo100001"
	paillierByteCodePath = "../../testdata/paillier-wasm-demo/contract-paillier.wasm"
	runtime              = common.RuntimeType_GASM

	// rust 合约
	//paillierContractName = "paillier-rust-10001"
	//paillierByteCodePath = "./testdata/counter-go-demo/chainmaker_contract.wasm"
	//runtime              = common.RuntimeType_WASMER

	paillierPubKeyFilePath = "../../testdata/paillier-key/test1.pubKey"
	paillierPrvKeyFilePath = "../../testdata/paillier-key/test1.prvKey"
)

func main() {
	TestPaillierContractCounterGo()
}

func TestPaillierContractCounterGo() {
	t := new(testing.T)
	client, err := examples.CreateChainClientWithSDKConf(sdkConfigOrg1Client1Path)
	require.Nil(t, err)

	fmt.Println("======================================= 创建合约（异步）=======================================")
	testPaillierCreate(client, examples.UserNameOrg1Admin1, examples.UserNameOrg2Admin1, examples.UserNameOrg3Admin1, examples.UserNameOrg4Admin1, false)
	time.Sleep(5 * time.Second)

	fmt.Println("======================================= 调用合约运算（异步）=======================================")
	testPaillierOperation(client, "paillier_test_set", false)
	time.Sleep(5 * time.Second)

	fmt.Println("======================================= 查询结果并解密（异步）=======================================")
	testPaillierQueryResult(t, client, "paillier_test_get")
}
```

创建合约：

```go
// 创建合约
func testPaillierCreate(client *sdk.ChainClient, admin1, admin2, admin3,
	admin4 string, withSyncResult bool) {
	resp, err := createUserContract(client, admin1, admin2, admin3, admin4,
		paillierContractName, examples.Version, paillierByteCodePath, runtime, []*common.KeyValuePair{}, withSyncResult)
	if err != nil {
		log.Fatalln(err)
	}

	fmt.Printf("CREATE contract-hibe-1 contract resp: %+v\n", resp)
}

func createUserContract(client *sdk.ChainClient, admin1, admin2, admin3, admin4 string,
	contractName, version, byteCodePath string, runtime common.RuntimeType, kvs []*common.KeyValuePair, withSyncResult bool) (*common.TxResponse, error) {

	payload, err := client.CreateContractCreatePayload(contractName, version, byteCodePath, runtime, kvs)
	if err != nil {
		return nil, err
	}

	endorsers, err := examples.GetEndorsers(payload, admin1, admin2, admin3, admin4)
	if err != nil {
		return nil, err
	}

	resp, err := client.SendContractManageRequest(payload, endorsers, createContractTimeout, withSyncResult)
	if err != nil {
		return nil, err
	}

	return resp, nil
}
```



调用合约方法进行链上同态运算：

```go
// 调用合约进行同态运算
func testPaillierOperation(client *sdk.ChainClient, s string, b bool) {
	pubKeyBytes, err := ioutil.ReadFile(paillierPubKeyFilePath)
	//require.Nil(t, err)
	if err != nil {
		log.Fatalln(err)
	}
	payloadParams, err := CreatePaillierTransactionPayloadParams(pubKeyBytes, 1, 1000000)
	resp, err := client.InvokeContract(paillierContractName, s, "", payloadParams, -1, b)
	//require.Nil(t, err)
	if err != nil {
		log.Fatalln(err)
	}

	if resp.Code != common.TxStatusCode_SUCCESS {
		fmt.Printf("invoke contract failed, [code:%d]/[msg:%s]\n", resp.Code, resp.Message)
	}

}

func CreatePaillierTransactionPayloadParams(pubKeyBytes []byte, plaintext1, plaintext2 int64) ([]*common.KeyValuePair, error) {
	pubKey := new(paillier.PubKey)
	err := pubKey.Unmarshal(pubKeyBytes)
	if err != nil {

	}

	pt1 := new(big.Int).SetInt64(plaintext1)
	ciphertext1, err := pubKey.Encrypt(pt1)
	if err != nil {
		return nil, err
	}
	ct1Bytes, err := ciphertext1.Marshal()
	if err != nil {
		return nil, err
	}

	prv2, _ := paillier.GenKey()

	pub2, _ := prv2.GetPubKey()
	_, _ = pub2.Marshal()

	pt2 := new(big.Int).SetInt64(plaintext2)
	ciphertext2, err := pubKey.Encrypt(pt2)
	if err != nil {
		return nil, err
	}
	ct2Bytes, err := ciphertext2.Marshal()
	if err != nil {
		return nil, err
	}

	ct1Str := base64.StdEncoding.EncodeToString(ct1Bytes)
	ct2Str := base64.StdEncoding.EncodeToString(ct2Bytes)

	payloadParams := []*common.KeyValuePair{
		{
			Key:   "handletype",
			Value: []byte("SubCiphertext"),
		},
		{
			Key:   "para1",
			Value: []byte(ct1Str),
		},
		{
			Key:   "para2",
			Value: []byte(ct2Str),
		},
		{
			Key:   "pubkey",
			Value: pubKeyBytes,
		},
	}
	/*
		old
		payloadParams := make(map[string]string)
		//payloadParams["handletype"] = "AddCiphertext"
		//payloadParams["handletype"] = "AddPlaintext"
		payloadParams["handletype"] = "SubCiphertext"
		//payloadParams["handletype"] = "SubCiphertextStr"
		//payloadParams["handletype"] = "SubPlaintext"
		//payloadParams["handletype"] = "NumMul"

		payloadParams["para1"] = ct1Str
		payloadParams["para2"] = ct2Str
		payloadParams["pubkey"] = string(pubKeyBytes)
	*/

	return payloadParams, nil
}
```

查询运算结果并解密：

```go
// 查询同态执行结果并解密
func testPaillierQueryResult(t *testing.T, c *sdk.ChainClient, s string) {
	//paillierMethod := "AddCiphertext"
	//paillierMethod := "AddPlaintext"
	paillierMethod := "SubCiphertext"
	//paillierMethod := "SubCiphertextStr"
	//paillierMethod := "SubPlaintext"
	params1, err := QueryPaillierResult(c, paillierContractName, s, paillierMethod, -1, paillierPrvKeyFilePath)
	require.Nil(t, err)
	fmt.Printf("QUERY %s contract resp -> encrypt(cipher 10): %d\n", paillierContractName, params1)
}

func QueryPaillierResult(c *sdk.ChainClient, contractName, method, paillierDataItemId string, timeout int64, paillierPrvKeyPath string) (int64, error) {

	resultStr, err := QueryPaillierResultById(c, contractName, method, paillierDataItemId, timeout)
	if err != nil {
		return 0, err
	}

	ct := new(paillier.Ciphertext)
	resultBytes, err := base64.StdEncoding.DecodeString(string(resultStr))
	if err != nil {
		return 0, err
	}
	err = ct.Unmarshal(resultBytes)
	if err != nil {
		return 0, err
	}

	prvKey := new(paillier.PrvKey)

	prvKeyBytes, err := ioutil.ReadFile(paillierPrvKeyPath)
	if err != nil {
		return 0, fmt.Errorf("open paillierKey file failed, [err:%s]", err)
	}

	err = prvKey.Unmarshal(prvKeyBytes)
	if err != nil {
		return 0, err
	}

	decrypt, err := prvKey.Decrypt(ct)
	if err != nil {
		return 0, err
	}

	return decrypt.Int64(), nil
}

func QueryPaillierResultById(c *sdk.ChainClient, contractName, method, paillierMethod string, timeout int64) ([]byte, error) {
	pairs := []*common.KeyValuePair{
		{Key: "handletype", Value: []byte(paillierMethod)},
	}

	/*
		old
		pairsMap := make(map[string]string)
		pairsMap["handletype"] = paillierMethod
	*/

	resp, err := c.QueryContract(contractName, method, pairs, timeout)
	if err != nil {
		return nil, err
	}

	result := resp.ContractResult.Result

	return result, nil
}

```

执行测试，结果如下：

```go
=== RUN   TestPaillierContractCounterGo
======================================= 创建合约（异步）=======================================
CREATE contract-paillier-1 contract resp: message:"OK" contract_result:<result:"5aa609367c8342e08459ec1ec1321b954d88569399284277bcc5887b72d8d2c5" message:"OK" > 
======================================= 调用合约运算（异步）=======================================
invoke contract success, resp: [code:0]/[msg:OK]/[txId:81ac75ccc8914ce694a0df6ddebe2c5fddea319d784a461cb52ec9b5fc373125]
======================================= 查询结果并解密（异步）=======================================
QUERY paillier-rust-10001 contract resp -> encrypt(cipher 10): -999999
--- PASS: TestPaillierContractCounterGo (19.23s)
PASS
```
