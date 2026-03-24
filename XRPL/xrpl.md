## XRPL 上 IOU 命名方式（GLUSD 举例）

### 1. **标准 3 字母代码（不适用 GLUSD）**

- XRPL 仅对 **3 个字母**的 Currency Code 有原生支持（如 `USD`、`CNY`、`BTC`）。
- `GLUSD` 超过 3 个字母，**不能直接作为标准代码**。

### 2. **160-bit 自定义 Currency Code（推荐）**

- 可以把 `"GLUSD"` 转成 UTF-8，再填充到 20 字节（160 位），然后转成 hex 表示。

- 例如：

  - `"GLUSD"` → HEX: `474C555344`
  - Padding 到 20 字节：`474C555344000000000000000000000000000000`

- 在 XRPL 上创建信任线（TrustLine）时，就写成：

  ```
  {
    "currency": "474C555344000000000000000000000000000000",
    "issuer": "rYourIssuerAddressHere"
  }
  ```

- 这样，钱包和浏览器在解析时会显示为 `GLUSD`。

### 3. **Issuer + Code 唯一性**

- 因为任何人都能发行一个 `"GLUSD"`，所以唯一性取决于 **Issuer 地址**。

- 最佳实践是：

  - 在 Issuer 账户的 `Domain` 字段设置并验证你的官网域名（例如 `glusd.com`）。

  - 这样钱包在显示时会标记为：

    ```
    GLUSD issued by rIssuer (glusd.com ✅)
    ```

------

## ✅ 命名最佳实践（适用于 GLUSD 这样的 IOU）

1. 使用 **自定义 160-bit Currency Code**，让 `GLUSD` 能完整显示。

2. 确保 **Issuer 地址唯一、固定**，避免换地址导致多个 GLUSD 混淆。

3. 设置并验证 **Domain**，提升可信度。

4. 对外宣传时写清楚：

   ```
   GLUSD (currency: 474C555344..., issuer: rIssuerAddress)
   ```





1. 前端先构建EVM格式交易，交易中包含EVM合约地址（前端需存储EVM的合约地址，合约ABI）、EOA账户地址（根据XRPL账户地址用hash算法推导）
2. 前端构建向XRPL托管账户的代币（IOU）转账的交易，交易备注（Memo）中附带EVM格式的交易数据；
3. 中继器（Relayer）实时监听XRPL链上交易，当检测到相关交易(Memo中做标识)后，解析Memo字段，提取EVM交易数据；
4. 中继器将提取的EVM交易数据发送至EVM节点执行；若上链执行失败控制XRPL托管账户将IOU退回
5. 若上链执行成功，中继器监听EVM节点的事件
6. 截取原始EVM交易事件的查询结果（如 `eth_getLogs`、`eth_getTransactionReceipt` 等）；
7. 将返回结果中的 `transactionHash` 与地址（如 `from`、`to`）等字段，替换为与之关联的 XRPL 原始交易信息（如 XRPL 交易哈希、来源地址等）；
8. 通过该方式，最终对外暴露的接口返回的是“XRPL 视角下”的交易上下文，从而实现 XRPL 与 EVM 层事件的逻辑绑定与统一追溯。





1. 前端先构建EVM格式交易，交易中包含EVM合约地址（前端需存储EVM的合约地址，合约ABI）、EOA账户地址（根据XRPL账户地址用hash算法推导）
2. 前端构建向XRPL托管账户的代币（IOU）转账的交易，将两个交易发送到kms服务中进行签名，kms返回两个链的交易签名
3. 前端发送EVM交易和XRPL交易到中继器，中继器将EVM交易发送到节点，监听交易执行成功后将XRPL交易发送到XRPL链
4. 后端监听EVM节点的事件进行业务处理，不关心XRPL链交易
5. 当中继器监测到XRPL交易失败后，执行EVM链分叉进行回滚
6. 后端按EVM链分叉情况处理



优点：通过分叉处理机制的运用，使后端业务只感知XRPEVM链，相当于新对接了一条EVM链，而中继器负责两条链的交易状态一致性问题



#### **场景目标**

1. 用户通过 **Evmos** 发起条件支付，将Evmos稳定币锁定在智能合约中。

2. **预言机（Oracle）** 监听到 Evmos 交易后，自动在 **XRPL** 创建托管账户（Escrow），用户需将 XRPL 稳定币转入该账户。

3. 当 Evmos 上条件满足时，Evmos 合约释放代币给Evmos 接收方，同时预言机自动签署 XRPL 交易，释放托管资金给XRPL 接收方。

   

#### **Evmos 智能合约设计**



#### 孪生代币



#### **预言机（Oracle）系统**

- 角色
  - 监听 Evmos 合约事件（如 `PaymentCreated`），自动在 XRPL 创建托管账户。
  - 监控 XRPL 托管账户是否收到资金，并向 Evmos 合约发送确认交易。
  - 当 Evmos 条件满足时，自动签署 XRPL 交易释放资金。
- 实现方案
  - **监听 Evmos 事件**：使用 Evmos 的 Web3 订阅或 The Graph 索引事件。
  - **创建 XRPL 托管账户**：通过 XRPL 的 `EscrowCreate` 交易，设置释放条件（如时间锁或 Evmos 的 `payment_id`）。
  - **自动签署 XRPL 交易**：预言机持有 XRPL 账户私钥，通过 SDK（如 `xrpl.js`）签署交易。



#### **XRPL 托管账户配置**

- 托管账户参数

  - 释放条件：与 Evmos 条件绑定，例如：

    - 时间锁：与 Evmos 的 `releaseTime` 一致。
    - 哈希锁：使用 Evmos 的 `payment_id` 作为加密条件。

  - **接收地址**：XRPL 上的目标账户（需与 Evmos 接收方地址关联）。

    



#### **状态同步**

关键步骤

1. Evmos → XRPL

   - 用户调用 `createPayment` 后，预言机监听事件，调用 XRPL 创建托管账户。
   - 预言机将 XRPL 托管账户地址返回给用户（通过事件或链下通知）。

2. XRPL → Evmos

   - 用户向 XRPL 托管账户转账后，预言机检测到资金到位，调用 Evmos 合约的 `markXRPLFunded` 方法。

3. 条件满足后的释放

   - Evmos 合约释放代币后，预言机检测到 `releasePayment` 事件，自动签署 XRPL 的 `EscrowFinish` 交易。

     

### 问题

#### **1. 原子性问题（操作一致性）**

- **问题**：若 Evmos 释放代币但 XRPL 释放失败，资金可能被永久锁定。

- 

  解决方案

  - **超时回滚**：在 XRPL 托管账户设置 `cancelAfter` 时间，若超时未释放则资金退回原账户。
  - **重试机制**：预言机监控失败交易并自动重试，或触发告警人工介入。

#### **2. 预言机中心化风险**

- **问题**：单一预言机可能被攻击或宕机。

  解决方案

  - 

#### **3. 地址映射与身份关联**

- **问题**：如何将 Evmos 的 EVM 地址与 XRPL 的 X-Address 关联。

  解决方案

  - **用户主动绑定**：用户在首次使用时关联两链地址（如签名验证）。
  - **智能合约生成**：通过确定性算法从 Evmos 地址派生 XRPL 地址。

#### **4. Gas 费用处理**

- **问题**：用户需支付 Evmos 和 XRPL 两笔 Gas 费。

  解决方案

  - **补贴机制**：项目方承担 XRPL 的 Gas 费用（通过预言机账户预充值 XRP）。





### POC

转账同步

**evmos与XRPL的跨链条件支付** 为例：

1. **明确核心问题**
   - 关键验证点：能否通过预言机实现两链状态同步？资产锁定与释放是否原子性？
2. **最小化实现**
   - 仅实现单一条件（如时间锁），忽略复杂逻辑（如多签、链下事件）。
   - 使用测试网（Evmos测试链 + XRPL Testnet）。
3. **开发与测试**
   - 搭建预言机监听 Evmos 事件 → 自动触发 XRPL 托管交易。
   - 模拟用户操作：锁定资产 → 条件满足 → 跨链释放。
4. **验证指标**
   - 成功率：100次跨链操作中成功次数。
   - 延迟：从 Evmos 条件满足到 XRPL 资金释放的时间。
5. **输出结论**
   - 可行性：是/否，需附上数据（如吞吐量、成本）。
   - 下一步建议：优化预言机响应速度或转向原型开发。



XRPL EVM BRIDGE方案
自己搭建，网关账户中心化问题，ROR在EVM链上而非主网



地址转换问题，现有合约的msg.sender，用户权限，

使用账户抽象方案，合约账户地址create2,指定为xrpl地址的转换

memo带上userOps信息，格式使用Axelar ，为后续改用XRPL EVM做准备

后端订阅EVM节点的事件

正对前端展示问题，交易哈希前端调用时存XRPL的交易哈希，展示地址时，也是XRPL的地址（如需转换时用转换函数）

签名验证，私有节点，不验证签名，中心化中继器上链，AUTH控制



工作量：验证XRPL交易备注构建

备注内容为evm链执行交易的参数

资金转账到中继器XRPL托管账户

中继器判断状态是否成功以及后续操作

中继器解析交易备注，发送到EVM节点执行

执行成功，发送事件后端记录成功

执行失败回滚，中继器账户将资金退回，后端无感，前端交易失败

合约增加跨链转账函数和通知中继器将托管账户中的资金转账到XRP接收账户，如结算时，通知主网中继器账户将资金转给接收方

发起send时，跨链铸造NFT通知，拆分时，跨链铸造NFT

问题：多维护一套私钥签名，EVM地址是由XRP地址推导出EVM私钥而来，还是用户注册就为用户生成一个，直接存好映射？

EVM地址无法转回XRP地址，结算时，中继器不知道将钱打给哪个XRP地址，除非维护一套XRP到EVM的地址映射表



使用地址编码转换，好处是不需要存映射表，直接按需计算，

缺点:该转换而来的EVM地址私钥位置，无法控制该账户操作EVM,交易都是由单一中继器签名上链，合约内的msg.sender改造，permit取消，RSV填空



XRPL EVM官方方案

还是需要用户管理两个链的私钥（需要由用户的助记词推出），而且需要用户多一步先跨链资产到目标链的交易，再发起目标链的合约调用

如果是那就变成更高级别的跨链，相当于一步用A链资金买B链资产的抽象水平，难度大



调研Axelar 跨链协议，符合Axelar GMP协议，未来将Destination改为Axelar网关账户就使用官方跨链服务，EVM合约实现`AxelarExecutable`Axelar GMP 的接口

部署账户抽象entrypoint

部署智能合约账户逻辑合约 init

各种permit的r.s.v问题改造

去除address是否是合约地址的判断

中继器合约调用机制





Amount

Destination：

Memos：

- The **destination chain ID** .
- The **contract address** on the destination chain to which the message is sent.
- evm的已签名交易（是通过kms签名还是前端直接签名，私钥为XRP地址转换而来）

所有字段[`Memos`](https://js.xrpl.org/interfaces/Payment.html#Memos)必须以十六进制格式进行编码。

