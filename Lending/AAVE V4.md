# AAVE V4

主要升级内容为「统一流动性层」，本质上来说，该功能系 Aave V3 版本 Portal 概念的扩展

尽管 Portal 可以让 Aave V3 变成一个无视链间流动性隔阂的 DeFi 协议，但其运作却需要依赖于一定的信任假设。简单来说，用户需要向一些被列入白名单的桥接协议（比如 Connext）提交桥接交易进行aToken的跨链，而非 Aave V3 核心协议。再次强调，终端用户当前无法仅通过 Aave 本身的核心协议来使用 Portal。



这就引出了「统一流动性层」的概念，这也是 Aave V3 到 V4 的最重要的架构变化。如下图所示，「统一流动性层」将采用模块化设计，统一管理「供应 / 借贷上限」、「利率」、「资产」和「激励」，并允许各个模块从中提取流动性。

在「统一流动性层」的框架之下，Aave 将借助 Chainlink 的跨链互操作性协议（CCIP）来构建「跨链流动性层」（CCLL），允许借款人在 Aave 支持的所有网络间即时访问所有的流动性。这一改进有望将 Portal 发展成为一个完全版的跨链流动性协议



# Horizon RWA Market

Horizon 是 Aave Labs 发起的一项计划，旨在为机构和合格的参与者打造一个强大的真实世界资产 (RWA) 交易市场

Horizon 市场的核心功能是允许用户提供经许可的**代币化 RWA 作为抵押品**，**同时借入无需许可的USDC、RLUSD和GHO等稳定币**。

只有提供许可型RWA抵押品的用户才能从无需许可的稳定币池中借款。这种模式巧妙地使机构实体能够利用其受监管的资产，在合规的框架内获取DeFi生态系统的流动性。

用户提供RWA资产后，会通过程序自动收到代表抵押头寸的aToken。**该市场的一个关键特征是这些收据代币不可转让，从而确保抵押头寸始终由原始提供者持有**，进而增强安全性和合规性。这种不可转让的特性防止了抵押头寸在二级市场的交易，**维护了权限系统的完整性。**

抵押品价值通过既定的定价机制确定，该机制考虑了每种风险加权资产类型的具体特征。风险参数，包括贷款价值比和清算阈值，根据基础资产的风险状况和监管要求进行配置。借款人必须维持充足的抵押率，该抵押率由其提供的风险加权资产（RWA）抵押品的风险参数决定。系统持续监控这些比率，并在抵押不足时触发清算程序，但清算机制会根据RWA抵押品的独特特征进行调整。

任何参与者都可以完全无需许可地向市场提供这些稳定币例如USDC、RLUSD和GHO以赚取收益。稳定币本身不能用作抵押品，从而清晰地区分了市场的许可抵押品部分和无需许可的流动性部分。

借贷流程遵循标准的Aave协议机制，利率由使用率曲线和市场动态决定。

![img](https://aave.com/docs/_next/static/media/horizon-permissions.2085ed83.png)



## 系统架构

![img](https://zyoncode.com/content/images/2024/06/image-27.png)

- Core Layer: 协议核心逻辑层,包含资金池、配置、数据提供、公共库等模块
- Periphery Layer: 协议外围功能层,包含预言机、奖励控制、手续费管理、钱包余额提供等模块
- Deployment Layer: 协议部署相关模块,帮助实现协议前端交互和部署流程



### 模块介绍与关键流程分析

#### Pool（执行引擎）

**Pool 是“无状态的金融执行器”**

- `supply`: 存款。用户调用传入资产类型和数额,合约记录存款、增发 aToken、更新储备金等
- `withdraw`: 取款。合约销毁对应 aToken、减少储备金、转移资产到用户账户
- `borrow`: 借款。合约检查抵押率,为用户增发债务 token,并转移借出资产到用户账户
- `repay`: 还款。合约记录还款数额,销毁对应债务 token,更新储备金
- `liquidationCall`: 清算。触发条件为抵押率低于清算阈值,合约出售抵押资产偿还债务

- ❌ **不能修改风险参数**
- ❌ **不持有治理权**

![img](https://zyoncode.com/content/images/2024/06/image-28.png)

**supply**

- 更新资产状态，包括liquidityIndex、variableBorrowIndex、lastUpdateTimestamp，累计资产总量
- 检查资产储备中资产量是否达到上限
- 更新资产相关利率，包含LiquidityRate（流动性利率）、StableBorrowRate（固定借款利率）、VariableBorrowRate（动态借款利率）
- 转入用户的资产token到对应的aToken地址
- 铸造aToken到用户地址

**borrow**

- 更新资产状态
- 检查是否满足借贷条件
  - 资产是否Active，是否Paused，是否Frozen，是否borrowingEnabled
  - 预言机是否设置了允许借贷
  - 计息模式检查
  - 借款总额是否超过借款上限，借款总额=固定利率总借款+动态利率总借款+当前请求借款
  - 资产隔离（isolation）模式：检查资产是否是可借款状态（borrowableInIsolation），隔离模式总债务是否小于隔离模式总债务上限
  - 用户EModeCategory不为空：检查资产的EModeCategory是否和用户的一致
  - 计算总抵押物价值，检查是否大于0
  - 当前LTV是否大于0
  - healthFactor（用户总抵押物/总债务）是否大于阈值（1）
  - 需要的抵押物（用户总债务+需要借款金额）/所有资产平均LTV，是否小于用户当前的总抵押物价值
  - 固定利率借款：借款金额是否小于当前资产剩余可借金额
- 稳定利率借款：mint稳定债务token到用户地址
- 动态利率借款：mint动态债务token到用户地址
- 更新资产相关利率
- 转出资产token到用户地址

**repay**

- 更新资产状态
- 检查是否满足还款条件（对应利率模式的债务大于0）
- 销毁对应债务的debtToken（StableDebtToken、VariableDebtToken）
- 更新资产相关利率
- 使用aToken还款：销毁对应的aToken
- 使用原始资产还款：将资产token转到对应的aToken合约地址

**withdraw**

- 更新资产状态
- 检查用户持有的aToken是否足够覆盖请求提现的金额
- 更新资产相关利率
- 销毁用户持有的aToken
- 转出资产token到用户的地址

**flashLoan**

- 检查请求借款的每个资产的状态
- 设置IFlashLoanReceiver对象（如果要使用闪电贷功能，调用方需要实现IFlashLoanReceiver接口）
- 计算当前闪电贷请求借款的所有资产的费用
- 将用户请求借款的资产转到用户地址
- 回调用户自定义的智能合约逻辑
- 如果调用者选择在同一区块还款（interestRateMode == InterestRateMode.NONE），处理还款逻辑
  - 计算给aave协议的费用
  - 计算给流动性池的费用
  - 更新资产状态
  - 更新资产相关利率
  - 将本金和所有费用转到资产对应的atoken地址
- 如果调用者选择不返还资产，将执行一次借款逻辑，生成一笔债务，并向调用者发放debtToken（ opens a debt position）



**liquidationCall**

- 更新债务资产状态
- 计算用户的healthFactor
- 计算用户总债务、动态利率债务、可清算债务额（healthFactor>0.95时为50%，小于等于0.95时为100%）
- 检查清算调用是否满足条件
  - 抵押物和债务资产是否为active和paused的
  - 预言机是否设置为允许清算
  - healthFactor是否小于1
  - 清算阈值是否大于0，用户是否允许抵押
  - 用户是否有债务
- 计算可被清算的抵押物
  - actualCollateralToLiquidate：实际被清算抵押物，当目标清算抵押物价值小于债务价值时为全部抵押物，如果抵押物大于债务价值时，为债务值加清算奖励（bonus），如果有平台手续费，需减去手续费
  - actualDebtToLiquidate：实际被清算债务，当目标清算抵押物价值小于债务价值时为全部抵押物价值减去清算奖励，如果抵押物大于债务价值，为清算债务值
  - liquidationProtocolFeeAmount：付给aave协议的费用
- 消除债务，销毁用户债务token，优先销毁动态利率债务，再销毁固定利率债务
- 更新债务资产利率
- 如果用户设置债务资产为隔离模式，更新隔离债务
- 如果清算者接受aToken，清算用户的aToken，将用户作为抵押物的aToken转到清算者地址
- 如果清算者不接受aToken，销毁aToken，将抵押资产原始token转到清算者地址
- 划转清算费用到aave清算费用地址
- 从清算者地址转出债务资产token（actualDebtToLiquidate）到对应aToken地址



**setUserUseReserveAsCollateral**

- 检查用户存款状态和对应资产状态
- 设置资产为抵押物
  - 检查用户在当前资产是否处于隔离模式
  - 设置资产为抵押物
- 取消资产为抵押物
  - 取消资产为抵押物
  - 检查取消后用户在当前资产的healthFactor和LTV是否符合要求

------

### PoolConfigurator（治理层）

- PoolConfigurator 为 Pool 合约提供配置方法。该合约的写入方法仅可由具备相应授权系统角色的地址调用，这些角色由 ACLManager 管理。
- 修改Pool **所有风险参数**
  - LTV
  - Liquidation Threshold
  - Reserve Factor
  - eMode 分类
  - Borrow / Supply 开关

📌 **这是 v2 → v3 的关键升级：**

> 执行权 & 决策权彻底分离

------

### Logic Libraries（规则引擎）

实现各种操作和功能基础逻辑的库。

### 主要 Library

| Library          | 职责             |
| ---------------- | ---------------- |
| ValidationLogic  | 所有前置检查     |
| ReserveLogic     | 利率、Index 更新 |
| BorrowLogic      | 借贷执行         |
| SupplyLogic      | 存款执行         |
| LiquidationLogic | 清算             |



------

### 资产状态核心：ReserveData（Aave 的心脏）

```
struct ReserveData {
  uint128 liquidityIndex;
  uint128 variableBorrowIndex;
  uint128 currentLiquidityRate;
  uint128 currentVariableBorrowRate;
  uint40 lastUpdateTimestamp;
}
```

每个reserve对应三种token：aToken、stableDebtToken、variableDebtToken，分别用来记录用户在该资产提供的流动性、稳定利率债务和动态利率债务。基于这几个token，可以算出用户的healthFactor，用户的借款、提现、清算动作能否执行都要基于healthFactor去做判断，是用户在与协议交互过程中的关键风控变量。

### 设计要点

- **Index 驱动一切利息**

- Token 不负责算利息

- 所有余额都是：

  ```
  realBalance = scaledBalance * index
  ```

📌 **这是 Aave v3 支持：**

- 高性能
- 跨链
- unbacked aToken
  的基础

------

### Token 架构：极简 + 会计化

### Token 类型

| Token             | 职责     |
| ----------------- | -------- |
| aToken            | 存款凭证 |
| VariableDebtToken | 浮动债   |
| StableDebtToken   | 固定债   |

### 设计哲学

- Token **不计算利息**
- Token **不判断风险**
- 只做：
  - mint
  - burn
  - balance tracking

👉 **Token ≈ 数据表，不是业务层**







![How AAVE works with visualization | by 0xape | Medium](https://miro.medium.com/v2/resize%3Afit%3A1400/0%2Aa-8Oxq2Vq41fecG6.png)





### Market Configurator

Market Configurator 是 Aave 协议的市场配置模块,用于管理员配置支持的资产类型、利率模型、清算阈值等系统参数。

关键流程:

- `initReserve`: 初始化资产,设置资产的利率模型、清算阈值、储备因子等参数
- `setReserveFactor`: 设置资产的储备因子,即收取的协议费用比例
- `enableBorrowingOnReserve`: 开启或关闭资产的借款功能

![image-20260119211446429](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260119211446429.png)








