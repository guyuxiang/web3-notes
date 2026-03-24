# SATOSHI TEST OF UNHOSTED WALLETS

**1. 基本理解**

**[Satoshi Test] 是一项针对 **[非托管钱包](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=非托管钱包&zhida_source=entity)（Unhosted Wallets）的合规性评估机制，其核心目标是验证用户**是否真正掌握钱包私钥**，以防止匿名洗钱或逃避监管。

**非托管钱包** 指用户自行管理私钥的加密钱包（如[MetaMask](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=MetaMask&zhida_source=entity)、[Ledger](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=Ledger&zhida_source=entity)），与交易所托管的钱包（如[Coinbase Wallet](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=Coinbase+Wallet&zhida_source=entity)）不同。由于非托管钱包的匿名性，监管机构需通过特定测试确保其使用符合反洗钱（AML）和反恐融资（CFT）要求。



**2. 测试背景与监管动因**

\- **监管挑战**：

非托管钱包允许用户绕过交易所的KYC（了解你的客户）审核，成为洗钱、逃税或非法交易的渠道。

\- **监管需求**：

美国FinCEN、欧盟[MiCA](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=MiCA&zhida_source=entity)等要求机构在向非托管钱包提供业务时验证用户身份，确保交易可追溯。



**3. Satoshi Test 的核心流程**

**(1.1) 签名式 Satoshi Test**

\- **挑战-响应机制**：

用户向平台提供非托管钱包的地址，平台要求用户签署一条特定消息（如随机生成的哈希值），若签名有效则证明用户拥有私钥。

**Step 1：平台生成挑战消息（Challenge）**

挑战消息通常包含：

- 随机数（nonce）
- 平台标识
- 时间戳
- 业务上下文（可选）

**Step 2：用户使用目标钱包进行签名**

在 EVM 体系中，通常使用：

```
const signature = await wallet.signMessage(message);
```

**Step 3：平台验证签名**

平台通过密码学方式验证：

- 签名是否由该地址私钥生成
- 是否与挑战消息完全匹配
- 是否在有效时间内

如果成立：

> ✅ 该钱包通过 Satoshi Test（签名式）



**(1.2) 转账式 Satoshi Test**

平台要求用户 **从其提供的目标非托管钱包发起一笔指定条件的链上转账**

**Step 1：平台生成「测试指令」**

平台（如交易所、合规 DeFi 前端）生成：

- 指定 **接收地址**
- 指定 **最小金额**（如 0.0001 ETH / 1 USDC）
- 指定 **时效窗口**（如 24 小时内）

------

**Step 2：用户从目标非托管钱包发起转账**

用户必须：

- 使用 **其提供的目标地址 **的钱包
- 按Step 1平台要求里的**接收地址**和**金额** **主动发起转账**

**Step 3：链上确认与匹配**

平台自动校验：

- `from` 地址 = 申报地址
- 金额 ≥ 指定最小值
- 时间 ≤ 有效窗口

一旦匹配成功：

> ✅ **Satoshi Test 通过**



**(2) 交易链路分析**

\- **区块链溯源**：

平台通过链上分析工具（如[Chainalysis](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=Chainalysis&zhida_source=entity)、[Elliptic](https://zhida.zhihu.com/search?content_id=255249121&content_type=Article&match_order=1&q=Elliptic&zhida_source=entity)）追踪资金来源，确保非托管钱包内的资产未涉及非法活动。

\- **案例**：若某钱包接收过被标记为勒索资金的UTXO，平台可拒绝其业务请求。

**监管意义：**

> 即使你“拥有私钥”，**资金不干净，仍可能被拒绝服务**



**4. 监管应用场景**

- 前端在用户交互前：
  - 要求钱包签名
  - 做地址风险评估
- 对高风险钱包：
  - 限制访问
  - 提高合规门槛
  - 直接拒绝服务



**Satoshi Test + KYC + 链上风控 = 合规闭环**


