# solana交互

RPC 方法：

```
sendTransaction
```

参数：

```
sendTransaction(
  transaction: <string>,
  config?: {
    encoding?: "base58" | "base64",
    skipPreflight?: boolean,
    preflightCommitment?: "processed" | "confirmed" | "finalized",
    maxRetries?: number,
    minContextSlot?: number
  }
)
```

返回值：

```
transaction signature (string)
```

示例返回：

```
5K3G8h7YfF...8kLw2
```

这个 signature 就是：

```
tx hash
```

可以在区块浏览器查交易。

# transaction 参数

`transaction` 是：

```
signed transaction
```

序列化后的字符串：

```
base64
```

例如：

```
AQABAgM...==
```

内部结构是：

```
Transaction {
   signatures
   message
}
```

其中：

```
message = instructions + accounts + blockhash
```

# config 参数

## 1 encoding

交易编码方式：

```
base58
base64
```

推荐：

```
base64
```

因为：

```
base58 更慢
```

------

## 2 skipPreflight

是否跳过 **模拟执行（simulation）**

默认：

```
false
```

RPC 会先执行：

```
simulateTransaction
```

检查：

- signature
- account
- instruction
- program error
- compute unit

如果 simulation 失败：

```
sendTransaction 会报错
```

跳过：

```
skipPreflight = true
```

适合：

- 高频交易
- MEV
- bot

------

## 3 preflightCommitment

模拟使用的 commitment：

```
processed
confirmed
finalized
```

推荐：

```
confirmed
```

------

## 4 maxRetries

RPC 自动重试次数：

```
默认：无限
```

例子：

```
maxRetries = 5
```

RPC 如果 leader 没接收：

会重新发送。

------

## 5 minContextSlot

确保：

```
RPC 节点 slot >= minContextSlot
```

否则拒绝。

用于：

```
避免旧节点
```













## slot

在 Solana 生态里，**一般用 `slot` 来表示“区块位置 / 区块高度”**，而不是 `block height`。

# signature

 **Solana 的交易 ID 本质上是签名，而不是交易内容的 hash**。

区块浏览器（例如 Solscan）也是用 **signature** 作为交易 ID。因为签名本身已经唯一，所以可以作为交易的唯一索引，这样可以少一次 hash实现高吞吐

Solana 交易结构大致是：

```
Transaction
 ├─ signatures[]
 └─ message
```

其中：

```
signature[0] = fee payer 的 Ed25519 签名
```

Solana 直接规定：

```
tx id = 第一条 signature
```



# 交易签名过程

1 构造 transaction message
2 序列化 message
3 使用私钥 Ed25519 签名
4 将 signature 放入 transaction
5 serialize transaction
6 sendTransaction



Transaction
 ├─ signatures[]
 └─ message



Message
 ├─ header
 ├─ account keys
 ├─ recentBlockhash
 └─ instructions

**`recentBlockhash` 是交易 message 里的一个字段，用于防止交易重放并限制交易的有效时间（TTL）**。

Solana 交易不是永久有效。

有效期：

```
≈150 blocks
≈60-90 秒
```

# recentBlockhash 如何获取

通过 RPC：

```
getLatestBlockhash
```

示例：

```
{
  "jsonrpc": "2.0",
  "result": {
    "context": { "slot": 246325123 },
    "value": {
      "blockhash": "5Y6K...abc",
      "lastValidBlockHeight": 221456987
    }
  }
}
```

返回两个关键值：

| 字段                 | 说明            |
| -------------------- | --------------- |
| blockhash            | recentBlockhash |
| lastValidBlockHeight | 交易过期高度    |

# Validator 如何验证

Validator 在接收交易时会检查：

```
recentBlockhash 是否在 blockhash queue
```

每个 validator 会维护一个：

```
recent blockhash queue
```

长度大约：

```
150 blocks
```

验证逻辑：

```
if blockhash not in queue:
    reject transaction
```





# 交易状态阶段

# 交易时间轴

一笔典型交易时间线：

```
T+0s   sendTransaction
T+0.4s processed
T+1-2s confirmed
T+15-30s finalized
```

## processed

含义：

```
交易已经被 leader 打包进一个 block
```

但：

```
还没有经过网络多数验证
```

特点：

| 属性         | 说明        |
| ------------ | ----------- |
| 是否可能回滚 | ✅ 可能      |
| 确认节点     | 当前 leader |
| 安全性       | 低          |
| 时间         | ~400ms      |

## confirmed

含义：

```
超过 2/3 stake validator 已确认该 block
```

也就是：

```
supermajority vote
```

特点：

| 属性         | 说明       |
| ------------ | ---------- |
| 是否可能回滚 | 极少       |
| 确认节点     | >66% stake |
| 安全性       | 高         |
| 时间         | ~1-2 秒    |

原因：

Solana validator 会对区块进行：

- vote transaction
- fork choice

通常需要：

```
2-3 slots
```

## finalized

含义：

```
区块已经被 root
```

也就是：

```
不可回滚
```

这是 Solana 最终确定状态。

特点：

| 属性         | 说明      |
| ------------ | --------- |
| 是否可能回滚 | ❌ 不可能  |
| 共识         | Tower BFT |
| 安全性       | 最终确定  |
| 时间         | ~12-32 秒 |

原因：

Solana 的 Tower BFT 需要：

```
31 vote lockout
```

因此 finalization 需要多个 slot。



# getTokenAccountsByOwner RPC 

返回指定代币所有者的所有 SPL Token 账户。

```
[
  ownerPubkey,
  {
    mint?: mintAddress,
    programId?: tokenProgramId
  },
  {
    encoding?: "jsonParsed"
  }
]
```



## programId 过滤的是账户类型

当使用：

```
{
  "programId": "Tokenkeg..."
}
```

意思是：

```
查询 owner 拥有的所有
由 Token Program 管理的账户
```

也就是：

```
所有 SPL Token Account
```

例如返回：

```
USDC token account
USDT token account
RAY token account
...
```

所以：

```
programId = 过滤账户类型
```

------

## mint 过滤的是 token 类型

当使用：

```
{
  "mint": "USDC mint"
}
```

意思是：

```
查询 owner 拥有的
这个 token 的账户
```

例如：

```
USDC token account A
USDC token account B
```

所以：

```
mint = 过滤 token 种类
```





# signatureSubscribe

监听 **某个交易的状态变化**。

## 请求示例

```
{
  "method": "signatureSubscribe",
  "params": [
    "TransactionSignature",
    {
      "commitment": "confirmed"
    }
  ]
}
```

------

## 返回示例

```
{
  "method": "signatureNotification",
  "params": {
    "result": {
      "err": null
    }
  }
}
```

含义：

```
交易执行成功
```

如果失败：

```
{
  "err": {
    "InstructionError": [...]
  }
}
```



# logsSubscribe

## 作用

订阅 **交易执行日志**。

日志来自程序执行时的：

```
Program log:
```

通常用于监听：

- 某个 program 的交易
- DEX swap
- DeFi 事件

------

## 请求示例

```
{
  "method": "logsSubscribe",
  "params": [
    {
      "mentions": ["ProgramAddress"]
    },
    {
      "commitment": "confirmed"
    }
  ]
}
```

------

## 过滤方式

支持三种：

| filter       | 含义       |
| ------------ | ---------- |
| all          | 所有交易   |
| allWithVotes | 包含投票   |
| mentions     | 包含某地址 |

最常用：

```
{
  "mentions": ["programId"]
}
```

意思：

```
任何调用该 program 的交易
```

------

## 返回示例

```
{
  "method": "logsNotification",
  "params": {
    "result": {
      "signature": "5h3...",
      "logs": [
        "Program log: swap start",
        "Program log: swap success"
      ]
    }
  }
}
```

重要字段：

| 字段      | 含义     |
| --------- | -------- |
| signature | 交易签名 |
| logs      | 程序日志 |





# accountSubscribe

## 作用

监听 **某个账户的数据变化**。

例如：

- 钱包余额变化
- token account 余额变化
- program state 变化

------

## 请求示例

```
{
  "method": "accountSubscribe",
  "params": [
    "AccountAddress",
    {
      "encoding": "jsonParsed",
      "commitment": "confirmed"
    }
  ]
}
```

------

## 返回示例

```
{
  "method": "accountNotification",
  "params": {
    "result": {
      "value": {
        "lamports": 1000000,
        "owner": "11111111111111111111111111111111"
      }
    }
  }
}
```

------

## 常见用途

### 钱包余额监听

监听：

```
SOL balance
```

------

### Token余额监听

监听：

```
Token Account
```

例如：

```
USDC ATA
```







# Solana Program Deploy 的真实流程

部署一个程序并不是一笔交易，而是 **4个阶段**：

### 1️⃣ 创建 buffer account

先创建一个 **buffer account** 用来暂存程序二进制。

交易：

```
createAccount
```

账户类型：

```
UpgradeableLoaderState::Buffer
```

------

### 2️⃣ 分块写入程序数据（大量 write）

`.so` 文件会被 **拆成很多 chunk**，每个 chunk 大概 **~900 bytes**。

每个 chunk 都需要一笔交易：

```
write(buffer, offset, bytes)
```

例如：

```
write offset 0
write offset 900
write offset 1800
write offset 2700
...
```

如果程序是：

```
150 KB
```

大约需要：

```
150000 / 900 ≈ 166 笔 write 交易
```

所以你看到：

```
write
write
write
write
write
```

是完全正常的。

------

### 3️⃣ deploy / finalize program

当所有 chunk 写完：

```
deploy(buffer → program account)
```

交易：

```
deploy_with_max_program_len
```

此时：

- 创建 program account
- 绑定 programdata
- 设置 upgrade authority

------

### 4️⃣ close buffer

最后：

```
close(buffer)
```

把剩余 lamports 退回钱包。





`logsSubscribe` 和 `accountSubscribe` 是 Solana 的两种 WebSocket 订阅方法，它们在不同的场景下非常有用。下面是它们的适用场景及区别：

### 1. `logsSubscribe` - 适用于监听日志和交易执行结果

**适用场景：**

- **监听交易日志**：`logsSubscribe` 用于订阅特定的日志输出，特别是在智能合约执行（如调用 SPL Token 相关函数、Solana 程序等）时。这种方式适用于你需要跟踪链上交易的执行过程，特别是在某些操作或函数执行时，你希望捕获合约的日志信息。
- **捕获交易状态**：如果你关心某些交易的状态（如某个交易是否成功、某个智能合约函数的执行结果等），`logsSubscribe` 是一个理想的选择。它可以让你订阅包含详细日志的事务，包括错误信息、状态变更等。
- **诊断和调试**：当你需要查看和分析 Solana 上发生的事件时，尤其是在开发和调试阶段，`logsSubscribe` 会非常有用。

**示例：**

- 你正在监听一个特定的合约地址，并希望追踪其状态变更、是否发生错误等信息。
- 你需要知道某个交易是否成功提交或失败，并获取失败的具体原因。

```
const { Connection, clusterApiUrl } = require('@solana/web3.js');

const connection = new Connection(clusterApiUrl('mainnet-beta'), 'confirmed');

// 订阅交易日志
const subscriptionId = connection.onLogs(
    "特定程序ID",
    (log, context) => {
        console.log("交易日志:", log);
    }
);
```

------

### 2. `accountSubscribe` - 适用于监听账户状态变化

**适用场景：**

- **监听账户余额变化**：`accountSubscribe` 适用于监听某个账户的状态变化，尤其是余额变动。你可以在用户钱包的余额或 SPL Token 余额发生变化时接收通知。这是跟踪账户活动的首选方法。
- **监控账户变化**：如果你需要监听某个特定账户的状态更新，或者检查账户的余额、授权等变动，`accountSubscribe` 提供了一种高效的方式来做到这一点。
- **钱包账户监控**：对于钱包应用、DeFi 应用等，需要实时监控用户账户的余额和状态，`accountSubscribe` 可以直接提供账户数据的变化。

**示例：**

- 你需要监听用户钱包地址的余额变化或资产变动。
- 你关心某个账户的余额变化（如接收到的支付、Token 转账等）。

```
const { Connection, clusterApiUrl } = require('@solana/web3.js');

const connection = new Connection(clusterApiUrl('mainnet-beta'), 'confirmed');

// 订阅账户变化
const subscriptionId = connection.onAccountChange("钱包地址", (accountInfo, context) => {
    console.log("账户信息变动:", accountInfo);
});
```

------

### 总结对比

| **方法**           | **适用场景**                                                 | **应用举例**                                               |
| ------------------ | ------------------------------------------------------------ | ---------------------------------------------------------- |
| `logsSubscribe`    | 监听交易和智能合约的日志，捕获执行的详细信息，适合诊断、调试和监控交易执行状态。 | 监听某个程序的交易日志，检查交易是否成功，或获取错误消息。 |
| `accountSubscribe` | 监听账户余额和状态的变化，适合实时监控账户变动。             | 监听钱包地址的余额变化或SPL Token转账。                    |

### 何时使用：

- 使用 `logsSubscribe` 时，你关心的是 **交易执行的详细日志** 和 **程序的状态信息**，这对于调试、追踪某些合约的行为很有用。
- 使用 `accountSubscribe` 时，你关心的是 **账户余额和账户状态的变化**，尤其是关注钱包或 Token 账户的余额变化。







# 共识

 Solana 中，共识机制并不是单一算法，而是 **多个机制组合**形成的高性能共识体系。核心由三部分组成：

```
Proof of History (PoH) + Tower BFT + Proof of Stake (PoS)
```

简单理解：

```
PoH   = 提供可验证时间
PoS   = 决定谁参与共识
Tower BFT = 达成最终共识
```
