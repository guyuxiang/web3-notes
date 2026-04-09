# Oracle

## 1. 市场

而预言机就是负责连接区块链链上和现实世界链下信息的桥梁，以此来实现区块链和现实世界的数据互通。预言机负责将外部数据安全、准确且可信地引入区块链系统。通过预言机，区块链系统可以在保持确定性操作的基础上，获取和利用外部数据，从而实现更广泛的应用和功能

**最新oracle市场份额：**

Chainlink：2017 年 9 月发布 ，2019 年 5 月正式在以太坊主网推出，经历了 2020 年的 “DeFi Summer” ，2021年 NFT 和 GameFi 生态的爆发之后，Chainlink 通过为市场趋势需求提供相应的功能，成为了整个预言机市场的领先者，并且最先注重开发者关系，资助吸引更多的开发者加入其生态系统。

Chronicle：由 MakerDAO 资助的预言机，专门为 MakerDAO 服务，属于内部预言机，属于makerDao架构中的一部分，获取资产的价格（如 ETH、USDC 等）以支持其去中心化稳定币 DAI 的生成和管理。

Pyth：由 Solana 生态支持，高性能链（如 Solana）上的应用提供精准的市场数据。

WINKLink：由 TRON（波场）开发，为 TRON 和相关生态提供链外数据服务。

![image-20241223135651257](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223135651257.png)



## 2. Chainlink介绍

### **2.1 去中心化**

由许多 Chainlink 节点组成的去中心化预言机网络（DON）：

- DevOps 节点：专门运行区块链基础架构的组织运营的节点。这些节点运营商在运行关键 Web3 基础设施、管理加密私钥以及提供服务换取 cryptocurrency 等方面经验丰富。DevOps 节点包括 Fish、P2P Validator 以及 Staked 等顶尖的质押池提供商。

- 企业节点：由传统Web公司运营的节点，其中包括德国电信子公司 T-Systems 和瑞士电信等国际电信公司，以及 LexisNexis 等全球化机构。

- 社区节点：这些节点来自 Chainlink 社区，其中包括 Chainlink Oracle Olympics 的优胜者、CryptoManufaktur、LinkRiver 以及 NorthWest Nodes。

  

### **2.2 DON**

Chainlink 2.0提出去中心化预言机网络（DON）的概念，DON 是由一组Chainlink 节点负责维护的网络

在经济激励机制上，采用显性和隐性经济激励机制。

**显性激励**

为权益质押，Chainlink节点质押一定数量的LINK通证参与DON。

Chainlink的权益质押与区块链上的权益质押不同，PoS区块链中采用质押机制的目的是对区块中的交易顺序达成全局共识，而Chainlink 2.0中的显性质押机制则是为了生成可靠且防篡改的预言机报告，并确保报告准确反映链下真实事件。

**隐性激励**

指未来收益机会，即只有服务质量高、声誉好的节点才能吸引潜在用户，获得未来收益机会。



**双层预言机网络**

第一层是chainlink网络的参与者，第二层是生态的裁决者

第一层的节点相互监督，在发现大规模异常时可以向第二层报告，由第二层的节点做出判断。双层预言机网络增加了一个在关键时刻起效的仲裁委员会，以部分牺牲去中心化的方式减少了对节点多数贿赂攻击的风险。



**保证金制度**

节点需要缴纳两部分保证金（deposits）：第一部分保证金会因报告与大众不同的数据而罚没（slash），第二部分保证金则会因错误上升到第二层网络裁决（faulty escalation）而罚没

Chainlink提出超线性权益质押，即攻击者需要提供远超节点质押的保证金，才能够有效攻击。有*n*个节点，每个节点都质押金额为*d*的保证金，监督节点将至少可以获得*dn/2*的保证金作为奖励，因此攻击者要给每个节点至少*dn/2的贿赂才能确保它们不上报。贿赂一层DON的总成本至少是*dn²/2。



**链下报告（OCR 2.0）**

采用点对点网络在链下聚合数据，所有节点的数据在链下聚合成一份预言机报告，然后通过一个节点将报告提交到链上

1. 所有节点运营商将其获取的中位数数据以及节点签名生成一个预言机报告（OCR）在DON网络上发布广播，DON网络是轻量级共识网络，每个节点验证其他节点的签名和报告，然后主节点（选举产生）收集其他节点的确认结果，再对所有数据进行一次聚合（如提取中位数）生成单个链下报告广播到所有节点
2. 报告者节点（随机时间表选择）将带有聚合报告的交易提交给链，链上再验证一遍报告签名和法定数量，修改合约存储完成一个更新轮次，如果指定节点未能在确定的时间内确认其传输，则会启动循环，以便其他节点也可以传输最终报告

每轮仅提交一笔交易减少了gas成本并让DON节点更具拓展性，可动态变更参与节点



## **3 Chainlink功能**

### **3.1 Data Feed(喂价)**

Data Feeds 结合[去中心化的数据模型多层聚合](https://docs.chain.link/architecture-overview/architecture-decentralized-model?parent=dataFeeds)和[链下报告OCR协议](https://docs.chain.link/architecture-overview/off-chain-reporting?parent=dataFeeds)，聚合众多数据源并将其发布到链上的数据合约中存储，其他Defi合约可以直接调用相关合约获取最新的数据。

**数据多层聚合流程：**

1. CoinGecko、Coinmarketcap 这种的数据网站对原生数据进行处理清洗
2. Chainlink 的节点运营商会从这些数据聚合器中获取价格数据并进行第二次聚合，通常来说会选取其中位数价格
3. 所有节点运营商上传其获取的中位数数据以及节点签名进行OCR报告上链更新![image-20241223142533472](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223142533472.png)

**链上获取数据流程：**

消费者应用合约 —> 代理合约 —> 聚合器合约

消费者应用合约：可以按需求获取最新数据或历史数据（根据round id获取）。

代理合约：指向聚合器合约地址， Chainlink Data Feeds 不时更新以添加新特性和功能以及响应外部因素，可以升级底层聚合器合约地址，而不会影响消费者合约的服务。

聚合器合约：从预言机网络定期接收数据更新，在链上存储聚合数据，被消费者合约检索

聚合器合约收到OCR报告提交后，会验证签名数组中签名是否由可信预言机节点生成，检查round id，检查节点数量阈值，解码报告中包含的数据并检查是否符合预期格式和范围，验证通过后，合约会更新存储报告的时间戳、round id，以及相关的报告上下文信息

预言机节点触发更新逻辑：

- 偏差阈值（Deviation Threshold）：当节点发现链下值与链上值的偏差超过定义的偏差阈值时，新一轮聚合开始。各个节点监控每个源的一个或多个数据提供者。

- 心跳阈值（Heartbeat Threshold）：自上次更新起经过指定时间后，新一轮聚合开始。

  

chinklink还提供了注册表合约和ENS更方便的获取Feed合约地址

链下也可以通过chainlink提供的web3包读取最新数据



**费用支付**

费用预付：
数据馈送的费用通常由协议开发者、DeFi 项目、基金会或数据提供商支付。这些费用用于支付 Oracle 节点服务、数据提供商费用以及链上交易的 Gas 费用。

用户免费使用：
消费者合约调用 Chainlink Data Feed 时，无需直接支付 LINK，只需通过智能合约的接口查询所需的数据。这是因为费用已经由其他方提前承担。



**喂价数据类型：**

- 价格：各类资产价格数据

  https://data.chain.link/feeds 该网站可查询Data Feed数据信息

  如抵押品价格被AAVE获取，合约价格被Synthetix获取

- SmartData：现实世界资产 (RWA) 数据

  如安全的铸币保证以及储备数据、净资产价值 (NAV) 和资产管理规模 (AUM) 数据等

- 利率、波动率：利率曲线数据、APY 数据和已实现的资产价格波动性等数据

- L2排序器状态：跟踪二层网络上的排序器在某一时间点的最后已知状态



**SmartData：**

储备金：提供稳定币、包装资产和现实世界资产的链下储备状况，通过指定外部 API 获取。

链下储备金使用以下方法提供数据：

- 第三方：审计师、会计师事务所或其他第三方审计并核实储备。这是通过将法定资产和投资资产组合成一个数值。
- 托管人：储备数据直接从银行或托管人处提取。托管人可以直接访问持有资产的银行或金库。一般来说，当提取的标的资产不需要额外估值并且只需在链上报告时，这种方法就有效。
- ⚠️ 自我报告：储备数据是从 API 读取的，代币发行人托管。资产发行人的自托管 API 报告的储备数据具有额外的风险。用户必须自行评估资产发行人风险。

跨链储备：通过查询源链客户端来报告跨链储备，通过在钱包地址管理器合约配置其他链的地址存储跨链储备

NAV：提供有关代币化资产、基金或投资组合的净资产价值 (NAV) 的实时、防篡改数据，通过在链上提供 NAV 数据，应用程序包括资产管理平台、DeFi 协议和依赖 NAV 进行重新平衡、铸造或赎回等操作的投资策略。

AUM：提供实体代表客户管理的资产总市值的当前数据。资产管理规模 (AUM) 是财务分析和决策过程中使用的重要指标。



### 3.2 VRF

一种可证明公平且可验证的随机数生成器，使智能合约能够在不影响安全性或可用性的情况下获取随机数，对于每个请求， Chainlink VRF 生成一个或多个随机值以及如何确定这些值的加密证明

VRF v2.5是目前最新的版本，使用的链下和链上组件：

- [VRF v2.5 协调器合约（链上组件）](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/VRFCoordinatorV2_5.sol) ：消费者应用合约与 VRF协调器合约交互。当被请求随机数时，VRF合约会发出事件，当接受到随机数后验证随机数以及生成随机数的证明，然后回调消费方合约 `fulfillRandomWords`函数。
- VRF 服务（链下组件）：通过订阅 VRF 协调器合约事件日志来监听请求，并根据块哈希和随机数计算随机数。然后，VRF 服务向`VRFCoordinator`合约发送一笔交易，其中包括随机数和生成方式的证明。

请求随机数的方法：

- [直接付费](https://docs.chain.link/vrf/v2-5/overview/direct-funding)：当请求随机值时，消费者合约直接使用本地代币或者LINK支付给随机数合约。必须在消费者合约中存有足够的资金来支付随机数请求，并且仔细估计每个请求的交易费用，以确保消费合约有足够的资金来支付请求的费用

  ![image-20241223173947661](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223173947661.png)

- [订阅付费](https://docs.chain.link/vrf/v2-5/overview/subscription)：创建一个订阅账户，并使用本地代币或者LINK 。然后，可以将多个消费合约注册到订阅帐户。当消费合约请求随机性数，交易成本将在获取成功后计算并相应扣除订阅账户余额。此方法允许从单个订阅中为多个消费者合约的请求提供资金。可以不必精确估计每个请求的成本，只要确保订阅账户有足够的资金，订阅模式还可以减少gas成本。

![image-20241223173024710](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223173024710.png)

消费者应用合约要继承[VRFConsumerBaseV2Plus](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol)并且实现fulfillRandomWords方法，该方法接受预言机合约发送的随机数，并执行自定义的处理逻辑

- 可以获取多个随机值，使用`numWords`参数请求，从单个 VRF 请求中获取多个随机值
- 可以并发请求和处理随机数，在requestId和响应之间创建映射，以跟踪发出每个请求的地址
- 可以编写不同的执行路径来处理 VRF 响应的随机数



应用场景：

- NFT信息
- 随机分配资源，如对奖励或收益进行随机分配
- 公平抽奖
- 竞争者随机选择



### 3.3 Functions

functions允许智能合约调用外部任意 API获取数据，并在去中心化预言机网络 DON 层面上实现了自定义运算，提高安全性和可靠性的同时，实现了高性能、低成本和可扩展性

- 连接到任何公共或私人数据 API（如汇率）
- 获取数据并在对其执行高级计算（如计算收益）
- 连接到企业系统（如企业 ERP 系统（如 SAP）获取信息）
- 连接到物联网设备数据
- 连接到中心化数据库、云服务
- 接入 Al （例如，在 OpenAl 的 ChatGPT APl 或为 DeFi 交易生成建议的云提供商）



脚本开发教程：
https://docs.chain.link/chainlink-functions/tutorials

可以在chainklink提供的平台上编写和模拟：
https://functions.chain.link/playground

脚本案例：

https://usechainlinkfunctions.com/



流程

1. 首先需要编写function js脚本，然后将其部署到 Chainlink 网络
2. 消费者合约通过向[FunctionsRouter合约](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/functions/v1_0_0/FunctionsRouter.sol)发送请求数据
3. DON中的预言机节点监听[FunctionsCoordinator合约](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/functions/v1_1_0/FunctionsCoordinator.sol)发出的事件，根据事件内容独立请求外部API并运行计算
4. 节点使用[OCR](https://docs.chain.link/architecture-overview/off-chain-reporting)协议聚合所有返回的结果，然后将单个聚合响应上链，通过回调函数传递回消费者合约

![image-20241223193543628](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223193543628.png)

订阅管理

类似VRF的订阅模式，创建一个订阅账户，并为它提供资金来支付消费者合约的 Chainlink Functions 请求

托管密钥。

为了启用对需要身份验证凭据的 API 的访问，Chainlink Functions允许用户提供API密钥进行托管，在构建请求是带上密钥

隐私保护。

Chainlink 函数允许用户提供加密的信息，这些密文可以在 DON 执行 Functions 请求时通过称为阈值解密的多方解密过程来解密。这意味着每个节点只能在其他 DON 节点的参与下解密加密的信息。



**实际案例**

Parametric Weather Insurance利用 Chainlink Functions 从气象数据提供商（如 WeatherAPI 或 NOAA）获取实时天气信息。

根据降雨量、温度或风速等数据，触发保险赔付逻辑

ESG Tracking and Transparency从链下的环保数据源（如碳排放监控系统）获取公司或个人的碳排放数据。

根据数据，动态生成碳信用



### 3.4 Automation

是一种去中心化的任务调度服务，专为自动化触发智能合约的特定逻辑而设计，该网络具有冗余性，即使某些节点宕机，仍将执行任务。

如果满足一组特定条件，去中心化的预言机节点网络达成共识调用一个智能合约函数

- **基于时间（Time-based）的触发器**：使用基于时间的触发器根据时间表执行您的函数。
- **日志（Log）触发器**：使用日志数据作为触发器和输入。
- **自定义逻辑（Custom logic）触发器**：提供自定义的 Solidity 逻辑，让自动化节点（链下）判断以决定何时在链上执行函数。

![image-20241223213352454](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223213352454.png)



**应用案例**

- Aave当用户的抵押率低于安全水平（健康因子 < 1）时，自动触发清算逻辑

- Synthetix定时从奖励池中发放质押奖励给 SNX 持有者

- Uniswap V3自动监控池的交易量和波动率调整手续费率

- Compound定时分析借贷市场的供需，自动更新借贷利率参数，以维持市场平衡

- PancakeSwap自动检测流动性池的权重失衡，触发再平衡逻辑，优化流动性利用率。

- Yearn Finance根据收益情况自动将用户存款池中的收益重新分配

- NFTfi 是一个 NFT 借贷平台，实现贷款到期的自动化处理，触发清算或资产回收逻辑

  

### 3.5 Data Streams

Data Streams结合了Automation和Functions的功能，提供了一种新的交易模式-流交易

“用户先在链上提交交易意图，随后由 Automation 基于事件去取数/取报告，再把报告和最终执行原子地放进第二笔链上交易里完成。”

流交易架构 - “commit-and-reveal”

执行一笔需要用户在链上和链下同时协作的交易，提取相关链下数据后将其与交易执行打包在一起

1. 用户通过`initiateTrade`交易来发起初始交易
2. 链上合约做基本交易处理后发出日志触发事件
3. Automation监控合约活动，检测到事件后，处理链下API请求和数据计算，生成报告
4. 根据链上计算或链下计算出现不同结果做出不同交易操作
5. 预言机节点回调合约相关方法上链最终完成一笔交易

![image-20241223220430794](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241223220430794.png)

**案例**

- GMX目前是Arbitrum和Avalanche上TVL（总锁仓量）最大的去中心化永续合约交易平台。GMX一开始是自己运行和管理内部的价格预言机以及自动执行基础设施。然而，由于需要同时兼顾数据质量、规模和安全性，因此运营变得非常复杂，而且对成本和资源的消耗也很大。此外，这样做还带来了额外的信任假设，GMX最终决定通过去中心化来解决这一问题。GMX社区就立刻投票升级了预言机基础设施。GMX V2目前已集成了Chainlink Data Streams来访问高频市场数据并自动结算交易。具体而言，GMX采用Chainlink Data Streams来结算交易并触发止损和清算功能。
- JOJO Exchange推出了创新的智能合约订单，允许用户发起链上限价单，这些订单在链下进行匹配，然后在链上结算，提高了资本效率和交易灵活性。
- Vertex现在使用Chainlink Data Streams来执行清算未足抵押头寸和计算资金费率。



## 4. Chainlink RWA案例

哥伦比亚银行集团旗下Wenia集成Chainlink PoR，用来确保其哥伦比亚比索稳定币COPW的铸造功能。

https://mp.weixin.qq.com/s/JPEtkHIFZa3WJey2J07gtw

巴西中央银行 (BCB) 现已选择Chainlink 预言机来实现供应链管理自动化并改善贸易融资流程，引入电子提单(eBOL)代币化，自动触发对出口商的付款、结算(DvP)与款对款同步收付(PvP)等机制
https://mp.weixin.qq.com/s/N19s1YNOKMOI2VOxuS8_yg

Swift、瑞银、Chainlink 试点代币化基金结算
https://blog.chain.link/chainlink-project-guardian/#sbi_digital_markets__ubs_asset_management__and_chainlink_unlock_automated_fund_administration_and_transfer_agency

TrueUSD成为首个借助“储备金证明”来保障Minting安全的美元抵押Stablecoin
https://mp.weixin.qq.com/s/8RoS44wEsjRZs0Tym36tvw

Sygnum携手富达国际与Chainlink达成合作，向链上传输基金净值数据
https://mp.weixin.qq.com/s/U1xixfKeSsrxCqTGAi-vJw

圆币科技将采用Chainlink的储备金证明 (PoR) 功能，为HKDR的储备提供可靠的链上验证
https://mp.weixin.qq.com/s/qXRQR78310HKMDfatVn_5Q?poc_token=HINUamejVLtL-Wrmk2H6aiP9HMvmr5zeSEzAjZXY

美国证券存托清算公司DTCC Smart NAV试点报告：将可信数据引入区块链生态
https://mp.weixin.qq.com/s/N0OxWlIrB-adnp31vPddRQ

Superstate集成Chainlink基础设施，提升USTB通证化基金的透明性和功能性
https://mp.weixin.qq.com/s/0S7zgNMvfmSzWZ6qPMp64Q

PropyKeys的房地产代币化在Base网络上整合了Chainlink的自动化服务（Chainlink Automation），用于分发质押奖励

https://www.chaincatcher.com/article/2140822

IDA与Chainlink携手 提升其由港币1:1抵押的stablecoin HKDA的透明性

https://mp.weixin.qq.com/s/OyzODO_dZ5Zqc9j-68r9Xg

沃达丰DAB与Chainlink Labs合作将预言机集成到其国际贸易平台Economy of Things，验证国际贸易文件等

https://foresightnews.pro/article/detail/53300

Swift、瑞银资产管理和Chainlink创新试点项目将通证资产接入传统支付系统

https://mp.weixin.qq.com/s/_pVA6oAgreo6NG75YhQYwQ

21Shares集成Chainlink储备金证明，以提升ARK 21Shares Bitcoin ETF ARKB的透明性
https://mp.weixin.qq.com/s/abwPWVTUyHGyRuILRDRbfA

澳新银行、ADDX和Chainlink，在新加坡金融管理局的守护者计划 (Project Guardian) 中用例，聚焦通证化的商业票据在跨境交易中的资产生命全周期

https://mp.weixin.qq.com/s/7XsLRWN6FIWd2yE90EPShA

Matrixdock集成Chainlink储备金证明 (PoR) ，以提升其短期国库券通证的链上透明度
https://mp.weixin.qq.com/s/VRGe9d51z69x_89SOU_-Ww

Backed Finance是一个专注于将传统金融资产代币化的平台，通过整合chainlink部分代币化RWA喂价，提升代币化资产的透明度，确保资产价格的准确性

https://www.theblockbeats.info/flash/155004

SOOHO.IO和Chainlink宣布达成战略合作，共同在亚洲探索通证资产和央行数字货币用例
https://mp.weixin.qq.com/s/kaJj0EalQLCYfZA7EdkQqg

Synthetix 是一个链上衍生品协议跟踪各种资产，如加密货币、大宗商品、外汇、黄金、石油、法币、股票指数等，使用Chainlink预言机来确保其合成资产的价格始终准确反映标的资产的实际市场价值

https://chain.link/education-hub/synthetic-derivatives



## 5. 集成思路

### 5.1 储备金证明（PoR）

DTT稳定币发行使用PoR的优势：

- **提高安全性——**在智能合约层面，安全铸造为Token发行者增加了一层额外的安全保障。通过在通证智能合约中对总供应量实施PoR 检查，如果通证的总供应量和新铸造金额的总和高于PoR 报告的储备金额，通证的智能合约可以自动撤销铸造交易。

- **增强透明度——**安全铸造通过提供资产链下/跨链抵押的可靠数据来源，为通证持有者提供更高的生态透明度和信心，并由Chainlink实时在链上报告，可以开发Dashboard页面供客户查看。

- **降低生态风险——**如果检测到链下储备金抵押不足，oracle可以暂停链上合约操作，为协议创建熔断机制，以防止产生进一步坏账。



​						**Wenia集成Chainlink PoR来确保COPW稳定币铸造的示例**

![image-20241224161210869](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241224161210869.png)

1. **Wenia账户数据访问**：
   Wenia（属于 Bancolombia Group 的公司）的信托账户中存有哥伦比亚比索（COP）余额。

2. **H&T获取账户余额**：
   H&T资产管理公司会定期调取Wenia银行账户的哥伦比亚比索（COP）储备数据。

3. **Chainlink查询H&T API**：
   Chainlink网络会通过其去中心化的预言机节点，查询H&T的API来获取最新的COP储备余额信息。

4. **Harris & Trotter 返回储备金余额**：
   Chainlink 节点从 API 获取到储备金数据，并将其传递回 Chainlink 网络。

5. **Chainlink 更新储备余额到区块链**：
   当储备金余额发生偏差（例如账户资金变动）时，Chainlink 会触发链上交易，将最新的储备金数据发布到区块链上的 PoR 智能合约中。

6. **COPW智能合约代币发行**：
   发行铸造COPW稳定币是，智能合约通过dataFeed获取并验证储备数据，确保COPW供应量与实际储备数据相符。

7. **铸币逻辑控制**：

   7a：当COPW的供应量大于或等于储备时，智能合约会阻止新的COPW铸造，以避免超额供应。

   7b：当COPW的供应量小于储备时，智能合约允许铸造COPW，但仅限于未利用的储备金额。



Wenia使用的chainlink的data feed来更新储备金，也可以使用data streams的流交易模式来实时检测储备金。

1. 当发行方发起mint铸造请求交易

2. Chainlink Automation监听到事件自动使用Functions查询账户储备金信息

3. Automation将储备金数据上链继续执行铸造流程，智能合约根据数据决定是否完成铸造。

   

### 5.2 实物NFT

ABT项目中加入资产池的挂钩实物资产的NFT，可以通过oracle获取链下存在性证明和资产信息提高安全性和透明性。

Chainlink Functions将链下资产文档、企业系统、各种 IoT设备验证货物交付和状况、物流系统的数据接入智能合约，以安全、可靠且信任最小化的方式将相关元数据更新到资产类NFT中

对NFT的管理如铸造、转移、销毁操作使用data streams的流交易模式实时验证确保资产的安全性。

可以结合[ERC-6956](https://eips.ethereum.org/EIPS/eip-6956)协议，对于NFT 底层实物资产铸造、转让、销毁、批准等业务场景，通过ORACLE证明对资产控制权![image-20241224171630201](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241224171630201.png)



### 5.2 NAV、AUM计算

ABT项目中需要计算NAV，通过Oracle使所有上链的NAV、AUM数据的来源和计算方法都可以公开验证

1. 根据需求在chainlink设置数据更新的频率（实时、每小时、每日）
2. 多个Chainlink节点使用Functions通过API获取资产价格、外汇汇率等数据，节点会根据资产价格、负债、汇率等数据计算出NAV
3. 计算完成后进行聚合，最后节点通过智能合约将NAV数据发布到区块链上

​                                         		**美国证券存托清算公司DTCC集成Chainlink DataFeed完成NAV更新示例**

<img src="C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241224173127687.png" alt="image-20241224173127687" style="zoom:150%;" />

​						

​							**Sygnum集成Chainlink从富达国际获取数据并计算基金净值数据示例**

![image-20241224174333373](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241224174333373.png)



### 5.3 关键链上数据接入

合约通过Oracle接入企业/传统系统安全透明地更新关键链上数据

在Chainlink Functions配置ESG数据、审计师证明、所有权，定价等信息的API

根据需要触发chainlink节点获取聚合后上链进行发布和更新



### 5.4 交易和结算

通过配置Chainlink Automation定时触发，保障去中心化，避免人为干预并且避免单点故障。

- DTT项目中可由Oracle触发条件支付的自动化结算
- ABT项目中可由Oracle触发投资收益的自动分配



使用chainlink data streams在交易和结算过程中使用oracle获取数据或验证信息.

- 可使用chainlink预言机从多个可信外汇数据源（如国际银行或市场数据提供商）中获取实时外汇汇率信息
- 使用链下法币支付场景，可使用chainlink从银行或支付网关验证付款是否完成
- 通过chainlink获取链下监管系统的 KYC 和 AML 数据来决定交易是否执行



### 5.5 隐私、身份验证、KYC/AML、资金证明

[DECO](http://mp.weixin.qq.com/s?__biz=MzU0MTgyMDQwNQ==&mid=2247498667&idx=1&sn=c9ae3247667a25b894010029b8979b94&chksm=fb26af31cc5126276b60f74839b8c085817a300c2a9e2887f9e2332b5119e40c09014ee31217&scene=21#wechat_redirect)是一个利用零知识证明技术的隐私保护去中心化预言机协议，它利用零知识技术使机构和个人链下私有数据，在隐私保护的前提下被链上智能合约调用。从而不在公链上泄露用户的敏感信息，但已经能利用智能合约保障交易的安全。

![image-20241224211606300](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20241224211606300.png)

**Chainlink DECO身份验证工作流程：金融机构使用隐私保护技术验证客户是否符合要求，进行KYC/AML**

1. 金融机构的传统应用程序将其API凭证和客户的身份查询提交给由金融机构操作的DECO Prover。
2. DECO Prover与身份数据提供者交互，以查询关于客户的私人数据。
3. DECO Prover将关于客户的特定声明的正确性和溯源性证明提供给运行DECO Verifier的Chainlink Oracles节点。
4. Chainlink Oracles提供时间戳证明，并将其提交到链上的智能合约（例如，某个机构借贷协议）。
5. 智能合约基于零知识证明验证客户的声明，确保整个流程既安全又保护隐私。



**机构或客户资金证明：保护隐私的流动性证明而不会在链上透露确切余额**

1. 机构或客户向DECO Prover提交API凭证和资金证明查询申请，DECO Prover从金融机构的内部系统及其银行（例如，金融机构为银行或持有其他银行流动性头寸的基金）中检索汇总余额数据。
2. DECO Prover查询、检索并处理财务数据，生成一个关于账户余额总和的零知识证明（ZKP），同时不暴露具体的账户细节。
3. 通过Chainlink Oracles，DECO Verifier验证该证明，确保金融机构满足所需的财务条件，同时不泄露隐私。
4. 验证通过的证明被上传至链上，供去中心化金融应用程序使用，从而在保持合规性的同时隐私得参与DeFi生态。



**企业对于交易数据或库存水平的证明：保护商业信息的隐私**

【2024年11月6日，新加坡】——ADDX携手澳新银行和Chainlink，在新加坡金融管理局的守护者计划 (Project Guardian) 中宣布推出一个新的用例，聚焦通证化的商业票据在跨境交易中的资产生命全周期。采用隐私交易功能（Private Transactions）阻止第三方获取金融机构交易中的隐私数据，具体包括金额、对手方信息、买卖价格以及结算指令。



todo: 对比，随机数

1.26需求

L2隐私，跨链案例解析





## ERC‑7412

ERC‑7412（全称 “On-Demand Off-Chain Data Retrieval”）是一项为 Ethereum 生态中的智能合约设计的提案（EIP／ERC），其目标是**在合约执行期间按需获取可信的链外数据**。

智能合约在执行过程中，有时需要读取链外数据（如价格、跨链状态等）。ERC-7412 提出一种标准化机制：当合约发现所需链外数据尚未提供时，可以触发一个预定义的错误（`OracleDataRequired(address oracleContract, bytes oracleQuery, uint256 feeRequired)`）提示客户端“请先获取数据”。 [Ethereum Improvement Proposals+1](https://eips.ethereum.org/EIPS/eip-7412?utm_source=chatgpt.com)

客户端（钱包／应用）在捕获这个错误后，会从指定的 oracle 合约中检索签名好的链外数据，并使用一个“多调用（multicall）”事务先提交数据核验，再执行原本合约调用。这样既能按需拉取数据，又避免合约不停推送／存储全部数据。 [Ethereum Improvement Proposals+1](https://eips.ethereum.org/EIPS/eip-7412?utm_source=chatgpt.com)

合约支持收费模型（`FeeRequired(uint amount)`）—— oracle 可以要求在调用 `fulfillOracleQuery(bytes signedOffchainData)` 时收取原生代币作为费用。 [Ethereum Improvement Proposals](https://eips.ethereum.org/EIPS/eip-7412?utm_source=chatgpt.com)