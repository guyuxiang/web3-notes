Move组中开发者体验

Sui 和 Aptos 为 Layer 1 公链

Movement 为建立在以太坊上的 Layer 2，旨在把 Move 语言带入 ETH 生态

Sui 和 Aptos 的共识都是基于 DAG 的 BFT（拜占庭容错） ，但 Leader 选取等具体机制不同，而 Movement 则采用了雪崩协议 Avalanche 的 Snowman 共识。**不同的共识机制带来了不同的 TTF（交易确认时间），目前 Sui 的 Mysticeti 共识最快，能在 0.5 秒内确认**。Aptos 将在之后升级到 RAPTR 共识，表现同样值得期待。

对于并行交易，**Aptos 和 Movement 都采用了 Block-STM 的并行引擎**，是一种乐观并行化的机制，假设所有交易都可以并行处理，若遇到错误交易则重新执行。Sui 则采用的是一种「状态访问」的方法，将交易分类、排序、确认无冲突后执行。

**虽然同样采用 Move 语言，但 Move 语言已演变为 Sui Move 和 Aptos Move 两大变种。**Movement 理论上都兼容，但其实主要支持 Aptos Move。

最开始，**Aptos 基于 Diem 项目的代码基础，是地址模型，是链**。Sui 后推出主网，虽然也是来自 Diem 的团队，但重新设计了许多关键组件并提出了新方案：Sui 是以对象为中心的 DAG。**Aptos 后来也改为对象模型与 DAG。**



**资源管理模型**：Move 将资产视为资源，使其不可复制或销毁。这种独特的资源管理模型避免了智能合约中常见的双重支付或意外销毁资产问题。

**模块化设计**：Move 允许智能合约以模块化的方式构建，提高代码复用性，并且降低了开发复杂度。

**高安全性**：Move 在语言层面内置了大量的安全检查机制，防止常见的安全漏洞，比如重入攻击（reentrancy attacks）等。



1. **模块结构**：代币合约通常以模块形式组织，需要展示如何定义模块及其依赖。
2. **资源管理**：Move中的资源是不可复制的，必须正确处理，例如代币的存储和转移。
3. **函数可见性**：区分公共函数和私有函数，确保合约的安全性。
4. **事件处理**：虽然Move本身不直接支持事件，但可能需要通过其他方式记录状态变化。
5. **错误处理**：使用abort和错误码来处理异常情况，如余额不足。



Move中，模块发布后不可更改，依赖明确，而Solidity可以通过代理模式升级合约



Move的资产是作为一等公民，资源不能复制，这直接避免了双花问题

- **资源（Resource）**：资产（如代币）被定义为不可复制的类型，必须显式转移或销毁。
- **线性类型**：资源只能被使用一次，防止双花攻击。

```move
public fun transfer(sender: &signer, receiver: address, amount: u64) {
    let coin = withdraw(sender, amount); // 提取资源
    deposit(receiver, coin); // 存入资源
}
```

而solidity是基于数值的资产管理



- **模块化开发**：代码以模块（Module）形式发布，模块内函数和数据高度封装。
- **不可变模块**：已发布的模块不可修改，需通过新版本升级。





按交易生命周期去调研

发生交易

交易执行

交易监听



标准账户——这是一个典型的账户，对应一个地址和一对相应的公钥/私钥。
资源账户- 一个没有对应私钥的自主账户，供开发者存储资源或上链发布模块。

帐户地址为 32 字节。它们通常显示为 64 个十六进制字符，每个十六进制字符为一个半字节。有时地址以 0x 为前缀。
Aptos区块链默认采用Ed25519签名交易





### 以太坊账户

- 地址：每个账户由一个160位长度的地址标识。
- 余额：每个帐户都有余额。
- Nonce：每个账户都有一个nonce，代表该账户曾经支付过gas的交易数量。
- 状态：每个账户都有一个状态，由存储根表示。如果账户是合约类型的账户，那么它还有一个包含代码哈希的字段，代码哈希只是字节码的哈希。
- 控制：合约账户由 EVM 代码控制，而 EoA 由私钥控制。



### Aptos 账户

- 地址：每个账户由一个256位长度的地址标识。
- 模块和资源：每个账户包含模块（智能合约）和资源（类型化数据结构）。
- 余额：账户余额并不像以太坊那样直接存储在账户下的单一字段中，而是通过资源来表示。账户拥有的每种资产都有一个资源——此资源包括该资产的账户余额。
- 序列号：每个账户都有一个序列号，代表从该账户发送的交易数量。
- 控制：模块包含由 Move VM 控制的字节码。







改造

将基于以太坊Solidity的DApp迁移到Aptos区块链是一个系统性工程，需要从前端、智能合约、后端三个层面进行全面改造。以下是详细的技术改造方案：

一、智能合约层改造

1. 语言迁移

- 从Solidity迁移到Move语言
- 核心差异：
  - Move基于资源模型（资源不可复制/隐式销毁）
  - 所有权系统显式管理（相比Solidity的地址所有权）
  - 模块化架构（module/script概念）
  - 形式化验证支持

1. 账户系统重构

- Aptos账户特性：
  - 账户需先初始化才能使用（需主动创建）
  - 身份验证密钥轮换机制
  - 多签支持原生集成
  - 16字节地址格式（0x开头）
- 改造要点：
  - 替换address类型为AccountAddress
  - 账户创建需调用aptos_framework::account::create_account
  - 实现密钥轮换逻辑（如需）





二、前端层改造

1. 交易构造重构
2. 节点交互适配

使用TypeScript SDK
https://aptos.dev/en/build/sdks/ts-sdk

交易从构建到链上执行需要经历 5 个步骤：构建、模拟、签名、提交、等待。

```javascript
// 构建交易
const transaction = await aptos.transaction.build.simple({
  sender: sender.accountAddress,
  data: {
	  // All transactions on Aptos are implemented via smart contracts.
	  function: "0x1::aptos_account::transfer",
	  functionArguments: [destination.accountAddress, 100],
  },
  gasUnitPrice: 100, // 每个 gas unit 的价格
  maxGasAmount: 10000, // 最大 gas 限制
  expirationTimestamp: Math.floor(Date.now() / 1000) + 600, // 10 分钟后过期
});

// 模拟
const [userTransactionResponse] = await aptos.transaction.simulate.simple({
  signerPublicKey: signer.publicKey,
  transaction,
});

// 签名
const senderAuthenticator = aptos.transaction.sign({
  signer: sender,
  transaction,
});

// 提交
const committedTransaction = await aptos.transaction.submit.simple({
  transaction,
  senderAuthenticator,
});

// 等待
const executedTransaction = await aptos.waitForTransaction({ transactionHash: committedTransaction.hash });
```

三、后端服务改造

1. 区块链节点交互
2. 交易监听服务

使用Kaptos SDK
https://aptos.dev/en/build/sdks/community-sdks/kotlin-sdk

aptos每个事件流都有一个唯一的 `event_key`，可通过合约的账户资源获取：

```json
{
  "data": [
    {
      "version": "12345",
      "sequence_number": "0",
      "data": {
        "from": "0xsender...",
        "to": "0xreceiver...",
        "amount": "100"
      }
    }
  ]
}
```

轮询策略：
记录已处理的最新 sequence_number，每次请求从该值 +1 开始。
设置合理的轮询间隔（如 5-10 秒），避免触发 API 速率限制。



账户

需链上初始化（支付Gas），显式分配存储结构

初始化过程允许用户设置权限模块（如自定义签名验证逻辑），默认[Ed25519](https://ed25519.cr.yp.to/)

好处是支持支持多签、密钥轮换等，Aptos 区块链支持以下身份验证方案：

1. [Ed25519](https://ed25519.cr.yp.to/)
2. [Secp256k1 ECDSA](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-49.md)
3. [K-of-N 多重签名](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-55.md)
4. 专用的、现已遗留的 MultiEd25519 方案



账户初始化问题

![image-20250226161814029](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20250226161814029.png)

账户授权问题

需要合约自定义实现验签授权逻辑



事件存储和监听问题



|   类别   |                  任务名称                  |
| :------: | :----------------------------------------: |
| 开发准备 |     Rust环境配置和Aptos开发工具链使用      |
| KMS改造  |        KMS对接Aptos GO SDK签名改造         |
| KMS改造  |         KMS实现EIP-712类似功能改造         |
| 合约改造 |              DTTERC20合约改造              |
| 合约改造 |               Encash合约改造               |
| 合约改造 |             RORERC721合约改造              |
| 合约改造 |        TransactionIDFactory合约改造        |
| 合约改造 |                DTT合约改造                 |
| 合约改造 |           RorEnhancement合约改造           |
| 合约改造 |             RorMarket合约改造              |
| 合约改造 |           UserPermission合约改造           |
| 合约改造 |               Config合约改造               |
| 合约改造 |       开发move没有的solidty库和函数        |
| 后端改造 |       对ERC20合约的直接调用逻辑切换        |
| 后端改造 |          19个合约事件解析逻辑切换          |
| 后端改造 | 19个合约事件消费、解析、广播与维护逻辑切换 |



1. aptos有哪些库毕竟好用，aptos ERC20/721合约库对应的aptos库?
2. 什么是模块化的设计思维?
3. move模块升级问题，升级时有没有资源存储冲突问题?
4. script有什么用?
5. 从合约中心化存储到用户中心化存储，设计时如何考虑数据所有权，有什么设计模式?
6. 日志是存在用户中还是模块中里？
7. Move 的全局状态修改是逐步进行的，若在中间步骤失败，已修改的部分不会回滚吗？
8. 开发过程中要注意哪些并发问题？
9. 单个模块的大小有没有限制？
10. EIP712 permit离线签名有没有move实现？
11. 有什么成熟的move项目可参考学习?