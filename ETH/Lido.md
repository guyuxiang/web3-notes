# Lido

## 以太坊原生质押的缺点

1. **流动性不足:** 在质押期间，用户无法移动、交易或将他们的 ETH 用作 DeFi 中的抵押品。这在他们能够从信标链中提取资金之前尤其昂贵。
2. **高参与费用:** 用户只能质押 32 ETH 的倍数，这排除了余额较小或不规则的用户。
3. **运营成本:** 尽管以太坊的核心开发人员认识到质押的低硬件和正常运行时间要求，许多用户仍然更愿意提供资本并将运营工作外包给第三方，这对去中心化产生了负面影响。

# 

以太坊的原生质押与信标链的启动同时出现，这标志着从工作量证明（PoW）到权益证明（PoS）的逐步过渡的开始。最初，质押是一种单向操作，涉及与存款合约的交互。质押后，在激活 The Merge 更新之前，无法从存款合约中提取资金，该更新引入了验证者自愿和强制退出的逻辑。这一限制突显了需要一个新方向来解决无法取消质押和使用资金的问题，从而催生了流动质押。

Lido的主要优势之一是接受任何规模的存款，返还 stETH，显著降低了 32 ETH 的入门门槛。

大型验证者从质押服务中获得资金。收入每日分配给 stETH 持有者。这会导致用户余额的变化，考虑到奖励和惩罚。

Lido DAO Token (LDO) 是用于管理 Lido 的 ERC-20 标准代币。

![Lido 架构](https://img.learnblockchain.cn/attachments/migrate/1719323916548)

Lido 协议架构组件的概括图

### 存款

![Lido存款](https://img.learnblockchain.cn/pics/20240625222825.png)

**池化质押**：

- 用户的 ETH 先进入 Lido 协议
- Lido 给你铸造 `stETH`
- 协议汇总资金，经 `StakingRouter` 分配给 staking modules ，
- 不同 staking modules 模块内部，再按模块自己的规则分给 Node Operators，Curated 模块偏 registry/顺序分配，CSM 偏 FIFO 队列分配
- 当轮到某个 NO 时，协议会从它“可用 key 集合”里取出**第一个未被使用的 signing key 和对应签名**，然后向官方 `DepositContract` 发起存款
- 节点运营者提供的是**预先生成好的验证者密钥材料和持续运行验证者服务**
- 质押收益被社交化到 `stETH` 持有人，扣除协议定义的奖励费率。



#### 节点运营者预操作

1. Node Operator 先离线生成 validator keys，私钥只由 Node Operator 自己生成、保存、管理， 之后它把**公钥和 deposit signature**提交到协议，`NodeOperatorsRegistry`；注册表记录其公钥等信息，并决定分配顺序。

**validator pubkey：**用来确定这笔存款要绑定到哪个验证者公钥

**deposit signature**：**这笔存款确实是对应这个验证者私钥持有者授权的**

`deposit signature` 签名内容：`(deposit_message, 链的 domain)` 的 root 做 BLS 签名。

这个 `deposit_message` 的核心内容可以理解为：

- `pubkey`
- `withdrawal_credentials`
- `amount`

后续**DepositContract用 pubkey 去验 signature**

如果验签成功，就说明：这份存款消息，确实是对应这个 pubkey 的私钥持有者签出来的。



2. 当 `StakingRouter` 判断现在该给某模块、某 NO 分配新的 32 ETH 时，它会从对应模块合约中取出尚未使用的 deposit data。

3. `StakingRouter` 使用刚取到的 **pubkey + signature + withdrawal credentials**，连同协议池中的 32 ETH，去调用以太坊官方 `DepositContract`。

`DepositContract.deposit(...)` 接收四个关键参数：

- `pubkey`
- `withdrawal_credentials`
- `signature`
- `deposit_data_root`

`pubkey` = 这个账户是谁

`withdrawal_credentials` = 以后钱该退到哪里

其中前 3 个字段组成一份 `DepositData`，而 `deposit_data_root` 是这份数据的哈希根



### 收益计算

**“总池记账 + 全局比例调整（rebase）”**

Lido 的底层收益，来自它参与以太坊验证者节点的奖励，Lido统计整个验证者池的总资产变化 → 按比例同步到所有 stETH 持有人



Lido 的 `AccountingOracle` 会汇总 Lido 参与验证者的状态与余额，以及协议金库中的资金

```
净收益 = 验证者余额增长
      + 执行层奖励
      - slash / 惩罚 / 离线损失
      - 协议层需要处理的其他扣减项
```

也就是说，Lido 的日收益更新逻辑大致是：

1. Oracle 收集 Lido 相关验证者总余额、退出数量、执行层奖励金库等数据
2. 报告提交到协议
3. 再把这部分净收益映射到 stETH / shares 体系里。



#### Oracle Committee（预言机委员会）

Lido 有一组固定成员（通常 9 个左右）：

每个成员运行：

- 共识层客户端（Beacon）
- 执行层客户端

1. 从 **validator pubkey 列表**中获取验证者，从 Beacon Chain 读取状态，调用共识层 API（本地节点）读取每个 validator balance（余额）、status（active / exited / slashed）

2. 从执行层读取priority fee、MEV 收益，因为出块时，收益进入 区块设置的 fee recipient 地址，对于Lido的节点运营商， 这个地址不是节点运营商自己，而是Lido 控制的地址，Oracle 节点会查询这些地址余额



**Lido stETH 余额计算公式是：**

```
balanceOf(account) = shares[account] * totalPooledEther / totalShares
```

也就是：

- `shares[account]`：你持有的内部份额
- `totalPooledEther`：协议池里对应的总 ETH
- `totalShares`：全体份额总数。

官方帮助文档说明，`stETH` 会按日 rebase，用户看到的是自己地址里的 `stETH` 余额增加；通常这个更新在每天中午 UTC 左右发生。

Lido 当前文档写明，协议对 **staking rewards** 收取 **10% 费用**，其中一半给节点运营者，一半给协议 treasury；而且这是对**奖励部分**收，不是对本金收。

Lido 采用 **rewards socialization model**，用户通常在存款后 24 小时内就能开始体现质押收益，而不需要等自己那一笔 ETH 对应的验证者单独激活。所以它更像一个**共享池份额制**，而不是你自己单独开了一本存款账户。



#### wstETH 为什么余额不变还能有收益

因为 `wstETH` 是 **non-rebasing** 版本。

官方文档说明：

- `stETH`：余额每天变
- `wstETH`：余额固定
- 收益通过 **wstETH 对 stETH 的兑换率上升** 来体现。



### 提取资金

![Lido 资金提取](https://img.learnblockchain.cn/pics/20240625222831.png)

#### 第 1 步：提交 withdrawal request

选择要提取的 `stETH` 或 `wstETH` 数量，用户请求后，协议会把你的 `stETH/wstETH` 锁进 **WithdrawalQueue** 合约，并为这次请求铸造一个 **unstETH NFT**。这个 NFT 代表你的排队位置和未来可领取的 ETH 请求。这个 NFT 就是你的“提现票据”

#### 第 2 步：Withdrawal Queue 排队处理

Lido 的提现队列是 **FIFO（先进先出）** 的。官方 withdrawals 说明和 WithdrawalQueue 合约文档都明确写了这一点。协议会按请求创建顺序逐个处理。

协议内部会优先使用 **Lido Buffer** 中已有的 ETH 来满足提现；如果 Buffer 不够，才会进一步安排验证者退出，以筹集足够的 ETH 来完成后续请求。

#### 第 3 步：请求被 finalize

当协议拿到了足够的 ETH，并且能够确定你的请求的 `stETH:ETH` 兑换结果时，你的这笔 withdrawal request 会被**finalize**。对应的 ETH 被预留出来，原先锁住的 `stETH` 被烧毁

#### 第 4 步：用户 claim ETH

当请求 ready 后，你到 `Claim` 页面发起第二笔交易。你烧掉 `unstETH NFT`，合约把对应 ETH 转给你



## 两个“退出路径”

### 路径 A：走 Lido 原生提现

特点是：

- 走协议自己的 Withdrawal Queue
- 需要排队
- 最终拿回的是原生 ETH
- 更接近“正式 unstake”。

### 路径 B：走 DeFi 二级市场交换

Lido 的 withdrawal 帮助中心也单独列了 “Fast Swaps Using DeFi Aggregators”。这意味着很多用户也会直接在市场上把 `stETH/wstETH` 兑换成 ETH，而不是走原生提现队列。这样可能更快，但会受到市场价格、滑点、流动性影响。