# Railgun

用户把 ERC-20 / NFT 先 `shield` 进 RAILGUN 合约，链上形成加密 note（可理解为私有 UTXO）；之后用户可以在这个私有余额里做私转、提现（unshield）、甚至调用外部 DeFi 合约；链上只验证零知识证明和状态更新，看不到真实付款方、收款方、金额明文以及私有余额历史。RAILGUN 官方文档把它描述为基于 UTXO、Merkle Tree、nullifier 和 zk-SNARK 的隐私系统，并明确支持通过跨合约调用访问外部 DeFi。

用户把资产从普通 `0x` 地址存入 RAILGUN；

合约把这次存入表示成一个或多个**加密 note / UTXO**；

所有 note 被放进一个**Merkle Tree**；

用户花费时，不公开自己花的是哪几个 note，而是提交一个 zk 证明，证明：

1. 我知道某些树里的 note；
2. 它们属于我；
3. 我没有重复花费；
4. 输入金额和输出金额守恒；
5. 交易规则满足协议约束。
    RAILGUN 官方文档明确说它的交易系统使用 UTXO，类似 Bitcoin / Zcash，并以 Merkle Tree 作为累加器来维护私有状态。

### 1) Commitment / Note

用户 shield 时，合约会根据输入数据计算一个 **commitment（也叫 note）**，它是一个哈希后的承诺值，加入到链上的 Merkle Tree 里。链上能看到“新增了一个叶子”，但不能从叶子反推出它对应的金额、资产、接收者等隐私信息。官方文档把 shield 过程描述为：用户提交公开数据后，RAILGUN 合约计算 commitment，并把它加入由合约维护的 Merkle Tree。



### 2) Merkle Tree

所有 shield 进来的 note、以及后续交易产生的新 note，都会进入同一个私有池对应的树结构。之后花费时，用户证明“我持有某个叶子及其 Merkle 路径”，而不用公开是哪一片叶子。RAILGUN 文档明确说所有系统内 token 共享同一个内部 batch-incremental Merkle Tree。



### 3) Nullifier

为了防止双花，RAILGUN 对每个被花费的 note 生成一个 **nullifier**。
 这个 nullifier 会公开上链，但外部观察者只能知道“有个 note 被花掉了”，**不知道它对应树里的哪一个叶子**。官方文档说明，nullifier 由 Spending Key 和 Merkle 路径索引等信息哈希得到，因此每个 note 的 nullifier 都唯一，而且只有持有 Spending Key 的一方才能把某个 nullifier 与具体 UTXO 对应起来。



### 4) Viewing Key / Spending Key

RAILGUN 的私有地址（0zk address）里编码了公开 Viewing Key 和公开 Spending Key，因此它比普通 0x 地址长。

- **Viewing Key**：用来看见属于自己的加密 note、余额、历史；
- **Spending Key**：用来真正花费这些 note。
   官方文档明确说明二者都编码在 0zk 地址中，且私钥不会暴露给 prover、verifier 或 broadcaster。



## 一次 shield / transfer / unshield 到底发生了什么

## 1. Shield：把公有资产放进私有池

用户从普通钱包发起 `shield()`：

1. 从 0x 地址把 token 转入 RAILGUN 相关合约；
2. 合约计算 note commitment；
3. commitment 进入 Merkle Tree；
4. 合约发出事件；
5. 只有持有 Viewing Key 或 Spending Key 的人，才能识别哪些加密输出是自己的。

RAILGUN 文档明确指出 `shield()` 会把 note 加入合约维护的 Merkle Tree，并发出 `Shield` 事件；相关数据由于哈希和 zk-SNARK 机制，只能被相应密钥持有人识别。

**重要边界**：
 shield 这一步不是“完全隐身”的。因为资产是从公有链地址进来的，所以**入口地址和入金金额在入口交易层仍然可见**。真正的隐私从资金进入私有池后的后续使用开始体现。官方文档也明确提到 shield 过程中发送钱包地址对链是可见的，这是从公账本转入私有系统所必需的。

## 2. Private Transfer：私有地址之间转账

当用户从一个 0zk 地址转给另一个 0zk 地址时：

1. 本地钱包扫描链上加密事件，找出自己可解密的 note；
2. 选取若干可花费的 UTXO 的 note 作为输入；
3. 生成 zk 证明，证明自己有权花费这些输入；（RAILGUN 的交易有效性由 zk 证明来保证：发送方确实持有这些 UTXO，而且它们还没被花过）
4. 公布这些输入对应的 nullifier，防止双花；
5. 本地生成新的输出 note 给收款方，必要时也生成找零 note 给自己；发送方用收款方的公钥信息加密这个 note， **链上只存 commitment（哈希），不存明文**
6. 合约验证证明，通过后更新 Merkle Tree。
7. 收款方扫描链，用自己的 Viewing Key 才能识别这是“给我的钱”

**所以从“密码学动作”上看，新输出 note = 收款方可解密的加密 UTXO + 对应 commitment 上链。**
**从“账本语义”上看，输入 note 被 nullifier 标记为已花费，输出 note 作为新叶子进入 Merkle Tree。外界只能看到“有旧输入被花了、有新叶子被加进树里”，但看不到“这个新叶子就是转给某个人的多少钱”。**

拿自己已有的私有 UTXO 作为输入，证明自己有权花费它们；然后在这笔交易里再创建新的输出 UTXO，其中一个输出归收款方，另一个通常是找零归自己。

RAILGUN 的 UTXO 列表是“用发送方和接收方的密钥加密”的，外部人看不到 UTXO 内容；官方站点也明确说“只有 Bob 能用自己的 private viewing key 解开 ciphertext，拿到 secret information”。所以可以确定：发送方在本地是把收款方的 0zk 地址里对应的公钥材料作为加密目标，生成一个只有收款方能识别/解密的新输出。

官方文档指出 `transact()` 会验证 nullifier 与内部 Merkle Tree 路径及电路输入输出的一致性，并强制要求 nullifier 不能已出现过，否则交易无效。

因此，链上观察者看到的是：

- 某些 nullifier 被消费了；
- 树上新增了一些 commitment；
- 证明校验通过了。

但看不到：

- 谁给谁转；
- 转了多少明文；
- 对应的是哪笔旧 note；
- 发送方和接收方的私有地址。
   这正是 RAILGUN 的核心隐私来源。

## 3. Unshield：从私有池取回公有世界

当用户要把资产从私有池提回某个 0x 地址时，本质上也是一次 `transact()`，只是输出不再是新的私有 note，而是**公开地把资产释放到目标外部地址**。
 这一步会暴露：

- 提现目标地址；
- 提现金额；
- 提现 token。

但依旧不会公开它对应私有池里哪一笔历史 note。这个“出口可见、池内路径不可见”的性质，是 RAILGUN 和大多数 shielded-pool 系统的典型边界。相关交易能力由 Wallet SDK 的 proof generation 和 transact 流程封装。

![image-20260407144445852](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260407144445852.png)

# 集成路线

## 🧱 智能合约层

通常你**不用改 Railgun 合约**，但你可以写：

- wrapper contract（封装逻辑）
- fee / policy 控制

------

## 💻 前端 / 后端

你必须实现：

### ① 钱包逻辑

- 生成密钥对
- 管理 notes

------

### ② zk proof 生成

- 调用 Railgun SDK
- 本地生成 proof

------

### ③ 状态同步

- 同步 Merkle Tree
- 扫描属于用户的 notes



## 0zk 地址

0zk 地址 = Viewing Key（看） + Spending Key（花） 的公开部分打包编码

因此长度比普通 `0x` 地址更长。

#### Viewing Key 的作用

👉 只读权限

- 扫描 Merkle Tree
- 解密属于你的 note
- 查看余额
- 查看历史

#### 带来的能力

#### 🔥 View-only Wallet

你可以：

- 把 Viewing Key 给审计 / 后端 / 监管
- 他们能看到你的资产
- 但永远不能动你的钱



#### Spending Key 的作用

👉 完整控制权

- 花费 note
- 生成 nullifier
- 构造 zk proof





### 生成过程

#### Step 1️⃣：生成 seed

#### Step 2️⃣：派生两个核心私钥

**1）Spending Key（花钱用）**
sk_spend

👉 控制资产的“所有权”

**2）Viewing Key（查看用）**
sk_view

👉 用来扫描链上、识别哪些 note 属于你

⚠️ 这一步非常关键：

**RAILGUN 是“读写分离”的密钥体系**

#### Step 3️⃣：生成对应公钥

```
pk_spend = PublicKey(sk_spend)
pk_view  = PublicKey(sk_view)
```

#### Step 4️⃣：编码成 0zk 地址

```
0zk_address = Encode(
  pk_view,
  pk_spend,
  network
)
```







## Broadcaster 在里面扮演什么角色

RAILGUN 的 0zk 地址本身不是普通 EOA，不能像 MetaMask 那样直接给节点发交易并支付 gas。
 所以它引入 **Broadcaster** 网络。官方文档说明 Broadcaster 会代表用户把私有交易转发到链上，并代付 gas；用户再从私有余额中支付 gas 和小额 premium 给 Broadcaster。

### 1) 隐私更强

如果由用户自己的公开钱包直接广播私有交易，广播层仍可能暴露“这个地址正在发起这笔私有行为”。
 由 Broadcaster 转发后，链上发起者看起来是 Broadcaster，而不是原始用户。官方文档把 Broadcaster 作为实现“full anonymity”的关键组成。

官方提供了 `@railgun-community/waku-broadcaster-client` 方向的组件用于接入 Broadcaster。





## Private Proofs of Innocence（POI）是什么，为什么现在集成时必须考虑它

RAILGUN 近年的一个重要工程现实是：**不是只有“能隐藏”就够了，还要能向交易对手或交易所证明资金不是来自某些已知恶意来源**。
 为此它引入了 **Private Proofs of Innocence（Private POI）**。官方文档把它描述为一个独立于 RAILGUN 合约的、基于 ZK 的“坏交易预防系统”，由 List Providers 提供公开的可疑地址/交易列表，用户在 shield 后会自动生成“我的资金不属于这些名单”的盲化证明。



# 生成 note 的实际过程（工程视角）

发送方在本地做这几件事：

## 1）拿到接收方的 0zk 地址

这个地址里编码了：

- Viewing Public Key
- Spending Public Key

👉 本质上就是接收方的“加密接收能力”

------

## 2）构造输出 note 的“明文结构”

可以抽象成：

```
note = {
  token: USDC,
  amount: 100,
  receiver_pubkey: Bob,
  randomness: r,
}
```

注意：

- `randomness` 非常关键（防止可关联性）
- 同样金额也会生成完全不同的 note

------

## 3）对 note 做两件事

#### （1）加密 → 给接收方看的

```
ciphertext = Encrypt(note, Bob_viewing_key)
```

👉 只有 Bob 能解密

------

#### （2）承诺 → 给链上验证的

```
commitment = Hash(note)
```

👉 放进 Merkle Tree

------

#### 4）一起打包进 zk 交易

发送方生成 zk proof，证明：

- 我拥有输入 note
- 输入金额 ≥ 输出金额
- nullifier 正确（防双花）
- commitment 是合法生成的

然后提交：

```
inputs:
  - nullifiers

outputs:
  - new commitments
  - encrypted ciphertexts
```

#### 5）接收方：

1. 扫描新区块里的所有输出
2. 用自己的 Viewing Key 尝试解密每个 ciphertext
3. 如果能解开：

```
→ 说明这是属于我的 note
→ 加入我的私有余额
```

否则就忽略



## 链上：

- ❌ 看不到谁给谁
- ❌ 看不到金额
- ❌ 看不到 note 内容

只看到：

- 某些 nullifier 被消费
- 新增了一些 commitment（叶子）
- proof 验证通过



## Cookbook 

 **RAILGUN Cookbook = “把公开 DeFi 调用编排成一笔私有交易”的执行层抽象**

它的核心作用是：

> 👉 让用户用私有余额（0zk）去调用你现有的公开智能合约（DEX / Vault / Lending 等）

# 二、Cookbook 解决了什么核心问题

如果没有 Cookbook，你要做：

- 手写 zk 交易结构
- 手写 cross-contract call 编码
- 手动处理：
  - unshield
  - approve
  - swap
  - 再 shield 回去
- 还要保证 token 不丢

👉 非常复杂 ❌

------

Cookbook 帮你做了：

### ✅ 1）把复杂流程抽象成 “Recipe”

### ✅ 2）自动处理：

- unshield → public world
- multi-call → DeFi
- shield → back to private

### ✅ 3）帮你生成：

- zk proof 所需数据结构
- Relay Adapt 调用参数

------

# 三、Cookbook 的核心概念：Recipe

## 👉 什么是 Recipe

👉 **Recipe = 一份“私有交易执行计划”**

你可以理解为：

```
“我要做什么操作”
→ 被翻译成
“RAILGUN 能执行的一整笔隐私交易”
```

------

## 一个典型 Recipe（Swap）

```
用户目标：
  用私有余额 swap 100 USDC → ETH

Recipe：
  1. unshield 100 USDC → Relay Adapt
  2. approve(Uniswap)
  3. swapExactTokensForTokens
  4. shield ETH → 用户私有余额
```

------

# 四、Cookbook 背后的执行引擎

Cookbook 最终生成的是给一个关键合约执行：

👉 **Relay Adapt**

------

## Relay Adapt 的作用

👉 它是“私有世界 ↔ 公共 DeFi”的桥

执行流程是：

### Step 1️⃣：Unshield

```
Private balance → Relay Adapt
```

------

### Step 2️⃣：执行 multi-call

```
[approve, swap, deposit, ...]
```

👉 任意 DeFi 调用都可以组合

------

### Step 3️⃣：Shield 回去

```
Relay Adapt → Private balance
```

------

⚠️ 特点：

- ✅ 全部在一个 transaction 里
- ❌ 任一步失败 → 全部回滚

------

# 五、Cookbook 的输出是什么（关键）

Cookbook 最重要的不是“执行”，而是：

👉 **生成交易数据结构**

核心输出包括：

## 1️⃣ crossContractCallsSerialized

```
multi-call 的编码数据
```

👉 给 Relay Adapt 执行

------

## 2️⃣ relayAdaptUnshieldERC20Amounts

```
需要 unshield 的 token + 数量
```

------

## 3️⃣ relayAdaptShieldERC20Addresses

```
需要重新 shield 的 token 列表
```

⚠️ 非常重要：

> ❗ 不在这个列表里的 token → 永久丢失

------

# 六、Cookbook 的完整执行流程（从用户视角）

## 🔁 Step 0：用户发起

```
用户点击：
  “私有 swap”
```

------

## 🔁 Step 1：构建 Recipe

前端：

```
recipe = buildSwapRecipe({
  tokenIn: USDC,
  tokenOut: ETH,
  amount: 100
})
```

------

## 🔁 Step 2：Cookbook 编译

```
recipe → RAILGUN transaction data
```

------

## 🔁 Step 3：Wallet SDK

负责：

- 选 note（UTXO）
- 生成 zk proof
- 构造 nullifier
- 组装交易

------

## 🔁 Step 4：Broadcaster

```
发送到链上
```

------

## 🔁 Step 5：链上执行

```
验证 proof
→ Relay Adapt 执行 multi-call
→ 更新 Merkle Tree
```