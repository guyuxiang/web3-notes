## Tenderly功能介绍

### **Tenderly节点**服务

和其他的节点服务类似, 都提供了标准的以太坊RPC服务, 特色功能有:

#### **Tenderly 的 RPC 方法**

Tenderly 节点除了提供标准的RPC方法外还有些特有的RPC方法

**1.交易模拟**

- [`tenderly_simulateTransaction`](https://docs.tenderly.co/node/rpc-reference/ethereum-mainnet/tenderly_simulateTransaction)在交易真正发送之前模拟交易的结果。可以模拟在最新的区块或者给定区块上执行的交易。

  单笔模拟:

  Tenderly 支持通过RPC和API进行单笔交易模拟。模拟在所选网络的最新状态下执行。

  https://docs.tenderly.co/simulations/single-simulations

- [`tenderly_simulateBundle`](https://docs.tenderly.co/node/rpc-reference/ethereum-mainnet/tenderly_simulateBundle)模拟在给定块上执行的一组交易并返回每个交易的结果。

  多笔捆绑模拟:

  Tenderly 支持通过RPC和API进行连续模拟多个交易的执行, 这些交易在同一个区块内逐个进行模拟。

  https://docs.tenderly.co/simulations/bundled-simulations



另外使用API中的`simulation_type`参数允许控制模拟结果中返回的细节级别。

- `full`（默认）：详细的原始和解码信息，包括调用跟踪、函数输入和输出、状态差异、日志等。
- `quick`：快速模拟, 提供原始数据，无需额外解码
- `abi`：包含解码的输入、输出和日志的最少信息



交易模拟对开发者好处:

- 在链上指定状态下模拟失败的交易以找出Bug的原因并更深入地了解其执行情况

- 可在链上的指定的数据状态下进行调试而不上链改变状态和增加调试记录

- 测试并验证智能合约错误修复或升级，以确保开发和生产过程中的代码正确性

- 模拟待处理交易的预期结果并优化所有交易的 gas 使用量




交易模拟对对用户好处:

使用模拟让用户在投入实际资产之前试运行交易。这样就可以让用户预览交易执行情况以及发生的详细余额和资产变化。

交易预览可以帮助用户建立对 Dapp 的信任，只批准成功的交易，并防止代价高昂的失败。

https://docs.tenderly.co/simulations/transaction-preview

模拟结果包括

- 显示资产和余额变化

- 提供准确的Gas使用量估算

- 告诉用户他们的交易是否失败并帮助他们节省 gas。

- 提供具有可读性的错误, 帮助用户理解为什么他们的交易失败。

- 模拟修改区块链条件，如时间戳和合约数据，以测试不同的场景。

- 向用户提供交易将访问的地址和存储槽。

- 以解码形式获取发出的事件、日志名称和相关参数。

  



**2.交易跟踪**

- [`tenderly_traceTransaction`](https://docs.tenderly.co/node/rpc-reference/ethereum-mainnet/tenderly_traceTransaction)获取指定交易的有关执行的信息，如状态、日志、内部交易等。



**3.气费估计**

以太坊网络上对 gas 估算的标准实现通常采用二分搜索算法, 通过逐步缩小可能的 gas 限制范围, 找到正确 gas 估计, 但通常需要大约 25 次交易执行才能估算出最佳的 gas 限制。对于一些复杂的函数, 估算可能会不准确且耗时, Tenderly设计的独特的算法将计算复杂度从 O(logN) 显著降低到 O(1)，从而显著提高了 gas 估算效率。

- [`tenderly_estimateGas`](https://docs.tenderly.co/node/rpc-reference/ethereum-mainnet/tenderly_estimateGas)计算100% 精确的gas估计值。

- [`tenderly_estimateGasBundle`](https://docs.tenderly.co/node/rpc-reference/ethereum-mainnet/tenderly_estimateGasBundle)计算单请求多交易(事务)100% 精确的gas估计值。

  



#### **节点拓展-用户自定义RPC 方法**

- 可以使用 JavaScript 或 TypeScript 编写自定义脚本,  来定制RPC 方法扩展 Tenderly Node 的功能
- 可以使用现有的Node 扩展库或者基于此进行定制编写自定义脚本

作为节点扩展构建的 RPC 方法具有**`extension_`**前缀。可以按照正常json rpc的方式去调用



#### **请求批处理**

JSON RPC 请求批处理允许您在单个 HTTPS 调用中发送多个 JSON-RPC 方法调用。批处理请求由一组单独的 JSON-RPC 请求组成。批处理 JSON-RPC 请求可减少执行多个 HTTP 请求的网络延迟，从而提高性能。

https://docs.tenderly.co/node/guides/request-batching

- **直接**通过发送包含单独 JSON RPC 调用数组的 HTTPS 请求，

- **[使用 Ethers](https://docs.tenderly.co/node/integrations-chain-interaction/ethers)**，通过实例化**`JsonRpcBatchProvider`**



#### 免费版使用限制(对不同RPC方法限制不同): 

读: 625万次/月/用户

- `eth_call`
- `eth_estimateGas`
- `eth_getFilterChanges`
- `eth_getFilterLogs`
- `eth_getLogs`
- `getProof`

或者

写: 125万次/月/用户

- `eth_sendRawTransaction`

或者

其余RPC: 2500万/月/用户



请求速率限制: 

不固定, 对不同RPC限制也不同, 大概每秒2~10次



### Tenderly开发调试

#### 项目管理

在Tenderly 中创建项目, 并将项目相关的合约地址添加到项目中, 可以实现对项目的仪表板管理

- 方便查看项目各个合约的源代码、ABI、字节码等信息
- 方便查看项目各个合约交易历史并支持使用过滤器排序和查找对您最重要的交易，这对开发和监控都很有帮助。
- 方便查看项目各个合约交易的模拟交易历史

另外还可以将某些重要的钱包账户地址添加到项目中

可以在仪表板中查看与钱包相关的所有交易、模拟交易或为钱包创建报警



组织管理

项目可以设置组织成员, 并且可以管理成员的权限方便一起处理项目





#### 虚拟测试网

Tenderly 提供了通过UI快速创建一个模拟测试网的功能, 并且可以Fork指定的主网或测试网, 还可以选择完全同步跟踪真实网络状态

TestNet创建完后会提供URL, 可以像在实时网络上一样使用，以毫秒为单位处理交易, 可以作为项目的临时环境, 

其JSON-RPC 接口和真正的区块链节点RPC一致

提供一个无限水龙头，可以使用它为任何帐户注入或设置任意数量的原生代币和 ERC-20 代币

提供了虚拟测试网的区块链浏览器, 并且可以指定为私有还是公共

可以继续对虚拟测试网进行Fork, 进行一些临时性的调试

支持回滚到指定快照位置



#### 合约验证

智能合约验证对于 Tenderly 的开发工具正常运行至关重要。

- **解码交易**：通过解码的调用跟踪、事件、状态变化和 gas 使用情况深入了解交易。
- **调试**：使用解码的信息进行有效的调试和交易分析，从而更容易识别和修复问题。
- **优化gas**：获取精确的gas使用情况分析来识别和实施优化，降低成本并提高性能。
- **增强协作**：安全地与团队成员和审计员共享已验证的合同，以促进协作并加快审计流程。

Tenderly 有提供hardhat插件来提供向Tenderly验证的流程, 并且支持可升级合约

[`@tenderly/hardhat-tenderly`](https://www.npmjs.com/package/@tenderly/hardhat-tenderly/v/2.2.1)

https://docs.tenderly.co/contract-verification/hardhat

![image-20240620194814947](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240620194814947.png)



#### 调试

Tenderly 提供了对交易debug的功能

可以查看交易生命周期内进行的每个调用和顺序

可以查看指定一个调用的堆栈

可以解码指定调用详细信息

- 调用函数的名称
- 操作码
- 合约和调用者的地址，以及调用者的余额
- 解码的输入和输出
- 天然气使用情况统计，显示使用的天然气总量、当前功能消耗的天然气以及剩余天然气

可以显示正在交互的合约的代码

使用“Next/Previous”按钮可进行跳转

评估功能可以直接在调试器中测试自定义的变量计算表达式。这非常适合检查全局变量、合约变量、函数参数和局部变量**。**

可以在调试页面将现有交易加载到交易模拟器中，可以调整输入进行模拟交易执行进行错误修复和故障排除

![image-20240620194750606](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240620194750606.png)



#### Gas分析

Gas Profiler 为您提供 Gas 消耗明细，类似火焰图的形式展示合约中每个函数在执行过程中如何使用 Gas

![image-20240620194717709](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240620194717709.png)



#### 监控警报

Tenderly可以监听区块链上的事件，并在事件发生时将实时通知发送到指定的目的地

这里的事件包含:

在所选择网络上一个或多个地址的行为

- 交易成功, 用于监控调用特定智能合约的交易或特定钱包发送或接收的交易。
- 交易失败, 用于及早发现钱包、智能合约和 dapp 的使用问题。
- 某个指定的函数调用
- 某个指定事件发出
- 出现指定事件参数
- ERC20 代币转移, 比如从一个地址到另一个地址的代币转移时
- 货币余额达到阈值, 可以了解何时需要充值
- 交易价值达到阈值
- 合约某状态变量发送变化
- 白名单外调用者或黑名单调用者



目的地包含:

- 电子邮件
- Telegram
- Discord
- 外部webhook
- Web3 Actions
- 第三方事件和错误监控平台如Sentry, PagerDuty



- **监控链上活动：**警报可用作监控工具，用于跟踪智能合约上的用户活动，例如失败或成功的交易或状态变化。当您需要跟踪区块链环境中的特定事件或变化时，这非常有用。
- **检测可疑行为：**警报可以告知您智能合约或钱包中的可疑行为，例如安全漏洞或欺诈活动。这可以帮助您采取适当的措施来保护您的资产并防止损失。
- **在紧急情况下做出更快的响应：**警报还可以帮助您检测错误、安全问题或失败的交易。当您需要在问题发生后立即采取行动解决问题时，这非常有用。

- **启用自动化流程：**使用警报作为 Web3 Actions的触发器来构建自定义自动化流程，使您能够使用自定义代码自动对链上事件做出反应。



#### Web3 Actions

Web3 Actions 是智能合约和链事件的可编程hook, 运行自定义的TypeScript/JavaScript脚本, 实现一些自动化的操作

, 还带有键值存储，用于存储需要保留的数据



可以和监控报警一起使用, 监控事件触发后, 执行Web3 Actions指定的JavaScript/TypeScript 代码, 立即处理该事件。

自定义 JavaScript 或 TypeScript 代码。可以通过 Tenderly Dashboard直接编写或者通过Tenderly CLI上传



案例:

**快速反应**

当智能合约出现紧急情况如被攻击时, Web3 Actions触发, 自动暂停合约相关功能

**构建自定义预言机**

每当智能合约通过链上事件发出数据请求时, Web3 Actions触发, 脚本获取到相关链下数据, 将数据推到链上



#### 图表分析

可以创建自定义图表深入了解智能合约的使用情况和相关的链上数据

- 可视化并分析智能合约的行为以发现模式并深入了解交易数据。
- 跟踪对您的项目很重要的指标和趋势，并确定具体的使用趋势。
- 使用自定义查询来请求复杂的链上数据，甚至通过 API 集成将其公开给您的用户。