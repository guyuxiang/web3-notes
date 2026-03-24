### 智能合约钱包Gonosis Safe

Gnosis Safe多签钱包是以太坊最流行的多签钱包，管理近千亿美元资产，合约经过审计和实战测试，支持多链（以太坊，BSC，Polygon等, 已融资1亿美金

![image-20240527203231600](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240527203231600.png)



#### safe链上合约架构

Gonosis Safe采用模块化架构, 各模块可插拔的, 因此可以根据具有业务需要使用相应模块

![image-20240527203327427](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240527203327427.png) 



#### 使用流程

可以直接使用官方提供TypeScript 脚本来进行账户创建, 另外脚本还支持账户管理, 交易提出和签名, 交易执行等功能

https://github.com/safe-global/safe-core-sdk

例, 创建一个多签智能合约钱包, 其中SafeAccountConfig结构体中包含了多个签名方以及多签阈值

```typescript
import { SafeAccountConfig } from '@safe-global/protocol-kit'

const safeAccountConfig: SafeAccountConfig = {
  owners: [
    await OWNER_1_ADDRESS,
    await OWNER_2_ADDRESS,
    await OWNER_3_ADDRESS
  ],
  threshold: 2,
  // Optional params
}

/* This Safe is tied to owner 1 because the factory was initialized with the owner 1 as the signer. */
const protocolKitOwner1 = await safeFactory.deploySafe({ safeAccountConfig })
```

该过程实际是调用工厂合约创建了一个新的钱包代理合约, 完成参数的初始化存储在该代理合约上, 并将该代理合约指向一个存在的GnosisSafe实现合约, 对钱包代理合约的所有函数调用都通过`delegatecall`的方式远程调用`GnosisSafe`合约



多签业务流程: 

1. 第一个签名者提出交易
   1. 创建交易：定义金额、目的地和任何其他数据
   2. 在提议之前对交易进行链下签名
   3. 将交易和签名提交给safe交易服务
2. 第二位签名者确认交易
   1. 从 Safe 服务获取待处理交易
   2. 执行交易的链下签名
   3. 将签名提交给服务
3. 任何人执行交易
   1. 在此示例中，第一个签名者执行交易
   2. 任何人都可以从 Safe 服务获取待处理的交易
   3. 执行交易的账户支付 gas 费



其中交易服务可以使用官方服务也可以自己部署

该服务使多位签名者协同完成多签，通过服务提供的RESTFul API接口将签名发送到服务以收集它们，获取有关的交易信息（如读取交易历史记录、待处理交易、启用的模块和保护等）以及其他功能。



#### 签名

Safe 支持不同类型的签名, 各签名者在链下进行各自的签名,

- ECDSA 签名

  该签名让用户可以直接使用小狐狸钱包进行签名

  对于EOA账户, 其在链下使用标准的ECDSA椭圆曲线加密算法对EIP-712定义的消息进行签名以批准交易, 签名结果为65字节的

  ```
  {32-bytes r}{32-bytes s}{1-byte v}
  ```

  在链上使用自带的recover函数即可进行签名恢复验证

  

- EIP-1271签名

  该签名让用户可以使用自定义实现的签名算法

  IERC1271是一个提供自定义验签名过程的合约对外的接口标准, 传参为待验证消息的hash和签名, 签名的验证过程可以自定义实现, 如果验签通过返回0x1626ba7e，即bytes4(keccak256("isValidSignature(bytes32,bytes)")

  ```solidity
  interface IERC1271 {
      function isValidSignature(bytes32 hash, bytes memory signature) external view returns (bytes4 magicValue);
  }
  ```

  

  signature字段在GnosisSafe的编码规则中为

  ```
  {32-bytes signature verifier}{32-bytes data position}{1-byte signature type}{32-bytes signature length}{bytes signature data}
  ```

  verifier: 实现 EIP-1271 接口的合约的地址

  position: 相对于签名数据开头的偏移量,用于指定签名数据

  type: 0, 用于标记为EIP-1271签名类型

  

  length: 指定签名长度

  data: 签名数据

  

  safe合约通过外部调用实现IERC1271的合约的`isValidSignature`方法对签名进行验证

  ```solidity
  ISignatureValidator(addr).isValidSignature(data,sig) == EIP1271_MAGIC_VALUE
  ```

  

- 预签名

  预签名是一种链上签名的解决方案, 其解决的是智能合约没有私钥，但要对一笔交易进行签名的问题, 签名和验签的过程都是在链上进行的

  相对其他两种签名, 多了上链的一步, 因此叫预签名

  在多签钱包合约中维护一个全局变量

  ```solidity
  mapping(address => mapping(bytes32 => uint256)) public approvedHashes;
  ```

  签名过程实际类似授权的过程, 要进行签名的合约调用钱包approveHash函数, 传入签名数据的hash, 该函数会判断调用者是否属于多签账户中的一个, 是的话将账户和签名数据hash的映射存储起来, 完成签名

  ```solidity
  function approveHash(bytes32 hashToApprove) external {
  	require(owners[msg.sender] != 0, "GnosisSafe/approveHash msg.sender is not an owner");
  	approveHashes[msg.sender][hashToApprove] = uint256(1);
  }
  ```

  验证签名时, 向mapping查询是否有此映射, 来确认某个账户对一笔交易哈希是否已经预先签名

  ```solidity
  if (v == 1) {
  	address currentOwner = address(uint160(uint256(r)));
  	require(approvedHashes[currentOwner][dataHash] != 0 || msg.sender == currentOwner);
  }
  ```

  GnosisSafe的预签名编码规则:

  ```
  {32-bytes hash validator}{32-bytes ignored}{1-byte signature type}
  ```

  validator: 预签名的账户地址

  ignored: 占位符填充0, 该字段未使用

  type: 1, 用于标记该签名为预签名



最后

多个签名者在使用各自的签名类型对交易签名后, 执行者将所有签名将合并为一个签名

```
"0x" + 
"cf06921ff2AD4c99dF77dcF741d6ce51061dE64300000000000000000000000000000000000000000000000000000000000000c300" + // EIP-1271签名
"Ccd5c9a1C54cE1521877Cc93124cb3513C2f7684000000000000000000000000000000000000000000000000000000000000000001" + // 预签名
"bde0b9f486b1960454e326375d0b1680243e031fd4fb3f070d9a3ef9871ccfd57d1a653cffb6321f889169f08e548684e005f2b0c3a6c06fba4c4a68f5e006241c" + // ECDSA签名
"00000000000000000000000000000000080000000000000000000000000000deadbeef"// EIP-1271的签名数据部分, 长度和签名数据
```

传输到智能合约钱包合约进行验证和执行。

验证过程:

执行一个for循环分别验证签名组合中的每一个签名，因为三种签名的编码格式都是{32-bytes}{32-bytes}{1-byte}, 

因此在每一个循环内部从sigatures里拿到对应的`r,s,v`值作为相应位置的变量进行后续验证



#### 社交认证签名

实现使用电子邮件地址、社交媒体账户或加密钱包账户对其进行身份验证, 让用户无需钱包即可开始使用

利用Web3Auth身份验证系统的MPC 技术,  web3auth生成三个密钥分片，而后分开存储在用户设备, 社交账户OAuth 登录分片, 备用设备中

EIP-1193定义了密钥管理软件提供的api接口, 以实现钱包的规范操作性, safe提供了兼容EIP-1193标准的包来使用Web3Auth能力

![image-20240528201648326](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240528201648326.png)



例, auth-kit包中的getProvider()返回EIP-1193标准的以太坊Provider, 可用于ether.js的签名

```javascript
const provider = new ethers.BrowserProvider(safeAuthPack.getProvider())
const signer = provider.getSigner()
await signer.signTransaction(tx)
```





#### 账户抽象

Gonosis Safe钱包是模块化架构,  因此可以在Safe智能合约钱包部署时或之后启用Safe4337模块

Safe4337模块实现了ERC-4337接口，包括验证和执行（s）的功能`UseOperation`，并且仅限于`EntryPoint`地址。



#### 免gas交易功能

safe智能合约钱包允许由第三方待付手续费或者使用 ERC-20 代币支付费用



#### 交易打包功能

safe智能合约允许将多笔交易按照自己的意愿来进行组合，并最终以单笔交易的操作来实现