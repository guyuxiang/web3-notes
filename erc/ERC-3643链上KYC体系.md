# ERC-3643链上KYC体系

证券监管法规错综复杂，各国之间也存在差异，但它们确实有一些共同之处：

- 其中一个相似之处在于，都需要**准确识别参与金融产品发行的各个利益相关者**，以便确定他们各自的责任和义务。
- 证券监管的另一个重要方面涉及**投资发行及相关交易**，例如二级市场投资者之间的证券分销、登记、赎回和转让，这些通常受到限制、约束或指导方针的约束。

可以把 ERC-3643 理解成一套“**ERC-20 Token + 身份注册层 + 合规规则层**”的组合协议。它仍然兼容 ERC-20，但 `transfer/transferFrom` 不再是“只看余额和授权”，而是会在转账时自动串起身份校验与合规校验。

**ERC-3643 协议解决了这两个方面的问题。** 

- 它代表具有链上**身份**的利益相关者，允许定义权限，并为每个利益相关者分配权利和义务。
- **此外，可以通过合规模块**轻松添加投资发行规则，并且可以通过其他智能合约丰富该协议，以适应代币所需的任何特定规则



![image-20251223105140433](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20251223105140433.png)



T-REX协议由以下几个关键部分组成：

- **[ONCHAINID](https://github.com/onchain-id/solidity)**：用户对应的链上身份智能合约，用于与代币或任何其他可能需要链上身份的应用程序进行交互时的验证。它存储与特定身份相关的密钥和声明。
- **可信发行方**：此合约包含与特定代币关联的所有受信任声明发行者的地址。
- **声明主题注册表**：此合约维护与Token相关的所有受信任声明主题的列表。
- **身份注册表**：该合约保存所有符合资格并获授权持有代币的用户的身份合约地址，并负责进行权益验证。
- **Compliance（合规模块）**：该合约独立运行，用于检查转账是否符合代币的既定规则。
- **T-REX代币合约**：该合约与身份注册机构交互，以检查投资者的资格状态，从而实现代币持有和交易。





## T-REX代币合约

这是用户真正交互的代币合约。它负责 `transfer / transferFrom / mint / burn / forcedTransfer / recoveryAddress / pause / freeze` 等生命周期动作，但它自己**不直接存完整的 KYC/AML 规则**，而是把“能不能持有、能不能转”委托给 Identity Registry 和 Compliance。Token 合约还允许 owner 设置 `IdentityRegistry` 和 `Compliance` 的地址。



## ONCHAINID

个人可以控制多个钱包。为了应对合规性和监管要求挑战，T-REX 协议实现了链上身份，从而确保个人层面的合规性，并提供可验证凭证的灵活性和可重复使用性。

**每个投资者有一个 ONCHAINID 合约，底层扩展自 ERC-734/735，用来保存 key 和 claims。**

**可信机构把 KYC/AML 等凭证写成 claim 附着到 ONCHAINID 上，验证时会检查 claim 的 topic、签名和数据有效性。**



#### **基于钱包的合规性的局限性**

- **每人可拥有多个钱包**：个人可以拥有多个钱包，这使得仅根据钱包地址来强制执行合规规则变得困难。
- **复杂的白名单机制**：简单的钱包白名单不足以确保满足每个人的监管要求。

#### **链上身份的优势**

- **个人层面的合规性**：通过在区块链上管理身份，可以直接对个人应用合规性规则，而无需考虑他们控制的钱包数量。
- **集中式身份管理**：ONCHAINID 允许创建全球可访问的身份，这些身份可以在区块链上进行管理和验证。



#### 链上身份的工作原理

ONCHAINID是一个基于区块链的身份管理系统

**每个 ONCHAINID 都拥有一个唯一的区块链标识符：其智能合约地址，它为每个参与者分配一个唯一的身份标识。**

**该身份标识与其所有关联的钱包相关联，从而实现全面的合规管理。**

身份创建：参与者创建一个存储在区块链上的 ONCHAINID 智能合约。

声明管理：受信任实体向这些身份颁发声明（可验证凭证），这些声明也存储在区块链上。这些声明验证各种方面，例如 KYC 和 AML 合规性。

验证：在交易过程中，将验证身份及其相关声明，以确保符合监管要求。



#### **可验证凭证**

在 ERC-3643 的架构中，**可验证凭证是由可信实体（例如受监管的 KYC/AML 服务商或政府机构）颁发给参与者的一种证明**，用于表示该参与者满足某些预设的合规条件（比如通过 KYC、是合格投资者等）

**凭证颁发**：可信第三方验证参与者（如 KYC），并将结果以 **可验证凭证（claims）** 的形式赋予 ONCHAINID。

**链上验证**：在代币转移或持有前，智能合约检查参与者的 ONCHAINID 是否拥有必要的可验证凭证。

**合规授权**：只有满足条件的参与者才能完成 ERC-3643 代币的接收或转移。



#### OnchainID Solidity 项目详解

https://github.com/onchain-id/solidity

这个仓库是 **OnchainID** 官方提供的 Solidity 实现，用于在区块链上实现**去中心化身份（On-chain Identity）**。它是很多合规型代币标准（例如 **ERC-3643 / T-REX**）的核心基础组件。

> 👉 **把“身份 + KYC + 声明（Claims）”做成链上可验证的智能合约系统**

传统 Web2：

- 身份在中心化数据库（银行 / KYC服务商）

OnchainID：

- 身份是一个**智能合约地址**
- 所有认证信息是**链上可验证的声明（Claim）**

#### 整体架构

#### 1️⃣ Identity（身份合约）

每个用户 = 一个 Identity 合约（类似智能账户）

核心能力：

- 持有多个 key（权限体系）
- 接收/存储 claims（认证信息）
- 支持代理执行（类似 AA）

👉 类似：

- ERC-4337 Smart Account（但更偏身份）

#### 2️⃣ Key Management（密钥体系）

OnchainID 定义了不同用途的 Key：

| Key 类型         | 作用                 |
| ---------------- | -------------------- |
| MANAGEMENT_KEY   | 管理权限（最高权限） |
| ACTION_KEY       | 执行交易             |
| CLAIM_SIGNER_KEY | 签发 claim           |
| ENCRYPTION_KEY   | 加密用途             |

👉 一个 Identity 可以：

- 多签控制
- 热/冷钱包分离
- 企业级权限结构

#### 3️⃣ Claim（声明体系）

这是整个系统的核心。

#### Claim 是什么？

👉 **“某个机构对你身份的一种证明”**

例如：

- “Binance 证明你通过了 KYC”
- “政府证明你是美国居民”
- “银行证明你是合格投资者”

------

#### Claim 数据结构（核心）

```
struct Claim {
    uint256 topic;
    uint256 scheme;
    address issuer;
    bytes signature;
    bytes data;
    string uri;
}
```

解释：

| 字段      | 含义                                        |
| --------- | ------------------------------------------- |
| topic     | 声明类型（KYC / AML / Accredited Investor） |
| issuer    | 谁签发的                                    |
| signature | 签名                                        |
| data      | 实际内容                                    |
| uri       | 链下数据                                    |

#### Claim 工作流程

流程如下：

1. 用户创建 Identity 合约
2. KYC 机构（Issuer）验证用户
3. Issuer 用私钥签署 Claim
4. Claim 存入用户 Identity 合约
5. 其他合约（如证券 Token）验证 Claim

#### 核心合约模块

#### 1️⃣ Identity.sol

👉 主体合约（最重要）

功能：

- addKey / removeKey
- addClaim / removeClaim
- execute（代执行）

------

#### 2️⃣ ClaimIssuer.sol

👉 Claim 签发者

职责：

- 管理签名公钥
- 提供 `isClaimValid()` 校验接口

#### 一笔合规转账流程：

1. 用户A 发起 transfer
2. Token 合约调用 IdentityRegistry
3. IdentityRegistry 获取：
   - A 的 Identity
   - B 的 Identity
4. 调用 Identity：
   - `getClaim(KYC_TOPIC)`
5. 调用 ClaimIssuer：
   - `isClaimValid()`
6. 校验通过 → 转账成功

#### 隐私问题

- Claim 默认是公开的
- 不适合敏感数据

👉 通常解决：

- hash + off-chain storage
- zk（但原项目没做）



## 可信发行机构注册表

可信发行机构注册中心 (TIR) 负责管理和验证有权发行权益的实体列表。该注册中心确保只有经过验证且可信的发行机构才能参与身份验证过程，从而增强协议的安全性和可靠性。

#### 可信发行机构注册表的工作原理

1. **列出可信发行人**：
   - TIR 列出了可信声明签发方的实体的地址。每个可信签发机构都需经过正式流程才能添加到注册表中，以确保其信誉和可靠性。
2. **验证流程**：
   - 在身份验证过程中，身份中心会将 **ONCHAINID 上的声明**与 **TIR 中的可信发行者列表**进行比对。如果声明是由 TIR 中列出的实体发行的，并且符合所需的声明主题，则该声明被视为有效。
3. **与其他组件的集成**：
   - TIR 与身份注册表存储和声明主题注册表集成，提供全面的验证流程。这确保所有声明均有效且由可信实体发布，从而维护协议的完整性。



## 声明主题注册表

声明主题注册表 (CTR) 负责**列出代币所需的声明主题**。每个代币都有其自身的声明主题注册表，其中规定了投资者必须在其 ONCHAINID 上拥有哪些类型的声明，才能被视为符合资格。CTR 与身份中心、身份注册表存储和可信发行者注册表协同工作，以确保全面的合规性验证。

#### 声明主题注册表的主要特点

1. **所需声明的定义**：
   - 这些声明可能包括KYC、AML、认证和其他监管要求。
2. **与身份验证集成**：
   - CTR 在身份验证过程中扮演着至关重要的角色。当`isVerified`在身份注册表上调用该函数时，它会从 CTR 中获取所需的声明主题以验证 ONCHAINID。
3. **动态管理**：
   - 声明主题注册表允许动态添加和删除声明主题。这确保了注册表能够适应不断变化的监管要求和不同代币的特定需求。

#### 声明主题注册表的工作原理

1. **列出声明主题**：
   - CTR 列出了特定代币所需的声明主题。每个声明主题都由一个唯一标识符表示。这些主题对于合规性和监管验证至关重要。
2. **验证流程**：
   - 在验证过程中，身份注册中心从身份注册中心存储中获取 ONCHAINID 地址，并从 CTR 中检索所需的声明主题列表。
   - 身份注册中心还会从可信发行者注册中心获取可信发行者列表，并将 ONCHAINID 上持有的声明与所需的声明主题进行比较。
   - 如果 ONCHAINID 具有所需的声明，并且这些声明是由受信任的发行者发布的，`isVerified`则该函数会检查加密签名以确保其有效性。
3. **确保合规性**：
   - 通过明确所需的声明主题，CTR 确保生态系统中的所有参与者都符合必要的监管标准。这种合规性对于维护代币交易的完整性和合法性至关重要。





## 身份注册表存储

身份注册存储是存储和管理身份数据的基础架构。它与身份注册中心协同工作，确保身份信息得到安全存储、便捷访问和高效管理。

同一个身份注册存储可供多个身份注册合约共享。

#### 身份注册表存储的工作原理

1. **数据存储**：
   - 身份注册表存储了与参与者**钱包地址**对应的 **ONCHAINID 地址**。这种映射确保每个钱包地址都能链接到一个 ONCHAINID，其中包含身份验证所需的声明和凭证。
2. **与身份注册机构集成**：
   - 当`isVerified`身份注册表调用该函数时，身份注册表会从其存储中获取与接收方钱包对应的链上ID地址。此过程对于在交易过程中验证参与者的身份至关重要。





## 身份中心

身份中心负责管理和验证生态系统参与者的身份。它确保只有合规且经过验证的参与者才能持有和转移代币，从而执行监管要求并增强平台的安全性。

#### 身份注册的主要特点

1. **合规执行**：

   - 身份注册系统与合规合约交互，以执行转让规则和限制。这种交互确保只有符合资格的投资者才能参与代币交易，从而维护证券型代币市场的完整性。

   

####  

#### 身份注册系统的工作原理

1. **注册流程**：
   - 参与者创建 ONCHAINID。ONCHAINID 与各种声明相关联，用于验证参与者的身份。
   - 可信声明签发方可向ONCHAINID签发声明，证明参与者符合监管要求。
2. **验证与合规性**：
   - 在代币转移或任何其他受监管的操作期间，身份注册机构会检查参与者的 ONCHAINID 和相关声明，以确保合规性。
   - 如果参与者符合所有必要条件，则该操作获得批准；否则，该操作将被拒绝，以维护系统的完整性。

####  

#### 转账验证流程

当发生转账并`isVerified`调用身份注册表中的函数来检查投资者的资格时，将执行以下步骤：

1. **获取链上ID**：
   - 身份注册表从**身份注册表**存储中获取与**钱包地址**对应的 **ONCHAINID 地址**。
2. **比较声明**：
   - 它将 ONCHAINID **持有的声明**与声明主题注册表和可信发行者注册表中规定的要求进行比较。
3. **验证声明**：
   - 身份注册机构通过核对声明单上的要求来检查声明的有效性。
4. **验证状态**：
   - 如果所有声明均有效且符合要求，则该`isVerified`函数返回 true，允许转账继续进行。否则，返回 false，阻止转账。



## **合规管理**

Compliance 管的是“**这笔交易规则上允不允许**”，而不是“这个人是不是合格投资者”。比如：单地址最大持仓、某国投资者人数上限、某时间窗内转账额度、只允许某些国家流通等。它提供只读的 `canTransfer(from,to,amount)` 做预检查，同时有 `transferred / created / destroyed` 三个状态更新钩子，用来在转账、增发、销毁后维护计数器或额度状态。

**灵活的规则管理**：

- 合规合约**允许动态添加和修改合规规则**。这种灵活性确保合约能够适应不断变化的监管环境和不同代币发行项目的具体要求。

#### 工作原理

**合规性验证**：

- 该`canTransfer`函数检查拟议的代币转移是否符合设定的规则。它会验证发送方和接收方的资格以及其他参数，并返回`true`转移是否合规的结果`false`。

**交易监控**

​	通过调用合规合约的  `transferred`, `created`, and `destroyed` 函数，监控所有代币的转移、创建和销毁操作。这些函数会更新合约内的状态变量，进而用于强制执行合规规则。



#### 模块化合规性附加组件

合规模块示例

**CountryAllowModule**：它可根据参与者的地理位置对代币转移进行精细化控制，使合规机构能够在链上管理特定国家/地区的交易权限。投资者与单个国家/地区关联，并保存在 ID 注册表中。

**CountryRestrictModule**：与上面提到的“CountryAllowModule”相反，所有者可以使用此模块将代币交易限制为特定国家/地区的用户。

**ExchangeMonthlyLimitsModule**：此模块旨在设置每月允许转移的代币数量限制。

**最大余额模块**：有时，大量代币存放在单个地址是不理想的，因为这可能导致价格操纵、投票系统不公平以及其他问题。最大余额模块可以帮助平台所有者限制用户可以持有的代币最大数量。

**SupplyLimitModule**：SupplyLimitModule 模块实现了常见于流行库中的供应上限功能。通过此模块，平台所有者可以将总供应量限制在一定范围内，从而防止代币的无限增发。

**TimeExchangeLimitsModule**：TimeExchangeLimitsModule 允许平台所有者在设定的时间范围内将代币交易限制在特定的交易所。用户（通过合规地址识别）可以拥有多个交易所 ID，并根据需要使用这些 ID 进行交易。

**TimeTransfersLimitsModule**：此模块允许平台所有者设置在给定时间范围内允许转移的代币数量限制。  

**TransferFeesModule**：协议费用对于平台的可持续发展至关重要。该模块允许系统管理员轻松设置费用并指定收款地址。因此，该模块可确保在代币转账过程中，按照指定的费率和收款地址收取费用。

**TransferRestrictModule**：TransferRestrictModule 合约本质上是在系统中创建了一个许可列表功能，使系统管理员能够无缝地管理用户对转账的访问权限。此外，它还通过批量操作提供了灵活性，可以高效地同时管理多个用户地址。



#### CountryAllowModule合约

 配置国家是调用 ERC3643CountryAllowModule 合约。
  具体方法在：                                                                                                                                                                                                                                                                                                                                                                                                                                             

  - contracts/kyc/erc3643/modules/ERC3643CountryAllowModule.sol                                                                                                                                                                                                                                                                                                                                                                                   

  可调用的方法：                                                                                                                                                                                                                                                                                                                                                                                                                                    
  - allowCountry(address compliance, uint16 country, bool allowed)                                                                                                                                                                  
  - batchAllowCountries(address compliance, uint16[] countries, bool allowed)                                                                                                                                                                                                                                                                                                                                                                                 

  参数含义：                                                                                                                                                                                                                                                                                                                                                                                                                                         
  - compliance                                                                                                                                                                                                                      
      - 传某个 vault 对应的 compliance 地址                                                                                                                                                                                         
      - 比如 seniorCompliance 或 juniorCompliance                                                                                                                                                                                   
  - country                                                                                                                                                                                                                         
      - 国家代码，uint16                                                                                                                                                                                                            
  - allowed                                                                                                                                                                                                                         
      - true 表示放开，false 表示移除    

  一个例子：

```solidity
  countryAllowModule.batchAllowCountries(seniorCompliance, [156, 840], true);
  countryAllowModule.batchAllowCountries(juniorCompliance, [156], true);
```



#### Compliance 如何检查

ERC3643ModularCompliance.canTransfer(...) 会遍历自己挂载的所有 module：                                                                                                                                                                                                                                                                                                                                                                     

  - 逐个调用 module.moduleCheck(from, to, value, compliance)                                                                                                                                                                        
  - 只要有一个返回 false，本次转账/claim 就不允许                        

 1. module 收到 moduleCheck(from, to, value, compliance)                                                                                                                                                                           
  2. 它先根据 compliance 找到这个 compliance 绑定的 IdentityRegistry                                                                                                                                                                
  3. 再调用：                                                                                                                                                                                                                       
      - identityRegistry.investorCountry(to)                                                                                                                                                                                        
  4. 取到目标地址 to 的国家码                                                                                                                                                                                                       
  5. 最后检查：                                                                                                                                                                                                                     
      - allowedCountries [compliance] [country]



## 初始化工作流程

**部署整套合约：Token、IR、IRS、CTR、TIR、Compliance。现在官方也提供 T-REX Factory，可一笔交易把整套 suite 部署出来，并完成基础配置。**

**owner 配参数：**

- Token 上设置 `setIdentityRegistry(IR)`、`setCompliance(Compliance)`
- IR 上设置 `setIdentityRegistryStorage(IRS)`、`setClaimTopicsRegistry(CTR)`、`setTrustedIssuersRegistry(TIR)`
- IRS 上 `bindIdentityRegistry(IR)`
- Compliance 上 `bindToken(Token)`。

**owner/agent 配合规基线：**

- **CTR 添加必须的 claim topics**
- **TIR 添加可信 issuer，并配置每个 issuer 允许签发哪些 topic。**

**投资者入网：**

- **投资者创建 ONCHAINID**
- **KYC/AML 机构作为 trusted issuer 往 ONCHAINID 写 claim**
- **IR 的 agent 调 `registerIdentity(wallet, onchainID, country)` 完成钱包与身份绑定。**



## 转账验证过程

### 第一步：用户发起 `token.transfer(to, amount)`

Token 先做本地检查：

- token 没被 `pause`
- 发送方和接收方地址没被冻结
- 发送方自由余额充足（总余额减去被冻结的部分）。

### 第二步：Token 调 `IR.isVerified(to)`

这里检查的是“**接收方是否有资格持有这个 token**”。

IR 里面会：

- **从 IRS 取出 `to` 绑定的 ONCHAINID 和 country**
- **从 CTR 取出必需 claim topics**
- 从 TIR 取出可信 issuer
- **校验 ONCHAINID 上对应 claims 是否存在且由可信 issuer 合法签发**
- 全部通过才返回 `true`。

### 第三步：Token 调 `Compliance.canTransfer(from, to, amount)`

这里检查的是“**这笔交易在规则层面是否允许**”。

例如：

- 接收方持仓是否会超上限
- 某国投资者人数是否会超限
- 交易时间窗额度是否超限
- 是否命中某个国家/交易所/期间限制模块。

### 第四步：真正记账

如果前面都通过，Token 执行 `_transfer(from, to, amount)`。这时才真正移动余额。随后 Token 会调 `Compliance.transferred(from, to, amount)`，让合规模块更新自己的内部状态，比如人数计数、额度消耗、时间窗统计等。



#### 身份标准相关ERC

T-REX 采用强大的身份管理标准，以确保符合监管要求：

ERC-734 和 ERC-735：这些标准用于身份创建和声明管理。它们支持可验证凭证的集成，确保只有符合资格的参与者才能持有和转让令牌。

ERC-734：专注于管理密钥持有者和身份相关操作。

ERC-735：处理声明，允许第三方对特定身份提出声明。