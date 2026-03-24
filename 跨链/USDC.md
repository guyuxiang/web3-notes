# USDC

跨链转账协议 (CCTP) 是一种无需许可的链上工具，可实现跨链的原生 USDC 转账。CCTP 会在源区块链上销毁 USDC，并在目标区块链上铸造新的 USDC，从而实现安全的 1:1 转账，无需传统的桥接流动性池或包装代币。

使用 Bridge Kit 简化 CCTP 的跨链转账。[Bridge Kit](https://developers.circle.com/bridge-kit)是一个 SDK，它利用 CCTP 作为其协议提供程序，只需几行代码即可在区块链之间传输 USDC。



CCTP 和 Gateway 提供不同的跨链转账方式。下表对这两种方式进行了比较。

| 属性             | CCTP                                                         | 网关                                                         |
| :--------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **用例**         | 将 USDC 从一个区块链转移到另一个区块链                       | 持有可在任何受支持的区块链上访问的统一 USDC 余额             |
| **传输速度**     | 快速转账：约 8-20 秒； 标准转账：15-19 分钟（以太坊/L2s）    | 余额建立后立即（<500毫秒）                                   |
| **平衡模型**     | 点对点接送                                                   | 统一跨链余额                                                 |
| **保管**         | 非监护                                                       | 非托管式账户，7 天无需信任即可提款                           |
| **支持的区块链** | [查看列表](https://developers.circle.com/cctp/concepts/supported-chains-and-domains) | [查看列表](https://developers.circle.com/gateway/references/supported-blockchains) |



## 原生USDC转账

无需封装代币或流动性池，即可在区块链之间转移原生 USDC。



## 可配置的传输速度

选择[快速传输](https://developers.circle.com/cctp/concepts/finality-and-block-confirmations#fast-transfer-attestation-times) 以获得速度优势，或选择[标准传输](https://developers.circle.com/cctp/concepts/finality-and-block-confirmations#standard-transfer-attestation-times) 以获得成本效益。



## 可编程挂钩

USDC到达目标区块链后，触发目标区块链上的自动化操作。



# 最终性和区块确认

CCTP 的区块确认要求和认证时间

在签署证明文件之前，Circle 会等待区块链交易达到相应的交易最终性级别。所需的最终性级别取决于您使用的是快速转账还是标准转账。

- 快速转账：交易确认并被纳入区块后，通常会在几秒钟内签发证明文件。由于快速转账的最终确认时间更快，因此需要遵守全球限额规定，以降低重组风险。
- 标准转账：在交易最终确定后，即不太可能通过链重组撤销交易时，会签发证明，通常在几分钟内即可完成。



# 转发服务

Circle Forwarding Service 是 CCTP 的一项服务，它简化了集成流程，无需您运行多链基础设施。



## 转移阶段

CCTP转移包括三个阶段：

1. **销毁**：USDC 在源区块链上销毁，并等待 Circle 签署证明。
2. **认证**：向 Circle API 请求认证，Circle的认证服务人员观察焚烧过程并签署证明文件。
3. **铸造**：已签署的证明文件提交至目标区块链以铸造 USDC。