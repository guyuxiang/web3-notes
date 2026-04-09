# Zama

> **Zama = 把 FHE（同态加密）工程化，让它可以在区块链/智能合约里用**

它不是单纯一个算法，而是一整套体系：

- FHE 算法（核心）
- 编译器 / SDK
- 协处理器（coprocessor）
- 密钥管理（MPC / KMS）
- 解密网关（gateway）
- 合约接口（如 ERC-7984）

Zama 的技术路线核心是：

> **TFHE（Fast Fully Homomorphic Encryption over the Torus）**

TFHE 的特点：

- 支持**布尔运算（AND / OR / NOT）**
- 支持小整数运算（如 8/16/32/64 bit）
- **bootstrapping  快速刷新非常快（关键优势）**



## 加密整数

TFHE 本身是 bit-level 的。

Zama 做了一个重要工程封装：

> **把多个 bit 组合成“加密整数”（euint）**

比如：

- `euint8`
- `euint32`
- `euint64`

一个 `euint64`：

- = 64 个加密 bit
- 每个 bit 都是 TFHE 密文

运算时：

- 加法 → bitwise + carry
- 比较 → bitwise comparator
- 逻辑 → 布尔电路

👉 本质：

> **把 CPU 的 ALU（算术逻辑单元）搬到密文世界**



## 快速刷新

在所有 FHE 里，最贵的操作是：

> **Bootstrapping（刷新密文）**

Zama 的核心优势就是：

> **把 bootstrapping 做到足够快，能支撑程序执行**

- 每次运算 → 噪声变大
- 噪声太大 → 解密失败

所以必须：

> **定期“清理噪声”**



## Zama 系统架构

### ① 前端 / SDK

- 生成密文（encrypt）
- 构造 input proof
- 发送交易

------

### ② 智能合约（EVM）

- 存储密文句柄（不是数据本身）
- 调用 FHE 操作（逻辑层）

------

### ③ Coprocessor（核心）

👉 最关键组件

负责：

- 实际执行 FHE 运算
- 处理：
  - 加法
  - 乘法
  - 比较
  - 逻辑
- 返回新的密文句柄

👉 可以理解为：

> **“加密版 CPU”**

------

### ④ KMS / MPC

- 管理私钥
- 防止单点泄露
- 支持 threshold 解密

------

### ⑤ Gateway（解密层）

- 控制谁能解密
- 执行解密请求
- 返回明文结果





### 引入 Coprocessor 计算网络

### 密文不在链上存储完整数据

- 链上只存 pointer
- 实际计算在外部

#### 链上（EVM / fhEVM）

负责：

- 记录：

  - ciphertext handle（密文引用）

- 调用：

  - FHE 操作（逻辑指令）

  

  链上的“符号执行”（Symbolic Execution）

当智能合约调用 FHE 运算（如 `FHE.add(x, y)`）时，主链节点并不执行真正的全同态加密运算。相反，fhEVM 库会进行**符号执行**

**生成新指针：** 主链通过对输入句柄（`bytes32`）、操作类型（如 `fheAdd`）和结果类型进行哈希运算（如 `keccak256`），确定性地生成一个新的 `bytes32` 结果句柄

**发出指令：** 合约随后触发一个事件（`FheOpEvent`），其中包含输入句柄、操作类型和新生成的输出句柄



**Zama 的同态加解密运算**
 **绝大部分不在智能合约里执行，而是在链下的“协处理器（coprocessor）”里执行。**

Coprocessor：

- 执行 TFHE 运算
- 做 bootstrapping
- **返回新密文**

> **EVM 根本跑不动 FHE**

计算量爆炸

FHE 运算涉及：

- 大整数运算
- 多项式卷积（FFT / NTT）
- 模运算
- bootstrapping（非常重）

### Zama 的核心设计：链上 + 链下分离

```
用户
 ↓
前端加密（SDK）
 ↓
智能合约（EVM）
 ↓（调用）
Coprocessor（链下FHE计算）
 ↓
存入数据库
```

**符号执行 (Symbolic Execution)：** 当智能合约在主链（Host Chain）上运行时，它并不执行真正的加密运算。相反，它使用**句柄 (Handles)** 来代表加密数据。合约通过哈希函数快速生成新句柄并触发事件，通知协处理器进行后续处理。这种方式确保了主链的高吞吐量和低 Gas 成本

**实际 FHE 执行 (Actual FHE Execution)：** 协处理器是部署在链下的组件，它持续监听主链事件。

**密文检索：** 协处理器根据事件中的 `bytes32` 句柄，从其**链下数据库**（例如 Zama 部署中使用的 AWS S3）中调取关联的实际密文数据

一旦收到任务，它会从数据库中调取对应的密文，利用高效的 **TFHE 方案**（支持精确且无限制的计算）协处理器使用 Zama 的 **TFHE-rs** 库,在强大的 CPU/GPU 硬件上完成复杂的同态运算，并将结果密文**存回数据库**，并将其索引在之前主链确定的**输出句柄**（那个 `bytes32` 指针）之下

**可验证性与扩展性：** 为了防止协处理器作恶，所有任务都是**公开可验证的**，任何人都可以重新计算并核对结果

**分布式密钥生成：** FHE 的私钥并非存储在单一实体中，而是通过 MPC 协议被切分成多个**秘密分片 (Secret Shares)**，由 *n* 个独立的 MPC 节点（通常为声誉良好的组织）分别持有，只有当足够数量（达到门限值 *t*）的节点协作时，才能完成解密，任何单一节点都无法获取完整私钥或独自解密数据

**Gateway** 在此架构中充当“管家”的角色。它负责协调主链、协处理器和 KMS 之间的交互：管理访问控制列表 (ACL) 以确保解密请求合法，并在各组件达成共识后发布最终结果

![image-20260407115008308](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260407115008308.png)

- [**FHEVM Solidity 库**](https://docs.zama.org/protocol/protocol/overview/library)：使开发人员能够使用加密数据类型和操作，以纯 Solidity 编写机密智能合约。
- [**宿主合约**](https://docs.zama.org/protocol/protocol/overview/hostchain)：部署在与 EVM 兼容的区块链上的可信链上合约。它们管理访问控制并触发链下加密计算。
- [**协处理器**](https://docs.zama.org/protocol/protocol/overview/coprocessor)– 去中心化服务，用于验证加密输入、运行 FHE 计算并提交结果。
- [**网关**](https://docs.zama.org/protocol/protocol/overview/gateway) **——**协议的中央协调器。它验证加密输入，管理访问控制列表（ACL），跨链桥接密文，并协调协处理器和KMS。
- [**密钥管理服务 (KMS)**](https://docs.zama.org/protocol/protocol/overview/kms) – 一个阈值 MPC 网络，用于生成和轮换 FHE 密钥，并处理安全、可验证的解密。
- **中继器**– 一种轻量级的链下服务，通过转发加密或解密请求来帮助用户与网关进行交互。



## 余额判断

在 fhEVM 中，由于账户余额是加密的，智能合约无法直接使用传统的 `if (balance >= amount)` 语句来判断余额是否充足。系统通过一种称为**同态逻辑选择**的机制来处理这一过程，确保在不解密数据的情况下完成验证。

以下是具体的判断和转账流程：

1. 加密逻辑比较

合约首先会调用加密比较操作（如 `FHE.lte`，即 Less Than or Equal）。例如，它会将转账金额 `amount` 与发送者余额 `balances[from]` 进行比较。

- 使用 `select` 算子进行分支模拟

由于无法在加密值上直接进行逻辑跳转（Branching），fhEVM 使用 `select` 算子来模拟分支逻辑。

- “无影响”执行

接下来的转账操作会照常执行，无论余额是否充足，交易都不会在链上直接失败（Revert）：

**扣款：** `balances[from] = FHE.sub(balances[from], txAmount)`。**入账：** `balances[to] = FHE.add(balances[to], txAmount)`。**结果：** 如果余额不足，由于 `txAmount` 是加密的 `0`，减去 0 和加上 0 后，双方的余额在逻辑上保持不变，但对外界观察者来说，这看起来像是一次正常的加密转账。



# Zama 里有哪些“关键密钥”？

一个完整的隐私转账，会涉及 4 类密钥👇

------

## 1️⃣ 用户密钥（User Key / Wallet Key）

👉 就是你熟悉的：

- Ethereum 私钥（EOA）

用途：

- 签名交易
- 调用合约

👉 **不负责隐私！只负责身份 & 授权**

------

## 2️⃣ FHE 公钥（Encryption Public Key）🔥

👉 用来：

- 把金额加密成密文

例如：

```
amount = 100
→ Enc(pk, 100)
```

特点：

- 通常是“系统级”或“合约级”
- 所有人都可以用它加密

------

## 3️⃣ FHE 私钥（Decryption Key）🔥核心

👉 用来：

- 解密链上的密文数据

⚠️ 关键点：

- **不会给单个用户**
- 通常由：
  - 多方（MPC）
  - 或去中心化网络
     持有

------

👉 用途：

- 当需要 reveal 时：
  - 解密余额
  - 解密结果

------

## 4️⃣ 权限密钥（Access Control Key / Viewing Key）🧠

👉 控制：

- 谁可以“看到”数据

例如：

- 用户自己能看余额
- 监管机构可审计（可选）

------

👉 实现方式：

- re-encryption（重加密）
- 或 access policy