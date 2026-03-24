# Account Abstraction 面试速记

这份文档用于面试前快速回顾，重点不是细节完整，而是：

- 能快速讲清项目结构
- 能说清 `EntryPoint` 主流程
- 能区分 account / paymaster / nonce / 7702 的职责
- 能回答常见追问

## 1. 项目一句话

这是一个 ERC-4337 Account Abstraction 核心合约仓库，围绕 `UserOperation` 提供：

- 统一入口 `EntryPoint`
- 智能账户基类 `BaseAccount`
- Paymaster 基类 `BasePaymaster`
- 资金账本 `StakeManager`
- 分 key nonce 管理 `NonceManager`
- sender 部署/初始化辅助和 EIP-7702 支持

一句话解释它的工作方式：

**Bundler 收集用户的 `UserOperation`，提交给 `EntryPoint`，`EntryPoint` 先校验账户和付款方是否认可这笔操作，再让账户执行目标调用，最后统一结算 gas。**

## 2. 核心角色怎么分工

### `EntryPoint`

职责：

- 验证 `UserOperation`
- 必要时创建/初始化 sender
- 执行账户调用
- 结算 gas
- 给 bundler 付款

面试表达：

**`EntryPoint` 是 ERC-4337 的执行和结算中心，不是钱包，也不是工厂。**

### `Account`

职责：

- 验签
- 定义自己的 nonce 规则
- 执行真正的业务调用
- 必要时补充 EntryPoint deposit

面试表达：

**账户负责“我是否认可这笔操作”，并在执行阶段代表用户去调用目标合约。**

### `Paymaster`

职责：

- 决定是否替用户付 gas
- 在 `postOp` 做记账、收费或额度扣减

面试表达：

**Paymaster 负责“谁来付 gas”，不负责替账户执行业务逻辑。**

### `Bundler`

职责：

- 收集 mempool 里的 `UserOperation`
- 调用 `handleOps`
- 垫付交易 gas
- 从 `EntryPoint` 取回最终手续费

## 3. `handleOps` 怎么讲

最标准的回答：

1. `EntryPoint.handleOps(ops, beneficiary)` 接收一批 `UserOperation`
2. 先进入 validation 阶段，对所有 op 做预校验
3. 全部校验通过后，发 `BeforeExecution`
4. 再逐个执行每个 op
5. 执行结束后统一结算 gas
6. 把累计手续费转给 bundler 指定的 `beneficiary`

关键点：

- **先全量校验，再逐个执行**
- validation 失败会导致整个 bundle 回滚
- 单个 op 执行失败通常不会让整个 bundle 回滚，只会记失败并照样收 gas

## 4. Validation 阶段都校验什么

`EntryPoint._validatePrepayment(...)` 大致做这些事：

1. 拷贝 userOp 静态字段
2. 计算 `userOpHash`
3. 校验 gas 字段范围
4. 计算 `prefund`
5. 调账户 `validateUserOp(...)`
6. 更新 nonce
7. 检查 verification gas limit
8. 如有 paymaster，调 `validatePaymasterUserOp(...)`
9. 记录 `preOpGas` 和 paymaster `context`

面试关键词：

- `userOpHash`
- `prefund`
- `validationData`
- `preOpGas`
- `context`

## 5. Execution 阶段都做什么

执行阶段不是 `EntryPoint` 直接去调目标合约，而是：

1. `EntryPoint` 调账户
2. 账户再调用目标合约

具体实现里：

- `_executeUserOp(...)` 会构造一次对 `innerHandleOp(...)` 的内部 call
- `innerHandleOp(...)` 中再让账户执行 `callData`

这样设计的好处：

- 分离执行上下文
- 更好处理内部 revert 和 OOG
- 更容易结算 `postOp`

## 6. Settlement 阶段怎么讲

`_postExecution(...)` 负责：

1. 计算真实 gasPrice
2. 统计实际 gas
3. 处理未使用 gas penalty
4. 如果有 paymaster 且 `context` 非空，调用 `postOp`
5. 计算 `actualGasCost`
6. 从 prefund 中扣除真实费用
7. 退还剩余 deposit
8. 给 bundler 付款

最常见追问：

### `prefund` 是不是最终收费？

不是。

- `prefund` 是按最坏情况预扣的上限
- 最终按 `actualGasCost` 结算
- 多余部分退回 account 或 paymaster 的 deposit

## 7. `BaseAccount` 怎么讲

`BaseAccount` 的作用是给账户实现提供统一骨架。

它已经固定了 `validateUserOp(...)` 主流程：

1. `_requireFromEntryPoint()`
2. `_validateSignature(...)`
3. `_validateNonce(...)`
4. `_payPrefund(...)`

所以子类最关键的扩展点是：

- `_validateSignature`
- `_validateNonce`
- `_payPrefund`
- `_requireForExecute`

面试表达：

**`BaseAccount` 解决的是 4337 账户的共性接入问题，不负责具体 owner 模型和签名策略。**

## 8. `_payPrefund` 怎么解释

很多人容易误以为 `_payPrefund` 是显式调用了 `depositTo`。

实际上流程是：

1. 账户给 `EntryPoint` 直接转 ETH
2. `EntryPoint` 继承 `StakeManager`
3. `StakeManager.receive()` 自动执行 `depositTo(msg.sender)`
4. 这时 `msg.sender` 是账户地址
5. 所以钱被记成账户在 EntryPoint 的 deposit

最短回答：

**账户不是主动调 `depositTo(address(this))`，而是给 EntryPoint 裸转账，EntryPoint 的 `receive()` 自动记账。**

## 9. `BasePaymaster` 怎么讲

`BasePaymaster` 是 paymaster 的骨架，不是完整实现。

它负责：

- 保存 `entryPoint`
- 限制只有 EntryPoint 可以调 `validatePaymasterUserOp` / `postOp`
- 提供 `deposit` / `withdraw` / `stake` 管理函数

子类必须自己实现：

- `_validatePaymasterUserOp(...)`
- 如返回非空 `context`，还必须实现 `_postOp(...)`

面试表达：

**`BasePaymaster` 把 EntryPoint 对接和权限边界包好了，业务赞助策略留给子类。**

## 10. deposit 和 stake 的区别

这是高频题。

### deposit

- 用于实际支付 `UserOperation` gas
- account 和 paymaster 都可以有

### stake

- 主要用于协议安全要求
- 更偏“保证金”语义
- 常见于 paymaster

一句话：

**deposit 是花的钱，stake 是锁着的保证金。**

## 11. nonce 支不支持并发

支持，但不是默认只有一条顺序流。

`NonceManager` 是：

- `mapping(address => mapping(uint192 => uint256)) nonceSequenceNumber`

所以一个账户可以有很多个 nonce key，每个 key 一条独立 sequence。

因此：

- 协议层支持并发
- 账户实现可以通过 `_validateNonce(...)` 把它限制成单通道顺序模式

面试表达：

**4337 的 nonce 本身支持多 key 并发，但账户实现可以选择不开放。**

## 12. EIP-7702 怎么讲

面试里最简洁的说法：

- `EntryPoint` 通过 `initCode` 是否带 `0x7702` marker 来识别 7702 路径
- 7702 sender 不走普通工厂部署
- `EntryPoint` 会从 `sender.code` 里解析 delegate 地址
- 必要时对 `sender` 自身做一次初始化调用
- 后续 validation / execution 仍然按账户流程继续

一句话：

**7702 模式下，EntryPoint 不是创建账户，而是确认 delegate 并可选初始化。**

## 13. `SimpleAccount` 怎么讲

这是最适合拿来举例的账户实现。

特点：

- 单 owner
- ECDSA 验签
- owner 可以直接调 `execute`
- 支持给 EntryPoint 充值/提现 deposit
- UUPS 可升级

面试表达：

**`SimpleAccount` 是最小单签 4337 账户示例，展示了怎样继承 `BaseAccount` 并把抽象点落地。**

## 14. 高频面试问答

### Q1: `EntryPoint` 为什么要先 validation 再 execution？

因为 bundler 需要先确认整个 bundle 都可执行，避免执行到一半才发现某个 op 根本不合法。

### Q2: 用户的 `callData` 是不是 `EntryPoint` 直接调用目标合约？

不是。  
`EntryPoint` 调的是账户，账户再调用目标合约。

### Q3: execution revert 会不会让整个 bundle 回滚？

通常不会。  
单个 op 可能执行失败，但仍然会走结算并支付 gas。  
validation 失败才更容易导致整个 bundle 回滚。

### Q4: paymaster 付款时，钱从哪里扣？

从 paymaster 在 EntryPoint 里的 deposit 预扣和结算。

### Q5: 为什么 `validateUserOp` 签名失败通常返回 `1`，而不是 revert？

因为 simulation 阶段需要可区分“签名无效”和“系统错误”。  
签名无效是业务上的失败，不一定是异常。

### Q6: 账户合约的 nonce 是不是自己维护？

唯一性更新主要由 EntryPoint 维护。  
账户只负责额外的 nonce 规则检查。

### Q7: paymaster 返回 `context` 有什么意义？

表示后续需要执行 `postOp(...)`。  
如果返回空 bytes，通常表示不需要后处理。

## 15. 最后一分钟复习版

只记下面这些就够应付大部分问题：

- `EntryPoint` = 校验 + 执行 + 结算中心
- `BaseAccount` = 账户骨架，统一 `validateUserOp`
- `BasePaymaster` = paymaster 骨架，统一 EntryPoint-only 权限和资金管理
- `StakeManager` = deposit / stake 账本
- `NonceManager` = 多 key nonce
- `SenderCreator` = sender 部署/初始化中介
- `EIP-7702` = 不部署 sender，只确认 delegate 并可选初始化

主流程只背一遍：

1. bundler 调 `handleOps`
2. EntryPoint 先全量 validation
3. 再逐个 execution
4. 最后统一 settlement
5. bundler 收手续费

最易错三点：

- `prefund` 不是最终收费
- `deposit` 不是 stake
- `EntryPoint` 不直接调目标合约，账户才是执行主体
