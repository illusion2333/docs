# Bulletproofs 开发手册

本文分为两部分：

1. bulletproofs 相关方法、工具介绍：对 ChainMaker 的 cmc 工具、合约 SDK、和 common 库提供的算法介绍。
2. 用例：介绍了如何使用 ChainMaker Bulletproofs，从数据加密、编写智能合约到使用 SDK（本文使用 Go SDK）发送交易进行介绍。

## 接口、工具、合约SDK介绍

common 提供 Bulletproofs 零知识范围证明的基础能力。

CMC 工具提供生成 opening、commitment、proof 和对同态运算结果生成 proof 的功能。

合约 SDK 提供了 commitment 链上同态运算和验证证明的接口方法，目前支持 Bulletproofs 算法的合约 SDK 有：Go、Rust。

### common算法部分介绍

#### Prove 相关函数

ProveRandomOpening：使用随机生成的 opening 生成对 X 的承诺和证明

Arguments:

- x：原始数值

return:

- []byte：proof
- []byte：commitment
- []byte：opening
- error：可能出现的错误

```go
ProveRandomOpening(x uint64) ([]byte, []byte, []byte, error)
```



ProveSpecificOpening：使用指定的 opening 生成对 X 的承诺和证明

Arguments:

- x：原始数值
- opening：指定的opening

return:

- []byte：proof
- []byte：commitment
- error：可能出现的错误

```go
ProveSpecificOpening(x uint64, opening []byte) ([]byte, []byte, error)
```



Verify：验证证明 proof 的有效性

Arguments:

- proof：对承诺隐藏的数值在范围 [0,2^64) 内的证明
- commitment：对隐藏数值的承诺

return:

- bool：返回 true 验证通过，证明 commitment 的原始值在[0,2^64)范围内，否则返回 false。
- error：可能出现的错误

```go
Verify(proof []byte, commitment []byte) (bool, error)
```



ProveAfterAddNum：使用相同的 opening 对运算后的结果 commitment 生成 proof

```go
ProveAfterAddNum(x, y uint64, openingX, commitmentX []byte) ([]byte, []byte, error)
```



ProveAfterCommitment：使用两个 opening 的和对运算后的结果 commitment 生成 proof

```go
ProveAfterAddCommitment(x, y uint64, openingX, openingY, commitmentX, commitmentY []byte) ([]byte, []byte, []byte, error)
```



ProveAfterSubNum：使用相同的 opening 对运算后的结果 commitment 生成 proof

```go
ProveAfterSubNum(x, y uint64, openingX, commitmentX []byte) ([]byte, []byte, error)
```



ProveAfterSubCommitment：使用两个 opening 的和对运算后的结果 commitment 生成 proof

```go
ProveAfterSubCommitment(x, y uint64, openingX, openingY, commitmentX, commitmentY []byte) ([]byte, []byte, []byte, error)
```

ProveAfterMulNum：使用 opening 与 y 的乘积对 commitment 与 y 的乘积生成 proof

```go
ProveAfterMulNum(x, y uint64, openingX, commitmentX []byte) ([]byte, []byte, []byte, error)
```



#### Commitment 相关函数

CommitmentOps 的大部分开发是需要用到的接口都是集成在链上，通过合约 SDK 提供能力，这里仅介绍一个获取 opening 的接口方法：PedersenRNG。

PedersenRNG：生成一个真正的随机标量，作为生成承诺的 opening

return:

- []byte：opening
- error：可能出现的错误

```go
PedersenRNG() ([]byte, error)
```


### CMC工具子命令 bulletproofs 介绍

> CMC 默认是没有 bulletproofs 相关功能的，需要自己安装好前置依赖库之后，在*chainmaker-go/tools/cmc*目录下进行编译：
> `go build -tags=bulletproofs -mod=mod`

简介

CMC 中的 paillier 命令是在不写客户端代码的情况下快速体验 bulletproofs 功能的辅助命令，用于数据的初始化、链下同态运算和对数据生成proof。

使用`./cmc paillier -h`查看使用帮助：

```shell
./cmc bulletproofs -h
ChainMaker bulletproofs command

Usage:
  cmc bulletproofs [command]

Available Commands:
  genOpening     Bulletproofs generate opening command
  pedersenVerify Bulletproofs pedersenVerify command
  prove          Bulletproofs prove command
  proveMethod    Bulletproofs proveMethod command

Flags:
  -h, --help   help for bulletproofs

Use "cmc bulletproofs [command] --help" for more information about a command.
```

genOpening 简介

genOpening 生成 opening 并输出到终端，查看命令使用帮助：

```shell
./cmc bulletproofs genOpening -h
Bulletproofs generate opening command

Usage:
  cmc bulletproofs genOpening [flags]

Flags:
  -h, --help   help for genOpening
```

prove 简介

prove 对一个数（在[0, 2^64)内的数）生成 commitment、proof 和 opening，opening 可指定，查看命令使用帮助：

```shell
./cmc bulletproofs prove -h
Bulletproofs prove command

Usage:
  cmc bulletproofs prove [flags]

Flags:
  -h, --help             help for prove
      --opening string   opening
      --value int        value (default -1)
```

参数介绍：

```
--opening：指定生成 commitment 和 proof 的 opening，若干不提供该参数，则会使用随机生成的 opening。
--value：要证明的值，证明的范围是 [0,2^64)，这里支持 [0,2^63)
```

proveMethod 简介

proveMethod 在需要对链上一个结果提供证明时，可以在链下使用该命令，进行相同的同态计算，并对结果生成 proof，以便发送到链上同链上结果进行验证。查看命令使用帮助：

```shell
./cmc bulletproofs proveMethod -h
Bulletproofs proveMethod command

Usage:
  cmc bulletproofs proveMethod [flags]

Flags:
      --commitmentX string
      --commitmentY string
  -h, --help                 help for proveMethod
      --method string        prove method: ProveAfterAddNum ProveAfterAddCommitment ProveAfterSubNum ProveAfterSubCommitment ProveAfterMulNum
      --openingX string
      --openingY string
      --valueX int           valueY (default -1)
      --valueY int           valueY (default -1)
```

参数说明：
```
--method：要调用的方法名，支持使用帮助中列出的方法
--valueX：原始值X
--valueY：原始值Y
--commitmentX：对X的承诺（必填，当同一个数进行运算时，默认会使用 commitmentX 的值，而不会去尝试使用 commitmentY 的值）
--commitmentY：对Y的承诺（可选，当两个commitment进行同态运算时，需要提供）
--openingX：值X使用的 opening（必填，当同一个数进行运算时，默认会使用 openingX 的值）
--openingY：值Y使用的 opening (可选，当两个 commitment 进行同态运算是，需要提供)
```

pedersenVerify 简介

pedersenVerify 在链下用于校验 opening、commitment 和原始值之间是否有绑定关系。查看命令使用帮助：

```shell
./cmc bulletproofs pedersenVerify -h
Bulletproofs pedersenVerify command

Usage:
  cmc bulletproofs pedersenVerify [flags]

Flags:
      --commitment string   commitment
  -h, --help                help for pedersenVerify
      --opening string      opening
      --value int           value (default -1)
```

参数介绍：
```
--value：原始值X
--commitment：对 X 的承诺 commitment
--opening：生成 commitment 时使用的 opening
```

### 智能合约SDK

#### go

BulletproofsContext 接口提供了 bulletproofs 同态运算和验证的能力，接口方法如下：

PedersenAddNum：无须知道 commitment 的原始值 X，计算 X+Y 的 commitment

Arguments:

- commitment：用于隐藏实际数值 X 的 commitment
- num：Y，uint64 的字符串表示，超出 uint64 范围在计算时会返回错误

return:

- []byte：结果 commitment
- ResultCode：函数执行状态码，0: success, 1: failed

```go
PedersenAddNum(commitment []byte, num string) ([]byte, ResultCode)
```



PedersenAddCommitment：无须知道 commitment1、commitment2 的原始值 X、Y，计算 X+Y 的 commitment

Arguments:

- commitment1：用于隐藏实际数值 X 的 commitment
- commitment2：用于隐藏实际数值 Y 的 commitment

return:

- []byte：结果 commitment
- ResultCode：函数执行状态码，0: success, 1: failed

```go
PedersenAddCommitment(commitment1, commitment2 []byte) ([]byte, ResultCode)
```



PedersenSubNum：无须知道 commitment 的原始值 X，计算 X-Y 的 commitment

Arguments:

- commitment：用于隐藏实际数值 X 的 commitment
- num：Y，uint64 的字符串表示，超出 uint64 范围在计算时会返回错误

return:

- []byte：结果 commitment
- ResultCode：函数执行状态码，0: success, 1: failed

```go
PedersenSubNum(commitment []byte, num string) ([]byte, ResultCode)
```



PedersenSubCommitment：无需知道 commitment1、commitment2 的原始值 X、Y，计算 X-Y 的 commitment

Arguments:

- commitment1：用于隐藏实际数值 X 的 commitment
- commitment2：用于隐藏实际数值 Y 的 commitment

return:

- []byte：结果 commitment
- ResultCode：函数执行状态码，0: success, 1: failed

```go
PedersenSubCommitment(commitment1, commitment2 []byte) ([]byte, ResultCode)
```



PedersenMulNum：无须知道 commitment 的原始值 X，计算 X*Y 的 commitment

Arguments:

- commitment：用于隐藏实际数值 X 的 commitment
- num：Y，uint64 的字符串表示，超出 uint64 范围在计算时会返回错误

return:

- []byte：结果 commitment
- ResultCode：函数执行状态码，0: success, 1: failed

```go
PedersenMulNum(commitment []byte, num string) ([]byte, ResultCode)
```



Verify：验证证明 proof 的有效性

Arguments：

- proof：对承诺隐藏的数值在范围 [0,2^64) 内的证明
- commitment：对隐藏数值的承诺

result:

- []byte：验证正确时返回字符串"1"的字节数组，失败时返回字符串"0"的字节数组
- ResultCode：函数执行状态码，0: success, 1: failed

```go
Verify(proof, commitment []byte) ([]byte, ResultCode)
```



#### rust

trait方法
pedersen_add_num：无须知道 commitment 的原始值 X，计算 X+Y 的commitment

Arguments:

- commitment：用于隐藏实际数值X的 commitment
- num：Y，u64 的字符串表示，超出 u64 范围在计算时会返回错误

return:

- return1：结果commitment
- return2：函数执行状态码，0: success, 1: failed

```rust
fn pedersen_add_num(&self, commitment: Vec<u8>, num: &str) -> Result<Vec<u8>, result_code>;
```



pedersen_add_commitment：无须知道 commitment1、commitment2 的原始值 X、Y，计算 X+Y 的 commitment

Arguments:

- commitment1：用于隐藏实际数值 X 的 commitment
- commitment2：用于隐藏实际数值 Y 的 commitment

return:

- return1：结果commitment
- return2：函数执行状态码，0: success, 1: failed

```rust
fn pedersen_add_commitment(
    &self,
    commitment1: Vec<u8>,
    commitment2: Vec<u8>,
) -> Result<Vec<u8>, result_code>;
```



**pedersen_sub_num**：无须知道 commitment 的原始值 X，计算 X-Y 的 commitment

Arguments:

- commitment：用于隐藏实际数值 X 的 commitment
- num：Y，u64 的字符串表示，超出 u64 范围在计算时会返回错误

return:

- return1：结果commitment
- return2：函数执行状态码，0: success, 1: failed

```rust
fn pedersen_sub_num(&self, commitment: Vec<u8>, num: &str) -> Result<Vec<u8>, result_code>;
```



PedersenSubCommitment：无须知道 commitment1、commitment2 的原始值 X、Y，计算 X-Y 的 commitment

Arguments:

- commitment1：用于隐藏实际数值 X 的 commitment
- commitment2：用于隐藏实际数值 Y 的 commitment

return:

- return1：结果commitment
- return2：函数执行状态码，0: success, 1: failed

```rust
fn pedersen_sub_commitment(
    &self,
    commitment1: Vec<u8>,
    commitment2: Vec<u8>,
) -> Result<Vec<u8>, result_code>;
```



pedersen_mul_num：无须知道 ommitment 的原始值 X，计算 X-Y 的 commitment

Arguments:

- commitment：用于隐藏实际数值 X 的 commitment
- num：Y，u64 的字符串表示，超出 u64 范围在计算时会返回错误

return:

- return1：结果commitment
- return2：函数执行状态码，0: success, 1: failed

```rust
fn pedersen_mul_num(&self, commitment: Vec<u8>, num: &str) -> Result<Vec<u8>, result_code>;
```



**verify**：验证证明 proof 的有效性

Arguments：

- proof：对 commitment 在范围 [0,2^64) 内的证明
- commitment：对隐藏数值的承诺

result:

- return1：验证正确时返回字符串"1"的字节数组，失败时返回字符串"0"的字节数组
- return2：函数执行状态码，0: success, 1: failed

```rust
fn verify(&self, proof: Vec<u8>, commitment: Vec<u8>) -> Result<Vec<u8>, result_code>;
```



## 用例

### 1、编写智能合约

#### go

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
	LogMessage("[bulletproofs] PutState: key=bulletproofs_test" + ",name=" + handleType + ",value=" + result_data_str + ",result:" + strconv.FormatInt(int64(result), 10))
	LogMessage("[bulletproofs] ========================================end")
	if result_code == 0 {
		SuccessResult("bulletproofs_test_set success")
	} else {
		ErrorResult("bulletproofs_test_set failure")
	}
}

//export bulletproofs_test_get
func bulletproofs_test_get() {
	LogMessage("[bulletproofs] ========================================start")
	LogMessage("[bulletproofs] input func: paillier_test_get")
	handletype, _ := Arg("handletype")
	value, result := GetState("bulletproofs_test", handletype)
	LogMessage("[bulletproofs] GetState: key=bulletproofs_test" + ",name=" + handletype + ",value=" + value + ",result:" + strconv.FormatInt(int64(result), 10))
	LogMessage("[bulletproofs] change event_test_get[value]: " + value)
	LogMessage("[bulletproofs] ========================================end")
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

编译得到*contract-bulletproofs.wasm*字节码文件：

```shell
tinygo build -no-debug -opt=s -o contract-bulletproofs.wasm -target wasm
```



#### rust

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
pub extern "C" fn bulletproofs_test_set() {
    sim_context::log("[bulletproofs] ========================================start");
    sim_context::log("[bulletproofs] bulletproofs_test_set");

    let ctx = sim_context::get_sim_context();
    let handle_type = ctx.arg_default_blank("handletype");
    let para1 = ctx.arg_default_blank("para1");
    let decode_para1 = decode(para1.as_bytes()).unwrap();
    let para2 = ctx.arg_default_blank("para2");
    // let decode_para2 = decode(para2.as_bytes()).unwrap();

    sim_context::log(&format!(
        "[bulletproofs] bulletproofs_test_set [handletype]: {}",
        handle_type
    ));

    sim_context::log(&format!(
        "[bulletproofs] bulletproofs_test_set [para1]: {}",
        para1
    ));

    sim_context::log(&format!(
        "[bulletproofs] bulletproofs_test_set [para2]: {}",
        para2
    ));

    let test = ctx.get_bulletproofs_sim_context();
    let result: Result<Vec<u8>, i32>;

    if handle_type == "PedersenAddNum" {
        // let decode_para2: u64 = para2.parse().unwrap();
        // let decode_para2 = para2.parse::<u64>().unwrap();
        result = test.pedersen_add_num(decode_para1, &para2)
    } else if handle_type == "PedersenAddCommitment" {
        let decode_para2 = decode(para2.as_bytes()).unwrap();
        result = test.pedersen_add_commitment(decode_para1, decode_para2)
    } else if handle_type == "PedersenSubNum" {
        // let decode_para2: u64 = para2.parse().unwrap();
        result = test.pedersen_sub_num(decode_para1, &para2)
    } else if handle_type == "PedersenSubCommitment" {
        let decode_para2 = decode(para2.as_bytes()).unwrap();
        result = test.pedersen_sub_commitment(decode_para1, decode_para2)
    } else if handle_type == "PedersenMulNum" {
        // let decode_para2: u64 = para2.parse().unwrap();
        result = test.pedersen_mul_num(decode_para1, &para2)
    } else if handle_type == "BulletproofsVerify" {
        let decode_para2 = decode(para2.as_bytes()).unwrap();
        result = test.verify(decode_para1, decode_para2)
    } else {
        ctx.error(&format!(
            "finish event_test_set failure: error para: {}",
            handle_type
        ));
        return;
    }

    if result.is_err() {
        ctx.error("finish event_test_set failure");
        return;
    }

    let data = result.unwrap();
    let data_u8 = data.as_slice();
    let data_str = encode(data_u8);
    let put_code = ctx.put_state("bulletproofs_test", &handle_type, data_str.as_bytes());

    ctx.log(&format!(
        "[bulletproofs] PutState: key=bulletproofs_test, name={}, value={}, result={}",
        handle_type, data_str, put_code
    ));
    ctx.log("[bulletproofs] ========================================end");
    // ctx.ok("finish event_test_set success".as_bytes());
    ctx.ok(data_str.as_bytes());
}

#[no_mangle]
pub extern "C" fn bulletproofs_test_get() {
    sim_context::log("[bulletproofs] ========================================end");
    sim_context::log("[bulletproofs] bulletproofs_test_get");

    let ctx = sim_context::get_sim_context();
    let handle_type = ctx.arg_default_blank("handletype");
    let result = ctx.get_state("bulletproofs_test", &handle_type);
    if result.is_err() {
        sim_context::log("[bulletproofs] bulletproofs_test_get error");
        ctx.error("finish bulletproofs_test_get failure");
        return;
    }
    let data = result.unwrap();
    let result = String::from_utf8(data);
    let result_str = result.unwrap();
    ctx.log(&format!(
        "[bulletproofs] GetState: key=bulletproofs_test_get, name={}, value={}, result={}",
        handle_type, result_str, 0
    ));

    ctx.log("[bulletproofs] ========================================end");

    if handle_type == "BulletproofsVerify" {
        ctx.ok(&*decode(result_str.as_bytes()).unwrap());
    } else {
        ctx.ok(result_str.as_bytes());
    }
}
```

编译得到*chainmaker_contract.wasm*字节码文件，在目录*chainmaker-contract-sdk-rust/target/wasm32-unknown-unknown/release*下：

```shell
make build
```



### 2、使用SDK编写测试用例：

> 注意：
>
> SDK并未提供 bulletproofs 相关接口，开发者需要调用 common 库中的*chainmaker.org/sdk-go/common/crypto/bulletproofs*包
>
> 把上一步编译好的合约文件放到指定位置。

总测试函数：

```go
const (
	sdkConfigOrg1Client1Path = "../sdk_configs/sdk_config_org1_client1.yml"
	createContractTimeout    = 5
)

const (
	// rust
	//bulletproofsContractName = "bulletproofs-rust-1001"
	//bulletproofsByteCodePath = "../../testdata/counter-go-demo/chainmaker_contract.wasm"
	//bulletproofsRuntime      = common.RuntimeType_WASMER

	// go
	bulletproofsContractName = "bulletproofsgo1001"
	bulletproofsByteCodePath = "../../testdata/bulletproofs-wasm-demo/contract-bulletproofs.wasm"
	bulletproofsRuntime      = common.RuntimeType_GASM

	// 链上合约SDK接口
	BulletProofsOpTypePedersenAddNum        = "PedersenAddNum"
	BulletProofsOpTypePedersenAddCommitment = "PedersenAddCommitment"
	BulletProofsOpTypePedersenSubNum        = "PedersenSubNum"
	BulletProofsOpTypePedersenSubCommitment = "PedersenSubCommitment"
	BulletProofsOpTypePedersenMulNum        = "PedersenMulNum"
	BulletProofsVerify                      = "BulletproofsVerify"

	// 测试数据
)

var (
	// 测试数据
	A            uint64 = 100
	X            uint64 = 20
	commitmentA1 []byte
	commitmentA2 []byte
	proofA1      []byte
	proofA2      []byte
	openingA1    []byte
)

func main() {
	TestBulletproofsContractCounterGo()
}

func TestBulletproofsContractCounterGo() {
	t := new(testing.T)
	client, err := examples.CreateChainClientWithSDKConf(sdkConfigOrg1Client1Path)
	require.Nil(t, err)

	fmt.Println("======================================= 创建合约（异步）=======================================")
	testUserBulletproofsContractCounterGoCreate(client, examples.UserNameOrg1Admin1, examples.UserNameOrg2Admin1, examples.UserNameOrg3Admin1, examples.UserNameOrg4Admin1, false)
	time.Sleep(5 * time.Second)

	funcName := BulletProofsOpTypePedersenAddNum
	//funcName := BulletProofsOpTypePedersenAddCommitment
	//funcName := BulletProofsOpTypePedersenSubNum
	//funcName := BulletProofsOpTypePedersenSubCommitment
	//funcName := BulletProofsOpTypePedersenMulNum

	fmt.Printf("============================= 调用合约 链上计算并存储 =============================\n")
	testBulletproofsSet(client, "bulletproofs_test_set", funcName, true)
	time.Sleep(5 * time.Second)

	fmt.Printf("============================= 查询计算结果 =============================\n")
	testBulletProofsGetOpResult(t, client, "bulletproofs_test_get", funcName, false)
	time.Sleep(5 * time.Second)

	fmt.Printf("============================= 调用合约验证 proof 和 查询的 commitment =============================\n")
	testBulletproofsVerify(client, "bulletproofs_test_set", BulletProofsVerify, true)
	time.Sleep(5 * time.Second)

	fmt.Printf("============================= 查询验证结果 =============================\n")
	testBulletProofsGetVerifyResult(t, client, "bulletproofs_test_get", BulletProofsVerify, false)
	time.Sleep(5 * time.Second)
}
```

创建合约：

```go
// 创建合约
func testUserBulletproofsContractCounterGoCreate(client *sdk.ChainClient, admin1, admin2, admin3,
	admin4 string, withSyncResult bool) {

	resp, err := createUserContract(client, admin1, admin2, admin3, admin4,
		bulletproofsContractName, examples.Version, bulletproofsByteCodePath, bulletproofsRuntime, []*common.KeyValuePair{}, withSyncResult)
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



调用合约 链上计算并存储计算结果

```go
// 调用合约 链上计算并存储计算结果
func testBulletproofsSet(client *sdk.ChainClient, method string, opType string, b bool) {
	// 构造payloadParams
	payloadParams, err := constructBulletproofsSetData(opType)
	if err != nil {
		return
	}
	resp, err := client.InvokeContract(bulletproofsContractName, method, "", payloadParams, -1, b)
	if err != nil {
		fmt.Println(err)
	}

	if resp.Code != common.TxStatusCode_SUCCESS {
		fmt.Printf("invoke contract failed, [code:%d]/[msg:%s]\n", resp.Code, resp.Message)
	}
}

func constructBulletproofsSetData(opType string) ([]*common.KeyValuePair, error) {
	// 1. 对原始数据生成承诺和证明
	var err error
	proofA1, commitmentA1, openingA1, err = bulletproofs.ProveRandomOpening(A)
	if err != nil {
		return nil, err
	}

	//_, commitmentX, openingX, err := bulletproofs.ProveRandomOpening(X)
	//if err != nil {
	// return nil, err
	//}

	// 2. 计算并生成证明
	proofA2, _, err = bulletproofs.ProveAfterAddNum(A, X, openingA1, commitmentA1)
	//proofA2, _, err = bulletproofs.ProveAfterSubNum(A, X, openingA1, commitmentA1)
	//proofA2, _, _, err = bulletproofs.ProveAfterAddCommitment(A, X, openingA1, openingX, commitmentA1, commitmentX)
	//proofA2, _, _, err = bulletproofs.ProveAfterSubCommitment(A, X, openingA1, openingX, commitmentA1, commitmentX)
	//proofA2, _, _, err = bulletproofs.ProveAfterMulNum(A, 10, openingA1, commitmentA1)
	//proofA2, _, _, err = bulletproofs.ProveAfterMulNum(A, X, openingA1, commitmentA1)
	if err != nil {
		return nil, err
	}

	// 3. 原始 commitment-proof 对儿 和 新生成的 proof 上链
	// 3.1. 构造上链 payloadParams
	base64CommitmentA1Str := base64.StdEncoding.EncodeToString(commitmentA1)
	XStr := strconv.FormatInt(int64(X), 10)
	//base64X := base64.StdEncoding.EncodeToString([]byte(XStr))
	//base64X := base64.StdEncoding.EncodeToString(commitmentX)

	payloadParams := []*common.KeyValuePair{
		{
			Key:   "handletype",
			Value: []byte(opType),
		},
		{
			Key:   "para1",
			Value: []byte(base64CommitmentA1Str),
		},
		{
			Key:   "para2",
			Value: []byte(XStr),
		},
	}

	return payloadParams, nil
}
```



查询计算结果

```go
// 查询计算结果
func testBulletProofsGetOpResult(t *testing.T, client *sdk.ChainClient, method string, opType string, b bool) {
	var err error
	commitmentA2, err = queryBulletproofsCommitment(client, bulletproofsContractName, method, opType, -1)
	require.Nil(t, err)
	fmt.Printf("QUERY %s contract resp -> : %s\n", bulletproofsContractName, commitmentA2)
}

func queryBulletproofsCommitment(client *sdk.ChainClient, contractName, method, bpMethod string, timeout int64) ([]byte, error) {

	resultBytes, err := queryBulletProofsCommitmentByHandleType(client, contractName, method, bpMethod, timeout)
	if err != nil {
		return nil, err
	}

	if bpMethod != BulletProofsVerify {
		resultBytes, err = base64.StdEncoding.DecodeString(string(resultBytes))
		if err != nil {
			return nil, err
		}
	}

	return resultBytes, nil
}

func queryBulletProofsCommitmentByHandleType(client *sdk.ChainClient, contractName, method, bpMethod string, timeout int64) ([]byte, error) {
	pair := []*common.KeyValuePair{
		{Key: "handletype", Value: []byte(bpMethod)},
	}

	resp, err := client.QueryContract(contractName, method, pair, timeout)
	if err != nil {
		return nil, err
	}

	result := resp.ContractResult.Result

	return result, nil
}
```



提供链下生成的 proof 和 查询到的链上计算结果 commitment，调用合约进行验证

```go
// 调用合约验证 proof 和 查询的 commitment
func testBulletproofsVerify(client *sdk.ChainClient, method string, opType string, b bool) {
	// 构造payloadParams
	payloadParams, err := constructBulletproofsVerifyData(opType)
	if err != nil {
		return
	}
	resp, err := client.InvokeContract(bulletproofsContractName, method, "", payloadParams, -1, b)
	if err != nil {
		fmt.Println(err)
	}

	if resp.Code != common.TxStatusCode_SUCCESS {
		fmt.Printf("invoke contract failed, [code:%d]/[msg:%s]\n", resp.Code, resp.Message)
	}
}

func constructBulletproofsVerifyData(opType string) ([]*common.KeyValuePair, error) {
	// 1. 对原始数据生成承诺和证明
	//var err error
	//proofA1, commitmentA1, openingA1, err = bulletproofs.ProveRandomOpening(A)
	//if err != nil {
	//	return nil, err
	//}
	//
	//// 2. 计算并生成证明
	//proofA2, _, err := bulletproofs.ProveAfterAddNum(A, X, openingA1, commitmentA1)
	//if err != nil {
	//	return nil, err
	//}

	// 3. 原始 commitment-proof 对儿 和 新生成的 proof 上链
	// 3.1. 构造上链 payloadParams
	base64CommitmentA2Str := base64.StdEncoding.EncodeToString(commitmentA2)
	base64ProofA2Str := base64.StdEncoding.EncodeToString(proofA2)
	//base64ProofA2Str := base64.StdEncoding.EncodeToString(proofA1)

	payloadParams := []*common.KeyValuePair{
		{
			Key:   "handletype",
			Value: []byte(opType),
		},
		{
			Key:   "para1",
			Value: []byte(base64ProofA2Str),
		},
		{
			Key:   "para2",
			Value: []byte(base64CommitmentA2Str),
		},
	}

	return payloadParams, nil
}
```



查询验证结果

```go
// 查询验证结果
func testBulletProofsGetVerifyResult(t *testing.T, client *sdk.ChainClient, method string, opType string, b bool) {
	result, err := queryBulletproofsCommitment(client, bulletproofsContractName, method, opType, -1)
	require.Nil(t, err)
	fmt.Printf("QUERY %s contract resp -> : %s\n", bulletproofsContractName, result)
}

```



执行测试：

```go
======================================= 创建合约（异步）=======================================
2021-06-23 15:13:28.369	[DEBUG]	[SDK]	sdk/sdk_user_contract.go:31	[SDK] create [ContractCreate] to be signed payload
2021-06-23 15:13:28.369	[DEBUG]	[SDK]	sdk/sdk_user_contract.go:60	[SDK] create [ContractManage] to be signed payload
2021-06-23 15:13:28.405	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"OK" 
CREATE bulletproofs-rust-1001 contract resp: message:"OK" contract_result:<result:"0caaf7f3f20641cfbd2ad17c6a440f0446cefa4344334260882ea412764bd8df" message:"OK" > 
============================= 调用合约 链上计算并存储 =============================
2021-06-23 15:13:33.469	[DEBUG]	[SDK]	sdk/sdk_user_contract.go:197	[SDK] begin to INVOKE contract, [contractName:bulletproofs-rust-1001]/[method:bulletproofs_test_set]/[txId:50e11d2c676744f2b14aded5bebb6cd8bd2f5c289c8a4df682995c455186f6a4]/[params:map[handletype:PedersenMulNum para1:XgR9wCSTBGZIegcUb7N171NDzWX/Vs7T3WIbgMSnG0c= para2:20]]
2021-06-23 15:13:33.470	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"OK" 
2021-06-23 15:13:33.470	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:50e11d2c676744f2b14aded5bebb6cd8bd2f5c289c8a4df682995c455186f6a4]
2021-06-23 15:13:33.471	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: code:CONTRACT_FAIL message:"txStatusCode:4, resultCode:1, contractName[SYSTEM_CONTRACT_QUERY] method[GET_TX_BY_TX_ID] txType[QUERY_SYSTEM_CONTRACT], no such transaction, chainId:chain1" contract_result:<code:FAIL message:"no such transaction, chainId:chain1" > 
2021-06-23 15:13:33.972	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:50e11d2c676744f2b14aded5bebb6cd8bd2f5c289c8a4df682995c455186f6a4]
2021-06-23 15:13:33.972	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: code:CONTRACT_FAIL message:"txStatusCode:4, resultCode:1, contractName[SYSTEM_CONTRACT_QUERY] method[GET_TX_BY_TX_ID] txType[QUERY_SYSTEM_CONTRACT], no such transaction, chainId:chain1" contract_result:<code:FAIL message:"no such transaction, chainId:chain1" > 
2021-06-23 15:13:34.473	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:50e11d2c676744f2b14aded5bebb6cd8bd2f5c289c8a4df682995c455186f6a4]
2021-06-23 15:13:34.473	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\n\305\003\n\214\001\n\006chain1\022:\n\026wx-org1.chainmaker.org\022 \347|\2228\305\0364F\331B\371K\330\200<\304\363Q%O\207q\371r\024m{\374n\013\347\364\"@50e11d2c676744f2b14aded5bebb6cd8bd2f5c289c8a4df682995c455186f6a4(\235\276\313\206\006\022\221\001\n\026bulletproofs-rust-1001\022\025bulletproofs_test_set\0325\n\005para1\022,XgR9wCSTBGZIegcUb7N171NDzWX/Vs7T3WIbgMSnG0c=\032\013\n\005para2\022\00220\032\034\n\nhandletype\022\016PedersenMulNum\032G0E\002!\000\356\302\350\275\340~B\000\315\276r4\361]\240\242\330\250/<#V\274\250E\361\031\330/\327\232\356\002 T-\333\344Y\223F\374\004(\303Z\362\273:\341Mb\335<\342M\373\037\220\372F\367\001U\352\233\"W\0223\022,aL+AlqjxZ/dTSDDy7TuPVIaIbGna49l8Me8u+HxeFhw= \320\333\345\010\032 i\007\334\032\215\257Uk\023\004\352]\233\277\213\037\357\210\267\000\332\221\264\234c\307'\3444\273Y\363\020," message:"OK" > 
invoke contract success, resp: [code:0]/[msg:OK]/[contractResult:result:"aL+AlqjxZ/dTSDDy7TuPVIaIbGna49l8Me8u+HxeFhw=" gas_used:18443728 ]
============================= 查询计算结果 =============================
2021-06-23 15:13:39.474	[DEBUG]	[SDK]	sdk/sdk_user_contract.go:240	[SDK] begin to QUERY contract, [contractName:bulletproofs-rust-1001]/[method:bulletproofs_test_get]/[txId:67dcb1ffcad54fa28062fe184c9a27e2258e597779634f17b5c23cd522eb88de]/[params:map[handletype:PedersenMulNum]]
2021-06-23 15:13:39.477	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"aL+AlqjxZ/dTSDDy7TuPVIaIbGna49l8Me8u+HxeFhw=" gas_used:13003432 > 
QUERY bulletproofs-rust-1001 contract resp -> : h�����g�SH0��;�T��li���|1�.�|^
============================= 调用合约验证 proof 和 查询的 commitment =============================
2021-06-23 15:13:44.478	[DEBUG]	[SDK]	sdk/sdk_user_contract.go:197	[SDK] begin to INVOKE contract, [contractName:bulletproofs-rust-1001]/[method:bulletproofs_test_set]/[txId:c8d9f0ae6faa413d84662324fe23935374cbebe0a2fa4d21a6b3b5f4b19b3a59]/[params:map[handletype:BulletproofsVerify para1:xtngQVNXzDIMmsXBW5656k/35X4Pa+mukB8wmVYGPxMG0w2mNo/gZ++mSBXocQfAW2q+l5uiQCLkZdroxCIEMAwCgeC98coHu5isUWzHVrObxuw4gTRhSjv3MpEO3dZSFpmEzwDzJr41syyRrVBHdoh24D/5PmyJvVGjdQgf+CZVm2UU50Rb8ODOIr/YHJh4qdXDPMG09FOLeU3pDMFpCYVKGBJcUCidUC+dU74zHXN56Xt4jsgzSsMqsX3hY5AP8invNY4wwp708fqq+kj864fvG7A7SUpkQk5C2izjqgaInTDvdU9h6Velz7P8cjG3bt9CeUMjqoOgMEwmrWIbJo7RECeLidOORw5rrAVyPKY9zKQk3LIgIqIt40dc7IZF6LLZ+EZbr+s/Yxe1wYR5ZUU6LU418E0UUMQ5y2AJMFjMRjBIpXHgCTdOo1+fEG4vyNUjlBhkS7ULZvoalxVfD2DPquye3XdT0yEFf7VTscM2TTx35XDV0vlI64wQsM0neGLRvqL/9nhu3f+44G/mqAavZiG7tmoCa9W6xYYxKSJG50duCmqlcrYVXABpFhAmHpW6p6cIgM9ePuRzyrkbCSAOKlXt1QINHouVsnj0G4cZDbliwvkQppn2RfC/uLBqlFejmqF4bpTgyEj7y206FtK2Fz8oevBQioiKO9mq435e7YRphiZHmFyrYFHvWO5EM6ZQ7ihh27axIGDHPNfnZjxMLM19DTTXK17vjpk3HksjkdWF53erB7WGJa7UDb1vZq1OUb3eqSW1O/Sk1Ck1YtfXquCsqpYsIw/g1asir1TSpouobOFopgEI2vobD7DSq1j5BrEcPipyssVhum54A/KEckOZ0USY0gZs4oAEBwF7OYqG4xoTh2FKgox593kI para2:aL+AlqjxZ/dTSDDy7TuPVIaIbGna49l8Me8u+HxeFhw=]]
2021-06-23 15:13:44.480	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"OK" 
2021-06-23 15:13:44.480	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:c8d9f0ae6faa413d84662324fe23935374cbebe0a2fa4d21a6b3b5f4b19b3a59]
2021-06-23 15:13:44.482	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: code:CONTRACT_FAIL message:"txStatusCode:4, resultCode:1, contractName[SYSTEM_CONTRACT_QUERY] method[GET_TX_BY_TX_ID] txType[QUERY_SYSTEM_CONTRACT], no such transaction, chainId:chain1" contract_result:<code:FAIL message:"no such transaction, chainId:chain1" > 
2021-06-23 15:13:44.982	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:c8d9f0ae6faa413d84662324fe23935374cbebe0a2fa4d21a6b3b5f4b19b3a59]
2021-06-23 15:13:44.984	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: code:CONTRACT_FAIL message:"txStatusCode:4, resultCode:1, contractName[SYSTEM_CONTRACT_QUERY] method[GET_TX_BY_TX_ID] txType[QUERY_SYSTEM_CONTRACT], no such transaction, chainId:chain1" contract_result:<code:FAIL message:"no such transaction, chainId:chain1" > 
2021-06-23 15:13:45.484	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:c8d9f0ae6faa413d84662324fe23935374cbebe0a2fa4d21a6b3b5f4b19b3a59]
2021-06-23 15:13:45.487	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: code:CONTRACT_FAIL message:"txStatusCode:4, resultCode:1, contractName[SYSTEM_CONTRACT_QUERY] method[GET_TX_BY_TX_ID] txType[QUERY_SYSTEM_CONTRACT], no such transaction, chainId:chain1" contract_result:<code:FAIL message:"no such transaction, chainId:chain1" > 
2021-06-23 15:13:46.487	[DEBUG]	[SDK]	sdk/sdk_system_contract.go:29	[SDK] begin to QUERY system contract, [method:GET_TX_BY_TX_ID]/[txId:c8d9f0ae6faa413d84662324fe23935374cbebe0a2fa4d21a6b3b5f4b19b3a59]
2021-06-23 15:13:46.489	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"\n\241\n\n\214\001\n\006chain1\022:\n\026wx-org1.chainmaker.org\022 \347|\2228\305\0364F\331B\371K\330\200<\304\363Q%O\207q\371r\024m{\374n\013\347\364\"@c8d9f0ae6faa413d84662324fe23935374cbebe0a2fa4d21a6b3b5f4b19b3a59(\250\276\313\206\006\022\225\010\n\026bulletproofs-rust-1001\022\025bulletproofs_test_set\032\212\007\n\005para1\022\200\007xtngQVNXzDIMmsXBW5656k/35X4Pa+mukB8wmVYGPxMG0w2mNo/gZ++mSBXocQfAW2q+l5uiQCLkZdroxCIEMAwCgeC98coHu5isUWzHVrObxuw4gTRhSjv3MpEO3dZSFpmEzwDzJr41syyRrVBHdoh24D/5PmyJvVGjdQgf+CZVm2UU50Rb8ODOIr/YHJh4qdXDPMG09FOLeU3pDMFpCYVKGBJcUCidUC+dU74zHXN56Xt4jsgzSsMqsX3hY5AP8invNY4wwp708fqq+kj864fvG7A7SUpkQk5C2izjqgaInTDvdU9h6Velz7P8cjG3bt9CeUMjqoOgMEwmrWIbJo7RECeLidOORw5rrAVyPKY9zKQk3LIgIqIt40dc7IZF6LLZ+EZbr+s/Yxe1wYR5ZUU6LU418E0UUMQ5y2AJMFjMRjBIpXHgCTdOo1+fEG4vyNUjlBhkS7ULZvoalxVfD2DPquye3XdT0yEFf7VTscM2TTx35XDV0vlI64wQsM0neGLRvqL/9nhu3f+44G/mqAavZiG7tmoCa9W6xYYxKSJG50duCmqlcrYVXABpFhAmHpW6p6cIgM9ePuRzyrkbCSAOKlXt1QINHouVsnj0G4cZDbliwvkQppn2RfC/uLBqlFejmqF4bpTgyEj7y206FtK2Fz8oevBQioiKO9mq435e7YRphiZHmFyrYFHvWO5EM6ZQ7ihh27axIGDHPNfnZjxMLM19DTTXK17vjpk3HksjkdWF53erB7WGJa7UDb1vZq1OUb3eqSW1O/Sk1Ck1YtfXquCsqpYsIw/g1asir1TSpouobOFopgEI2vobD7DSq1j5BrEcPipyssVhum54A/KEckOZ0USY0gZs4oAEBwF7OYqG4xoTh2FKgox593kI\0325\n\005para2\022,aL+AlqjxZ/dTSDDy7TuPVIaIbGna49l8Me8u+HxeFhw=\032 \n\nhandletype\022\022BulletproofsVerify\032G0E\002 i\273?\010k\270\352.jX\234$\036\242h\335\034O0\342I^*\250\344\222\035 !p\371\036\002!\000\247a!\320\331\021\274\311n\344\025\352\265\350\"EsOg\335\232\013\006M\034\306\363\201\360\264\366\345\"/\022\013\022\004MQ== \307\355\270\017\032 0i.\272^\367\177)\307/\307\004\304\037\364\357\366M\321\352\364\205f2*\211\201\237\320\271\242G\020-" message:"OK" > 
invoke contract success, resp: [code:0]/[msg:OK]/[contractResult:result:"MQ==" gas_used:32388807 ]
============================= 查询验证结果 =============================
2021-06-23 15:13:51.489	[DEBUG]	[SDK]	sdk/sdk_user_contract.go:240	[SDK] begin to QUERY contract, [contractName:bulletproofs-rust-1001]/[method:bulletproofs_test_get]/[txId:d78d77c847ba4cc697556a68ae39498b83750e8dc0e64382b7738395aa1b9692]/[params:map[handletype:BulletproofsVerify]]
2021-06-23 15:13:51.493	[DEBUG]	[SDK]	sdk/sdk_client.go:368	[SDK] proposalRequest resp: message:"SUCCESS" contract_result:<result:"1" gas_used:13146253 > 
QUERY bulletproofs-rust-1001 contract resp -> : 1
--- PASS: TestBulletproofsContractCounterGo (28.34s)
PASS
```
