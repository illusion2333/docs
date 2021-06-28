
## Q&A
### 合约开发建议采用哪种开发语言？建议使用的开发工具是什么？

> 建议开发语言：rust，合约内可以引用大多数外部依赖（如：含随机数的不可用）。
> 建议开发工具：vscode，+  插件：rust-analyzer 

### 智能合约中PutState有两个参数是什么意思？

```go
func PutState(key string, field string, value string) ResultCode
```

> 
> 实际存储到leveldb的key为：contractName + “#” + key + field
> 
> 长度限制： key:64、  field:64、 value:1M
> 
> 且key、field符合正则`^[a-zA-Z0-9._-]+$`，只能为英文字母、数字、点、横杠、下划线
> 
> 两个参数的原因：一个逻辑性的命名空间概念，key作为namespace一般为有规律的值

### 不同组织间有没有共同的ca？

> 不同组织间的CA证书可以使用同一个。但是不建议这样做，建议是一个组织一个CA证书。

```yml
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

### 组织间的数据能否实现哪些数据可以公开给对方，哪些数据不能公开给对方

> 上链数据均共享。可以根据场景需要，采用混合加密、分层身份加密、同态加密、零知识证明等方式保护数据隐私

### 合约代码是否需要每个组织节点都部署？

> 合约代码部署也是一个交易。发送给某个节点后，该节点会把交易广播到自己的网络中。其他节点也就有了这个交易了。交易上链需要各个节点达成共识，其他共识节点也会执行该交易。

