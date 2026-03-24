# AAVE V2

## 1 项目背景概述

AAVE v2 是一个基于以太坊区块链构建的开放式借贷平台，允许用户存入各种 ERC-20 代币并从中赚取利息，同时也允许以支付利息的形式借用市场中的代币。通过引入 "利率市场" 的概念，AAVE v2 实现了去中心化的资金池管理和自动化的利率调整机制。此外，AAVE v2 还提供了闪电贷、抵押贷款和代币交换等高级功能，以满足用户的多样化需求，进一步巩固了其在 DeFi 领域的领先地位。



## 2 项目架构分析

![AAVEv2 协议架构图](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/raw/main/images/1.png)

AAVE v2 的整体架构设计围绕用户、资金流动管理、抵押机制、清算流程以及利率策略等关键功能展开，旨在提供高效且安全的去中心化借贷服务。以下是综合分析：



### **用户操作流程**

- **用户**：用户可以进行存款、借款、偿还、提取、借贷利率模式交换、闪电贷以及委托信用等多种操作。用户与协议交互时，会根据其操作自动铸造或销毁相应的 aTokens，代表其在协议中的存款权益，并根据利率策略获得收益。
- **委托信用**：用户可以将自己的信用额度委托给其他用户，扩展了协议的灵活性和使用场景。

![img](https://i-blog.csdnimg.cn/blog_migrate/d8f8719d43b731e72a3fcb28288db62e.png)

### **核心组件**

- **LendingPool**：作为核心模块负责处理所有用户的操作请求，包括存款、借款、偿还、借贷利率模式交换、闪电贷和清算，并更新利率和状态。
- **Collateral Manager**：管理抵押资产，确保用户借款行为安全可控。当抵押资产不足时，会触发清算流程来保护系统的整体流动性。
- **Libraries**：封装储蓄金逻辑，验证逻辑，通用功能逻辑，如清算和借贷操作的计算，为 LendingPool 提供支持。



### **债务与代币化**

### 份额币（Share Tokens）

将资产存入借贷池的用户被激励将其资金长期保留在借贷池中，他们的存款会产生利息。利息随着时间的推移而累积，协议按用户在借贷池中的存款比例来计算，并由其各自的存款用户索取。用户在借贷池中保持其资产的时间越长，他们应计的利息就越多。

当一个用户将资产存入资金池时，他们的 "份额" 会稀释所有用户的份额，协议也会相应地反映这一点。然而，协议并不直接跟踪和更新每个用户在资金池中的份额，在每一次提款或存款的时候。在链上这样做会造成巨大的浪费，而且对储户来说成本过高，储户得为每次更新动作买单。

协议被设计为**只处理储户份额的变化**，而不需要主动更新其他用户的份额。

"份额币 "的设计会自动调整其他 "股东（存款人）" 的股份稀释，以反映 "股份" 的铸造和燃烧，分别与他们的标的资产的存入或提取对应

因为`aToken`代表了资金池的份额，而不是直接的价值



当用户将资产存入池中时，实际要铸造的**aToken**数量是：

<img src="https://img.learnblockchain.cn/2023/04/22/7183.png" alt="img" style="zoom:33%;" />

但使用**balanceOf**查看aToken，返回的是乘liquidityIndex后的数额，也就是用户真实拥有的底层代币数量



aToken的balanceOf函数的定义为：一个用户的aToken的数量为该用户的本金与该本金产生的利息之和。



当用户提取他们的标的资产时，`liquidityIndex` 被用作一个乘数来计算交易中所持有的代币数量。



- **Debt Tokens**：用来跟踪用户的借款负债，与贷出资金数额 1:1。债务代币分别有固定利率和可变利率选项（如 DebtDAI Stable、DebtDAI Variable 等），且债务代币不可转移。

- **aTokens**：用户存入资产时会生成 1:1 的 aTokens 锚定底层资产，这些 aTokens 会不断增值以反映存款所赚取的利息。其中由此引入与本金余额一起存储为一个比率，称为缩放余额 scaled balance（ScB）。
  公式如下：

  [![ScB计算公式](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/raw/main/images/2.png)](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/blob/main/images/2.png)

  举例：

  1. 初始阶段 (流动性累计指数 = 1.0)： 存入 100 DAI → 得到 100 aDAI 简单 1:1 兑换

  2. 一段时间后 (流动性累计指数 = 1.1)： 第一笔存款时的 100 DAI 已经增值到 110 aDAI 

     此时如果新存入 100 DAI 的计算: Scaled Balance = 100/1.1 ≈ 90.91 实际获得 = 90.91 × 1.1 = 100 aDAI

  3. 继续存款 (存入 55 DAI)： 新的 Scaled Balance = 100 + 55/1.1 = 150 实际余额 = 150 × 1.1 = 165 aDAI

  4. 取款操作 (取出 88 DAI)： 最终 Scaled Balance = 150 - 88/1.1 = 70 ,销毁80的aDai 剩余DAI数量 = 70 × 1.1 = 77 
  
     
  
     aDAI 关键公式：aDai余额 = 实际Dai余额 = ScB × 流动性累计指数
  
  
  
  aToken支持ERC-2612: Permit Extension for EIP-20 Signed Approvals

### **价格和利率**

- **Oracles Proxy**：依赖外部预言机（Chainlink）提供资产市场价格数据，用于评估用户抵押资产的价值，确保借贷行为的定价准确性和系统的稳定性。

- **Lending Rate Oracle**：根据系统的状态和市场情况，提供动态的借贷利率，优化资本利用率和流动性。

  

### **配置与管理**

- **Configurator**：用于配置系统参数，如不同资产的风险参数和借贷限额，管理储备金的各种操作，包括激活、借款、抵押、冻结、更新参数及在紧急情况下启用或禁用功能。确保协议可以根据市场变化进行动态调整。



### **其他关键功能**

- **Liquidation Manager**：当用户抵押品价值下降至清算门槛以下时，管理清算操作，保护系统的资金安全。清算人可以通过清算操作获得奖励。
- **Reserves Balances**：存储系统的储备资金数据，用于计算和调整利率策略。

[![Supply Info](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/raw/main/images/3.png)](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/blob/main/images/3.png)



### **利率策略**

**Interest Rate Strategy**：根据市场和用户需求，动态调整利率以实现最佳资本配置，同时考虑流动性风险，确保系统在不同市场条件下的灵活性和稳定性。
尽管存在两种利率模型稳定型和浮动型，但是其模型计算都类似于一个的拐点型模型。在拐点最优利用率下的 slope1 和超过最优利用率的 slope2 分段计算。且在这个条件行也分为固定利率模型和可变利率模型。

![Interest rate model](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/raw/main/images/5.png)

## 浮动利率（Variable Rate，浮动借款利率）

### 定义

浮动利率是**随市场实时变化**的借款利率，主要由该资产在 Aave 资金池中的**供需关系**决定。

### 特点

- 📈 **利率随时波动**
   当借款需求增加、流动性减少时，利率会上升；反之下降。

- 💰 **通常低于稳定利率（在市场平稳时）**

- ⚠️ **不确定性较高**，在市场波动或资金紧张时，利率可能快速飙升

  

### 稳定利率（Stable Rate，稳定借款利率）

### 定义

稳定利率是一种**相对固定的利率**，在你借款时确定后不变，目的是提供更可预测的借款成本。

### 特点

- 🔒 **利率相对稳定**，不随短期市场波动频繁变化
- 🔄 在以下情况下**可能被调整**：
  - 市场整体利率发生剧烈变化
  - 用户的借款规模较大，对协议风险产生影响
- 💸 **初始利率通常高于浮动利率**
- 稳定利率由**marketBorrowRate** 为基准计算，marketBorrowRate 由 Aave 治理写入“**长期借款参考利率**”





### 利率模型

在 **Aave V2** 中，所有利率都围绕一个核心变量：

> **资金利用率（Utilization Rate, U）**

**一句话概括：**
 👉 *借得越多、池子里剩的钱越少，利率就越高*

#### 资金利用率 U

### 公式

<img src="C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260115210111490.png" alt="image-20260115210111490" style="zoom:50%;" />

### 含义

- **U 低**：资金充裕，借款容易 → 利率低
- **U 高**：资金紧张，借款拥挤 → 利率高（抑制继续借款、吸引存款）



**每次发生会影响资金池状态的操作（存/取/借/还/清算/flashloan 等）都会更新利率**





#### 分段线性模型（Kink Model）

设定参数：

- **Base Rate**：基础利率
- **Slope₁**：低利用率区间斜率
- **Slope₂**：高利用率区间斜率
- **Uₒₚₜ**：最优利用率（如 80%）

#### 当 U ≤ Uₒₚₜ：

$$
VariableRate=Base + \frac{U}{U_{opt}} \times Slope_1VariableRate
$$

#### 当 U > Uₒₚₜ：

$$
VariableRate= Base + Slope_1 + \frac{U-U_{opt}}{1-U_{opt}} \times Slope_2VariableRate
$$



**Slope₂ 远大于 Slope₁**
 → 防止资金被“借空”，保障协议安全





**全池平均利率（Average Borrow Rate）**

池子里有两类债务：

- Variable debt
- Stable debt

设：

- 总浮动借款：`V`
- 浮动利率：`R_v`
- 总稳定借款：`S`
- **全池稳定借款的平均利率**：`R_s`（StableDebtToken 里维护的 `avgStableRate`）

那么：
$$
{ \text{AverageBorrowRate} = \frac{ V\cdot R_v + S\cdot R_s }{ V + S } }
$$




**全池平均稳定利率（avgStableRate）**

同样是加权平均更新：
$$
newAvg= = \frac{ oldAvg \cdot oldTotal + newRate \cdot newAmount}{oldTotal + newAmount}
$$


每个用户有自己的 `currentRate`

但资金池需要一个 **O(1)** 就能拿到的全局值，就有了全池平均稳定利率



**用户稳定利率**

当用户再次用 stable 借同一资产（或在 rebalance 时重铸），会调用 `StableDebtToken.mint(...)`，里面会把“旧债 + 新债”的利率做 **加权平均**：

<img src="C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260115213452855.png" alt="image-20260115213452855" style="zoom:50%;" />





## 存款利率（流动性利率）（Liquidity Rate）

存款利率不是独立算的，而是**由借款利率反推**

### 简化公式

$$
DepositRate = U \times \text{AverageBorrowRate} \times (1 - \text{ReserveFactor})
$$



- **ReserveFactor**：协议抽成（如 10%）
- 借得越多 → 存款收益越高



##  “稳定利率”为什么还会变：Rebalance（再平衡）

Aave V2 不是“永远固定不变”的稳定利率；当池子利用率太高、存款收益太低等条件满足时，任何人都可以触发对某个用户的 **rebalanceStableBorrowRate**。

LendingPool 的实现是：

1. `reserve.updateState()`
2. 把用户稳定债务 **burn 掉**
3. 按 **当前 `reserve.currentStableBorrowRate`** 重新 mint 同样的 stableDebt



## 指数 index

#### **流动性累积指数（存款累积指数）（Liquidity Index）**

**作用对象**：
 👉 **所有存款人（aToken 持有者）**

**它表示什么？**

> 从该资产创建至今，**1 单位存款“累计增长了多少”**，也就是从池子首次发生用户操作时，累计到现在，每单位存款本金，变成多少本金（含利息收入）

- 初始值：`1e27`（Ray 精度）
- 只会 **单调递增**
- 用来把 **aToken 数量 → 实际可提取本金**



Aave V2 使用 **线性利息（不是复利）**：
$$
InterestFactor=1+r×Δt
$$
其中：

- r：当前利率（年化，Ray）
- Δt：距离上次更新的时间（秒 / YEAR）

------

### 2️⃣ 更新 Liquidity Index

$$
{ LI_{new} = LI_{old} \times (1 + liquidityRate \times \Delta t) }
$$

其中
$$
Δt=\frac{Tnew-Told}{Tyear}
$$


- `liquidityRate`：存款利率（已经扣过 reserve factor）
- 更新时机：**任何会触发 `reserve.updateState()` 的操作**



## 可变借款累积指数（Variable Borrow Index）

**作用对象**：
 👉 **所有 variable 借款人**

**它表示什么？**

> 从该资产创建至今，**1 单位 variable 借款“累计膨胀了多少”**

- 初始值：`1e27`
- 同样只增不减
- 用来把 **variableDebtToken 数量 → 实际欠款**

**线性利息（不是复利）**：
$$
InterestFactor=1+r×Δt
$$
其中：

- r：当前利率（年化，Ray）
- Δt：距离上次更新的时间（秒 / YEAR）

### 更新 Variable Borrow Index

$$
{ VBI_{new} = VBI_{old} \times (1 + variableBorrowRate \times \Delta t) }
$$



- `variableBorrowRate`：浮动借款利率（全额）
- 不扣 reserve factor







### 国库里的 aToken 是怎么 mint 的？

在 `reserve.updateState()` 时，协议会算：
$$
\text{TreasuryAccrued} = \text{TotalBorrows} \times \text{BorrowRate} \times \Delta t \times \text{ReserveFactor}
$$
然后：

- 给 `treasuryAddress`mint 对应数量的 **aToken**



### 风险参数

Risk Parameters

参考 [Risk Parameters](https://docs.aave.com/risk/asset-risk/risk-parameters#risk-parameters-analysis)

| Name     | Symbol | Collateral | Loan To Value | Liquidation Threshold | Liquidation Bonus | Reserve Factor |
| :------- | :----- | ---------: | ------------: | --------------------: | ----------------: | -------------: |
| DAI      | DAI    |        yes |           75% |                   80% |                5% |            10% |
| Ethereum | ETH    |        yes |         82.5% |                   85% |                5% |            10% |

#### 抵押率 Loan To Value（LTV）

Loan to Value 抵押率，表示价值 1 ETH 的抵押物，能借出价值多少 ETH 的资产

**最大抵押率**
	是用户抵押的各类资产的抵押率的加权平均值
$$
MaxLTV=\frac{ ∑Collaterali in ETH × LTVi} {Total Collateral in ETH}
$$

### 

$$
{ LTV_{user} = \frac{TotalDebt}{\sum (CollateralValue_i \times LTV_i)} }
$$

- 用在：

  - `borrow`
  - `withdraw / balanceDecreaseAllowed`

- 要求：

  LTVuser≤1LTV_{user} \le 1LTVuser≤1



#### 清算阈值  **Liquidation Threshold（LT）**

用于HF的计算：

**
$$
HF = \frac{\sum (CollateralValue_i \times LT_i)}{TotalDebtValue}
$$


- **HF < 1**：允许被清算
- **HF ≥ 1**：不允许清算



## 为什么 LT 一定要大于 LTV？

几乎所有资产配置都是：

LT>LTVLT > LTVLT>LTV

### 原因只有一个：**给用户留缓冲区**

#### 如果 LTV = LT 会发生什么？

- 你刚借满 → HF = 1
- 价格轻微波动
- **立刻触发清算**

👉 这是灾难性的用户体验 + 系统风险

### 中间这段空间是什么？

$$
安全缓冲 = LT - LTV
$$

它的作用是：

- 用户：
  - 不能再借（触达 LTV）
  - 但还不会被清算
- 系统：
  - 有时间让用户补仓 / 还款
  - 避免清算风暴



## 流程梳理

### **存款**

1. 用户通过调用LendingPool合约的deposit函数进行存款，该函数接受四个参数：资产地址、存款金额、接收方地址及推荐码。
2. 首先验证合约未处于启用状态，然后通过 ValidationLogic.validateDeposit 验证存款金额必须大于 0，同时确认确认储备处于激活状态且未被冻结。
3. 接着系统会更新储备状态，调用 reserve.updateState() 更新流动性指数和可变借款指数， 并计算时间段内产生的利息，其中一部分利息会被铸造并转入协议国库。
4. 随后通过 reserve.updateInterestRates 根据最新的供需关系动态调整流动性利率、稳定借款利率和可变借款利率(都由 DefaultReserveInterestRateStrategy.calculateInterestRates 函数计算更新)。
5. 资产转移环节，系统将用户的基础资产转入 aToken 合约，同时铸造等额 aToken 给用户所提交的 onBehalfOf 地址。其中，aToken 采用缩放机制 (scaled balance) 处理利息累积。如果是用户首次存款，系统会自动将该资产标记为用户的抵押品。



### **提现**

1. 用户通过调用 withdraw 函数进行提现操作。首先取指定资产的储备数据，包括对应的 aToken 地址，检查此用户在 aToken 中的余额。
2. 接下来，调用 ValidationLogic.validateWithdraw 函数来验证提现请求，包括检查提现金额是否有效、用户余额是否足够、储备是否处于活动状态等。其中通过 GenericLogic.balanceDecreaseAllowed 中对用户的健康系数以及提现是否影响抵押品进行检查。在 balanceDecreaseAllowed 函数中 calculateUserAccountData 和 calculateHealthFactorFromBalances 函数计算出取出资金后的清算阀值并检查用户总抵押，总借贷数额以及用户当前的健康系数，以此来判断是否用户健康系数处于流动性阀值的安全状态。
3. 随后更新储备的状态，并更新利率，将提现金额传递给函数。若用户请求的提现金额等于其当前余额，则更新用户配置，将该储备标记为不再作为抵押使用。最后销毁用户的 aToken，并将提现的资产转账到指定的地址。

![health factor](https://github.com/slowmist/AAVE-V2-Security-Audit-Checklist/raw/main/images/12.png)


先回忆“用户加权清算阈值”的定义：
$$
LTuser =  \frac{\sum (C_i \cdot LT_i)}{\sum C_i}
$$
其中 Ci 是每种抵押品价值（ETH计价）。



因为：
$$
totalCollateralInETH⋅avgLT=  \sum(C_i) \cdot \frac{\sum(C_i \cdot LT_i)}{\sum(C_i)}
$$
所以它等价于“**取款前的分子：加权抵押价值之和**”。



你取走的是某一种抵押品，它在分子里贡献的是：
$$
Δnumerator = amountToDecreaseInETH \cdot LT_{asset}
$$
所以要从原来的分子里减掉它。



取款后新的分母就是剩余抵押总价值：
$$
\sum C_i - amountToDecrease
$$
最终得到：
$$
LTafter = \frac{\sum(C_i\cdot LT_i) - amountToDecrease \cdot LT_{asset}} {\sum C_i - amountToDecrease}
$$
✅ 这就是**取款后的用户加权清算阈值**。

取款后健康因子怎么计算

`calculateHealthFactorFromBalances` 的经典形式是：
$$
HF = \frac{Collateral \cdot LT}{Debt}
$$
因此这里得到：
$$
HFafter = \frac{ collateralAfter \cdot LT_{after} }{ totalDebt }
$$
最终校验

```solidity
return healthFactorAfterDecrease >= GenericLogic.HEALTH_FACTOR_LIQUIDATION_THRESHOLD;
```

在 Aave V2 里 `HEALTH_FACTOR_LIQUIDATION_THRESHOLD` 通常就是 **1（按 Wad 精度表示）**，语义是：

- **HF ≥ 1**：不可清算（安全）
- **HF < 1**：可清算（危险）

所以这句的业务含义就是你写的注释：

> **要求取款后的健康因子仍然大于等于清算阈值（=1）**，否则不允许取款/转出抵押品。



### **借贷**

1. 用户借款通过 borrow 函数进行借贷，执行借款会先从价格预言机获取资产的当前价格，将借款金额转换为 ETH 等价。
2. 随后通过 ValidationLogic.validateBorrow 检查以及 GenericLogic.calculateUserAccountData 用户借款是否合法，计算包括 onBehalfOf 地址的总抵押资产、总债务、当前贷款价值比率（LTV）、清算阈值和健康因子以及市场的稳定性等，是否有足够的抵押资产借贷。
3. reserve.updateState 更新储备状态，如利率和借款指数（这一步类似于 compound 中的 accrueInterest），用于计算并更新利息。
4. 随后根据用户选择进行的 interestRateMode 稳定利率或浮动利率）生成债务。选择不同的利率模型的代币合约来铸造代币。同时，铸造代币时会检查如果 onBehalfOf 地址不是调用者，则会在 代币合约中减去其对调用用户的借贷授权。如果是用户的首次借款，会将其配置为活跃借款者。
5. DebtToken 铸造给用户后，协议会通过 updateInterestRates 更新借款利率，反映借款后的新利率和储备池的变化。如果用户请求释放借款的底层资产，协议会将资产直接转移给用户。

在 **Aave V2** 中：

> **用户 LTV（Loan To Value）=「你当前借款价值」÷「你可用于借贷的抵押价值上限」**

### 单资产 LTV（参数）

- 每个抵押资产在治理中配置一个 `loanToValue`
- 例如：
  - ETH：80%
  - USDC：85%

👉 这是**资产层的风险参数**

### 用户 LTV（运行时计算）

当你有**多个抵押资产**时，协议计算的是：
$$
LTV_{user} = \frac{\text{TotalDebtInETH}} {\sum(\text{CollateralValue}_i \times LTV_i)} 
$$
👉 这是**账户层的实时状态**

####  计算用户“可借额度上限”

$$
BorrowingPower = \sum(\text{CollateralValue}_i \times LTV_i)
$$

**校验借款是否合法（LTV 约束）**
$$
CurrentDebt+BorrowAmount≤BorrowingPower
$$
也就是：

> **借完这笔之后，你的用户 LTV 不能超过 100%**



### **还款**

用户通过 repay 函数进行还款，首先获取用户的当前债务（包括稳定债务 stableDebt 和浮动债务 variableDebt）。

根据用户选择的利率模式（稳定或浮动），由 ValidationLogic.validateRepay 验证用户的还款操作合法性。包括用户的债务余额是否足够进行还款，根据选的利率模式是否有对应债务；要求还款amount > 0，如果 amount 是 `type(uint256).max` → 视为 **“全部还清（repay all）”** validateRepay **不关心 LTV / HF**，因为：还款只会 **降低风险**不可能因为还款触发清算

根据用户选择的利率模式来确定还款的具体债务类型（稳定利率或浮动利率）。协议会计算： paybackAmount = min(amount, userDebt)， 如果用户要还的金额小于当前债务余额，系统会使用用户提供的还款金额进行部分还款；否则，将偿还所有债务。更新储备的状态 updateState，用于计算并更新协议中的利息、借贷量以及借贷指数。

随后燃烧相应的稳定债务代币，并通过 updateInterestRates 更新借款利率。此时，如果用户的所有债务（包括稳定和浮动债务）在还款后为零，则会将该用户的借款状态标记为 false，表示用户不再借款（这一步会减少后续 accountData 计算开销）。

最后转移底层资产（资金交割），用户将还款金额从其账户转移到协议的 aToken 合约地址（作为池子的可用流动性）。



### 清算

用户通过 lendingpool 的 liquidationCall 函数进行清算，函数通过代理模式调用 LendingPoolCollateralManager 的 liquidationCall 函数，确保函数的成功执行。首先 GenericLogic.calculateUserAccountData 获取抵押品资产及债务资产的储备数据和用户的配置信息，计算用户的健康因子，并通过 getUserCurrentDebt 获取用户的当前稳定和可变负债。 ValidationLogic.validateLiquidationCall 函数验证清算调用的合法性，包括检查用户的健康因子、债务状态和抵押品配置。

若健康因子小于阀值，已作为抵押品，且两种债务都不为 0 则验证通过。接着计算用户的最大可清算债务，并确定实际需要清算的债务数量。如果清算的债务超过用户的可用抵押物，将调整清算金额。

如果清算人选择接收被清算人抵押的底层资产，需要确保抵押物储备中有足够的流动性。更新债务储备的状态，并根据清算人是否接收 aToken 情况，燃烧相应数量的可变和稳定债务代币。

更新债务的利率，反映清算后的市场情况。清算人奖励如果选择接收 aToken，清算人将获得相应数量的 aToken。如果不接受 aToken，则更新其抵押状态和抵押物的利率，从用户账户中燃烧掉对应数量的 aToken ，将底层资产转移给清算人。最后，将清算所需的债务资产从清算人转移到相应的储备 aToken 中，完成清算过程。



**健康因子**
$$
HF = \frac{\sum (CollateralValue_i \times LT_i)}{TotalDebtValue}
$$


- **HF < 1**：允许被清算
- **HF ≥ 1**：不允许清算



**最大可清算债务（close factor）**

Aave V2 会限制一次最多能清算多少债务（避免一笔把用户全清光、也减少市场冲击）。这就是 **Close Factor**（在 V2 中通常是 50%，取决于 HF 是否低到某个阈值以下，具体实现版本略有差异）。
$$
DebtToCover = \min(debtToCover, \; userDebt \times closeFactor)
$$


## 判断抵押品可拿多少或者债务最多能还多少

### 函数目的

`_calculateAvailableCollateralToLiquidate` 做的是**把“我愿意替他还 debtToCover 的债”**转换成：

- 清算人理论上能拿走的抵押品数量 `maxAmountCollateralToLiquidate`
- 但如果抵押品不够，就改成“把他全部抵押拿走时，最多需要还多少债”`debtAmountNeeded`

------

**先算“理论上能拿走的抵押品数量”（含清算奖励）**
$$
{ CollateralAmount = \frac{DebtValue}{CollateralPrice} \times (1 + bonus) }
$$
其中：

- `DebtValue = debtToCover × debtAssetPrice`
- `CollateralPrice` 是抵押品价格
- `liquidationBonus` 是奖励（Aave 用 percentMul/Div 表示，典型如 10500=+5%）
- `10^decimals` 是为了把不同 token 精度对齐（比如 USDC 6 位，WETH 18 位）

**含义**：清算人替你还债，能用折扣拿走更多抵押品，这个“多出来的部分”就是 bonus 奖励。



#### 情况 ：理论可拿抵押 > 用户抵押余额

> 如果清算的债务超过用户的可用抵押物，将调整清算金额

所以只能：

- 把用户 **全部抵押品** 给清算人：`collateralAmount = userCollateralBalance`
- 反推：拿走这些抵押品对应最多“需要覆盖的债务”是多少：

直观公式：
$$
DebtNeeded = \frac{CollateralValue}{(1 + bonus)}
$$
**含义**：如果抵押不足，清算人不能按原计划还那么多债；他最多只能还到“把所有抵押拿走”刚好对应的债务量（而且要除掉 bonus，因为清算人拿走抵押是带折扣的）。



### **闪电贷**

用户通过 lendingpool 的 flashLoan 函数进行闪电贷。作为借贷协议的闪电贷，可以允许当前闪电贷立刻还款或是作为债务来后续还款，其中以传入的 modes 参数不同而决定。0 为立刻还款，1 为作为稳定型债务，2 为浮动型债务。 

函数首先通过 ValidationLogic.validateFlashloan 检查输入参数匹配，计算闪电贷所需的溢价成本，并直接将所需 aToken 转给接收者地址。调用接受者的 executeOperation 操作实现预设的闪电贷。AAVE 实现的闪电贷操作已包括了兑换，兑换清算，以及兑换偿还操作。再 executeOperation 以上操作完成后，记录需偿还闪电贷金额和相应的费用。如果用户选择以非债务模式归还资金：系统更新储备状态，累积储备流动性以及更新流动性指数。最后再从请求者转移资金和费用至储备池。 若用户选择以债务模式处理，则调用 _executeBorrow，开启相应的债务头寸。



### **转换债务模式**

在 AAVE v2 中，用户可以通过 swapBorrowRateMode 函数在稳定利率模式和浮动利率模式之间切换。首先通过 getUserCurrentDebt 函数获取用户在目标资产上的当前稳定利率债务和浮动利率债务，确定用户的债务状况。接着调用 ValidationLogic.validateSwapRateMode 函数验证切换操作是否合法。其中检查用户是否有足够的稳定或浮动债务以支持模式切换，确保切换目标模式符合资产的配置和用户的债务情况。调用 reserve.updateState 更新资产储备的状态，确保储备数据最新。随后就是对于两种债务代币的相互转换，燃烧稳定债务代币铸造浮动债务代币或是燃烧浮动债务代币铸造稳定债务代币。转换完成后 reserve.updateInterestRates 更新目标资产利率，确保反映当前市场状态和用户债务的变化。



**信贷委托 (Credit Delegation)：**

允许 AAVE 的存款人将其信用额度委托给其他人，而接收者无需提供抵押品即可借款。这种机制将 DeFi 的“超额抵押借贷”推向了“信用借贷”。通常用于机构之间或 DAO 之间的信任协作。







# 使用 Aave 协议创建多头杠杆和卖空等金融策略

**多头杠杆（Long）**：

> 用已有资产做抵押 → 借稳定币 → 买更多目标资产 → 再抵押 → 放大敞口
>
> <img src="C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260119152006218.png" alt="image-20260119152006218" style="zoom:33%;" />

**卖空（Short）**：

> 抵押稳定币 → 借目标资产 → 卖出换稳定币 → 等价格下跌 → 买回还债





## 循环杠杆的杠杆率如何计算（核心公式）

### 1️⃣ 定义变量（统一口径）

| 符号       | 含义                        |
| ---------- | --------------------------- |
| `C₀`       | 初始自有资产价值（USD）     |
| `LTV`      | 抵押借款比（如 75% → 0.75） |
| `n`        | 循环次数                    |
| `Exposure` | 最终资产总敞口              |
| `Leverage` | 杠杆倍数                    |

------

### 2️⃣ 第一次借款（不循环）

```
Exposure₁ = C₀
Leverage₁ = 1×
```

------

### 3️⃣ 循环 n 次后的总敞口

每一轮你都会：

```
新增资产 = 上一轮抵押 × LTV
```

这是一个**几何级数**：

```
Exposureₙ = C₀ × (1 + LTV + LTV² + … + LTVⁿ)
```

------

### 4️⃣ 理论最大杠杆（n → ∞）

当你循环到极限时：

```
Exposure_max = C₀ / (1 − LTV)
```

👉 **杠杆倍数公式（最重要）**：

```
Leverage_max = 1 / (1 − LTV)
```

------

### 5️⃣ 直观示例

| LTV          | 理论最大杠杆 |
| ------------ | ------------ |
| 70%          | 3.33×        |
| 75%          | 4.00×        |
| 80%          | 5.00×        |
| 90%（eMode） | 10.0×        |



## 清算是如何计算的

### 1️⃣ 清算不看“几倍杠杆”，只看 **健康因子 HF**

**清算条件：**

```
HF < 1
```

------

### 2️⃣ HF 的定义（标准版）

```
HF = (抵押品价值 × 清算阈值 LT) / 债务价值
```

其中：

- `LT`（Liquidation Threshold）通常 > LTV
   （如 LTV 75%，LT 80%）

------

## 四、把 HF 套进循环杠杆（关键推导）

### 假设（常见 ETH Loop）

- 抵押资产 = ETH
- 借出资产 = 稳定币
- 价格变量 = ETH 价格 `P`

------

### 1️⃣ 债务规模（稳定）

在极限附近：

```
Debt ≈ Exposure_max× LTV = C₀ × LTV / (1 − LTV)
```

------

### 2️⃣ 抵押品价值随价格变化

```
Collateral_value = Exposure × (P / P₀)
```

------

### 3️⃣ 清算触发条件（HF = 1）

```
(Exposure × (P_liq / P₀) × LT) / Debt = 1
```

整理后得到：

```
P_liq / P₀ = Debt / (Exposure × LT)
```

------

### 4️⃣ 用 LTV 化简（非常重要）

代入 Exposure 与 Debt 后：

```
P_liq / P₀ = LTV / LT
```

👉 **关键结论**：

> **循环杠杆的清算价比例
>  只取决于 LTV 和 LT，
>  与你循环了几次、杠杆几倍无关。**

------

## 五、一个完整数值例子（ETH 多头 Loop）

### 参数

- 初始资金 `C₀ = $10,000`
- LTV = 75%
- LT = 80%
- 当前 ETH 价格 = $2,000

------

### 1️⃣ 理论最大杠杆

```
Leverage_max = 1 / (1 − 0.75) = 4×
```

------

### 2️⃣ 清算价格

```
P_liq / P₀ = 0.75 / 0.80 = 93.75%
P_liq ≈ $1,875
```

📌 含义：

- **ETH 跌 ~6.25% 就会被清算**
- 杠杆再高，清算线也不会更远

------

## 六、为什么这点非常反直觉（但极其重要）

很多人以为：

> “我只做 2× 杠杆，会比 4× 安全很多”

❌ **在 Aave Loop 里，这是错的**

真正决定安全的是：

- **你借到多接近 LTV 上限**
- **HF 留了多大缓冲**

------

## 七、实操中的“安全杠杆”怎么选？

### 经验法则（非常实用）

- **不要用满 LTV**
- 目标 HF：

| 风格 | 建议 HF |
| ---- | ------- |
| 激进 | ≥ 1.15  |
| 平衡 | ≥ 1.30  |
| 保守 | ≥ 1.50  |
