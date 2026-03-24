# 代币化存款Oracle

按 **“银行系统 + Oracle + 智能合约 + 风控/审计”** 四层来设计。
因为“把利息同步到链上”本质不是喂一个价格，而是把 **银行账本里的收益状态，可信地映射到链上资产净值**。

一句话先定性：

**这类 Oracle 不是市场价格预言机，而是“账本状态预言机 / NAV Oracle / Interest Accrual Oracle”。**

------

# 一、先明确目标

假设银行发行一个代币化存款 `JPMD`，代表客户在银行的美元存款份额。

你要同步到链上的，不是“今天利率是多少”这么简单，而通常是下面几类状态之一：

### 方案 A：同步累计净值指数

比如：

```text
interestIndex = 1.00000000
interestIndex = 1.00013698
interestIndex = 1.00027399
```

链上根据 index 计算用户资产。

这是最推荐的。

------

### 方案 B：同步每期利率

比如：

```text
dailyRate = 0.00013698
```

链上自己累计。

这个能做，但不如 A 稳。

------

### 方案 C：同步 totalAssets

比如：

```text
totalAssets = 1,256,333,221.91 USD
```

链上用 `totalAssets / totalSupply` 算净值。

这适合 ERC4626 / share 模型。

------

# 二、最推荐的总体设计

我建议银行做成 **“双账本 + 签名型 Oracle + 延迟生效 + 审计可追溯”** 模型。

整体架构：

```text
┌─────────────────────────────┐
│      Bank Core Ledger       │
│  存款主账 / 利息计算引擎      │
└─────────────┬───────────────┘
              │
              │ 生成官方收益快照
              ▼
┌─────────────────────────────┐
│   Interest Calculation Hub  │
│  日终/小时净值、利率、总资产   │
└─────────────┬───────────────┘
              │
              │ 多方审批 / 风控校验
              ▼
┌─────────────────────────────┐
│   Oracle Signer Cluster     │
│  HSM/MPC 签名节点集群         │
└─────────────┬───────────────┘
              │
              │ signed report
              ▼
┌─────────────────────────────┐
│   On-chain Oracle Contract  │
│  验签 / 时序校验 / 更新状态    │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│ Deposit Token / Vault       │
│  根据 index 或 totalAssets   │
│  更新兑换率或用户余额         │
└─────────────────────────────┘
```

------

# 三、链下部分怎么设计

## 1）核心银行账本层

这里是数据源，必须是唯一真相源。

通常包括：

- 客户存款主账
- 计息规则引擎
- 资金配置系统
- 总账 / 子账
- 日终清算系统

Oracle 绝不能直接从多个业务系统零散取数，而应该从 **已对账、已结算的 canonical ledger** 取数。

要同步的核心字段建议统一成一个快照对象：

```json
{
  "product_id": "JPMD-USD",
  "as_of": "2026-03-16T00:00:00Z",
  "total_principal": "1000000000.00",
  "accrued_interest": "136986.30",
  "total_assets": "1000136986.30",
  "total_supply": "1000000000.00",
  "interest_index": "1.0001369863",
  "version": 1024
}
```

这里最重要的是：

- `as_of`：这份快照对应哪个时点
- `version`：单调递增，防重放
- `interest_index` 或 `total_assets`：最终链上使用值

------

## 2）利息计算引擎

这个模块负责把银行内部的计息规则标准化，输出链上可消费的结果。

常见计息逻辑：

- 年化利率，按日计提
- 浮动利率，按区间切换
- 工作日 / 自然日规则
- 节假日顺延
- 复利 / 单利
- 扣税 / 费用前后口径

这里一定要避免把复杂规则搬到链上。
**正确做法是：链下计算最终结果，链上只验证发布权限和数据时序。**

也就是说：

```text
复杂金融逻辑在银行内部
简化后的结果状态上链
```

------

## 3）Oracle 签名集群

这一层不要只做一个中心化 signer。
推荐做成 **M-of-N 签名集群**，并用 HSM 或 MPC 管理密钥。

例如：

- 5 个 oracle signer
- 至少 3 个签名才有效

签名节点可以来自不同控制域：

- 财务控制域
- 风控控制域
- 技术运维控制域
- 合规控制域
- 外部受托审计节点（可选）

这样即使某个系统被攻破，也不能单独伪造利息数据。

签名前先做三类校验：

### 数值校验

比如：

- `interest_index` 必须大于等于上次值
- 单日增幅不能超过阈值
- `total_assets >= total_supply`
- 利率跳变不能超出 policy

### 时序校验

比如：

- `as_of` 必须晚于上次
- `version` 严格递增
- 不允许回滚到旧快照

### 对账校验

比如：

- 与总账一致
- 与产品账一致
- 与托管/清算账一致

------

# 四、链上 Oracle 合约怎么设计

链上合约应该尽量简单，不做复杂金融运算，只做：

- 签名验证
- 版本校验
- 时间戳校验
- 风险阈值限制
- 数据落链

推荐存这几个字段：

```solidity
struct Report {
    bytes32 productId;
    uint64 asOf;
    uint64 version;
    uint256 totalAssets;
    uint256 totalSupply;
    uint256 interestIndex;
}
```

其中：

- `interestIndex` 建议用 1e18 精度
- `version` 单调递增
- `asOf` 用 UTC 时间戳

合约状态：

```solidity
uint64 public latestVersion;
uint64 public latestAsOf;
uint256 public latestInterestIndex;
uint256 public latestTotalAssets;
uint256 public latestTotalSupply;
```

------

## 关键校验逻辑

### 1）验签

验证至少 `M` 个授权 signer 对同一 report 签名。

### 2）防重放

要求：

```solidity
require(report.version > latestVersion);
require(report.asOf > latestAsOf);
```

### 3）单调性

如果是累计净值模型：

```solidity
require(report.interestIndex >= latestInterestIndex);
```

正常情况下利息累计指数不该下降。
除非你产品允许扣费、坏账、罚息，这种情况需要专门设计“可下降但有限幅”的规则。

### 4）风控阈值

防止一次恶意更新把净值从 `1.0001` 拉到 `3.5`。

例如：

```solidity
uint256 maxBpsChangePerUpdate = 50; // 单次最多 0.5%
```

校验：

```solidity
newIndex <= oldIndex * (10000 + maxBpsChangePerUpdate) / 10000
```

### 5）时间窗口

限制只能更新最近窗口内的数据：

```solidity
require(block.timestamp <= report.asOf + MAX_DELAY);
```

防止老数据重新进链。

------

# 五、Token/Vault 如何消费 Oracle 数据

这里有两种主流模式。

## 模式 1：Exchange Rate / Share 模型

最适合银行场景。

用户持有的是 shares，不是固定 1:1 本金 token。
Oracle 更新 `interestIndex` 或 `totalAssets` 后，用户可赎回价值上升。

例如：

```text
Alice 持有 1000 shares
初始 index = 1.0
赎回价值 = 1000 USD

更新后 index = 1.01
赎回价值 = 1010 USD
```

优点：

- 不需要给每个地址单独发利息
- 链上 gas 很低
- 非常适合机构产品
- DeFi 兼容更好

------

## 模式 2：Rebase 模型

Oracle 更新后，用户余额自动增大。

例如：

```text
Alice: 1000 -> 1001.369863
```

这个对“看余额”的体验直观，但对一些协议兼容性不如 share 模型。

银行级产品我更建议第一种。

------

# 六、推荐的数据发布节奏

不要每秒同步。
银行计息通常不是高频市场数据，不需要 price feed 那种秒级刷新。

建议节奏：

### 方案 1：日更

每天日终跑一次，更新一次累计净值。

适合大多数存款产品。

### 方案 2：小时级

适合想强调“全天候结算”的机构产品。

### 方案 3：事件驱动 + 定时兜底

以下事件触发更新：

- 利率切换
- 大额申购赎回
- 日终结息
- 异常修复

再加一个固定日更兜底。

------

# 七、最关键的安全设计

这个系统最大的风险不是“价格不准”，而是 **账本伪造、时间回滚、权限滥用、错误净值扩散**。

所以必须做以下设计。

## 1）双轨发布：pending / active

不要 Oracle 一发，Token 立刻生效。
建议做双阶段：

- `submitReport()`
- `activateReport()` 经过 delay 后生效

比如 30 分钟 timelock。

这样风控系统或人工监控发现异常时可以暂停。

------

## 2）紧急暂停

如果发现异常：

- 停止新 report 激活
- 暂停 mint
- 暂停 redeem，或只保留受控赎回
- 暂停抵押品使用

要有 `pause()` 和 `guardian pause()`。

------

## 3）审计哈希上链

建议把完整链下快照文件做哈希：

```text
reportHash = keccak256(full_report_json_or_pdf)
```

链上只存简化字段 + `reportHash`。

这样未来审计时可证明：

- 当时链上使用的数据
- 对应哪份银行内部报表
- 是否被篡改

------

## 4）角色隔离

至少分开这些角色：

- 数据生成者
- 签名者
- 链上提交者
- 暂停管理员
- 参数管理员

不能一个热钱包全包。

------

## 5）多链一致性

如果以后发到多个链：

- Base
- Ethereum
- 私链
- 其他 L2

不要每条链各算各的。
应该用同一个 canonical report，在各链同步同一 `version` 和 `hash`。

否则会出现跨链净值不一致问题。

------

# 八、推荐的消息格式

建议链下签名用 EIP-712，便于链上验签与审计。

示意结构：

```solidity
InterestReport(
    bytes32 productId,
    uint64 asOf,
    uint64 version,
    uint256 totalAssets,
    uint256 totalSupply,
    uint256 interestIndex,
    bytes32 reportHash
)
```

为什么要 EIP-712：

- 防止签名被别的场景复用
- 域分隔明确
- 链 ID、合约地址都能绑定
- 审计友好

------

# 九、一个更完整的生产级流程

完整流程建议这样：

### 第一步：银行账本结息

核心系统生成 `as_of=T` 的产品快照。

### 第二步：内部对账

与总账、产品账、清算账核对。

### 第三步：风控规则检查

检查利率跳变、净值跳变、总资产异常。

### 第四步：生成标准 report

产出标准 JSON / protobuf / CSV + 哈希。

### 第五步：M-of-N 签名

Oracle signer 集群对 report 签名。

### 第六步：上链提交

Relayer 把 report 和签名提交给 Oracle Contract。

### 第七步：进入 pending

链上记录 pending report，不立即生效。

### 第八步：延迟观察

监控系统、人审、自动规则核验。

### 第九步：激活生效

Oracle 状态更新为 active，Vault/Token 读取新 index。

### 第十步：记录审计轨迹

将本次 `version` 与链下报表、审批记录、签名日志对应归档。

------

# 十、智能合约模块拆分建议

不要把所有逻辑堆到一个合约里。
建议拆成：

## 1）Oracle Contract

只负责 report 管理和状态发布。

## 2）Rate Consumer / Vault

只负责根据 index 计算兑换率。

## 3）Risk Controller

负责 pause、阈值参数、timelock。

## 4）Access Control

统一角色管理。

这样升级、审计、隔离都更好。

------

# 十一、推荐的链上核心字段

如果你想做得最稳，我建议链上只放这 6 个核心值：

- `productId`
- `version`
- `asOf`
- `interestIndex`
- `totalAssets`
- `reportHash`

其中真正给业务用的是：

- `interestIndex` 或 `totalAssets`

其他是风控和审计字段。

------

# 十二、一个简化版 Solidity 思路

示意，不是完整生产代码：

```solidity
struct InterestReport {
    bytes32 productId;
    uint64 asOf;
    uint64 version;
    uint256 totalAssets;
    uint256 totalSupply;
    uint256 interestIndex; // 1e18
    bytes32 reportHash;
}

contract InterestOracle {
    mapping(address => bool) public isSigner;
    uint256 public signerThreshold = 3;

    uint64 public latestVersion;
    uint64 public latestAsOf;
    uint256 public latestInterestIndex;
    bytes32 public latestReportHash;

    function submitReport(
        InterestReport calldata report,
        bytes[] calldata sigs
    ) external {
        require(report.version > latestVersion, "old version");
        require(report.asOf > latestAsOf, "old timestamp");
        require(report.interestIndex >= latestInterestIndex, "index decreased");

        _verifySigs(report, sigs);
        _checkRiskBounds(report);

        latestVersion = report.version;
        latestAsOf = report.asOf;
        latestInterestIndex = report.interestIndex;
        latestReportHash = report.reportHash;
    }
}
```

生产版还应该有：

- pending/active 双状态
- timelock
- pause
- signer 去重
- EIP-712 typed data
- max delay / max increase / max decrease
- upgrade 管控

------

# 十三、我给你的架构结论

如果我是这类项目的总架构师，我会明确采用下面这个方案：

**链下：银行账本计算累计净值指数
链上：多签 Oracle 发布 `interestIndex`
资产合约：采用 share/exchange-rate 模型
风控：timelock + pause + 阈值限制 + 审计哈希**

这是最稳、最符合银行合规、也最容易接入 DeFi 的方案。

------

# 十四、为什么不建议“银行 API 直接上链”

很多人第一反应是：

```text
后端定时调用银行 API，把利率写到链上
```

这个太弱，问题很多：

- 单点信任
- 无多方制衡
- 无审计闭环
- 容易热钱包被盗
- 容易误推错误数据
- 不能证明链上值对应哪份正式账本

所以生产级系统必须做成：

**正式账本快照 → 标准 report → 多方签名 → 延迟生效 → 可审计落链**

------

# 十五、给你一个最实用的落地建议

你如果真要设计 PoC 或正式系统，第一版不要上来就做复杂 rebase。
先做：

1. `InterestOracle`
2. `DepositVault(ERC4626风格)`
3. `interestIndex` 日更
4. M-of-N signer
5. timelock + pause
6. reportHash 审计锚定

这样第一版就已经很像银行级架构了。

下一步再扩展：

- 多币种
- 多链同步
- 外部审计节点
- 零知识证明账本一致性
- 合规白名单地址
- 自动抵押品折扣率联动

如果你要，我下一步可以直接给你画一版 **“银行存款代币 + 利息 Oracle + Vault 合约”的完整时序图和 Solidity 模块设计**。