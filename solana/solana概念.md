# Solana核心概念

# [账户](https://solana.com/zh/docs/core/accounts)

Solana 网络上的所有数据都存储在账户中。您可以将 Solana 网络视为一个包含单一账户表的公共数据库。账户与其地址之间的关系类似于键值对，其中键是地址，值是账户。

每个账户都有相同的基本[结构](https://solana.com/zh/docs/core/accounts#account-structure)，并且可以通过其[地址](https://solana.com/zh/docs/core/accounts#account-address)找到。

![三个账户及其地址的示意图，包括账户结构定义。](https://solana.com/assets/docs/core/accounts/accounts.png)

## [账户地址](https://solana.com/zh/docs/core/accounts#账户地址)

账户地址是一个 32 字节的唯一 ID，用于在 Solana 区块链上定位账户。账户地址通常以 base58 编码字符串的形式显示。大多数账户使用 [Ed25519](https://ed25519.cr.yp.to/) [公钥](https://solana.com/zh/docs/core/accounts#public-key) 作为其地址，但这并不是强制性的，因为 Solana 还支持[程序派生地址](https://solana.com/zh/docs/core/accounts#program-derived-address)。

 Ed25519是一种计算快，安全性高，且生成的签名内容小的一种不对称加密算法。新一代公链几乎都支持这个算法。



## [账户结构](https://solana.com/zh/docs/core/accounts#账户结构)

每个 [`Account`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/account/src/lib.rs#L48-L60) 最大容量为 [10MiB](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/system_instruction.rs#L85)，包含以下信息：

- `lamports`：账户中的 lamports 数量
- `data`：账户数据
- `owner`：拥有该账户的 program 的 ID
- `executable`：指示账户是否包含可执行二进制文件
- `rent_epoch`：已弃用的 rent epoch 字段



### Lamports

账户的余额以 [lamports](https://solana.com/docs/references/terminology#lamport) 为单位。

每个账户必须有一个最低 lamport 余额，称为 [租金](https://solana.com/docs/references/terminology#rent)，这允许其数据存储在链上。租金与账户的大小成正比。

虽然这个余额被称为 rent，但它更像是押金，因为当账户关闭时，可以全额取回余额。（"rent" 这个名称来源于现已弃用的 rent epoch 字段。）

### data

该字段通常被称为“账户数据”。此字段中的 `data` 被视为任意内容，因为它可以包含任意字节序列。每个 program 会定义此字段中存储数据的结构。

- 程序账户：此字段包含可执行程序代码或[程序数据账户](https://solana.com/zh/docs/core/accounts#program-data-accounts)的地址，该账户存储可执行程序代码。
- 数据账户：此字段通常存储状态数据，供读取使用。

从 Solana 账户读取数据涉及两个步骤：

1. 通过 [address](https://solana.com/zh/docs/core/accounts#account-address) 获取账户
2. 将账户的 `data` 字段从原始字节反序列化为相应的数据结构，具体结构由拥有该账户的 program 定义。

### owner

此字段包含账户所有者的程序 ID。

每个 Solana 账户都有一个 [program](https://solana.com/docs/core/programs) 作为其 owner。

**只有该账户的 owner program 可以更改账户的 `data` 或根据 program 指令扣除 lamports。**

`owner = System Program`
 👉 你不能写（只能转 lamports）

`owner = Token Program`
 👉 只有 SPL Token 能写

`owner = 你的 Program`
 👉 **只有你能写**





### executable

此字段指示账户是[程序账户](https://solana.com/zh/docs/core/accounts#program-accounts)还是[数据账户](https://solana.com/zh/docs/core/accounts#data-accounts)。

- 如果 `true`：该账户是 program account
- 如果 `false`：该账户是 data account



### 租赁 epoch

**`rent_epoch` 字段已弃用。**

过去，此字段用于跟踪账户何时需要支付租金。然而，此租金收取机制现已被弃用。



## [账户类型](https://solana.com/zh/docs/core/accounts#账户类型)

账户分为两大类：

- [程序账户](https://solana.com/zh/docs/core/accounts#program-accounts)：包含可执行代码的账户
- [数据账户](https://solana.com/zh/docs/core/accounts#data-accounts)：不包含可执行代码的账户

程序代码与其状态的分离是 Solana 账户模型的一个关键特性。（类似于操作系统，通常将程序和其数据分为不同的文件。）



### [程序账户](https://solana.com/zh/docs/core/accounts#程序账户)

每个程序都由一个[Loader加载器程序](https://solana.com/docs/core/programs#loader-programs)拥有，用于部署和管理账户。当部署一个新的[程序](https://solana.com/docs/core/programs)时，会创建一个账户来存储其[可执行](https://solana.com/zh/docs/core/accounts#executable)代码。这被称为程序账户。（为了简化，可以将程序账户视为程序本身。）

在下图中，你可以看到一个 loader program 被用来部署一个 program account。program account 的 `data` 字段包含可执行的程序代码。

![程序账户、其四个组成部分及其加载器程序的示意图](https://solana.com/assets/docs/core/accounts/program-account-simple.svg)

### [数据账户](https://solana.com/zh/docs/core/accounts#数据账户)

数据账户不包含可执行代码，而是用于存储信息。

**常见的数据账户细分：**

#### [程序数据账户](https://solana.com/zh/docs/core/accounts#程序数据账户)

使用 loader-v3 部署的 Programs 在其 `data` 字段中不包含程序代码。相反，它们的 `data` 指向一个单独的 **program data account**，其中包含程序代码。（见下图。）

data = 程序字节码 + 元数据（比如 upgrade authority）

![一个程序账户及其数据。数据指向一个单独的程序数据账户](https://solana.com/assets/docs/core/accounts/program-account-expanded.svg)

在程序部署或升级期间，缓冲账户用于临时存储上传内容。

Buffer Account（缓冲账户）部署/升级时用于暂存程序字节码：

- `owner = BPFLoader`



#### [程序状态账户](https://solana.com/zh/docs/core/accounts#程序状态账户)

程序使用数据账户来维护其状态。为此，必须创建一个新的数据账户。可以是私钥账户，也可以是PDA账户

**私钥账户**

1. 调用 [System Program](https://solana.com/docs/core/programs#the-system-program) 来创建一个账户。（然后 System Program 将所有权转移给新程序。）
2. 根据其 [instructions](https://solana.com/docs/core/instructions) 初始化账户数据。

**PDA账户**

大多数程序都会有一个显式的初始化指令：`initialize / init` 指令

```
initialize(
  payer,
  state_account (PDA),
  system_program
)
```

执行过程：

1. 用户发交易调用 `initialize`
2. 程序 CPI 调 System Program：
   - `create_account`
   - `owner = program id`
   - `space = state size`
3. 写入初始状态数据

![由程序账户拥有的数据账户示意图](https://solana.com/assets/docs/core/accounts/data-account.svg)



#### [系统账户](https://solana.com/zh/docs/core/accounts#系统账户)

**本质**：就是你的“钱包地址”。

**所有者**：由系统程序 (`System Program`) 拥有。

**数据为空**：它的 `data` 字段长度为 0。
**特点**：只能存取原生代币 **SOL**，可以作为交易的支付方（Gas Fee 付费者）。

当 SOL 第一次被发送到一个新地址时，会在该地址创建一个由 System Program 拥有的账户。



**系统账户可以转换成数据账户**

**准备阶段**：你生成一个 Keypair，往里面充点 SOL（作为系统账户存在）。

**分配空间 (`Allocate`)**：你调用指令，告诉系统这个账户要存 100 字节的数据。

**所有权转移 (`Assign`)**：这是最关键的一步。你执行 `Assign` 指令，把这个账户的 **Owner** 从 `System Program` 改成你的 **Program ID**。

**身份转变完成**：一旦 Owner 变了，它就不再是“系统账户”了，它变成了受你程序控制的**“程序状态账户”**。



数据账户既有data，又可以有sol，比如WSOL的token account



### 特殊账户[系统变量Sysvar ](https://solana.com/zh/docs/core/accounts#sysvar-账户)

这是一类特殊的只读账户，存储了 Solana 集群的实时状态，程序可以直接读取它们。

- **Clock**: 当前的时间戳和槽位（Slot）。
- **Rent**: 当前的租金费率标准。
- **Stake History**: 质押的历史数据。

Sysvar 账户存在于预定义的地址，并提供对集群状态数据的访问。它们会动态更新网络集群的相关数据。查看完整列表：[Sysvar Accounts](https://docs.anza.xyz/runtime/sysvars)。





### [交易](https://solana.com/zh/docs/core/transactions)

一个 [`Transaction`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/src/transaction/mod.rs#L207) 包含以下信息：

- `signatures`：一个 [签名](https://solana.com/zh/docs/core/transactions#signatures) 数组
- `message`：交易信息，包括待处理的指令列表

**顺序**执行多条指令，交易是**原子性**的：如果单条指令失败，整个交易将失败，并且不会发生任何更改。

交易的总大小限制为 [1232](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/packet/src/lib.rs#L29) 字节，此限制旨在避免典型互联网基础设施上的数据包分片



### signatures

交易的 `signatures` 数组包含 `Signature` 结构体。每个 [`Signature`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/signature/src/lib.rs#L30) 为 64 字节，由账户私钥对交易的 `Message` 签名生成。每个包含在任意交易指令中的 [签名账户](https://solana.com/zh/docs/core/transactions#account-addresses) 都必须提供签名。

第一个签名属于支付交易[基础费用](https://solana.com/zh/docs/core/docs/core/fees#base-fee)的账户，并且是交易签名。交易签名可用于在网络上查找交易的详细信息。



### message

交易的 `message` 是一个 [`Message`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/message/legacy.rs#L131) 结构体，包含以下信息：

- `header`：消息 [头部](https://solana.com/zh/docs/core/transactions#header)
- `account_keys`：交易指令所需的 [账户地址](https://solana.com/zh/docs/core/transactions#account-addresses) 数组
- `recent_blockhash`：作为交易时间戳的 [区块哈希](https://solana.com/zh/docs/core/transactions#recent-blockhash)
- `instructions`：[指令](https://solana.com/zh/docs/core/transactions#instructions) 数组



### header

消息的 `header` 是一个 [`MessageHeader`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/message/mod.rs#L97) 结构体。包含以下信息：

- `num_required_signatures`：交易所需的签名总数
- `num_readonly_signed_accounts`：需要签名的只读账户总数
- `num_readonly_unsigned_accounts`：不需要签名的只读账户总数



### account_keys

消息的 [`account_keys`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/message/legacy.rs#L138) 是一个账户地址数组，采用[紧凑数组格式](https://solana.com/docs/references/terminology#compact-array-format)发送。

数组前缀表示其长度。数组中的每一项都是一个公钥，指向指令所用的账户。

`accounts_keys` 数组必须完整，并严格按照如下顺序排列：

1. 签名者 + 可写
2. 签名者 + 只读
3. 非签名者 + 可写
4. 非签名者 + 只读

account_keys[0] = fee payer

![显示账户地址数组顺序的图示](https://solana.com/assets/docs/core/transactions/compat_array_of_account_addresses.png)



为什么必须写在 header 里？

#### 1️⃣ 签名验证不需要解析指令

验证器

- 先看 header
- 知道前 N 个账户必须签名
- 直接做签名校验
- ❌**不需要解析 Instruction / Program**



因为accounts_keys是顺序排列的，

因此通过num_required_signatures和num_readonly_signed_accounts可以知道在已签名账户中，前几个账户是可写的，之后 num_readonly_signed_accounts 个是只读的

使用num_readonly_unsigned_accounts同理，计算出在后面在未签名账户中，那几个是可写的，最后 num_readonly_unsigned_accounts 个是只读的

整个 `account_keys` 被划分为 **4 段**：

```typescript
[ signed & writable ]
[ signed & readonly ]
[ unsigned & writable ]
[ unsigned & readonly ]
```

### 目的

因为Solana的执行模型是并行的，显式声明签名数量和只读账户数量，是为了

- 让运行时在解析指令之前就能完成签名校验
- 账户锁定
- 做并行调度的冲突检测，从而支持高性能、确定性的并行执行。



### recent_blockhash 

消息的 `recent_blockhash` 是一个哈希值，作为交易的时间戳，并防止重复交易。

一个 blockhash 在 [150 个区块](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/clock/src/lib.rs#L134) 后会过期。（假设每个区块为 400ms，相当于一分钟。）区块过期后，交易也会过期，无法被处理。

[`getLatestBlockhash`](https://solana.com/docs/rpc/http/getlatestblockhash) RPC 方法允许你获取当前 blockhash 以及该 blockhash 有效的最后区块高度。



### instructions

消息的 [`instructions`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/message/legacy.rs#L146) 是所有待处理指令的数组，采用[紧凑数组格式](https://solana.com/docs/references/terminology#compact-array-format)发送。数组前缀表示其长度。数组中的每一项都是一个 [`CompiledInstruction`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/instruction.rs#L22) 结构体，并包含以下信息：

1. `program_id_index`：指向 [`account_keys`](https://solana.com/zh/docs/core/transactions#account-addresses) 数组中某个地址的索引。该值表示处理该指令的程序地址。
2. `accounts`：指向 `account_keys` 数组中地址的索引数组。每个索引都指向该指令所需账户的地址。
3. `data`：指定要在程序上调用的指令的字节数组，同时包含指令所需的其他数据（如函数参数）。

![指令的紧凑数组](https://solana.com/assets/docs/core/transactions/compact_array_of_ixs.png)



### 指令

一个[指令](https://solana.com/docs/terminology#instruction)是 Solana 上最小的执行逻辑



调用指令本质上是调用一个公共函数，任何使用 Solana 网络的人都可以调用。

指令调用程序，该程序调用 Solana 运行时以更新状态，是更新全局 Solana 状态的调用

每个指令可以是一个单独的函数调用，每个指令用于执行特定的操作。指令的执行逻辑存储在[程序](https://solana.com/docs/core/programs)中，每个程序定义其自己的指令集。

要与 Solana 网络交互，需要将一个或多个指令添加到[交易](https://solana.com/docs/core/transactions)中并发送到网络进行处理。

在交易中按顺序执行。交易是原子性的，这意味着如果其中任何指令失败，整个交易将失败，你只需支付交易费用。



[SOL 转账示例](https://solana.com/zh/docs/core/instructions#sol-转账示例)

发送方的钱包发送包含其[签名](https://solana.com/docs/references/terminology#signature)和包含 SOL 转账指令的消息的交易。交易发送后，系统程序处理转账指令并更新两个账户的 lamport 余额。

![SOL 转账图示](https://solana.com/assets/docs/core/transactions/sol-transfer.svg)

一个 [`Instruction`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/instruction/src/lib.rs#L94) 包含以下信息：

- `program_id`：该指令业务逻辑的程序的公钥地址，实际为节省数据大小，是上面account_keys的数组索引

- `accounts`：一个账户数组，实际为节省数据大小，是上面account_keys的数组索引，（该字段是为了只要不修改同一个账户，交易就可以并行执行指令。）

- `data`：包含指令所需额外 [数据] 的字节数组

  

accounts（SDK构造交易时用的抽象结构）包括以下信息：

- pubkey：账户的公钥地址
- is_signer：如果账户必须签署交易，则设置为 `true`
- is_writable：如果指令会修改账户数据，则设置为 `true`

它**不是链上结构**,只是 SDK（Rust / JS）用来汇总所有 instruction 的账户,最后会排序生成 `account_keys + header + instruction.accounts`



data

指令的 `data` 是一个字节数组，用于指定要调用的程序指令，也包含指令所需的参数。





### 示例交易结构

```json
{
    "transaction": {
        "message": {
            "header": {
                "numRequiredSignatures": 1,
                "numReadonlySignedAccounts": 0,
                "numReadonlyUnsignedAccounts": 1
            },
            "accountKeys": [
                "EPLUagqZZAuAtJ5LSbK7eeXjqeTdesd4q8WhoqVrfG3g",
                "9Txf5pi5jzm7FydFAsQafk7xn5wY9yN2UNm5LW15qvcK",
                "11111111111111111111111111111111"
            ],
            "recentBlockhash": "2qYPgehzMKXcMt4Ku1tKAk9DACKUbtYEY9EUEN42cseT",
            "instructions": [
                {
                    "programIdIndex": 2,
                    "accounts": [
                        0,
                        1
                    ],
                    "data": "3Bxs4NN8M2Yn4TLb"
                }
            ],
            "indexToProgramIds": {}
        },
        "signatures": [
            "3jUKrQp1UGq5ih6FTDUUt2kkqUfoG2o4kY5T1DoVHK2tXXDLdxJSXzuJGY4JPoRivgbi45U2bc7LZfMa6C4R3szX"
        ]
    }
}
```





# [交易手续费](https://solana.com/zh/docs/core/fees)

每笔 Solana 交易都需要支付交易手续费，费用以 SOL 结算。

交易手续费分为两部分：基础费用和优先费用。

基础费用用于补偿 validator 处理交易。优先费用是可选的，用于提高当前 leader 处理你交易的概率。



### 基础费用

每笔交易每包含一个签名需支付 5000 [lamports](https://solana.com/docs/references/terminology#lamport)。该费用由交易的第一个签名者支付。只有 System Program 拥有的账户可以支付交易手续费。基础费用分配如下：

- **50% 销毁：** 一半会被[销毁](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/fee-calculator/src/lib.rs#L70)（从 SOL 流通总量中移除）。
- **50% 分配：** 另一半[支付给处理该交易的 validator](https://github.com/anza-xyz/agave/blob/e621336acad4f5d6e5b860eaa1b074b01c99253c/runtime/src/bank/fee_distribution.rs#L58-L62)。

（0.000000001 [sol](https://solana.com/docs/terminology#sol) = 1 lamport）

### 优先费用

[优先费用](https://github.com/anza-xyz/agave/blob/v2.2.14/compute-budget/src/compute_budget_limits.rs#L47-L48)是一种可选费用，用于提高当前 leader（validator）处理你交易的概率。validator 会获得[100% 的优先费用](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0096-reward-collected-priority-fee-in-entirety.md)。你可以通过调整交易的[计算单元](https://solana.com/docs/references/terminology#compute-units)（CU）价格和 CU 限额来设置优先费用。



优先费用的计算方式如下：

```
Prioritization fee = CU limit * CU price
```

优先费用用于确定你的[交易优先级](https://github.com/anza-xyz/agave/blob/v2.2.14/core/src/banking_stage/transaction_scheduler/receive_and_buffer.rs#L646)，相对于其他交易。其计算公式如下：

```
Priority = (Prioritization fee + Base fee) / (1 + CU limit + Signature CUs + Write lock CUs)
```



### 计算单元限额

[默认情况下](https://github.com/anza-xyz/agave/blob/v2.1.13/compute-budget/src/compute_budget.rs#L149-L197)，每条指令分配 [200,000 CU](https://github.com/anza-xyz/agave/blob/v2.1.13/compute-budget/src/compute_budget_limits.rs#L10)，每笔交易分配 [140 万 CU](https://github.com/anza-xyz/agave/blob/v2.1.13/compute-budget/src/compute_budget_limits.rs#L14)。你可以在交易中包含一个 [`SetComputeUnitLimit`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/src/compute_budget.rs#L42-L44) 指令来更改这些默认值。每个引用的账户在每个区块最多可使用 12,000,000 计算单位。这个限制防止一个账户在单个区块中过多地锁定写入，进一步防止本地费用市场被一个账户占据。



要为您的交易计算合适的 CU 限额，建议按照以下步骤操作：

1. 通过[模拟](https://solana.com/developers/guides/advanced/how-to-request-optimal-compute)交易，估算所需的 CU 单位数
2. 在估算值基础上增加 10% 的安全余量

**优先费是由请求的计算单元（CU）限额决定的，*而不是*实际使用的计算单元数量。如果您设置的 CU 限额过高或使用默认值，可能会为未使用的计算单元付费。**



### 计算单元价格

计算单元价格是为每个请求的 CU 支付的可选 [micro-lamports](https://solana.com/docs/references/terminology#micro-lamports) 数量。您可以将 CU 价格视为一种小费，用于激励 validator 优先处理您的交易。要设置 CU 价格，请在交易中包含一个 [`SetComputeUnitPrice`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/src/compute_budget.rs#L48-L50) 指令。

默认的 CU 价格为 0，这意味着默认的

如需帮助确定适合您交易的 CU 价格，请参阅下表中提供的实时 CU 价格建议。

[QuickNode](https://www.quicknode.com/)

[Helius](https://www.helius.dev/)



交易的另一个限制是你可以在单个指令中进行的**指令调用深度**。这个限制目前设置为 4，这意味着在交易将回滚之前，你只能在深度为 4 的位置调用指令。这使得 Solana 上不存在重入问题，与你在以太坊上需要担心的问题相比。



### 租金

更像是押金而不是费用。当你在网络上创建账户或分配空间时，你必须为网络存入一些 SOL 以保持你的账户。租金是根据网络上存储的字节数计算的，并且为分配空间收取额外的基础费用。重要的是要注意，租金费用不会丢失；如果你关闭账户并允许集群重新收回分配的空间，那么这些租金费用可以被收回。

## Rent-Exempt（免租）机制（重点）

### 💡 核心规则

> **只要一次性存够“最低 lamports”，账户就永久免租**

```
rent_exempt_minimum = Rent::minimum_balance(data_len)
```

- 跟 **账户数据大小** 有关
- 跟时间无关
- 一次性支付

👉 **99% 的程序 / 状态账户都是 rent-exempt**



### 创建新账户时

钱是在 `System Program::CreateAccount` 指令执行时，由 runtime 自动从 `payer` 扣 lamports 并记到新账户里的。

**校验签名**

runtime 检查：

- `payer` 是否是 signer
- `new_account` 是否是 signer（非 PDA 情况）
- lamports 是否足够

Transaction
 └── Instruction: SystemProgram::CreateAccount
       ├── from: payer（用户钱包）
       ├── to: new_account（程序 / PDA / buffer）
       ├── lamports: rent_exempt_minimum
       ├── space: data_len
       └── owner: 某个 program





# [程序](https://solana.com/zh/docs/core/programs)

在 Solana 上，智能合约被称为程序。程序是一种无状态的 [账户](https://solana.com/docs/core/accounts#program-account)，其中包含可执行代码。这些代码被组织为称为指令的函数。用户通过发送包含一个或多个 [指令](https://solana.com/docs/core/instructions) 的 [交易](https://solana.com/docs/core/transactions) 与程序进行交互。一笔交易可以包含来自多个程序的指令。



当程序被部署时，Solana 会使用 [LLVM](https://llvm.org/) 将其编译为可执行与可链接格式（[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)）。ELF 文件包含以 Solana 字节码格式（sBPF）编译的程序二进制文件，并保存在链上的可执行账户中。sBPF 是 Solana 定制的 [eBPF](https://en.wikipedia.org/wiki/EBPF) 字节码版本。



### 编写程序

大多数程序都是用 [Rust](https://rust-book.cs.brown.edu/title-page.html) 编写的，常见的开发方式有两种：

- [Anchor](https://www.anchor-lang.com/docs)：Anchor 是为 Solana 快速、便捷开发而设计的框架。它利用 [Rust 宏](https://doc.rust-lang.org/book/ch20-05-macros.html) 来减少样板代码，非常适合初学者。
- [原生 Rust](https://solana.com/docs/programs/rust)：直接用 Rust 编写程序，不依赖任何框架。这种方式更灵活，但复杂度也更高。



### 更新程序

要[修改](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/bpf_loader/src/lib.rs#L704)现有程序，必须使用有升级权的限账户（通常是最初[部署程序](https://solana.com/docs/programs/deploying)的账户）如果升级权限被撤销并设置为 `None`，则该程序将无法再被更新。

```
solana program deploy <program_filepath>
```

将其状态降级为不可变：

```
solana program set-upgrade-authority <program_address> --final
```



### 验证程序

Solana 支持[可验证构建](https://solana.com/docs/programs/verified-builds)，用户可以检查链上程序代码是否与其公开源代码一致。Anchor 框架提供了[内置支持](https://www.anchor-lang.com/docs/verifiable-builds)来创建可验证构建。

要检查现有程序是否已验证，请在 [Solana Explorer](https://explorer.solana.com/address/PhoeNiXZ8ByJGLkxNfZRnkUfjvmuYqLR89jjFHGqdXY) 上搜索其 program ID。或者，你也可以使用 Ellipsis Labs 的 [Solana Verifiable Build CLI](https://github.com/Ellipsis-Labs/solana-verifiable-build)，独立验证链上程序。



### [内置程序](https://solana.com/zh/docs/core/programs#内置程序)

[The System Program](https://solana.com/zh/docs/core/programs#the-system-program)

System Program 是唯一可以创建新账户的账户。默认情况下，所有新账户都归 [System Program](https://github.com/anza-xyz/agave/tree/v2.1.13/programs/system/src) 所有，尽管许多账户在创建时又会被分配给新的所有者。



| 功能                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [新账户创建](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/system/src/system_processor.rs#L146) | 只有 System Program 可以创建新账户。                         |
| [空间分配](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/system/src/system_processor.rs#L71) | 设置每个账户数据字段的字节容量。                             |
| [分配程序所有权](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/system/src/system_processor.rs#L113) | System Program 创建账户后，可以将指定的程序所有者重新分配给其他 program account。自定义程序就是通过这种方式获得 System Program 创建的新账户的所有权。 |
| [转账 SOL](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/system/src/system_processor.rs#L215) | 将 lamports（SOL）转移到其他账户。                           |

system program 的地址是 `11111111111111111111111111111111`。



### [Loader 程序](https://solana.com/zh/docs/core/programs#loader-程序)

每个程序都归另一个账户所有——即其 loader。Loader 用于部署、重新部署、升级或关闭程序，也用于完成程序和转移程序权限。

Loader 程序有时也被称为“BPF Loaders”。



目前有五个 loader 程序，如下表所示。

| Loader | Program ID                                    | 说明                                                         | 指令链接                                                     |
| ------ | --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| native | `NativeLoader1111111111111111111111111111111` | 拥有其他四个 loader                                          | —                                                            |
| v1     | `BPFLoader1111111111111111111111111111111111` | 管理指令已禁用，但程序仍可执行                               | —                                                            |
| v2     | `BPFLoader2111111111111111111111111111111111` | 管理指令已禁用，但程序仍可执行                               | [指令](https://docs.rs/solana-loader-v2-interface/latest/solana_loader_v2_interface/enum.LoaderInstruction.html) |
| v3     | `BPFLoaderUpgradeab1e11111111111111111111111` | 程序部署后可更新。可执行文件存储在单独的 program data account 中 | [指令](https://docs.rs/solana-loader-v3-interface/latest/solana_loader_v3_interface/instruction/enum.UpgradeableLoaderInstruction.html) |
| v4     | `LoaderV411111111111111111111111111111111111` | 开发中（未发布）                                             | [指令](https://docs.rs/solana-loader-v4-interface/latest/solana_loader_v4_interface/instruction/enum.LoaderV4Instruction.html) |

使用 loader-v3 或 loader-v4 部署的程序，在部署后可能仍可修改，具体取决于其升级权限。



当新程序部署时，默认会使用最新的 loader 版本。

### 

### [预编译程序](https://solana.com/zh/docs/core/programs#预编译程序)

除了 loader 程序外，Solana 还提供以下预编译程序。

#### [验证 ed25519 签名](https://solana.com/zh/docs/core/programs#验证-ed25519-签名)

ed25519 程序用于验证一个或多个 ed25519 签名。

| 程序         | 程序 ID                                       | 描述                                                    | 指令                                                         |
| ------------ | --------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| Ed25519 程序 | `Ed25519SigVerify111111111111111111111111111` | 验证 ed25519 签名。如果有任何签名验证失败，则返回错误。 | [指令](https://docs.rs/solana-ed25519-program/latest/solana_ed25519_program/index.html) |



#### [验证 secp256k1 公钥恢复](https://solana.com/zh/docs/core/programs#验证-secp256k1-公钥恢复)

secp256k1 程序用于验证 secp256k1 公钥恢复操作。

| 程序           | 程序 ID                                       | 描述                                       | 指令                                                         |
| -------------- | --------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| Secp256k1 程序 | `KeccakSecp256k11111111111111111111111111111` | 验证 secp256k1 公钥恢复操作（ecrecover）。 | [指令](https://docs.rs/solana-secp256k1-program/latest/solana_secp256k1_program/index.html) |

| 程序           | 程序 ID                                       | 描述                                                         | 指令                                                         |
| -------------- | --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Secp256r1 程序 | `Secp256r1SigVerify1111111111111111111111111` | 验证最多 8 个 secp256r1 签名。接收签名、公钥和消息。如果有任何验证失败，则返回错误。 | [指令](https://docs.rs/solana-secp256r1-program/latest/solana_secp256r1_program/all.html) |



### [核心程序](https://solana.com/zh/docs/core/programs#核心程序)

下表中的程序为网络提供核心功能。

| 程序                     | 程序 ID                                       | 描述                                                         | 指令                                                         |
| ------------------------ | --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **System**               | `11111111111111111111111111111111`            | 创建新账户、分配账户数据、将账户分配给所属程序、从 System Program 拥有的账户转移 lamports，并支付交易费用 | [SystemInstruction](https://docs.rs/solana-program/latest/solana_program/system_instruction/enum.SystemInstruction.html) |
| **Vote**                 | `Vote111111111111111111111111111111111111111` | 创建和管理跟踪 validator 投票状态和奖励的账户                | [VoteInstruction](https://docs.rs/solana-vote-program/latest/solana_vote_program/vote_instruction/enum.VoteInstruction.html) |
| **Stake**                | `Stake11111111111111111111111111111111111111` | 创建和管理代表委托给 validator 的质押和奖励的账户            | [StakeInstruction](https://docs.rs/solana-sdk/latest/solana_sdk/stake/instruction/enum.StakeInstruction.html) |
| **Config**               | `Config1111111111111111111111111111111111111` | 向链中添加配置信息，并指定允许修改该配置的公钥列表。与其他程序不同，Config 程序没有定义任何单独的指令。它只有一个隐式指令：“store”。其 instruction data 是一组控制账户访问权限和存储数据的密钥 | [ConfigInstruction](https://docs.rs/solana-config-program/latest/solana_config_program/config_instruction/index.html) |
| **Compute Budget**       | `ComputeBudget111111111111111111111111111111` | 设置交易的计算单元限制和价格，允许用户控制计算资源和优先级费用 | [ComputeBudgetInstruction](https://docs.rs/solana-compute-budget-interface/latest/solana_compute_budget_interface/enum.ComputeBudgetInstruction.html) |
| **Address Lookup Table** | `AddressLookupTab1e1111111111111111111111111` | 管理地址查找表，使交易能够引用比账户列表中可容纳的更多账户   | [ProgramInstruction](https://docs.rs/solana-sdk/latest/solana_sdk/address_lookup_table/instruction/enum.ProgramInstruction.html) |
| **ZK ElGamal Proof**     | `ZkE1Gama1Proof11111111111111111111111111111` | 为 ElGamal 加密数据提供零知识证明验证                        | —                                                            |



# [PDA程序派生地址](https://solana.com/zh/docs/core/pda)



PDA 是使用程序 ID 和一组可选的预定义输入确定性创建的地址。PDA 看起来与公钥地址类似，但没有对应的私钥。

Solana 运行时允许程序为 PDA 签名而无需私钥。



## [背景](https://solana.com/zh/docs/core/pda#背景)

Solana 的 [keypair](https://github.com/anza-xyz/solana-sdk/blob/sdk@v2.2.2/keypair/src/lib.rs#L26) 是 [Ed25519 曲线](https://ed25519.cr.yp.to/)（椭圆曲线加密）上的点。它们由公钥和私钥组成。公钥成为账户地址，私钥用于为账户生成有效的[签名](https://solana.com/docs/core/transactions#signatures)。

PDA 被有意派生为落在 Ed25519 曲线之外。这意味着它没有有效的对应私钥，无法执行加密操作（例如提供签名）。然而，Solana 允许程序为 PDA 签名而无需私钥。

![非曲线地址](https://solana.com/assets/docs/core/pda/address-off-curve.svg)



[派生一个 PDA](https://solana.com/zh/docs/core/pda#派生一个-pda)

![程序派生地址](https://solana.com/assets/docs/core/pda/pda.svg)

在使用 PDA 创建账户之前，您必须首先派生地址。

Solana SDK 支持使用下表中显示的函数创建 PDA。每个函数接收以下输入：

- **程序 ID**：用于派生 PDA 的程序地址。该程序可以代表 PDA 签名。
- **可选种子**：预定义的输入，例如字符串、数字或其他账户地址。

| SDK                            | 功能                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| `@solana/kit` (Typescript)     | [`getProgramDerivedAddress`](https://github.com/anza-xyz/kit/blob/v2.1.0/packages/addresses/src/program-derived-address.ts#L157) |
| `@solana/web3.js` (Typescript) | [`findProgramAddressSync`](https://github.com/solana-foundation/solana-web3.js/blob/v1.98.0/src/publickey.ts#L212) |
| `solana_sdk` (Rust)            | [`find_program_address`](https://github.com/anza-xyz/solana-sdk/blob/sdk@v2.2.2/pubkey/src/lib.rs#L617) |

### [标准 bump](https://solana.com/zh/docs/core/pda#标准-bump)

bump seed 是附加到可选种子后的一个额外字节。派生函数从 255 开始迭代 bump 值，每次递减 1，直到找到一个生成有效非曲线地址的值。第一个生成有效非曲线地址的值被称为“标准 bump”。



该函数使用程序 ID 和可选种子，然后通过迭代 bump 值尝试创建一个有效的程序地址。bump 值的迭代从 255 开始，每次递减 1，直到找到一个有效的 PDA。找到有效的 PDA 后，函数返回 PDA 和 bump seed。



这意味着，在给定相同的可选 seeds 和 `programId` 的情况下，具有不同值的 bump seed 仍然可以派生出有效的 PDA。

![PDA 派生](https://solana.com/assets/docs/core/pda/pda-derivation.svg)

###  SPL（[Solana Program Library](https://github.com/solana-program)）代币



- [Token Program](https://solana.com/zh/docs/tokens#token-program) 包含了在网络上与代币（包括同质化和非同质化）交互的全部指令逻辑。
- [Mint Account](https://solana.com/zh/docs/tokens#mint-account) 代表某一特定代币，并存储该代币的全局元数据，如总发行量和铸造权限（有权创建新代币单位的地址）。
- [Token Account](https://solana.com/zh/docs/tokens#token-account) 用于跟踪特定所有者在特定 mint account 下的代币持有情况。由keypair创建
- [Associated Token Account](https://solana.com/zh/docs/tokens#associated-token-account) 是通过所有者和 mint account 地址进行PDA派生出的一种 Token Account。



### [Token Program](https://solana.com/zh/docs/tokens#token-program)

Solana 生态系统有两个主要的 Token Program。以下是这两个程序的源代码。

### Token Program（原版）

- 基础代币功能（铸造、转账等）
- 不可变且被广泛使用

### Token Extension Program（Token 2022）

- 包含原始 Token Program 的所有功能
- 通过“扩展”增加新特性



Token Program 包含了在网络上与代币（包括同质化和非同质化）交互的全部指令逻辑。Solana 上的所有代币本质上都是由 Token Program 拥有的 [数据账户](https://solana.com/docs/core/accounts#data-account)（程序状态账户）

![Token Program](https://solana.com/assets/docs/core/tokens/token-program.svg)

### [Mint Account](https://solana.com/zh/docs/tokens#mint-account)

Solana 上的代币由 Token Program 拥有的 [Mint Account](https://github.com/solana-program/token/blob/6d18ff73b1dd30703a30b1ca941cb0f1d18c2b2a/program/src/state.rs#L16-L30) 地址唯一标识。该账户作为某一特定代币的全局计数器，并存储如下数据：

- **Supply**：该代币的总发行量
- **Decimals**：代币的小数精度
- **Mint authority**：有权铸造新代币、增加总供应量的账户
- **Freeze authority**：有权冻结 Token Account 中代币、阻止其转账或销毁的账户

![Mint Account](https://solana.com/assets/docs/core/tokens/mint-account.svg)



### [Token Account](https://solana.com/zh/docs/tokens#token-account)

 [token account](https://github.com/solana-program/token/blob/6d18ff73b1dd30703a30b1ca941cb0f1d18c2b2a/program/src/state.rs#L87-L108)，用于追踪每个代币单位的个人持有情况

token account 会存储如下数据：

- **Mint**：该 token account 持有的代币
- **Owner**：有权从该 token account 转移代币的账户
- **Amount**：该 token account 当前持有的代币数量

![Token Account](https://solana.com/assets/docs/core/tokens/token-account.svg)

#### 普通Token Account的创建过程

**1. 准备地址：生成 Keypair**

首先，在链下（比如在你的前端代码或 CLI 中）生成一个新的 **Keypair**。

- 这个 Keypair 的公钥（Public Key）就是未来的 **Token Account 地址**。
- 注意：这是它被称为“普通”的原因，因为它的地址是随机生成的，而不是推导出来的。

**2. 第一步：向系统程序申请空间 (System Program)**

你必须先调用 **System Program** 的 `CreateAccount` 指令。

- **主要目的**：在链上开辟一块固定大小的空间（对于 Token Account，固定为 **165 字节**）。
- **支付租金**：你（创建者）需要转入足够的 SOL 到这个新地址，以达到“租金豁免”标准。
- **指定所有权**：这是最关键的一步。在创建时，你必须指定该账户的 **Owner** 为 **Token Program**（地址：`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`）。

**3.第二步：初始化账户数据 (Token Program)**

此时，账户在链上已经存在了，且归 Token Program 管，但里面是一片空白（全是 0）。

- 接下来，你需要调用 **Token Program** 的 `InitializeAccount`（或 `InitializeAccount3`）指令。
- **参数**：你需要告诉程序，这个账户属于哪个 **Mint**（代币类型，如 USDC）以及谁是这个账户的 **Authority**（拥有管理权的钱包地址）。
- **动作**：Token Program 会检查这个账户是否由它拥有且数据长度正确，然后将 Mint 地址、Owner 地址、余额（初始为 0）等状态写入那 165 字节的空间里。



每个钱包想要持有某个代币（mint）时，都需要一个对应的 token account，并将钱包地址设置为 token account 的 owner。

一个钱包可以拥有多个相同代币（mint）的 token account

![账户关系](https://solana.com/assets/docs/core/tokens/token-account-relationship.svg)

请注意，每个 token account 的数据都包含一个 `owner` 字段，用于标识谁拥有该 token account 的权限。这与基础 [Account](https://solana.com/docs/core/accounts#account-type) 类型中指定的程序 owner 不同，后者对于所有 token account 来说都是 Token Program。



### [ATA关联代币账户](https://solana.com/zh/docs/tokens#关联代币账户)

**背景目的**：当别人想给你转 USDC 时，他不知道你的Token Account是多少?

Associated Token Account 简化了查找特定 mint 和所有者的 token account 地址的流程

它通过一套固定的公式，根据你的wallet address和mint address，推导出一个**唯一的、标准化的**Token Account。

这意味着：只要知道你的**钱包地址**。以及**代币的 Mint 地址**（比如 USDC 的合约地址）。任何人都能算出你的 ATA 地址，而不需要你去“告诉”他们。





Associated Token Account 是通过从所有者地址和 mint account 地址派生出的地址创建的。

ATA 本质上是一个 **PDA**（程序派生地址）

ATA地址通过 **PDA推导**：

```
ATA = PDA(
    seeds = [
        wallet_address,
        TOKEN_PROGRAM_ID,
        token_mint_address
    ],
    program_id = ASSOCIATED_TOKEN_PROGRAM_ID
)
```

$$
seeds = \text{Hash}(\text{Wallet Address} + \text{Token Program ID} + \text{Mint Address})
$$
需要注意的是，Associated Token Account 其实就是一个具有特定地址的 token account。

一个钱包只会有一个ATA关联代币账户，这个Associated Token Account 理解为某个 mint 和所有者的“默认” token account。

![Associated Token Account](https://solana.com/assets/docs/core/tokens/associated-token-account.svg)



一个钱包owner、一个mint地址，针对token和token2022 程序可以计算出两个ATA



**发送者创建**：如果你给朋友转一个他从未持有的代币，你的交易指令通常会顺便帮他创建一个 ATA（所以你会发现转账时手续费稍贵了一点点，那是在付租金）。



# TA创建时发生了什么

ATA创建流程：

1️⃣ 客户端调用
 spl-associated-token-account

2️⃣ ATA Program 做三件事：

**① 计算 ATA 地址**

```
find_program_address(
  [wallet, TOKEN_PROGRAM_ID, mint],
  ASSOCIATED_TOKEN_PROGRAM_ID
)
```

**② 创建账户**

通过 System Program：

```
system_program::create_account
```

**③ 初始化 Token Account**

调用
 spl-token

```
initialize_account
```

并设置：

```
account.owner = TOKEN_PROGRAM_ID
```

所以ata账户级别的owner是TOKEN_PROGRAM



### 普通 Token Account vs. ATA

| **特性**     | **普通 Token Account**  | **Associated Token Account (ATA)** |
| ------------ | ----------------------- | ---------------------------------- |
| **地址生成** | Keypair随机生成，不好找 | 确定性生成，公式可算               |
| **数量限制** | 一个钱包可以有无数个    | 一个钱包一个代币对应一个           |





### SPL授权委托

Token Account 里和委托相关的字段

```
struct TokenAccount {
  mint: Pubkey,
  owner: Pubkey,              // 真正的账户所有者
  amount: u64,                // 当前余额

  delegate: Option<Pubkey>,   // 被批准的代理人
  delegated_amount: u64,      // 代理人最多可花多少

  state: AccountState,
  close_authority: Option<Pubkey>,
}
```

委托信息 **存在 Token Account 里**



调用Token Program的Approve 写入委托信息

SPL Token 的一个 Token Account 在同一时间只能有「一个」delegate



## [理解 Token 权限](https://solana.com/zh/docs/tokens/basics/set-authority#理解-token-权限)

### [Mint Account 权限](https://solana.com/zh/docs/tokens/basics/set-authority#mint-account-权限)

- **Mint Authority**：控制新代币的铸造。可以将代币铸造到任何 token account。通常在初始发行后撤销该权限，以实现固定供应的代币。
- **Freeze Authority**：控制冻结和解冻 token account 的能力。可以阻止任何 token account 转账代币。通常会撤销该权限，以确保用户的代币无法被冻结。

### [Token Account 权限](https://solana.com/zh/docs/tokens/basics/set-authority#token-account-权限)

- **Account Owner**：对 token account 拥有完全控制权。可以转账代币、销毁代币、授权代理人以及在余额为零时关闭账户。
- **Close Authority**：当账户余额为零时，可以关闭 token account。默认情况下为账户所有者，但也可以委托给其他账户。



该 [`SetAuthority`](https://github.com/solana-program/token/blob/a7c488ca39ed4cd71a87950ed854929816e9099f/program/src/instruction.rs#L153) 指令可更改或撤销 mint account 和 token account 的权限。只有当前权限持有者可以将权限转移到新地址，或通过将权限设置为 null 永久撤销。一旦撤销，权限无法恢复。



### 包装 SOL (WSOL)

Native Mint 仍然是一个 Mint Account

```
Mint Account
├─ owner = SPL Token Program
├─ decimals = 9
├─ supply = 0   （逻辑上无意义）supply 由 lamports 决定
├─ mint_authority = None
├─ freeze_authority = None
```

包装 SOL (WSOL) 是一种代币账户，用于将账户中的 lamports 数量作为代币余额进行跟踪。WSOL 使与 DeFi 协议的集成成为可能，可以将 SOL 作为 SPL 代币进行转移。

Token Program 的硬编码检查使得wSol无法被普通issuer

```
if mint_pubkey == NATIVE_MINT_ID {
    return Err(ProgramError::InvalidInstruction);
}

pub const NATIVE_MINT_ID: Pubkey = pubkey!("So11111111111111111111111111111111111111112");
```





WSol的Token Account

```
Account (owner = SPL Token Program)
├─ lamports     ← 实际 SOL
├─ data.amount  ← SPL Token 里的余额
```

**lamports ≠ data.amount（默认）**

**当转入时必须手动同步**

[`SyncNative`](https://github.com/solana-program/token/blob/a7c488ca39ed4cd71a87950ed854929816e9099f/interface/src/instruction.rs#L387) 指令用于同步已包装 SOL（WSOL） token account 的余额与实际存储在其中的 SOL（ lamports ）数量。

当你向 WSOL token account 转入原生 SOL 时，token 余额不会自动更新。你需要调用 SyncNative ，以反映正确的 WSOL 余额。



**转账 wSOL** 

SPL Token Program 在处理 `transfer` 指令时，
 对 Native Mint 走了一条“特殊分支逻辑”

Token Program 减少 `amount`同时减少 `lamports`

**两者始终一致**



## [关闭账户](https://solana.com/zh/docs/tokens/basics/close-account#关闭账户)

[`CloseAccount`](https://github.com/solana-program/token/blob/a7c488ca39ed4cd71a87950ed854929816e9099f/program/src/instruction.rs#L212C5-L212C17) 指令会永久关闭一个 token account，并将所有剩余的 SOL（ rent ）转入指定的目标账户。关闭前， token account 的余额必须为零。只有 token account 的所有者或指定的关闭权限人才能执行此操作。



### 冻结账户

[`FreezeAccount`](https://github.com/solana-program/token/blob/a7c488ca39ed4cd71a87950ed854929816e9099f/program/src/instruction.rs#L228) 指令会阻止某个特定 token 账户的所有 token 转账或销毁操作。

一旦被冻结，该账户将无法发送、接收 token，也无法关闭，直到被解冻。

只有该 token 铸造合约的冻结权限（freeze authority）才能冻结账户。

 [`ThawAccount`](https://github.com/solana-program/token/blob/a7c488ca39ed4cd71a87950ed854929816e9099f/program/src/instruction.rs#L243) 指令用于解除冻结，使之前被冻结的 token account 恢复全部功能。解冻后，该账户可以像正常一样发送和接收代币。只有该代币 mint 的冻结权限方才能解冻账户。

如果在铸造账户上撤销（设置为 null）冻结权限，则该 token 将无法再被冻结。





## Token Extensions Program（Token 2022）

Token Extensions（也称为 Token-2022）是在 Solana 区块链上一个高级代币程序，扩展了现有 Token Program 的能力。它旨在为开发人员提供增强的灵活性和额外的功能，而不会影响当前代币的安全性。

该程序涵盖了其前身Token Program的所有功能（与原始 Token 指令和账户布局保持兼容），同时提供新的指令和功能。这些扩展在铸造和账户中引入了新字段

#### 当前的铸造扩展包括：

| 特性           | 描述                                                         |
| :------------- | :----------------------------------------------------------- |
| 转账费用       | 与转移代币相关的费用（或税收）。                             |
| 关闭铸造       | 该 [`MintCloseAuthority`](https://github.com/solana-program/token-2022/blob/6f2473344d70271f632c3e9b7e945be00186c536/interface/src/instruction.rs#L569) 扩展允许关闭铸币账户并回收其 rent。 |
| 计息代币       | 随时间累积利息的代币。                                       |
| 非可转移代币   | 一旦发行就不能转移的代币（即，灵魂绑定代币）。               |
|                |                                                              |
| 转账钩子       | 需要与代币指令交互的程序（例如，版权费的执行）。             |
| 元数据指针     | 指向包含铸造信息的元数据账户的指针。                         |
| 元数据         | 存储有关铸造的信息，例如名称、符号和标志（或其他自定义字段）。 |
| 转账备注       | 当在 token account 上启用时， [`MemoTransferExtension`](https://github.com/solana-program/token-2022/blob/6f2473344d70271f632c3e9b7e945be00186c536/interface/src/instruction.rs#L623) 要求所有传入的 token 转账都必须附带一条备忘录。token account 的所有者可以启用或禁用此扩展功能。 |
| 默认状态       | [`DefaultAccountStateExtension`](https://github.com/solana-program/token-2022/blob/6f2473344d70271f632c3e9b7e945be00186c536/interface/src/instruction.rs#L593) 为为某个 mint 创建的所有新 token account 设置默认状态。启用后，此扩展会自动将每个新 token account 初始化为指定的状态，例如冻结。这样，token 创建者就可以强制执行特定行为，比如要求所有新账户默认被冻结，直到被明确解冻。 |
| 超级权限       | [`InitializePermanentDelegate`](https://github.com/solana-program/token-2022/blob/6f2473344d70271f632c3e9b7e945be00186c536/interface/src/instruction.rs#L678) 会为某个 mint 分配一个拥有所有 token account 管理权限的代理。与 token account 拥有者可以撤销的普通代理不同，永久代理无法被 token account 拥有者更改或移除。永久代理可以在该 mint 的任意 token account 中转移或销毁代币。当前永久代理可以将永久代理角色更改为新的账户。 |
| 计息代币       | 该 [`InterestBearingMintExtension`](https://github.com/solana-program/token-2022/blob/6f2473344d70271f632c3e9b7e945be00186c536/interface/src/instruction.rs#L654) 允许你直接在 mint account 上设置利率。利息会根据网络时间戳持续复利，但每个 token account 中实际存储的 token 余额不会发生变化。总价值（token 余额加上累计利息）只是一个计算结果，并不会实际铸造新 token。此扩展不会产生新的 token。 |
| Rebase Token   | 缩放 UI 数量扩展允许发行方为代币的 UI 数量应用一个可更新的倍数。 |
| 保密转账       | 保密转账允许你在 token account 之间转移代币时不公开转账金额。这对于保护隐私的交易非常有用。只有转账金额和 token 余额是私密的，token account 地址仍然是公开的。 |
| CPI Guard      | 可以防止通过跨程序调用（CPI）对代币账户进行意外转账。启用后，此扩展确保只有通过直接调用 Token Extensions Program 的转账指令才能转移代币，阻止所有其他程序通过 CPI 转移代币。 |
| Token 组和成员 | 相当于ERC-1155                                               |
| 可暂停         | [`PausableExtension`](https://github.com/solana-program/token-2022/blob/6f2473344d70271f632c3e9b7e945be00186c536/interface/src/instruction.rs#L733) 允许指定的暂停权限方暂停某个 mint 上的所有代币活动。暂停权限方可以随时暂停或恢复该 mint。 |



### 老 Token

```
use spl_token::ID as TOKEN_PROGRAM_ID;
```

### Token 2022

```
use spl_token_2022::ID as TOKEN_2022_PROGRAM_ID;
```

创建 ATA 也要用 **associated-token-account-2022** 版本。



------

## 1. Token-2022 程序账户 (Program Account)

- **地址**：`TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`
- **角色**：它是“逻辑中心”。
- **实现逻辑**：
  - **不可变代码**：程序账户本身只存储编译后的 BPF/SBF 字节码，不存储任何状态（余额、总量等）。
  - **指令集兼容**：Token-2022 实现了与旧 Token 程序完全相同的指令集（如 `Transfer`, `MintTo`, `Burn`），这意味着原本支持旧代币的钱包和代码，只需更换 `Program ID` 就能在基础功能上无缝衔接。
  - **扩展处理器**：它多了一套处理“扩展（Extensions）”的逻辑，专门解析 Mint 账户和 Token 账户尾部的额外数据。

------

## 2. Mint 账户 (Mint Account)

Mint 账户相当于代币的“印钞机”和“说明书”，定义了代币的总量、精度等全局属性。

### 具体实现结构：

Token-2022 的 Mint 账户采用了一种 **“固定头部 + 变长尾部 (TLV)”** 的存储方式。

1. **基础数据 (Base Data - 前 82 字节)**：

   为了保持兼容，前 82 位存储的是传统 SPL Token 的核心字段：

   - `Mint Authority`: 谁有权铸币。
   - `Supply`: 当前总供应量。
   - `Decimals`: 代币精度（如 9 表示 $10^9$ 为一个币）。
   - `Is Initialized`: 是否已初始化。
   - `Freeze Authority`: 谁有权冻结账户。

2. **TLV 扩展区 (Tag-Length-Value)**：

   这是 Token-2022 的核心创新。在 82 字节之后，程序会根据你启用的功能追加数据：

   - **Tag (类型)**：标识这是什么扩展（如：转账手续费、元数据指针）。
   - **Length (长度)**：告诉程序这段扩展数据占多少空间。
   - **Value (内容)**：实际的配置信息（如：手续费比例 0.3%）。

**常见的 Mint 扩展实现：**

- **Transfer Fee**: 每一笔转账自动扣除一部分费用。
- **Metadata**: 直接将代币名称、符号、URI 存入 Mint 账户，不再需要外部的 Metaplex 账户。
- **Transfer Hook**: 定义一个外部程序，在每次转账时强制调用该程序逻辑（如：黑名单校验）。

------

## 3. Token 账户 (Token Account)

Token 账户（常表现为 ATA，Associated Token Account）用于记录**具体某个用户**持有的代币数量。

### 具体实现结构：

与 Mint 账户类似，它也分为基础区和扩展区。

1. **基础数据 (Base Data - 前 165 字节)**：

   - `Mint`: 指向该账户属于哪种代币。
   - `Owner`: 谁拥有这个账户（通常是用户的钱包地址）。
   - `Amount`: 余额（以最细分单位表示）。
   - `Delegate`: 授权额度。
   - `State`: 账户状态（Uninitialized, Initialized, Frozen）。

2. **扩展区 (Extensions)**：

   当 Mint 账户开启了某些功能时，Token 账户也需要存储对应的状态：

   - **Immutable Owner**: 锁定所有权，防止账户被“转移”给别人（安全性增强）。
   - **Memo Required**: 强制要求转账时必须附带备注信息。
   - **CPI Guard**: 限制跨程序调用（CPI）时的某些操作，防止恶意合约盗取资产。



# RPC

交易状态

Not Found（未找到）节点尚未看到这笔交易。

Processed（已处理）交易被某个节点处理过，但还未确认。表示已经被 leader 处理，但未进入持久状态；可能会被回滚；

Confirmed（已确认）交易已经包含在某个 block 中，且有一定数量的确认（通常为 1 个确认）（三分之二）。但仍有极小的概率被重组回滚；

Finalized（最终确认）交易已被超级多数（2/3）以上的 validator 认可，**不可被回滚**。



在正常网络状态下：

- Solana 的 block 时间是 ~400ms；
- 一个 block 只需约 1 秒就能获得 1-2 次确认；
- `finalized` 通常在 32 个确认（大约 15 秒左右）时发生；
- 如果你已经达到了 `confirmed` 状态，并保持在线且网络未发生大规模重组，**回滚的概率极小（<0.01%）**。



在 **Solana RPC 请求**中，很多方法都支持一个参数叫做 `commitment`，它就是用来指定**你想查询的交易状态（区块确认级别）**。

```json
{
        "jsonrpc": "2.0",
        "id": 1,
        "method": "accountSubscribe",
        "params": [
            "CM78CPUeXjn8o3yroDHxUtKsZZgoy4GPkPPXfouKNH12",
            {
            "encoding": "jsonParsed",
            "commitment": "finalized"
            }
        ]
    }
```

- `commitment` 不是所有方法都必需，但**建议在所有状态查询相关的方法中使用**；
- 如果你不指定，默认使用节点的 `commitment`（通常是 `"finalized"`）；
- 写入类方法（如 `sendTransaction`）本身不会使用 commitment，但你后续监听它是否确认时会用到 commitment。

Websocket 接口中每个 Subscribe 方法，都对应的有一个 Unsubscribe 方法，当发送改方法时，服务器后续不再推送消息。



## 为什么 Solana 需要 PDA？

在以太坊里：

- 合约 = 地址 + 状态
- 合约天然能控制自己的 storage

在 Solana 里：

- 程序（Program）是**无状态的**
- 状态必须放在 **Account**
- Account 必须有一个“所有者（owner）”

 那问题来了：

> **如何创建一个“只能被某个程序控制”的账户？**

PDA 就是答案。

### 程序可以“代表 PDA 签名”

通过：

```
invoke_signed(...)
```

程序向 runtime 证明：

> “这个 PDA 是我派生的，我有权使用它”



### [Devnet 端点](https://solana.com/zh/docs/references/clusters#devnet-端点)

- `https://api.devnet.solana.com` - 单个由 Solana Labs 托管的 API 节点；有速率限制



本地化费用市场

根据 [Solana Beach](https://solanabeach.io/validators)，**目前有 1314 个验证者节点



### 账户模型

与以太坊不同，Solana 旨在利用高端机器中的多个核心。考虑到这一点，账户模型被设计为利用多个核心，创建一个可以使交易相互**并行化**的系统。

与以太坊不同，以太坊中的每个智能合约都是一个包含执行逻辑和存储绑定在一起的账户，而 Solana 的智能合约是完全**无状态**的。

**Solana 计数器程序**

```rust
#[program]
pub mod counter_anchor {
use super::*;


pub fn initialize_counter(_ctx: Context<InitializeCounter>) -> Result<()> {
  Ok(())
}

pub fn increment(ctx: Context<Increment>) -> Result<()> {

ctx.accounts.counter.count =   ctx.accounts.counter.count.checked_add(1).unwrap();
  Ok(())
 }
}

#[derive(Accounts)]
pub struct InitializeCounter<'info> {
#[account(mut)]
pub payer: Signer<'info>,
#[account(
init,
space = 8 + Counter::INIT_SPACE,
payer = payer
)]

pub counter: Account<'info, Counter>,
pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
  #[account(mut)]
  pub counter: Account<'info, Counter>,
}

#[account]
#[derive(InitSpace)]
pub struct Counter {
  count: u64,
}
```

在 Solidity 中，你有 `int private count = 0;`，而在 Solana 上的 Rust 智能合约中，你有一个名为 `initialize_counter` 的结构，该初始计数器创建一个具有 `count` 为 0 的账户，然后你可以将此账户传递给 `increment` 以增加 `count`。这样可以避免在智能合约本身内部具有状态。

有单独的账户存储程序外部的数据。要执行程序中的逻辑，你需要传递要执行操作的账户。在这个 `counter` 程序的情况下，当调用 `increment` 函数时，你将向程序传递一个 `counter` 账户，程序将增加 `counter` 账户中的值。





### 本地费用市场

拥有 Solana 账户模型的另一个幸运副作用是能够基于状态争用建模费用。如前所述，交易可以并行执行。但是，它们仅根据正在写入的账户来并行执行。例如，假设 Solana 上正在进行热门的 NFT 铸造活动。通常，这种热度会提高每个使用链的人的价格，但是在这种情况下，所有未参与 NFT 铸造的人不受影响。

顾名思义，费用市场是每个账户的本地费用。如果你在发送 USDC 转账时，而其他人都在铸造最热门的新 NFT，你将不受影响，并继续支付你在 Solana 上习惯的低费用。这适用于 Solana 中的任何应用程序，避免了你在以太坊上习惯的常见全局费用市场，同时降低了每个人的成本。



### 内存池在哪里？

与以太坊不同，Solana 上没有内存池。Solana 验证者将交易转发给排定的最多四个领导者。虽然 Solana 没有内存池，但它仍然有优先费用来帮助排序交易。没有内存池会导致交易从领导者跳到领导者，直到区块哈希过期，但它减少了跨集群传递内存池的开销。



# CPI([Cross Program Invocation](https://solana.com/zh/docs/core/cpi))

指的是一个 Solana 程序直接调用另一个程序的指令。这使得程序具备可组合性。

![跨程序调用示例](https://solana.com/assets/docs/core/cpi/cpi.svg)

在进行 CPI 时，账户权限会从一个程序扩展到另一个程序。假设程序 A 收到一个包含签名账户和可写账户的指令。然后，程序 A 对程序 B 发起 CPI。此时，程序 B 可以像程序 A 一样使用这些账户，并保留原有权限（也就是说，程序 B 可以用签名账户签名，也可以写入可写账户）。如果程序 B 继续发起自己的 CPI，这些权限也可以继续传递下去，最多可传递 4 层。

程序指令调用的最大高度被称为 [`max_instruction_stack_depth`](https://github.com/anza-xyz/agave/blob/v2.1.13/compute-budget/src/compute_budget.rs#L38) ，其值由 [MAX_INSTRUCTION_STACK_DEPTH](https://github.com/anza-xyz/agave/blob/v2.1.13/compute-budget/src/compute_budget.rs#L13) 常量设定为 5。

堆栈高度从初始交易的 1 开始，每当一个程序调用另一个指令时增加 1，因此 CPI 的调用深度限制为 4。



## [带有 PDA 签名者的 CPI](https://solana.com/zh/docs/core/cpi#带有-pda-签名者的-cpi)

当 CPI 需要 PDA 签名者时，会使用 [`invoke_signed`](https://github.com/anza-xyz/agave/blob/v2.1.13/sdk/program/src/program.rs#L51-L73) 函数。该函数接收用于派生签名者 [PDA](https://solana.com/docs/core/pda) 的 signer seeds。Solana 运行时会在内部调用 [`create_program_address`](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/bpf_loader/src/syscalls/cpi.rs#L552) ，并传入 `signers_seeds` 以及调用方程序的 `program_id`。当 PDA 验证通过后， [会被添加为有效签名者](https://github.com/anza-xyz/agave/blob/v2.1.13/programs/bpf_loader/src/syscalls/cpi.rs#L554)。