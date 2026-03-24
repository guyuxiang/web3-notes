# Web3 供应链金融智能合约整体设计



## **1. 执行摘要**

在传统供应链金融中，信息孤岛、虚假贸易、信用传递困难以及跨境清结算延迟是核心痛点。本平台利用以太坊生态技术栈，通过 **ERC-3643 合规标准**、**可验证凭证（VC）** 和 **代币化资产（RWA）**，构建一个透明、自动化且符合全球监管要求的下一代金融基础设施。

## **2. 核心架构层级**

![Gemini_Generated_Image_fwrouufwrouufwro](C:\Users\顾宇翔\Documents\WXWork\1688857451512266\Cache\Image\2025-12\Gemini_Generated_Image_fwrouufwrouufwro.png)



## **3. 合规身份层 (Identity & Compliance)**

平台的核心准入机制基于 **ERC-3643** 协议与 **去中心化身份（DID）**。

- **链上 KYC 体系**：通过 `Identity Registry` 建立地址与合规状态的映射。实现“一次 KYC，多方复用”，大幅降低企业的准入成本。
- **可插拔的规则引擎**：用于校验特定司法辖区限制、投资者合格性及制裁名单。
- **可验证凭证 (VC)**：企业完成一次 KYB（银行 / 合规服务商）签发 VC，企业持有包含注册信息、评级等声明的 VC。在交互时，通过数字签名或 **零知识证明 (ZKP)** 证明合规性（如“非制裁实体”），而无需暴露底层隐私数据。



**ERC-3643 合规资产模型**

ERC-3643 提供 **受监管资产的标准化合规模块**：

**核心组件**

- **Identity Registry**：地址 ↔ DID ↔ 合规状态
- **Compliance Modules**：
  - KYC 是否完成
  - 是否属于特定司法辖区
  - 是否为合格投资者
- **Transfer Validator**：每次转账前强制校验



## **4. 货币与价值层 (Monetary Layer)**

支持多维度的链上资金形式，确保流动性的高度可编程性。

- **合规稳定币**：基于 ERC-3643 标准发行，法币 1:1 锚定，仅限白名单机构持有。

- **储备金 Oracle**：集成多源预言机（托管行、审计机构），实时披露储备金状态。具备偏离阈值自动熔断机制，确保系统性安全。

- **代币化存款 (Tokenized Deposits)**：将商业银行负债映射上链，支持 T+0 实时结算，实现与银行核心系统的无缝对账。

- **代币化授信额度（Credit Line Token）**:代表银行授予企业的代币化信贷额度，可作为支付资产



## **5. 供应链金融业务引擎**

### **5.1 资产代币化：应收账款 (Account Receivable)**

平台将应收账款转化为**符合ERC-3643 协议** 的**动态贴现定价 ERC20 代币**。

- **ONCHAINID**：记录核心信息：

  - 面值 (Face Value)
  - 到期日 (Maturity Date)
  - 债务人 ID (Debtor)
  - 贴现率 (Discount Rate)

- **ERC-3643 协议**：将“合规判断”从 **人工流程** 转为 **智能合约可执行规则**

- **动态贴现模型**：内置贴现现金流法（DCF）函数，代币价格随到期日的临近自动向面值回归，支持精准的链上定价，提供Oracle查询。

  - 应收账款定价的核心逻辑是：**现在的 1 块钱比未来的 1 块钱更值钱。**
  
    即使账款面值（Face Value）是 1 USDT，但在它到期支付之前，它的市场价值永远小于 1 USDT。这个差额就是投资者的收益来源（折价买入，按面值赎回）。
  
  
  #### 核心定价公式
  
  假设该笔应收账款的年化贴现率为 r，距离到期日剩余天数为 t，则该 ERC-20 代币的当前价格 P 为：
  
  
  $$
  NAV_{Invoice} = 面值 \times (1 - 贴现率 \times \frac{剩余天数}{365})
  $$
  
  $$
  P = 1 \times \left(1 - r \times \frac{t}{365}\right)
  $$
  
  
  - **1 (面值)：** 假设 1 个代币对应 1 购买力单位（如 1 USDT）。
  - **r (贴现率)：** 由风险评估（债务人信用、行业风险）决定的年化收益要求。
  - **t (剩余天数)：** t = 到期时间 - 当前时间。
  
  
  
  **链上资产市场：** 记录核心信息：
  
  - 面值 (Face Value)
  - 到期日 (Maturity Date)
  - 债务人 ID (Debtor)
  - 贴现率 (Discount Rate)
  - 电子签名

### **5.2 多级流转与支付逻辑**

平台支持核心企业信用的穿透式传递。

| **模式**         | **核心逻辑**                                                 |
| ---------------- | ------------------------------------------------------------ |
| **买方发起**     | 核心企业利用其信用背书，为主导的供应链提供融资。             |
| **卖方发起**     | 供应商基于已确认的应收账款进行融资请求。                     |
| **组合支付引擎** | 支持稳定币、代币化存款及代币化信贷的原子化组合支付，由合约自动处理优先级。 |

### **5.3 条件支付 (Conditional Payment)**

利用智能合约实现支付的精准触发：

- **时间条件**：到期自动释放资金。
- 各方审批条件
- **事件条件**：基于物流状态（eBL）或发票验证（eInvoice）触发。

### 5.4 多级流转与自动化结算

- 应收账款可多次转让
- 每一跳合规校验
- 完整链上追踪

**清结算引擎：**

- 合约根据 ERC20 的持有比例，自动按优先级分发给供应商、流转方和 ABT 投资者，实现 Atomic Settlement（原子化结算）

### 5.5 法律与存证

- **eIDAS 合规**：支持合格电子签名，确立链上交易的法律效力。
- **分布式存储**：合同 Hash 上链，原始文件存储于 IPFS 或加密的企业级存储中。

------





## **6. 结构化金融与二级市场**

### **6.1 资产池与 ABT (Asset-Backed Tokens)**

平台支持将多个应收账款代币打包成资产池，并进行分层设计（Tranching）：

- **优先级 (Senior)**：较低风险，优先获得现金流清偿。
- **劣后级 (Junior)**：吸收首损，获得更高预期收益。
- 资产动态出入资产池
- 支持投资者使用多种资金认购



### 6.2 Portfolio NAV 的计算流程

整个投资池（Portfolio）的 NAV 是池内所有资产价值的总和。对于包含多种应收账款 ERC-20 和部分闲置资金的池子，公式如下：
$$
NAV_{Total} = \sum_{i=1}^{n} (Balance_i \times P_i) + Cash_{Stablecoin}
$$


### 计算步骤：

1. **资产识别：** 统计 Portfolio 合约中持有的所有应收账款 ERC-20 代币种类及余额。
3. **计算现值：** 根据当前区块时间（Block.timestamp）计算每种代币的当前贴现价格 P。
4. **汇总：** 将所有代币的现值加上池内未使用的稳定币余额。

可以通过以下两种方式更新：

- **链上实时计算（View Call）：** 投资者在前端看到的是实时根据 `block.timestamp` 计算出的动态 NAV。
- **定时快照（On-chain State）：** 每天由 Keepers 或发行方调用 `updateNAV()`。这对于 Senior/Junior 的利息分配至关重要。

### 瀑布式清偿

执行瀑布式清偿逻辑，确保现金流首先覆盖 Senior 级代币持有者，剩余部分分发给 Junior 级。

​	现金流 → Senior → Junior



## **7. 二级市场**

平台支持为应收账款代币或ABT提供多种交易模式

### **7.1 交易机制**

二级市场支持多种交易模式优化资产定价效率，所有操作均受合规模块（Transfer Validator）控制：

- **OTC  **
- **RFQ**：针对大额机构交易。
- **荷兰拍卖（Dutch Auction）**
- **盲拍 / 封闭式拍卖设计（Sealed Bid Auction）**
- **定期拍卖（Periodic Auction）**

------



## **8. 基础设施与信任保障**

### 8.1 OTC 多法币出入金 

- 接入第三方OTC通道
- 多法币接入
- 合约自动换汇

### **8.2 信用体系与仲裁**

- **灵魂绑定代币 (SBT)**：根据历史履约记录、还款速度和交易量，为企业发放不可转让的 SBT，为后续业务开展和第三方引用提供链上凭证，动态更新，可被第三方读取。
- **链上仲裁**：开发去中心化仲裁协议，由评审团依据链上证据对物流或质量纠纷进行裁决。

### **8.3 跨链与隐私**

- **跨链机制**：实现资产在不同区块链生态间的无缝移动。
- **隐私保护**：通过隐私交易技术隐藏敏感的商业交易金额与对手方信息。

### 8.4 第三方 DeFi 协议的接入

- 如Uniswap swap、AAVE V4的借贷





# ABT设计

**发行方 / 管理人（Manager）**：创建 portfolio、配置参数、投放投资组合资产、发起结算

**投资者（Investor）**：用稳定币认购，选择 SeniorToken 或 JuniorToken

**估值与喂价者（NAV Reporter / Oracle）**：每日更新各资产价格、计算 NAV 并上链记录（可多签/委员会）

**托管/策略执行（Strategy / Adapter）**：把资金投到不同资产代币或收益策略（Aave、RWA、LP 等）



### 生命周期（典型 4 阶段）

1. **募资期 Fundraising**：允许稳定币、代币化存款、代币化信贷认购；可设置上限/白名单/最小额
2. **启动期 Live**：募集结束，按认购结果铸造并发放**SeniorToken / JuniorToken**；资金进入策略/资产组合
3. **运行期 Running**：每日记录 NAV；允许/不允许中途赎回（看产品定位）
4. **到期结算 Settlement**：停止策略、收回资产→换回稳定币→按瀑布分配，投资者赎回凭证





### SeniorToken（低风险）

- 本质：**优先级债权/优先收益份额**
- 目标：更稳的收益（可设“目标年化/上限”或“浮动但优先”）
- 风险：资产亏损时 **先由 Junior 吸收**，Senior 最后受损（但不是绝对无风险）

### JuniorToken（高风险）

- 本质：**劣后权益/杠杆收益份额**
- 目标：承担波动，获取剩余收益（上不封顶 / 更高收益）
- 风险：亏损**优先吞**，可能归零



### 总 NAV（以稳定币计价）

对 portfolio 持仓资产集合 AAA：
$$
NAVt=baseBalancet+∑i∈A(qtyi,t×pricei,t)−liabilitiestNAV_t = \text{baseBalance}_t + \sum_{i \in A} (qty_{i,t} \times price_{i,t}) - liabilities_tNAVt=baseBalancet+i∈A∑(qtyi,t×pricei,t)−liabilitiest
$$


- `price_{i,t}` 来自自建报价（RWA 可能只能“管理人报价+审计签名”）
- `liabilities_t` 如果你引入杠杆/借贷需要计入（否则可为 0）





### 分层净值（如果你要展示 Senior/Junior 单独 NAV）

你需要一个“分层债务/权益”模型。最常见的是 **Senior 目标收益 + 劣后吸收**：

设：

- `S0` = Senior 初始本金（募集期 Senior 认购额）
- `J0` = Junior 初始本金
- `r_s` = Senior 目标日利率（或年化换算）
- `t` = 运行天数

Senior 应计“应收额”（像债务余额）：

![image-20251230155029245](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20251230155029245.png)



然后按瀑布确定当日可覆盖的 Senior 价值：

![image-20251230155118264](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20251230155118264.png)

计算Junior 价值：

![image-20251230155127176](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20251230155127176.png)



再换算成每份额净值（用于前端展示/赎回参考）：

![image-20251230155013750](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20251230155013750.png)

> 这样实现了：资产整体收益先满足 Senior 目标增长，超出部分归 Junior；亏损先打到 Junior，Senior 只有在 NAV 低于 Senior 应收额时才会受损。





## 到期结算与收益分配（瀑布分配）

### 结算前步骤

1. `startSettlement()`：状态切换 Settling，暂停新认购/赎回/调仓
2. `unwind()`：策略退出、赎回外部协议资产，尽量换回 baseAsset
3. 得到 `NAV_final`（以 baseAsset 计价，可等同于 Vault baseAsset 实际余额）

### 结算瀑布（推荐）

定义：

- `fees`：管理费/绩效费等先扣（或后扣，见下面）
- `S_due = S0 * (1 + r_s)^T`（Senior 到期应收）
- `NAV_net = NAV_final - fees`

分配：

- 给 Senior：`S_pay = min(NAV_net, S_due)`
- 给 Junior：`J_pay = NAV_net - S_pay`

> 若 `NAV_net < S_due`：Senior 发生损失，Junior 通常归零。

### 投资者赎回接口

- `redeemSenior(shares)`：按 `S_pay / totalSeniorShares` 兑付 baseAsset
- `redeemJunior(shares)`：按 `J_pay / totalJuniorShares` 兑付 baseAsset
- 兑付后销毁 s/j token（burn）



### ABT的代币化资产可以设计为两种产品：

- **封闭式（更像基金/票据）**：不支持中途赎回，只能二级交易 token
- **开放式**：支持按 NAV 赎回，但需要：
  - 赎回排队/冷却期
  - 流动性缓冲（cash buffer）
  - 提前赎回费与限额（防挤兑）



### 费用模型

常见费用：

1. **管理费（Management Fee）**：按 AUM 年化计提（每日/每区块累积），最终在结算时扣
2. **绩效费（Performance Fee）**：只对 **Junior 的正收益** 收取更合理（避免侵蚀 Senior 保障）
3. **认购/赎回费（可选）**：抑制短期资金进出（如果支持中途赎回）

建议简化为：

- 管理费：从 `NAV` 里每日累积到 `accruedFee`
- 绩效费：结算时若 `J_pay > J0`，对 `J_profit = J_pay - J0` 收取 x%

