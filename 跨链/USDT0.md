# USDT0

## Legacy Mesh（USDT0 特有）

Legacy Mesh 是一个跨链 USDT 流动性网络，连接已部署 USDT 的网络：以太坊、Arbitrum、Celo、Tron 和 TON 上的传统部署。

它支持 USDT 跨链转移，无需铸造或销毁。相反，它采用基于信用的系统，在每条链的智能合约池之间锁定和解锁流动性。



因为USDT原先已经在多条链上部署，且不是OFT标准的代币，合约也不可升级，因此为了保留了现有的 USDT 实现方式，同时实现了彼此之间以及与所有支持 USDT0 的链之间的无缝连接，因此在传统链和Arbitrum之间选择“锁定 / 铸造（Lock & Mint）”模式，在新链使用“铸造/销毁”模式



### 多跳路由[](https://docs.usdt0.to/overview/the-legacy-mesh#multihop-routing)

架构上为了减少多链复杂度，选择了中心辐射架构，以 Arbitrum 为中心枢纽

通过传统网状网络进行的数据传输可以使用以 Arbitrum 为中心枢纽的两跳路由机制：

1. **第 1 跳：传统链 → Arbitrum Hub** 您的 USDT 从源链（例如，Tron、TON、Solana）转移到 Arbitrum，在那里统一转换为 USDT0。
2. **第二跳：Arbitrum  → 目标链** 然后，USDT0 将使用 OFT 标准的原生铸造/销毁机制从Arbitrum发送到任何连接的 USDT0 网络。



这种中心辐射式设计允许用户通过一次用户操作，将资金从任何旧版 USDT 链转移到任何 USDT0 链，而底层基础设施则通过 Arbitrum 进行路由。





### 跨链转账

**https://usdt0.to/transfer**



### 分析面板

https://analytics.usdt0.to/
https://layerzeroscan.com/oft/USDT0/USDT0

https://dune.com/usdt0/usdt0-metrics-dashboard





### 组件交互

该系统支持 USDT0 代币在以太坊 A 链和 B 链之间进行转移：

**以太坊**→ **Chain B**：

- **USDT0 适配器**将代币锁定在以太坊上
- LayerZero消息触发Chain B上的**USDT0 OFT ，以铸造等值的代币。**

**链 B** →**链 A**：

- **USDT0 OFT**在 Chain B 上销毁代币
- 一条消息触发A链上的**USDT0 OFT，**以铸造等值的USDT0。

**链 A** →**以太坊**：

- **USDT0 OFT**在 Chain A 上销毁代币
- 一条消息指示**USDT0 适配器**在以太坊上解锁代币。



![image-20260217221905852](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260217221905852.png)





所有 USDT0 代币都实现了以下标准接口：

- ERC20
- ERC20许可证（EIP-2612）
- EIP-3009（无气体传输）



### 组件

1. OAdapterUpgradeable

   （以太坊上）：

   1. 为以太坊实现 LayerZero OFT 功能
   2. 处理跨链消息的发送和接收
   3. 直接与以太坊上的源代币合约对接（USDT0/CNHt0 使用 TetherToken，XAUt0 使用 XAUt）
   4. **锁定/解锁**用于跨链转账的代币
   5. **在以太坊上，用户需要先授权 OFT 适配器才能花费他们的代币，然后再调用`send`.**

2. 可升级

   （在其他链上）：

   1. 为其他链实现 LayerZero OFT 功能
   2. 处理跨链消息的发送和接收
   3. 与 TetherTokenOFTExtension（或等效项）的接口
   4. 控制跨链转移的**铸造/销毁**

3. TetherTokenOFTExtension

   （在其他链上）：

   1. 为 OFT 提供铸币/烧币接口







##  安全配置（DVN）[](https://docs.usdt0.to/technical-documentation/developer/#3-security-configuration-dvns)

USDT0 代币采用双 DVN 安全配置，需要以下验证：

- LayerZero DVN
- USDT0 DVN（用于所有 USDT0 产品）

两个分布式验证网络（DVN）都必须验证有效载荷哈希值，跨链消息才能提交执行。这种设置通过对所有跨链转账进行独立验证来确保更高的安全性。

有关 LayerZero DVN 和安全堆栈的详细信息，请参阅：https://docs.layerzero.network/v2/home/modular-security/security-stack-dvns