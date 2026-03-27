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
交易签名列表里第一个签名结果
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





# 为什么自己写的程序更适合用 event

因为你能控制协议接口和日志设计。

如果只监听 instruction，你拿到的是：

- 用户调用了哪个方法
- 传了什么参数
- 带了哪些账户

这只能表示：

```
用户请求了什么
```

但链下系统真正更关心的通常是：

- 实际处理结果是什么
- 实际扣了多少
- 实际 mint 了多少
- 实际 fee 是多少
- 是否走了某条分支
- 某些派生值是多少

这些往往**不是 instruction 参数本身**，而是**程序执行后的结果**。

所以对于你自己的程序，最适合的是在指令执行成功后，主动 emit 一份你定义好的业务事件。



# 事件规范

  1. 事件语义稳定

  - 一条指令只表达一类核心动作，不要混太多副作用。
  - 指令名要直接对应业务语义，比如 add_to_whitelist、remove_from_whitelist、pause_mint。
  - 同类动作的账户顺序、参数顺序保持稳定。

  这样链下解析时，不需要为每个版本写一堆特例。

    2. 日志要结构化
    
    不要只打随意文本，尽量固定格式。比如：

  msg!("EVENT:WhitelistAdded mint={} token_account={} signer={}", mint, token_account, signer);
  msg!("EVENT:TransferAllowed mint={} source={} destination={} amount={}", mint, source, destination, amount);
  msg!("EVENT:TransferRejected reason=AccountNotInWhiteList mint={} source={} destination={}", mint, source, destination);

  规范建议：

  - 每条事件统一前缀，比如 EVENT:
  - 事件名固定，不要今天 WhitelistAdded 明天 AddWhiteList
  - 字段名固定，统一用 key=value
  - 同类事件字段顺序固定

  这样链下服务甚至不用完整解码指令，只看 log 就能先跑起来。

    3. 自定义错误码稳定且有语义
    
    你现在这种 Custom: 6002 最终还是要映射回业务含义，所以：

  - 错误码不要频繁变
  - 每个错误码只表达一种失败原因
  - README / 文档里明确错误码语义

  例如：

  - 6000: Unauthorized
  - 6001: IsNotCurrentlyTransferring
  - 6002: AccountNotInWhiteList

  链下就可以直接做失败分类。

    4. PDA seeds 设计可推导
    
    PDA 的 seeds 要让链下可以明确、稳定地自行推导。
    比如：

  - ["white_list"]
  - ["extra-account-metas", mint]

  更好的规范是：

  - seed 字符串固定
  - 能带业务主键就带业务主键
  - 不依赖隐式上下文

  链下服务最怕“得先查一个账户才能知道另一个账户怎么推”。

    5. 账户字段命名和最小必要信息
    
    链上账户结构要让链下读出来就知道它是什么。
    比如白名单账户至少有：

  - authority
  - mint 或明确说明是全局
  - version（如果未来会升级）
  - 核心业务字段

  如果账户结构未来可能演进，建议早点加：

  - version: u8
  - bump: u8（可选）
  - reserved: [u8; N]（可选）

  这样后面升级兼容性好很多。

    6. 账户里存“业务主键”，不要只靠 PDA 推断
    
    即使某个字段能从 PDA 推出来，也建议在账户里重复存一下关键值，比如：

  - mint
  - token_account
  - authority

  因为链下解析、审计、导出时会更简单，也更不容易出错。

    7. 指令账户顺序固定且文档化
    
    链下如果要从交易里还原事件，最依赖的就是：

  - 哪个账户是 mint
  - 哪个是 source
  - 哪个是 destination
  - 哪个是 authority

  所以要做到：

  - 同一类指令账户顺序不要改
  - README 明确写出来
  - 如果改了，明确版本边界

    8. 链下事件优先从“成功交易 + 指令 + 日志”三者交叉确认

    所以链上设计时最好保证：

  - 指令本身能看出是什么动作
  - 日志能补足业务字段
  - 账户能回溯状态

  不要把关键业务信息只藏在其中一种渠道里。

    9. 版本化
    
    如果你预计程序会升级，建议现在就定规矩：

  - 日志事件名不要乱改
  - 账户加 version
  - 链下解析器按 program_id + version 分支处理

  否则以后升级后，历史数据和新数据很难统一解析。

    10. 面向链下的最小规范模板
    
    你可以直接把这个当团队规范：

  - 指令名使用稳定业务语义
  - 所有关键动作打 EVENT:<Name> key=value...
  - 自定义错误码稳定且文档化
  - PDA seeds 固定且可公开推导
  - 账户结构包含 version
  - 关键业务主键在账户里显式存储
  - 指令账户顺序固定
  - README 中维护“指令/账户/日志/错误码”对照表

  对你这个项目，最值得立刻补的其实只有两件事：

    1. 给关键路径加结构化日志
       例如：

  - WhitelistAdded
  - WhitelistRemoved
  - TransferAllowed
  - TransferRejected

    2. 在 README 里补一张解析对照表
       写清楚：

  - 指令名
  - 账户顺序
  - 主要日志
  - 错误码含义
