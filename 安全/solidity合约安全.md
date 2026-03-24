# 合约安全

## **不小心将 API 密钥或私钥提交到 Github** 

虽然我们没有经常看到这种情况发生，但每次发生都会导致极其灾难性的后果。如果你将 API 密钥或私钥放在`.env`文件中，请始终将`.env`文件添加到`.gitignore`文件中。



## 溢出

整数类型的上溢或者下溢出，攻击者破坏原有的合约检查

防护：对于旧版本，请使用 OpenZeppelin 的 SafeMath 库。openzepplin的safeMath库

solidity 0.8.0以上版本溢出时会revert，已经修复了这个漏洞

但要注意使用uncheck和yul assembly就不会检查，还是要注意漏洞



## 访问控制绕过

不正确的访问控制允许未经授权的用户执行受限制的函数，

防护：使用基于角色的访问控制（例如，OpenZeppelin 的 AccessControl）



## 可升级合约中的存储冲突

**描述**：如果状态变量在版本之间未对齐，则使用代理模式（例如，UUPS）的可升级合约存在存储冲突的风险。

**Solodit 检查**：

- 在升级过程中使用一致的存储布局。
- 使用像 slither-check-upgradeability 这样的工具。



## 不要Delegate Call不可信的合约攻击

被调用的合约修改调用合约的状态存储，比如owner

如果被调用合约可能自毁，导致调用合约功能也失效



防护：因此要注意合约中有delegate call的合约不能调用任意第三方合约或者不能public被外部调用攻击

在使用`delegatecall`时，一定不要使用用户提供的地址调用。



## 安全的处理 ERC20 转账（解决非标准 ERC20 问题）

根据 ERC20 标准（EIP-20）：

> `transfer` 应该返回 `false` 表示失败，而不是必须 `revert`。

Uniswap 有很经典的 `TransferHelper` 思路，用来兼容各种不规范 ERC20。

因为很多 ERC20：

- 有的 `transfer` 返回 `bool`
- 有的不返回值
- 有的失败直接 revert

#### TransferHelper 风格的安全 token 调用

如果外部 token 调用后：

- `returndatasize() == 0`
  说明没返回值，但很多老 ERC20 默认算成功
- `returndatasize() == 32`
  说明返回了一个标准 `bool`
- 其他长度
  一般视为异常

```
function checkSuccess() internal pure returns (bool ok) {
    assembly {
        switch returndatasize()
        case 0 {
            ok := 1
        }
        case 32 {
            returndatacopy(0, 0, 32)
            ok := mload(0)
        }
        default {
            ok := 0
        }
    }
}
```



防护：

使用OpenZeppelin SafeERC20[12]来实现。

这是一个围绕 ERC-20 调用的包装库。不要感到困惑，这不是为了创建自己的 token ，而是为了安全地交易。SafeERC20 的实现基本上就是像上面的 Uniswap 版本一样，你可以像下面这样用它：

```
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/SafeERC20.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol";

contract TestContract {
    using SafeERC20 for IERC20;

    function safeInteractWithToken(uint256 sendAmount) external {
        IERC20 token = IERC20(address(this));
        token.safeTransferFrom(msg.sender, address(this), sendAmount);
    }
}
```



## 重入

被攻击者合约的函数功能里进行转账eth到攻击者合约地址后，执行攻击者合约的reciver()或fallback()函数，该函数可能被设计成继续调用被攻击者合约的函数进行循环调用



被攻击者合约的函数功能里进行外部调用其他合约时，比如transferAndCall，也会进入到被攻击合约的代码再调用回被攻击者合约的函数



另外还有跨函数重入，如果函数直接共享状态被意外操控也可能导致逻辑问题，所以就算用了防重入锁，开发也要遵循CEI规范



防护：

1.先扣款，再赚钱，确保处理完所有导致合约状态变化后在进行transfer或者外部调用

**Checks-Effects-Interactions 模式**：先检查条件、再更新状态、最后与外部交互

2.互斥锁，进入函数加锁，函数执行完解锁



transfer函数的gas限制为2300 gas，这意味着如果接收方合约没有实现fallback函数，或者fallback函数消耗的gas超过了2300，那么转账将失败并回滚所有更改。这可以防止重入攻击，但也可能导致一些问题，例如无法向某些合约发送以太币。

send函数的gas限制也为2300 gas，但它返回一个布尔值，指示转账是否成功。如果转账失败，它将返回false。但是，如果接收方合约没有实现fallback函数，或者fallback函数消耗的gas超过了2300，那么转账将失败并回滚所有更改。

由于这些限制，transfer和send函数已经被认为是不安全的，因此不应该使用它们。相反，您应该使用call函数来转移以太币。call函数没有gas限制，可以向任何地址发送以太币，并且可以指定要发送的gas数量。但是，您应该小心使用call函数，因为它可能会导致一些安全问题，例如重入攻击。）



## 价格操纵

价格操纵是指攻击者通过各种手段人为改变资产价格，然后利用被操纵的价格在 DeFi 协议中获利。DeFi 协议依赖价格数据进行借贷抵押率计算、清算触发、衍生品定价等关键操作，如果价格被操纵，协议就会做出错误的决策。

最常见的价格操纵方式是利用价格来源的缺陷。早期许多协议直接使用单一 DEX 的即时价格，攻击者可以通过大额交易短暂改变价格，在协议中完成操作后再恢复价格。

**背景**：Mango Markets 支持杠杆交易，使用链上订单簿获取价格

**攻击过程**：

1. 攻击者使用两个账户，在 Mango Markets 和其他交易所同时操作
2. 账户 A 在 Mango Markets 做多 MNGO 代币永续合约
3. 账户 B 在现货市场大量买入 MNGO，将价格从 0.03 美元推高至 0.91 美元（涨幅 30 倍）
4. Mango Markets 的价格机制更新，账户 A 的抵押品价值暴涨
5. 账户 A 借出平台上几乎所有资产（USDC、SOL、BTC 等）
6. 停止买入 MNGO，价格回落，但攻击者已提走资产



典型的价格操纵攻击流程：

1. 借入大量代币 A（通过闪电贷）
2. 在 DEX 用代币 A 大量购买代币 B，将 B 的价格推高
3. 在目标协议中，以被操纵的高价格作为抵押品借出资产
4. 在 DEX 卖出代币 B，恢复价格
5. 归还闪电贷，保留借出的资产

整个过程在一个区块内完成，成本仅为 Gas 费和手续费。



**防护措施**

现代 DeFi 协议采用多层防护：

1. **时间加权平均价格（TWAP）**：使用一段时间内的平均价格，而非即时价格
2. **多数据源聚合**：使用 Chainlink 等专业预言机服务，聚合多个独立数据源，避免依赖单一价格来
3. **价格变动限制**：设置单次更新的最大变动幅度，异常波动时暂停操作
4. **延迟更新机制**：关键操作（如清算、铸造）使用延迟价格，给套利者时间纠正异常价格



## 闪电贷

闪电贷（Flash Loan）是 DeFi 的创新金融工具，最初由 Aave引入，后被大多数借贷协议采用（如 dYdX、Uniswap V2 等）。它允许用户在单个交易内借入大量资金而无需抵押，只要在交易结束前归还即可。如果交易执行失败或未归还资金，整个交易会回滚。



从 bZx 借入大量 sUSD（合成美元）

在 Kyber 和 Uniswap 用 sUSD 兑换 ETH，操纵 sUSD/ETH 价格

利用被操纵的价格在 bZx 借出更多资产



## 恶意eth发送

当合约依赖一些this.balance的操作时，就是合约设计好payable，不然其他人随意赚钱

攻击者还是可以通过自毁自己的攻击者合约，将攻击者合约内的eth转入被攻击者合约从而影响其balance

或者通过预测被攻击者合约地址，create或create2，在部署前提前往该地址转入eth,引起该合约后续balance的判断



防护：智能合约要避开依赖this.balance的逻辑，可使用一个变量记录虚拟余额



## 熵随机源

使用未来块的哈希、时间戳、区块号、燃料上限都可能会被矿工攻击，并不是真随机



防护：使用外部接入的随机数，如commit-reveal，如chainlink

 **Chainlink VRF**，不是“合约自己生成随机数”，而是**通过预言机网络返回一个可验证的随机数**。官方把它定义为一种可验证、可防篡改的随机数生成方案，每次请求都会附带密码学证明，证明这个随机值确实是按规则生成的，链上会先验证证明，再把结果交给你的合约使用。

三步：

1. **你的合约发起随机数请求**
2. **Chainlink VRF 生成随机数 + 证明**
3. **Coordinator 回调你的合约，把随机数送回来**



`private_key`：Chainlink oracle 的私钥

`seed`：输入种子（来自链上数据）

`randomness`：随机数



seed = hash(
    keyHash,
    blockhash,
    requestId,
    sender
)



```
(randomness, proof) = VRF(privateKey, seed)
```



任何人都可以通过 **公钥验证 proof**：

```
Verify(public_key, seed, randomness, proof)
```

如果验证成功，说明：

- 随机数确实由该 oracle 生成
- 没有被篡改
- 计算过程正确



因为矿工没有 oracle 私钥。

所以：

```
无法预测随机数
```

oracle也无法提前知道链上区块信息，无法预测









## 未验证的CALL返回

使用transfer()函数发送eth如果失败会revert

但call()和send()函数进行外部调用，会返回一个布尔值标记调用成功与否，而不会因为外部函数revert而revert，如果不判断会继续执行代码导致逻辑错误

```
 (bool success, ) = msg.sender.call.value(amount)("");
```

防护：开发时要判断外部调用的返回结果



另外调用外部合约时要对错误进行处理



## MEV

区块生产者或交易排序者，通过重新排序、插入或删除交易，从区块中额外赚取的利润。

1 Sandwich Attack（三明治攻击）

假设用户要买 ETH：

```
用户交易：
swap 10000 USDC -> ETH
```

机器人看到后：

```
1 机器人先买 ETH
2 用户买 ETH（价格被抬高）
3 机器人卖 ETH
```

结构：

```
Bot Buy
User Buy
Bot Sell
```



2.**抢跑（Front-running）**

机器人检测到有利可图的交易，提交相似交易并支付更高 Gas 费，让自己的交易先执行。

场景 1：清算抢跑

1. 某个借贷协议中的抵押仓位达到清算线
2. 用户 A 提交清算交易到 mempool
3. MEV 机器人检测到这笔清算交易
4. 机器人提交相同的清算交易，但支付更高 Gas 费
5. 机器人的交易先执行，获得清算奖励（通常为抵押品的 5-10%）
6. 用户 A 的交易失败或空跑





防范;

1. Flashbots ,私有交易池，验证者可以通过 MEV-Boost 接收来自多个构建者（Builder）的区块提案，选择最优方案。这使得 MEV 价值能够部分回流给验证者和用户，而非完全被机器人获取。

2. 限制滑点，revert

3. 1inch Fusion 私有订单流

订单不广播到公开内存池，而是直接发送给可信的解析器（Resolver）网络，避免被 MEV Bot 监控。解析器竞争提供最优执行价格，用户无需担心被抢跑或三明治攻击。

4. Uniswap X

类似 1inch Fusion，将交易路由外包给专业的填充者（Filler），在链下竞争最优价格。填充者承担执行风险，用户只需签署意图，无需支付 Gas 费（除非交易失败）。



## Dos阻塞攻击

如果合约的循环要遍历一个数组，攻击者如果让这个数组变大很大，导致调用会耗尽gas，导致合约的某个操作一直无法执行



如果外部调用，别调用的合约的方法作恶导致调用失败，也会导致合约的某个操作一直无法执行

一个智能合约可以返回一个消耗大量Gas的大型内存数组



防护：

合约不能遍历访问一个巨大的数组，

如果有外部调用，应该防范失败，加一个兜底方法



## 浮点数精度

Ethereum Virtual Machine 只提供 **256-bit 整数运算指令**，没有浮点运算指令。

如果支持浮点数会带来几个问题：

1 非确定性

不同 CPU / 编译器的浮点计算可能存在细微差异，例如：

```
0.1 + 0.2 ≠ 0.3
```

区块链要求 **所有节点计算结果完全一致**。

1 Gas 成本

浮点计算需要：

- software emulation
- 复杂指令

Gas 会非常高。

------

3 精度问题

金融合约（DeFi）必须精确。

浮点数会出现：

```
0.30000000000000004
```

这是不可接受的。



遇到除法计算中出现要取整，会向下取中，可能会影响业务正确执行



防护：必须要保证分子足够大

也可以定义一个虚拟的高精度数，把相关变量用高精度变量存储计算，只在输出时再变换回来

避免**先除后乘**



## 对存储指针的写入不会保存新数据

这段代码看起来像是把myArray[1]中的数据复制到了myArray[0]中，但其实不是。如果你把函数的最后一行注释掉，编译器会说这个函数应该变成一个视图函数。对foo的写入并没有写到底层存储。

```
contract DoesNotWrite {
    struct Foo {
        uint256 bar;
    }
    Foo[] public myArray;

    function moveToSlot0() external {
        Foo storage foo = myArray[0];
        foo = myArray[1]; // myArray[0] 不会改变
        // we do this to make the function a state 
        // changing operation
        // and silence the compiler warning
        myArray[1] = Foo({bar: 100});
    }
}
```



## 计算问题

遇到乘法如何超过两个乘数类型的大小，会revert，导致无法执行

防护：如需增大整数的大小，通过显式向上转型每个变量



Solidity 截断不会回退Solidity 并不检查将一个整数转换为一个较小的整数是否安全。



## tx.origin

合约如果需要鉴权时，不应该使用tx.origin，防止攻击者使用攻击合约让用户调用做恶意转发从而绕过鉴权



官方也不推荐使用 `tx.origin`



## 资产转移

支付中使用pull而不是push，防止支付逻辑影响会被其他业务逻辑影响

将支付分离到另一个函数汇总，让用户去请求

调用外部合约时，可能有意或无意造成失败。为了最大程度的减小失败造成的损害，交易发起者可以将每次外部合约调用隔离在单独的交易中。特别是在转账交易中，最好让用户主动发起资金提取而不是自动向用户发起转账。(这也减小了 gas limit 的问题[11].)避免在一次交易中包含多个以太坊转账。









## 开发安全规范

开发过程中注意不要引入安全漏洞

代码应该保持简洁明了,避免过度复杂的逻辑。复杂的合约更容易隐藏漏洞,也更难审计。将系统拆分为多个职责单一的合约,降低单点风险。

对外部调用保持警惕,假设外部合约可能是恶意的。

使用时间锁和多签控制关键操作,避免单点控制。



使用经过验证的代码库(如 OpenZeppelin)而非自己实现基础功能,可以避免重复造轮子带来的风险。



编写全面的单元测试和集成测试,覆盖各种边界情况和异常场景。



使用静态分析工具在开发阶段就发现潜在问题。



有限规模上线(设置资金上限),逐步放开限制



Forta合约安全监控，持续监控协议运行状况,建立异常检测和报警

自动触发紧急暂停机制









### 不要用递归，因为堆栈深度有限，可能导致递归失败

### 谨慎使用循环





### 





# 安全审计和工具

主流的审计公司包括Trail of Bits、OpenZeppelin、ConsenSys Diligence、CertiK、GoPlus 等

但需要明确的是,审计不能保证绝对安全。

审计只能发现已知的漏洞模式,新型的攻击方式或复杂的经济模型缺陷可能被遗漏。

即使是经过多次审计的协议,也可能因为后续升级引入新问题或在实际运行中暴露问题。



Bug Bounty(漏洞赏金计划)是另一种重要的安全机制。项目方在 Immunefi、HackerOne 等平台发布赏金,邀请白帽黑客测试系统安全。



**Solodit 清单**：一个社区驱动的漏洞、检查和真实审计结果的存储库（可在 solodit.xyz 获得）。



自动化安全工具可以辅助检测。

Slither、Mythril 等静态分析工具可以扫描常见的代码模式问题。





**Mythril** 是一个基于 **符号执行（Symbolic Execution）** 的以太坊智能合约安全分析工具，用于自动发现合约漏洞。
 它可以分析 **Solidity源码、编译后的字节码、链上合约**



OpenZeppelin Defender

Echidna、Foundry 的模糊测试功能可以自动生成测试用例发现边界条件错误。

Certora 等形式化验证工具可以数学证明代码的某些属性,提供更强的安全保证。

