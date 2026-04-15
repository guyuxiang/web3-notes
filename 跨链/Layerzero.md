# Layerzero

### 协议架构

LayerZero 是一种全链互操作性协议，它为跨链消息传递提供了一个稳定、不可变的接口。

LayerZero 通过将接口、验证和执行分离到独立的层中，实现了可组合的跨链架构，同时又不损害安全性。

**五个独立层**

![image-20260212144339104](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260212144339104.png)

1. #### **业务逻辑接口（OApp）**

**全链应用程序 (OApp)**是 LayerZero 对使用 LayerZero 发送和接收跨链消息的智能合约的定义。

OApp 标准提供了一个一致的应用程序接口，用于调用 LayerZero 协议并传递此自定义业务逻辑：

OApp标准作为一个门面层，将原始的LayerZero协议接口封装为开发者友好的方法。

开发者无需直接调用endpoint.send()和endpoint.lzReceive()，而是使用_lzSend()和_lzReceive()方法，这些方法处理了常见模式，如手续费估算、消息验证和错误处理。
您的应用程序代码专注于业务逻辑（如自定义数据的编码/解码、状态转换等），而 OApp 包装层负责管理协议交互。

提供了跨链费用报价、发送、接受的方法框架

```solidity
// SPDX-License-Identifier: MIT
import {OApp, MessagingFee, Origin} from "@layerzerolabs/oapp-evm/contracts/oapp/OApp.sol";
import {OptionsBuilder} from "@layerzerolabs/oapp-evm/contracts/oapp/libs/OptionsBuilder.sol";

contract MyOmnichainApp is OApp {
    using OptionsBuilder for bytes;

    constructor(address endpoint) OApp(endpoint, msg.sender) {}

    function quoteMessage(uint32 dstEid, bytes memory message, bool payInLzToken)
        external view returns (MessagingFee memory fee) {
        bytes memory options = OptionsBuilder.newOptions()
            .addExecutorLzReceiveOption(200_000, 0);

        // Returns the cost of sending a message in either native gas token or ZRO
        return _quote(dstEid, message, options, payInLzToken);
    }

    function sendMessage(uint32 dstEid, bytes memory message) external payable {
        // Unordered by default; requests destination chain gas units for message delivery
        bytes memory options = OptionsBuilder.newOptions()
            .addExecutorLzReceiveOption(200_000, 0);

        // If you need ordering:
        // options = options.addExecutorOrderedExecutionOption();
        _lzSend(
            dstEid,
            // highlight-next-line
            message,
            combineOptions(dstEid, SEND, options),
            MessagingFee(msg.value, 0),
            payable(msg.sender)
        );
    }

    function _lzReceive(
        Origin calldata origin,
        bytes32 guid,
        // highlight-next-line
        bytes calldata message,
        address /*executor*/,
        bytes calldata /*extraData*/
    ) internal override {
        // your logic
    }
}
```







2. #### **协议接口（Endpoint）**

LayerZero Endpoint是区块链上所有跨链消息传递的**唯一入口和出口**。每条链都有一个 LayerZero  Endpoint合约，该合约可以与任何受支持链上的任何其他 LayerZero Endpoint合约之间发送和接收消息。

- **通用**：所有受支持的链上都使用相同的接口

- **不可更改**：无法升级或更改,LayerZero 的不可变性确保了接口永不改变，从而为基于 LayerZero 构建的应用程序提供永久兼容性。
- **无需许可**：任何人都可以调用（需支付相应费用）

端点提供通用的、不可变的协议接口：

```solidity
// Excerpt of ILayerZeroEndpointV2 - core messaging functions

// Core data structures
struct MessagingParams {
    uint32 dstEid;           // Destination endpoint ID
    bytes32 receiver;        // Receiver address (bytes32 for crosschain compatibility)
    bytes message;           // Message containing application-specific business logic
    bytes options;           // Execution options (gas, native drops, etc.)
    bool payInLzToken;       // Payment method (native chain token vs LZ token)
}

struct MessagingReceipt {
    bytes32 guid;            // Globally unique identifier for tracking
    uint64 nonce;            // Message sequence number for ordering
    MessagingFee fee;        // Actual fee charged
}

struct MessagingFee {
    uint256 nativeFee;       // Fee in native gas token
    uint256 lzTokenFee;      // Fee in LZ token (alternative payment)
}

struct Origin {
    uint32 srcEid;           // Source endpoint ID
    bytes32 sender;          // Sender address (bytes32 for crosschain)
    uint64 nonce;            // Message nonce for ordering
}

interface ILayerZeroEndpointV2 {
    // Events for tracking message lifecycle
    event PacketSent(bytes encodedPayload, bytes options, address sendLibrary);
    event PacketVerified(Origin origin, address receiver, bytes32 payloadHash);
    event PacketDelivered(Origin origin, address receiver);

// Quote fees before sending
function quote(MessagingParams calldata _params, address _sender) external view returns (MessagingFee memory);

// Core send primitive - emits message and returns receipt
function send(
    MessagingParams calldata _params,
    address _refundAddress
) external payable returns (MessagingReceipt memory);

// Mark message as verified
function verify(Origin calldata _origin, address _receiver, bytes32 _payloadHash) external;

// Check if message can be verified
function verifiable(Origin calldata _origin, address _receiver) external view returns (bool);

// Check if message passes the target receiver's checks
function initializable(Origin calldata _origin, address _receiver) external view returns (bool);

// Core receive primitive - delivers message to destination contract
function lzReceive(
    Origin calldata _origin,
    address _receiver,
    bytes32 _guid,
    bytes calldata _message,
    bytes calldata _extraData
	) external payable;
	// ... additional configuration and management functions
}
```
- 费用支付和验证
  1. 确保调用者已提供准确要求的**本地费用**或零利率费用。
  2. 执行时`endpoint.send(...)`，端点会**验证费用是否与所选消息库中的报价相符**。如果费用不足，则会撤销操作。

- 数据包构造和发送
  1. Endpoint计算出针对`(sender, dstEid, receiver)`的 nonce，并构建一个`Packet`包含`nonce`, `srcEid`, `sender`, `dstEid`, `receiver`, `GUID`, 和原始数据的结构`message`。
  2. 它会查找要使用的发送库，可以是每个 OApp 的覆盖设置，也可以是默认值`(sender, dstEid)`。
  3. send 库将数据序列化为结构体`encodedPacketd`的一个`Packet`并返回该`MessagingFee`结构体。
  4. 端点发出一个**`PacketSent(...)`事件**，以便 DVN 和执行器知道要处理哪个数据包。
- 验证跨链消息

- 调用`lzReceive(...)`
  1. 如果验证成功，目标端点将调用您的 OApp 的公共端点`lzReceive(origin, guid, message, executor, extraData)`。



**通道**

LayerZero 中的一个**通道**由四个组成部分唯一确定：

- **发送方 OApp**：发起消息的合约
- **源端点**：源链上的 LayerZero 端点
- **目标端点**：目标链上的 LayerZero 端点
- **接收方 OApp**：接收消息的合约

每一种独特的组合都会创建一个具有自身配置的独立通道：



**实际上，只需使用目标端点 ID (EID)**和**接收方**合约地址，即可从源链中识别每个通道。

源 EID 已知（存储在本地端点中），调用方是发出请求的 OApp



**通道特定随机数跟踪**

每个通道都维护自己独立的随机数序列，从而实现并行消息传递而不会出现顺序冲突：

不同通道之间没有顺序依赖关系

每个通道都有自己的安全序列，消息顺序仅在同一通信路径内起作用



**通道的消息包生成**

当应用程序发送消息时，端点会将原始消息数据封装在包含通道特定元数据的标准化数据包容器中。基于[EndpointV2 实现](https://github.com/LayerZero-Labs/LayerZero-v2/blob/4645b5795185a196713263311f76a497a3267dcc/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L108C5-L144C6)：

```solidity
// Endpoint constructs packet with unique identifiers per channel
Packet memory packet = Packet({
    nonce: latestNonce,                    // Sequential message number per channel
    srcEid: eid,                          // Source endpoint ID (this chain)
    sender: _sender,                      // Sender contract address (the OApp)
    dstEid: _params.dstEid,              // Destination endpoint ID
    receiver: _params.receiver,           // Receiver contract address
    guid: GUID.generate(latestNonce, eid, _sender, _params.dstEid, _params.receiver), // Globally unique ID
    message: _params.message              // Raw application data (bytes)
});
```

**主要特性**：

- **随机数**：每个通道按顺序分配随机数，用于排序和重放保护——每个通道维护自己的随机数序列
- **GUID**：由通道组件和 nonce 生成的全局唯一标识符
- **信道识别**：`(sender, srcEid, dstEid, receiver)`唯一标识通信路径
- **消息隔离**：将原始应用程序数据与协议元数据分离



3. #### 消息库层

**管理通道配置**

Endpoint通过模块化**消息库**提供管理通道配置的接口：

```solidity
struct SetConfigParam {
    uint32 eid;          // Target endpoint ID
    uint32 configType;   // Configuration type identifier
    bytes config;        // Encoded configuration data
}

interface IMessageLibManager {
    // Library selection per pathway
    function setSendLibrary(address _oapp, uint32 _eid, address _newLib) external;
    function setReceiveLibrary(address _oapp, uint32 _eid, address _newLib, uint256 _gracePeriod) external;

    // Library configuration per pathway
    function setConfig(address _oapp, address _lib, SetConfigParam[] calldata _params) external;
    function getConfig(address _oapp, address _lib, uint32 _eid, uint32 _configType)
        external view returns (bytes memory config);
}
```

每个 OApp 都可以配置库来定义其特定渠道的消息传递行为。这些库决定了如何验证、执行和传递消息，以满足该 OApp 的通信需求。



**可配置协议库（消息库）**

消息库是链上规则集，用于处理消息在链下发送以及在端点之间到达链上的流程。它们定义了完整的流程：消息处理、验证要求和交付协调，同时维护通用的端点接口。

消息库是链上粘合剂，它使验证器网络和交付服务能够实现插拔，同时保持安全性。不同的消息库可以实现完全不同的消息传递范式和验证需求。

**每个通道可配置的内容**

消息库为每个通道提供三种主要类型的配置：

- **最终性**：验证开始前需要多少个区块确认
- **验证**：哪些验证网络必须验证消息，以及以何种组合方式进行验证
- **执行**：哪些执行服务传递消息以及传递哪些参数



**X-of-Y-of-N 验证协调**

**消息库允许每个通道根据安全要求指定不同的 X/Y/N 配置**

LayerZero 库目前使用**X/Y/N**配置模式进行验证，其中：

- **X**：必须始终进行验证的验证器网络（非同质化）
- **Y**：所需验证者网络总数（必需验证者网络 + 可选验证者网络阈值）
- **N**：可用验证器网络总数

**示例**：**2/4/6**配置

- **2 个特定验证者网络**必须始终进行验证（X = 必需）
- **总共**需要 4 项验证（Y = 必需 + 可选阈值）
- **剩余的 4 个**可选网络中的任意 2 个都可以提供额外的验证。
- 池中**共有 6 个验证者网络可供选择（N = 总选项数）**

每个“验证者网络”都是一个独立的验证系统,实现一种验证方法（零知识证明、委员会共识、轻客户端等）。这就形成了一个**多重验证机制**

![image-20260212152742789](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260212152742789.png)

**推送消息配置（ULN）**

```solidity
struct UlnConfig {
    uint64 confirmations;        // Block confirmations required for finality
    uint8 requiredDVNCount;      // Number of required DVNs (0 = DEFAULT, NIL_DVN_COUNT = NONE)
    uint8 optionalDVNCount;      // Number of optional DVNs available
    uint8 optionalDVNThreshold;  // How many optional DVNs needed: (0, optionalDVNCount]
    address[] requiredDVNs;      // Required DVN addresses (sorted, no duplicates)
    address[] optionalDVNs;      // Optional DVN addresses (sorted, no duplicates)
}

struct ExecutorConfig {
    uint32 maxMessageSize;       // Maximum message size in bytes
    address executor;            // Executor contract address
}
```



4. #### 工作层

**去中心化验证网络 (DVN)**

每个 DVN 都是一个独立的验证服务，实现了一种验证方法（零知识证明、委员会共识、轻客户端、中间链等）。

**DVN提供商**

分布式验证网络（DVN）是独立的验证服务。常见的提供商包括 LayerZero Labs、Google Cloud、Polyhedra (ZK) 等。每家提供商都有不同的信任模型、延迟和成本特点。请查看[DVN 提供商页面](https://docs.layerzero.network/v2/deployments/deployed-contracts)，了解各连锁店的当前地址和可用性。



**执行器**

执行器是一种无需许可的服务，可将经过验证的消息传递到目标链。

任何人都可以运行执行器，这使得消息传递具有竞争力并能有效抵御审查。



**协调机制**：消息库定义规则，分布式验证网络 (DVN) 根据 X/Y/N 配置进行验证，执行器在满足验证要求后交付结果。

1. **消息库**定义了验证和执行的规则。
2. **DVN**根据其专门的验证方法验证消息。
3. **执行器**在满足验证要求后传递消息。
4. **链上执行**确保所有规则在消息执行前都得到遵守。

这种分离使得：

- **独立扩展**：DVN 和执行器可以独立扩展。
- **竞争激烈的市场**：多家供应商可以在成本和性能方面展开竞争
- **灵活的安全性**：应用程序可以根据路径选择验证方法。
- **可靠交付**：多种执行选项，并设有手动备选方案



**拉取模式的特殊性：**

[lzRead功能。](https://docs.layerzero.network/v2/concepts/applications/read-standard)

拉取消息模式，其中数据直接从目标合约查询，而无需涉及目标链的 LayerZero 端点。

对于 lzRead 拉取消息，发送方和接收方是同一个 OApp（请求/响应模式）。由于我们需要区分读取请求和发送到同一合约的标准推送消息，因此我们使用任意通道 ID：

```
// Custom channel ID for lzRead workflows
uint32 constant CUSTOM_READ_CHANNEL = 4294967295;  // Arbitrary identifier

// lzRead pattern:
// - Sender OApp: The contract requesting data (e.g., on Ethereum)
// - Receiver OApp: Same contract receiving the response (same Ethereum contract)
// - Custom EID: Distinguishes this as a read request, not a push message
// - Target: The contract being queried (e.g., on Arbitrum)
```



### 全链应用（OApps）与设计模式

#### 架构模式

**中心辐射式架构（非对称协调）**

其中一条链充当中央“枢纽”，负责协调逻辑，而其他链则充当“辐条”，负责执行逻辑。枢纽做出决策、聚合数据并分发命令。辐条向枢纽报告并执行接收到的指令。

![image-20260212154648072](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260212154648072.png)

所有节点只连接 **中心 Hub**

任意节点间通信，都要 **先到 Hub，再转发**

优点：
新增一个节点，只要对接 Hub

系统复杂度 ≈ 线性增长

USDT0就使用的中心辐射



**点对点架构（对称业务逻辑）**

所有合约都拥有相同的业务逻辑，并以平等身份运行。每个合约都可以与其他任何合约发起通信，并且所有合约都以相同的方式处理消息。没有中央协调器——每个合约都维护自己的状态，同时与对等合约保持同步。

![image-20260212154731308](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260212154731308.png)

理论去中心化最强，但复杂度高，如果有 N 条链，网状模型理论上要维护 **N×(N−1)/2** 条通路（桥/池/路由/风控/监控/应急预案）

没有单点故障影响全局的风险，但攻击面巨大：

- 桥多
- 合约多
- 逻辑分散



#### 消息流模式

**批量发送**

同时向多个目标链发送一条消息

关键 Gas 考量：批量发送需要覆盖基本的 OApp 费用逻辑，因为您将多个报价汇总成一笔付款，然后将其分配给每个发送调用：



**Ping-Pong (ABA 模式)**

链 A 向链 B 发送数据，链 B 在其业务逻辑中调用 LayerZero 端点`_lzReceive`，将数据发送回链 A

关键 Gas 考量：此模式需要链下 Gas 规划。您必须在链下计算 B→A 的返回成本，并将其包含在您的 A→B 执行选项中

**应用场景**：跨链认证、条件执行、数据验证



**调用组合器**

其中主消息存储一个组合消息以供稍后非原子执行：

OApp 调用`endpoint.sendCompose()`存储与 LayerZero 消息 GUID 关联的组合消息。组合合约在单独的事务中调用，这使得该过程成为非原子性的、故障隔离的过程。

**应用场景**：支持自动化操作的代币转移、多步骤 DeFi 操



条件消息处理：_lzReceive业务逻辑可以支持基于消息内容的条件处理。如果您的应用程序需要，您可以在消息中编码条件标识符，以确定应执行哪种类型的处理：



### 消息处理模式

**顺序执行**

OApp 通过将协议 nonce 与本地 nonce 跟踪进行比较来强制执行严格的序列顺序

**关键信息**：LayerZero 端点拥有自己的 nonce 跟踪机制，但默认情况下消息是无序传递的。为了实现有序传递，OApp 必须将协议 nonce（来自消息源）与其本地 nonce 跟踪机制进行比较，并强制执行顺序要求。**应用场景**：金融交易、工作流依赖关系、状态机



**速率限制**



### V2 VS V1

**V1**

- Oracle（默认 Chainlink）
- Relayer（默认 LayerZero Labs）

安全依赖：

> 选定的 Oracle + Relayer 不同时作恶

Relayer 默认中心化，安全模型相对“固定”

![How does Layer Zero Work?. The technology that Layer Zero brings… | by  Asikibayumeko | Medium](https://miro.medium.com/0%2ARjSrk5i5TCMXdTaT.png)

**V2**

- 自定义多个验证网络
- 设置验证阈值（比如 2/3）

安全模型完全模块化

![LayerZero V2 Deep Dive. Everything you need to know about V2… | by Mark  Murdock | LayerZero Official | Medium](https://miro.medium.com/v2/resize%3Afit%3A1400/0%2Asn1eeHrRx4JP1DYX)



### OApp实施

**部署**

1. **与本地端点集成**

将本地 Endpoint V2 地址传递给构造函数或初始化器。

2. **配置Peer**

- 在每个链上，所有者调用`setPeer(eid, peerAddress)`以注册给定Entrypoint ID 的目标连 OApp 地址。
- 在目标链上重复此操作：在其端点 ID 下注册源链的 OApp 地址。
- 因为信任是单向的，所以接收方 OApp 会`peers[srcEid] == origin.sender`在处理入站消息之前进行检查。



**开发**

**`send(...)`入口**开发者定义的逻辑

1. 执行本地状态更改（例如，销毁或锁定代币，记录意图）。
2. 将所有必要数据（地址、金额或任意指令）编码到字节数组中。
3. 可选择接受执行选项（gas 限制、原生 gas 转移或 LayerZero Executor 服务）。



**`lzReceive(...)`入口**开发者定义的逻辑

1. 将字节数组解码为原始数据类型（地址、金额或指令）。
2. 执行预期的链上业务逻辑（例如，铸造代币、解锁抵押品、更新余额）。
3. 如果存在可组合的钩子，您的 OApp 可以调用它`sendCompose(...)`来打包进一步的跨链调用。

​	

- **访问控制和同级检查**只有EntryPoint才能调用`lzReceive`。



### 全链代币标准

**OFT（全链同质化代币）**

一种使用 LayerZero 消息传递的同质化代币标准，用于在源链上进行扣款**（销毁或锁定）**，并在目标链上进行贷记**（铸造或解锁）**，从而在所有连接的网络中保持单一统一的全球供应量。



**ONFT（Omnichain 非同质化代币）**

ONFT 是一种非同质化代币 (NFT) 标准，它使用 LayerZero 消息传递技术在链间转移 NFT，同时保持其唯一性和所有权。ONFT 支持销毁和铸造两种模式，以及基于适配器的锁定/铸造/解锁模式。



#### OFT技术参考

使用相同的 LayerZero 消息传递来移动数据，但添加了特定于Token的不变式作为 OApp 业务逻辑的一部分：

```solidity
// OFT: sends tokens with invariant logic
function send(SendParam memory params) external {
    // 1. Debit tokens locally (burn or lock)
    (uint256 sent, uint256 received) = _debit(msg.sender, params.amountLD, params.dstEid);

    // 2. Send message with token data
    bytes memory message = OFTMsgCodec.encode(params.to, _toSD(received), params.composeMsg);
    //highlight-next-line
    _lzSend(params.dstEid, message, options, fee, refundAddress);
}

// On destination: credit tokens
// highlight-next-line
function _lzReceive(..., bytes calldata message, ...) internal override {
    address to = message.sendTo().bytes32ToAddress();
    uint256 amount = _toLD(message.amountSD());

    // 3. Credit tokens on destination (mint or unlock)
    _credit(to, amount, origin.srcEid);
}
```



#### 语义

- **统一供应模型：**
  对于同质化代币，该标准确保跨链代币供应量保持一致。在发送方，代币要么被*销毁*，要么被*锁定*——有效地将其从流通中移除——而在接收方，相同数量的代币会被*铸造*或*解锁*。这种代币的“流动”创建了一个统一的全球供应量。



- **原子借记/贷记**：

  借记：从源链中移除代币（销毁或锁定）

  贷记：将代币添加到目标链（铸造或解锁）

  本地借记立即发生，远程贷记在消息送达时发生。



- **小数处理**：确保不同十进制系统的链之间精度一致
- 因为不同链、不同 token 实现的 decimals 可能不一样，但跨链消息需要一个统一精度，否则消息编码和对端还原会不一致

​	例如，EVM 链通常使用 18 位小数，而 Solana 通常使用 6 位或 9 位小数。OFT 通过两级十进制系统来处理这个问题：

​	**本地小数**：每个特定链上代币的原生精度，`localDecimals`字段告诉 OFT 合约，该特定区块链使用多少位小数来表示代币的最小单位。

​	**共享小数**：为了确保数值表示的一致性，LayerZero 消息中使用的归一化精度，`sharedDecimals = 6`，跨链转账的最小金额为**0.000001 个代币**。此精度下限可防止高精度链（18 位小数）发送的金额太小，无法使用低精度链（6 位小数）发送。

在跨链发送代币之前，OFT 逻辑会将“本地”金额转换为标准化的“共享”单位。到达目标 OFT 后，目标 OFT 会将该共享单位重新转换回其自身小数精度的本地表示形式。

**归一化过程如下**：

1. 计算转化率：

   小数转换率 =10^本地小数−共享小数）

2. 通过将地板除（例如，EVM 上的整数除法）来清除灰尘：

   flooredAmountLD=⌊amountLD / 小数转换率⌋×小数转换率

3. 计算并向发送方**返回剩余的灰尘量**：

   dust= amountLD −flooredAmountLD

   在从发件人账户**扣款**之前，这笔款项`dust`会退还到发件人的余额中

4. 将本地小数`amountLD`中的金额**转换为源链上的共享单位**：

​	amountSD = amountLD / decimalConversionRate

5. 将金额以共享小数形式（`amountSD`）作为 LayerZero 消息的一部分进行传输。
6. 在目标链上，重建本地数量（`amountLD`）

​	amountLD  = amountSD * decimalConversionRate

要转移小于 0.000001 个代币的小量金额，需要提高`sharedDecimals`精度，但这必须在您的 OFT 部署中的所有链上保持一致。



- **费用估算与支付：**
  **内置的费用报价机制**可估算跨链转账的成本。无论您转账的是同质化代币还是 NFT，发送方都会获得准确的费用估算，其中包括源链 gas 费、协议费用和目标链执行费用。



- **可配置的执行选项：**
  两种代币标准都允许开发者设置执行选项（例如 **gas 限制或回退配置**），并强制执行这些选项，以保证为目标链上的转账提供足够的资源。



- **管理控制：**
  强大的访问控制——通过管理员和委托角色——确保只有授权方才能更新配置（例如对等节点、费用设置、安全设置和执行参数），从而为所有跨链操作维持高安全标准。



- **Hook**：

  **可重写函数**OFT 的核心`_debit`和`_credit`方法已声明`virtual`（或非 EVM 语言中的等效项），允许开发人员在自定义子类/模块中覆盖它们。比如增加**访问控制**的逻辑



- **跨链价值转移 + 调用**：

  可以将**任意数据**与 **OFT 转账**捆绑在一起进行跨链，转账的同时触发其他逻辑的调用，或者使用`composeMsg`功能组合调用





#### OFT方案选择

- 对于新代币，可以使用常规的 OFT，它利用所有链上的**销毁/铸造机制。**

**代币合约本身**包含所有桥接逻辑（发送/接收）以及标准代币功能（铸造、销毁、转移）。

- 对于没有所有者或铸币权的现有代币或者不想升级现有代币合约的，可以使用**OFT 适配器变体**，该变体利用原始链上的**锁定/解锁机制**。适配器合约可以锁定源链上的代币并在目标链上进行**铸币（非原生）**，从而实现跨链转移，而无需修改原始代币合约。

**代币合约**与**桥接逻辑**是**分离**的。代币本身并不包含发送/接收功能，而是由**适配器合约**处理所有跨链操作。适配器持有（锁定）用户代币，与目标链上的配对 OFT 合约通信，将等值的代币转移给接收者，原始代币合约仍然不知道 LayerZero 或跨链流动。



# LayerZero V2 OFT 

#### OFT

该`_debit`函数`OFT.sol`销毁一定数量的 ERC20 代币，同时`_credit`在目标链上铸造 ERC20 代币。![OFT销毁和增发机制示意图：代币在网络A上销毁（减少），在网络B上增发（增加），箭头连接两者，表示跨链转移。](https://mintcdn.com/layerzero/l5FYciYAmKUwmOFF/images/learn/oft_mechanism_light.jpg?fit=max&auto=format&n=l5FYciYAmKUwmOFF&q=85&s=f1789e8bc96aac6d6a4949e10798c85d)`OFT.sol`扩展`OApp.sol`并继承了基础功能`ERC20`，同时提供跨链消息传递和标准代币功能：![类继承关系图，展示了 OFT.sol 继承自 OApp.sol 以实现跨链消息传递，并继承自 ERC20 以实现标准代币功能。](https://mintcdn.com/layerzero/6mCd38kpC06wrzrn/images/oft-inheritance-light.svg?fit=max&auto=format&n=6mCd38kpC06wrzrn&q=85&s=2ae373ecbd50a58d0c26c92172632f6b)

如果您的使用场景涉及代币转移以外的跨链消息传递，请考虑使用[**OApp 标准**](https://docs.layerzero.network/v2/developers/evm/oapp/overview)以获得最大的灵活性。



#### OFT消息传递的工作原理

![image-20260212162412427](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260212162412427.png)

**OFT 消息流**：

1. **代币扣款**：`_debit()`在本地销毁或锁定代币。
2. **消息编码**：`OFTMsgCodec.encode()`创建标准化的Token消息
3. **LayerZero Send**：`_lzSend()`通过 LayerZero 协议发送消息
4. **验证与交付**：DVN 进行验证，执行器进行交付（标准 LayerZero 流程）
5. **消息解码**：`OFTMsgCodec.decode()`从消息中提取Token数据
6. **代币贷记**：`_credit()`在目的地铸造或解锁代币
7. **（可选）触发组合调用。**参阅[Omnichain 可组合性](https://docs.layerzero.network/v2/concepts/applications/composer-standard)。

**核心优势**：

1. **统一供应**：一种代币，多条链，所有部署的总供应量保持一致。
2. **无包装代币**：每条链上的原生表示，无需桥接组件
3. **直接转账模式**：无需中间代币即可进行链间直接代币转账
4. **可组合**：可将令牌与任何消息一起发送到任何地址，从而实现复杂的工作流程。



### 实施OFT

**部署**

OFT 合约必须部署在当前存在或将来存在该代币的每个网络上。

**通道配置**

所有 OFT 部署都必须配置定向通道才能成功进行消息传递。这意味着部署者必须：

- **设置连接消息通道**（建立跨链消息的底层路径）。
- 使用**OApp 级别对 OFT 部署进行配对**`setPeer(...)`，以便每个合约都知道其在目标链上的可信对应合约。

https://docs.layerzero.network/v2/concepts/technical-reference/oapp-reference#security-and-channel-management





# LayerZero V2 Solana

在 Solana 上创建跨链代币的开发者将需要部署他们自己的 OFT 程序实例，并且该实例拥有他们自己的 OFT 存储账户。

每个**OFT 存储帐户都由一个OFT 程序**管理

**OFT Program** 和 **Solana Token Program** 进行交互进行mint/burn

 OFT 存储账户是一个程序派生地址 (PDA) 账户，负责存储您的代币的特定 LayerZero 配置，并支持 Solana 代币的跨链转账。

![image-20260415112956289](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260415112956289.png)

![image-20260415113050804](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260415113050804.png)

**Solana OFT 标准**使用6 个主要账户：

| 帐户名称         | 可执行文件 | 描述                                                         |
| :--------------- | :--------- | :----------------------------------------------------------- |
| OFT程序          | `true`     | OFT 程序本身，即控制 OFT 如何与 LayerZero 端点和 SPL 令牌交互的可执行无状态代码。 |
| mint账户         | `false`    | 这是OFT 的 SPL 代币的[铸币账户](https://solana.com/docs/core/tokens#mint-account)。它存储特定代币的关键元数据，例如总供应量、小数精度、铸币权限、冻结权限和更新权限。 |
| mint权限多重签名 | `false`    | 1/N[多重签名](https://spl.solana.com/token#example-mint-with-multisig-authority)作为 SPL 代币的铸造授权。OFT 存储始终是签名者之一。也可以添加其他签名者。 |
| 第三方托管账户   | `false`    | **OFT 商店**拥有的对应**Mint 账户的**代币账户。用于**OFT 适配器**部署，以及在启用费用时存储费用。对于 OFT 和 OFT 适配器，托管地址是 OFT 商店 PDA 派生的一部分。托管账户是普通的代币账户，而非关联代币账户。 |
| OFT存储账户      | `false`    | 一个[PDA](https://solana.com/docs/core/pda)账户，用于存储每个OFT的相关数据，例如底层SPL代币铸造、SPL代币计划、终端计划、OFT的费用结构和扩展功能。该账户是托管账户的所有者。OFT商店是铸造机构多重签名的签署方。 |
| PeerConfig       | `false`    | PDA账户用于存储每个远程链[的](https://solana.com/docs/core/pda)配置信息，包括对等地址、强制选项、速率限制器和费用设置。该账户源自OFT商店和远程[EID](https://docs.layerzero.network/v2/concepts/glossary#endpoint-id)。 |



### 设置进入 EVM 链的选项

典型的 OFT`lzReceive`调用和铸造操作在大多数 EVM 链上都会消耗`60000`gas，因此您可以强制执行此选项，要求调用者在源链交易中支付`60000`gas 限额，以防止目标链出现 gas 不足的问题。

```
import {addressToBytes32, Options} from '@layerzerolabs/lz-v2-utilities';
// ...
// add the following 3 lines anywhere before the `oft.quote()` call
const GAS_LIMIT = 60_000 // Gas limit for the executor
const MSG_VALUE = 0 // msg.value for the lzReceive() function on destination in wei
const _options = Options.newOptions().addExecutorLzReceiveOption(GAS_LIMIT, MSG_VALUE)
// ...
// replace the options value in oft.quote()
const { nativeFee } = await oft.quote(
    umi.rpc,
    {
        payer: umiWalletSigner.publicKey,
        tokenMint: mint,
        tokenEscrow: umiEscrowPublicKey,
    },
    {
        payInLzToken: false,
        to: Buffer.from(recipientAddressBytes32),
        dstEid: toEid,
        amountLd: BigInt(amount),
        minAmountLd: 1n,
        options: _options.toBytes(), // <--- here
        composeMsg: undefined,
    },
// ...
// replace the options value in oft.send()
    const ix = await oft.send(
        umi.rpc,
        {
            payer: umiWalletSigner,
            tokenMint: mint,
            tokenEscrow: umiEscrowPublicKey,
            tokenSource: tokenAccount[0],
        },
        {
            to: Buffer.from(recipientAddressBytes32),
            dstEid: toEid,
            amountLd: BigInt(amount),
            minAmountLd: (BigInt(amount) * BigInt(9)) / BigInt(10),
            options: _options.toBytes(), // <--- here
            composeMsg: undefined,
            nativeFee,
        },
// ...
```

LayerZero 的 **Message Execution Options（`_options`）** 本质上是一个**跨链消息执行层的“参数指令集”**，用于告诉 **DVNs（验证网络）和 Executor（执行器）**：

> `_options` 是一个 **bytes 编码的执行策略配置**，用于控制跨链消息在目标链上的执行行为。

它会被 LayerZero 的执行组件读取：

- DVN（Decentralized Verifier Network）
  - 负责验证消息
- Executor
  - 负责在目标链调用你的合约（`lzReceive`）

### _options 的本质结构

虽然是 `bytes`，但本质是一个 **TLV（Type-Length-Value）结构编码**

可以理解为：

```
_options = [
  option_type | length | value,
  option_type | length | value,
  ...
]
```

常见 Option 类型（重点）：

### 常见 Message Execution Options

#### 1️⃣ LZ_RECEIVE（目标链执行 gas）

👉 最重要的一个

表示：

> 给目标链 `lzReceive()` 提供多少 gas

示意：

```
addExecutorLzReceiveOption(uint256 gas, uint256 value)
```

- `gas`：执行 gas
- `value`：附带 native token（例如 ETH）

📌 作用：

- 防止执行失败（gas 不够）
- 控制费用

------

#### 2️⃣ LZ_COMPOSE（组合调用）

用于 **跨链后继续调用其他合约（组合调用）**

```
addExecutorLzComposeOption(uint16 index, uint256 gas, uint256 value)
```

📌 场景：

- 跨链 → DEX swap → staking
- 跨链 → NFT mint → metadata update

👉 类似“跨链后的多步执行 pipeline”

------

#### 3️⃣ NATIVE_DROP（空投 gas）

```
addExecutorNativeDropOption(address receiver, uint256 amount)
```

📌 作用：

> 在目标链给某地址转一点 native token（比如 ETH）

用途：

- 给用户 gas（常见 UX 优化）
- onboarding 新用户

------

#### 4️⃣ ORDERED_EXECUTION（顺序执行）

保证消息按顺序执行（避免乱序）

📌 默认 LayerZero 是**无序执行（并发）**





enforcedOptions 和 extraOptions 

  - 目标链执行时要预留多少 gas：sendGas
  - 执行时附带多少原生币 value：sendValue





## Solana OFT程序执行

#### oft_store的创建原理

  - 参与派生的 program id 是 oftProgramId
  - 参与派生的 seed 是 OFT 固定前缀 + 这枚 token 的 mint 地址 “seeds = [OFT_SEED, token_mint.key().as_ref()],”
  - 返回值是 [pda, bump]

所以 oft_store 其实扮演的是这枚 mint 的 OFT 主账户，里面存：

  - 这枚币是谁：token_mint
  - 精度换算怎么做：ld2sd_rate
  - 管理员是谁：admin
  - 是否暂停：paused
  - endpoint 程序是谁：endpoint_program
  - 它自己的 bump：bump



#### 执行

multisig 账户作为 authority，同时用 invoke_signed 让 oft_store PDA 作为 multisig signer 参与 mint_to

```
 mint_to 调用：

  let ix = spl_token_2022::instruction::mint_to(
      ctx.accounts.token_program.key,
      &ctx.accounts.token_mint.key(),
      &ctx.accounts.token_dest.key(),
      mint_authority.key,
      &[&ctx.accounts.oft_store.key()],
      amount_received_ld,
  )?;

```

  - 第 4 个参数 mint_authority.key：谁是 authority
    这里传的是 multisig 账户地址
  - 第 5 个参数 &[&ctx.accounts.oft_store.key()]：谁是这次参与签名的 multisig signer
    这里传的是 oft_store PDA

PDA 没私钥，不能像钱包那样签，所以程序必须通过 invoke_signed 告诉运行时：

  “这个 PDA 是我的，我知道它的 seeds，请把它当成已签名账户。”

```
  let seeds: &[&[u8]] = &[OFT_SEED, oft_store_seed.as_ref(), &[ctx.accounts.oft_store.bump]];

  solana_program::program::invoke_signed(
      &ix,
      &[
          ctx.accounts.token_mint.to_account_info(),
          ctx.accounts.token_dest.to_account_info(),
          mint_authority.to_account_info(),
          ctx.accounts.oft_store.to_account_info(),
      ],
      &[&seeds],
  )?;

```

运行时把 oft_store 视为本次 CPI 的 signer





#### peer

如果 Solana OFT 要连 Sepolia、Base、Arbitrum，那么链上会有多份 PeerConfig，每一份都对应一个目标 eid

```
pub struct PeerConfig {
      pub peer_address: [u8; 32],
      pub enforced_options: EnforcedOptions,
      pub outbound_rate_limiter: Option<RateLimiter>,
      pub inbound_rate_limiter: Option<RateLimiter>,
      pub fee_bps: Option<u16>,
      pub bump: u8,
  }
```

peer 是一个 PDA，派生方式在多个指令里都写得很明确：

```
  peer PDA = PDA(
    program_id = OFT program,
    seeds = ["Peer", oft_store, remote_eid]
  )
```

 例如：

  - Peer(oft_store, SepoliaEid) 是一份配置
  - Peer(oft_store, BaseEid) 是另一份配置

  所以你可以说：

  - 逻辑上，PeerConfig 属于某个远端 EID
  - 实现上，这个 EID 主要体现在 PDA 派生上，而不是 struct 字段里

远端 peer 地址 是什么

  peer_address 是这条路径在远端链上的 OApp / OFT 地址，类型是固定 32 字节：

  pub peer_address: [u8; 32],

  见 programs/oft/src/state/peer_config.rs:9

  对 EVM 链来说，实际地址是 20 字节，所以项目会先做 bytes32 padding，再写进去。



####  接收方向

远端链发消息到 Solana 时，lz_receive 会校验消息里的 sender 必须等于该路径登记的 peer_address：

它保证只有“被你事先登记过的远端 peer”发来的消息才会被接受。
所以 peer_address 其实就是这条跨链路径的对端身份白名单。



#### enforced options 是什么

  enforced_options 是管理员为这条路径预设的 LayerZero options：

  - send：普通 send 的强制 options
  - send_and_call：带 compose message 的 send-and-call 强制 options

 默认内容是通过 LayerZero 的 Options 生成的，当前主要是：

  - 目标链 lzReceive 的 gas
  - 目标链执行时附带的 value



 **它在发送和报价时都会参与**

  所以 enforced_options 的作用是：

  - 给这条链路设置默认 / 最低要求
  - 保证即便调用方没带 extra options，路径本身也知道目标链该如何执行
  - 直接影响 quote 出来的 nativeFee



## 跨链费用计算

solana->eth

    1. 先报价 nativeFee
    2. 再把报价结果作为参数传给 send
    3. OFT 程序把它透传给 LayerZero Endpoint CPI



在 sendNative() 里，这个nativeFee值会被序列化进 OFT 程序的 send 指令数据

```
  data: getSendInstructionDataSerializer().serialize({
      dstEid,
      to,
      amountLd,
      minAmountLd,
      options: options ?? new Uint8Array(),
      composeMsg: composeMsg ?? null,
      nativeFee,
      lzTokenFee: lzTokenFee ?? 0n,
  })
```

 OFT 程序没有自己写任何 system_instruction::transfer(...)OFT 本身不做 lamports 扣款逻辑，而是把支付责任交给 Endpoint CPI


  -  payer.publicKey 被显式传给了 Endpoint 的 account-meta 构造逻辑
  - Endpoint 在 CPI 里会把它当作费用支付方
  - 然后按 native_fee 指定的金额处理 lamports



  用户发起一笔 Solana 交易
    -> 调用 OFT send
      -> OFT burn token
      -> OFT CPI 到 Endpoint
        -> Endpoint 根据 native_fee 完成收费和消息发送



#### 如果 nativeFee 不够会怎么样

  虽然 Endpoint 内部代码不在这个仓库里，但按这套模式，结果通常就是：

  - quote 得到一个建议费用
  - send 时把这个数传进去
  - 如果你传少了，Endpoint CPI 会失败
  - 整笔 Solana 交易回滚
  - token burn 也不会最终生效