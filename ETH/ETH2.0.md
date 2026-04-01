# ETH2.0

**Execution Layer（执行层）** → 原 ETH1（EVM、交易、合约）

**Consensus Layer（共识层）** → 新增 PoS（Beacon Chain）

随着升级到版本 2，网络被分成了两个相互支持的链。一个处理交易（执行层——EL），而另一个（共识层——CL）管理以有向无环图形式构建主链，通常与区块链相关。为了保持这一结构的运作，其参与者使用**网络客户端**，而网络客户端又由**验证者客户端**和**执行客户端**组成，每个客户端在各自的层级上运行。

**常见客户端：**

- Prysm
- Lighthouse
- Teku

![共识层与执行层交互](https://img.learnblockchain.cn/attachments/migrate/1719324318821)

两个层级通过 RPC 请求交换信息。一个称为 [Engine-API](https://github.com/ethereum/execution-apis/blob/main/src/engine/common.md) 的 API 定义了两个客户端之间传递的指令。

- 图表显示，EL 首先从 CL 获取信标状态，然后将其与 EL 状态同步，使用 EVM 处理来自 TX Mempool 的交易，更新 EL 状态，并通过 RPC 将其发送到 CL。
- 根据 EL 的结果，CL 创建新的信标区块，证明并最终确定它们，并将结果记录在主共识链中。
- 两个层级都形成了一个点对点网络并并行运行。每个层级都有其[客户端](https://learnblockchain.cn/tags/以太坊客户端)（分别为 EL 网络客户端和 CL 网络客户端），每种客户端都有其网络栈。EL 网络客户端通过点对点网络分发**交易区块数据**，而 CL 网络客户端分发**信标区块数据**。这种数据分发仅在通过内置机制验证后才会在网络中开始



链架构

```
共识层（Beacon Chain）
        ↓
执行层（Ethereum 执行状态链）
```

![The Ethereum Merge. By Sumi Maria Abraham, Research &… | by Kerala  Blockchain Academy | KBA Ethereum Stories | Medium](https://miro.medium.com/1%2Ad98MjnYTLjY6130VGgiz7w.png)

### 共识层在增长：

- slot / epoch 递增
- 验证者状态变化
- staking余额变化
- attestation记录

------

### 执行层在增长：

- 交易增加
- 状态树更新（账户余额、合约状态）
- gas费记录



## Slot / Epoch

- **1 slot = 12 秒**
- **1 epoch = 32 个 slots**
- 所以 **1 epoch = 6.4 分钟** 左右。

每个 epoch，RanDAO 机制确保所有活跃的验证者均匀分布在各个 slot 中。

- **Epoch** 是一个条件时间单位，在此期间以太坊协议完成其操作的完整周期。每个 epoch，协议处理来自交易的信息，形成一个新块，对其进行证明，并将其包含在主链中，从而确保 epoch 之间的连续性。
- **Slot** 是将新块添加到信标链的机会窗口。每 12 秒，假设系统性能最佳，一个新块将被添加到链中。Slot 有点类似于区块时间，但如果验证者未能达成一致或交易处理不理想，slot 也可能是空的。
- **RanDAO** 是一种生成伪随机输出的算法，负责在信标链中选择区块提议者（根据其余额值）和区块见证者（根据其在同步委员会中的分布）。所有参与者首先本地选择一个伪随机数，然后每个参与者发送其选择的数字的哈希值。随后，参与者依次揭示其选择的数字，并对揭示的数字执行 XOR 操作，该操作的结果成为协议的输出。

![img](https://img.learnblockchain.cn/attachments/migrate/1719324319751)

如图所示，在正常的网络操作中，每个区块经过三个状态：

1. **区块提议** | 当考虑中的区块的签名在 Epoch 0 期间被输入到信标链中时，分配此状态。在下一个 epoch，验证者必须为其投票。
2. **区块见证（证明）** | 如果在 Epoch 1 开始时，至少 2/3 的验证者为 Epoch 0 中提议的区块投票，则分配此状态。
3. **区块最终确定** | 如果 Epoch 1 中提议的区块已达到证明状态，则在 Epoch 2 结束时，将 Epoch 0 中的区块分配为最终确定状态。



### Casper FFG Finality 推动终局性

- 2 个 epoch 确认 final
- 一旦 final → 不可回滚



### LMD-GHOST 分叉选择

决定“哪条链是主链”



### Slashing（惩罚机制）

- 双签
- 离线

会扣 ETH（甚至踢出）



### 当前参与共识的数据查看途径

### Beacon Chain Explorer

- 你可以通过 [Beaconcha.in](https://beaconcha.in/) 或 eth2beaconchain.ethercluster.com 来查看当前网络的参与验证者信息，包括验证者状态、奖励、惩罚等。



## 共识过程

### 日常循环

### A. epoch 开始或临近开始

- 计算这个 epoch 的 proposer 分配
- 计算各 slot 的 attestation committees
- 更新哪些人有 sync committee 职责。

### B. slot 开始

- 所有节点都有当前本地视图：head / justified / finalized
- 当前 slot 的 proposer 被确定。

### C. proposer 出块

- 跑 LMD-GHOST 选父块/head
- 从执行层获取 execution payload
- 组装 beacon block
- 广播给网络。

### D. 其他节点验证块

- 验签、验 slot、验 proposer、验状态转换
- 调执行层验证 payload
- 有效则导入本地分叉树。

### E. attesters 投票

- 对 source / target / head 签 attestation
- 广播到网络
- 聚合者聚合签名。

### F. fork choice 更新

- 各节点根据最新 attestation 运行 LMD-GHOST
- 更新本地 head。

### G. epoch 边界结算

- process_epoch 统计参与率
- 发奖励、扣惩罚
- 推进 justified / finalized
- 处理激活/退出/slashing 等状态更新。

### H. 循环进入下一个 slot / epoch

- 新 proposer、新 committees、新投票
- 链持续前进





### 进入日常运行：每个 epoch 开始前会先做“排班”

在每个 epoch 内，会做几件事：

- 把所有 active validators 分配到各个 **slot committee**
- 每个 slot 指定 **一个 block proposer**
- 每个 validator 知道自己在哪个 slot 要 attestation
- 一小部分验证者可能被抽进 **sync committee**，为轻客户端签 header。



## 一个 slot 里具体发生什么

下面讲最核心的日常循环。你可以把一个 slot 理解成一次完整的“小共识回合”。

### 第 1 步：所有节点先有一个当前视图

在 slot 开始前，每个共识节点都维护着：

- 当前认为的 **head**
- 当前 **justified checkpoint**
- 当前 **finalized checkpoint**
- 所见到的最新 blocks / attestations。



### 第 2 步：该 slot 的 proposer 被确定

每个 slot 都会从验证者集合中选出 **1 个 proposer**。官方文档说明这是用公开已知的伪随机机制选出来的。

这个 proposer 的任务是：

- 收集自己看到的最新链头
- 从执行层拿交易和执行结果
- 组装一个新的 beacon block
- 广播给全网。



### 第 3 步：proposer 先跑 fork choice，决定“我要在哪个父块上出块”

这里不是随便接到最新块后面，而是要跑 **LMD-GHOST fork choice**。

LMD-GHOST 的直觉是：

- 看每个验证者“最新一次投票”支持的是哪条分叉
- 从已知的 justified 起点往下走
- 每一层都选择“子树总投票权最重”的那条路径
- 最终得到当前 head。

所以 proposer 真正做的是：

> “根据我现在收到的最新 attestation，算出当前最该接着构建的 head，然后在这个 head 上提一个新块。”



### 第 4 步：proposer 向执行层要 payload

共识层 proposer 不直接执行交易，而是通过执行客户端准备 **execution payload**。

实际逻辑可以理解成：

1. 共识层先定出父 beacon block / head
2. 调用执行层，让执行层基于相应 execution head 构造 payload
3. 执行层把：
   - 交易列表
   - 状态 root
   - receipt root
   - gas 使用结果
   - fee recipient 相关内容
      返回给共识层
4. 共识层把这些内容塞进 beacon block

所以现在出的“共识层区块”里，实际上嵌了执行层 payload。



### 第 5 步：proposer 组装并广播 beacon block

这个 block 至少包含几类关键内容：

- Beacon Chain 头部信息
- 对 execution payload 的承载
- 可能包含一些 attestation 聚合
- slashings / exits / deposits / withdrawals 等与验证者状态有关的操作。共识规范把 beacon block 和 epoch processing 定义为链状态转换核心。

广播后，全网其他共识节点开始验证它。



## 其他节点收到 block 后怎么处理

别的共识节点收到 proposer 的 block，不会直接接受，而是做一轮检查。

### 第 1 步：检查共识层基本合法性

例如：

- slot 对不对
- proposer 对不对
- 签名对不对
- 父块是否已知
- 状态转换前后是否符合规范。

### 第 2 步：检查 execution payload

节点会让本地执行客户端验证这个 payload：

- 交易是否可执行
- 状态转换是否正确
- execution block hash / state root 是否匹配

如果执行层说这个 payload 无效，那么这个 beacon block 也不能被接受。官方 nodes/clients 文档还提到 optimistic sync 下，共识层甚至可以先乐观导入 beacon blocks，等执行层追上后再确认 payload 有效性。

### 第 3 步：如果有效，导入本地链视图

导入后，节点会更新自己看到的候选分叉树，并等待 attestation 改变 head。



## 同一个 slot 里，attesters 开始投票

每个 slot 不只是 proposer 出块，**该 slot 被分配到的 attestation committees 还要投票**。官方 attestation 页面说明，attestation 是验证者对自己所见链状态的投票，里面包含 source、target，以及对链头的看法。

attestation 不是一句“我赞成这个块”这么简单，它实际上同时承担三种含义：

1. **Head vote**：我认为当前 head 是哪个块
2. **Source vote**：我认为最近 justified checkpoint 是哪个
3. **Target vote**：我认为当前 epoch 的边界 checkpoint 是哪个。

所以 attestation 同时服务于：

- **LMD-GHOST** 的 head 选择
- **Casper FFG** 的 justified / finalized 推进

这就是为什么 Gasper 是两套机制的组合，而不是单一算法。



## attestation 在网络里如何传播

不是每个验证者各发一条最终都链上全存，那样太重了。

流程更接近：

1. 各 attester 在指定时机签自己的 attestation
2. 广播到 P2P 网络
3. 聚合者（aggregator）把同一 committee 的 attestation 聚合
4. 后续 proposer 把聚合后的 attestation 打包进后续 block。官方奖励说明里明确提到验证者有时还承担 signature aggregation 职责。

这一步的意义是：

- 降低链上数据体积
- 提高全网能及时看到有效投票的概率



## fork choice 如何随着 attestation 动态变化

这是以太坊 PoS 的灵魂。

节点不是“谁先出块听谁的”，而是**不断根据各验证者最新 attestation 更新自己对 head 的判断**。

LMD-GHOST 可以直观理解为：

1. 从当前 justified checkpoint 开始
2. 看它的所有子块
3. 计算每个子树获得的“最新投票权重”
4. 选择最重的子树
5. 递归往下
6. 直到叶子，得到当前 head。

这里的关键是 **Latest Message Driven**：

- 不是一个验证者所有历史票都算
- 只看它**最新那一票**

所以如果网络临时分叉了，随着更多验证者广播 attestation，全网会逐渐收敛到同一个 head。





## epoch 结束附近：Casper FFG 推进 justified / finalized

现在讲第二层：终局性。

Ethereum 不是“每个块一提交就 final”，而是每个 epoch 边界有一个 **checkpoint**。Casper FFG 通过验证者对 source / target 的投票，判断：

- 某个 checkpoint 是否可以被 **justified**
- 更早的某个 justified checkpoint 是否可以进一步被 **finalized**。

你可以这样理解：

- **head**：现在大家多数票认为最新应接着走的链头
- **justified**：已经得到强支持的 checkpoint
- **finalized**：再回滚它需要非常严重的违规并可导致大规模 slashing

### FFG 的直观流程

省掉数学细节，直观上是这样：

1. 验证者在 attestation 里，对某个 `source checkpoint -> target checkpoint` 做投票
2. 当 target 获得足够投票权支持时，它可能成为 **justified**
3. 当一个已 justified 的 checkpoint 被后继 justified checkpoint 正确链接时，前者可能成为 **finalized**。



## epoch 边界还会做哪些“结算工作”

共识规范里的 `process_epoch` 会在 epoch 边界做一批状态更新。不同升级版本细节有变化，但主线包括：

- 统计参与率
- 结算奖励与惩罚
- 更新 justification / finalization
- 处理验证者激活、退出、slashings 后续状态
- 更新委员会和其他周期性状态。

所以 **slot processing** 更像日常流水，**epoch processing** 更像阶段性结算。



#### 正常奖励来自哪里

如果你：

- 准时 attestation
- 你的 source / target / head vote 正确
- 被选中 proposer 且成功提块
- 被选中 sync committee 且完成签名

就能获得相应奖励。官方奖励页面把这些职责按权重计入 base reward。

#### 普通惩罚

如果你：

- 该投票没投
- 出块时缺席
- sync committee 缺席

一般是**拿不到奖励并有轻微罚没**。

#### 严重惩罚：slashing

严重违规比如双签、围绕 Casper 终局规则的可证明恶意行为，会被 slashed。

slashed 之后：

- 会被强制退出
- 会扣减余额
- 在后续一段时间继续处于可追责状态。



## sync committee 在整个流程里做什么

这是 Altair 之后非常重要的一层，但它**不是主共识投票**，而是主要服务轻客户端。

官方资料说明：

- 每个 sync committee 随机选出 **512 个验证者**
- 大约每 **1.1 天**轮换一次
- 它们对最近区块头做签名
- 轻客户端用这些签名更轻量地跟踪链。





## 如果网络出问题，会发生什么

### 1）proposer 没出块

那这个 slot 可能就**空 slot**。官方文档明确提到，如果 proposer 因攻击或离线没能发布块，对其他人看来该 slot 只是空的。

后续 slot 仍继续，不会因为某个 slot 空了而停链。

### 2）网络短暂分叉

不同节点可能暂时看到不同 head。
 但随着更多 attestation 传播，LMD-GHOST 会让 head 收敛；再经过 Casper FFG，checkpoint 获得 justified / finalized。

### 3）大批验证者离线

这种情况下 finality 可能停滞。
 协议会通过惩罚机制逐步压缩离线者影响；近年的官方博客也提到测试网在大规模离线后会触发 inactivity leak 并在之后恢复 finality。





## 区块结构

**Merge 后并不是“只剩 Beacon Block，原来的 ETH1 区块结构被彻底去掉”**，而是变成了：

> **外层是 Beacon Block（共识层区块）**
>  **里面包含一个 Execution Payload（执行层区块内容）**

所以更准确地说：

- **共识层看到的是 Beacon Block**
- **执行层看到的仍然是类似原来 ETH1 的区块内容**

### 1）共识层区块：Beacon Block

它负责：

- slot
- proposer
- attestation / slashings / exits 等共识数据
- 指示当前链的共识推进

### 2）执行层负载：Execution Payload

它负责：

- 交易列表
- state root
- receipts root
- logs bloom
- gas used
- gas limit
- base fee
- block hash
- fee recipient
- withdrawals 等执行结果

**PoW 相关字段基本退出历史舞台，例如：**

- `difficulty`
- `nonce`
- `mixHash`







## 以太坊中的验证者生命周期

在执行层面，大多数操作由存款合约管理，该合约是任何想要激活验证者的人的入口。存款合约接收用户资金，并向共识层发送存款收据和激活新验证者的请求。

在处理验证者的停用请求时，共识层会计算每个验证者的奖励和惩罚，然后将资金转移到指定地址

![img](https://img.learnblockchain.cn/attachments/migrate/1719324321530)

以太坊网络验证者的生命周期分为 7个主要步骤：

1. **存入 32 ETH 到存款合约**
2. 存款信息被共识层识别
3. 进入 **activation queue**
4. 激活后变成 **active validator**
5. 可以被分配 proposer / attester / sync committee 职责
6. 退出时发 **voluntary exit**
7. 经过退出流程后，余额才能继续按规则提现。



**已激活**

- 当激活Epoch到来时，每个Epoch，验证者开始承担区块提议者或区块见证者的角色并履行分配的职责。此激活阶段有三种可能的结束场景：
- **[3.A]** 如果验证者的行为损害了网络状态，其他验证者会对其进行削减，导致其被强制退出并延迟 36 天返还存款。
- **[3.B]** 如果验证者频繁受到网络惩罚且其余额低于最低阈值，则会被强制退出。
- **[3.C]** 如果验证者发起停用，在 2¹¹个Epoch（约 9 天）后，可以继续进行。

**被削减并退出（已退出）**

- 如[3.A]所述，验证者会被延迟 36 天。
- **削减** 是销毁部分验证者的资金并将其从当前Epoch选择的活跃验证者集中移除。为了避免削减，区块提议者必须避免为一个插槽提议两个冲突的新区块版本，区块见证者必须避免签署两个冲突的证明。
- **退出** 只是将验证者从网络中停用并返还所有奖励。如果验证者违反了网络规则，在退出时会被削减。如果验证者行为诚实，则只会退出并返还奖励。

**未削减并退出（已退出）**

- 在[3.B]和[3.C]的情况下，验证者等待约 27 小时，然后获得“可提款”状态。

**可提款**

- 在此阶段，验证者的存款可供提款，周期结束。

![img](https://img.learnblockchain.cn/attachments/migrate/1719324322354)

**验证者集合不会瞬间大变动**，激活和退出都有队列限制，这是为了让 finality 安全性不被快速变化的验证者集合破坏。



## 共识层账户（Beacon Chain）

这是“验证者专用账户”

在 Beacon Chain 上：

每个验证者有：

- 一个 **validator index**
- 一个 **验证者公钥（不是ETH地址）**

👉 奖励（如 attestation）会：

- 直接增加这个验证者的余额（例如 32 → 32.01 ETH）



## 验证者奖励

验证者实际赚的钱来自 4 个部分：

### ✅ (1) 提议区块奖励（Block Proposal）

当你被选中出块：

- 获得：
  - 优先费（priority fees）
  - MEV（最大可提取价值）

👉 这部分**波动很大**（有时非常高）

------

### ✅ (2) 证明奖励（Attestation）

这是最稳定的收入来源：

验证者需要：

- 对区块投票（attest）
- 参与共识

奖励包括：

- 正确投票
- 投票及时（inclusion delay）

------

### ✅ (3) 同步委员会（Sync Committee）

随机选中的验证者：

- 每 256 epoch（约27小时）参与一次

奖励：

- 额外收益（但概率较低）

------

### ❌ (4) 惩罚（Penalty）

如果你：

- 离线
- 投错票
- 作恶（双签）

会被扣钱甚至被 slashing



## 奖励是怎么分发的？

###  共识层 vs 执行层（重点区别）

####  共识层（Beacon Chain）

通过 Beacon Chain：

- 记录：
  - attestation奖励
  - 基础奖励
- 奖励形式：
    直接增加验证者余额（链上记账）

------

####  执行层（Execution Layer）

涉及：

- gas fee（优先费）
- MEV

这些奖励：

- **直接打到验证者设置的地址（fee recipient）**



##  发放节奏（不是实时到账）

- 奖励按 **epoch（约6.4分钟）** 计算
- 每个 epoch 都在更新余额



### 全额退出（Full Exit）

- 退出验证者队列后
- 可取回全部本金 + 奖励



## “共识层余额”转到“执行层钱包”

共识层余额 ≠ 可转账资产

必须通过协议内置的 **withdrawal 机制**

最终变成执行层的一笔普通 ETH 转账



每个验证者在存款时都会设置一个：

👉 withdrawal credentials（提现凭证）

格式大致是：

0x00 → 老版本（BLS地址，不能自动提现）
0x01 → 新版本（指向 ETH 地址）



提现时执行层（EVM）会给对应withdrawal 地址增加余额