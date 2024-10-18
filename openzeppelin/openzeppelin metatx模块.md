## openzeppelin metatx模块

该模块为元交易模块（Metatransaction），是一种让用户不需要支付 gas 费就能够使用 DApp、发起交易、调用智能合约的手段。

用户在前端完成对交易内容的签名, 通过链下RPC协议发送给代付方, 待付方对交易内容初步解析验证后, 将该交易内容包装后发送到区块链节点, ERC2771Forwarder代理合约, 来解析用户签名的交易数据, 验证无误后发起交易到相应的目标合约, gas fee由代付方支付从而实现用户免gas交易

在代理合约实际支付逻辑 , 比如由用户的ERC20代币来抵扣交易ges fee



基于ECDSA和EIP712来实现对交易数据的签名和解析验证:

![image-20240417181629823](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240417181629823.png)





#### ERC2771Forwarder合约

代理人对用户的签名进行包装

```solidity
struct ForwardRequestData {
    address from;                    // 用户地址
    address to;                      // 业务合约地址
    uint256 value;                   // 转账金额
    uint256 gas;                     // 设置的gas
    uint48 deadline;               
    bytes data;                     // 原始交易数据
    bytes signature;                // 用户交易签名
}
```



代理人调用合约的execute函数来进行验证用户交易签名, 验证通过后会调用业务合约

```solidity
function execute(ForwardRequestData calldata request) public payable virtua
```



批量交易执行:

```solidity
function executeBatch(ForwardRequestData[] calldata requests, address payable refundReceiver) public payable virtual
```



#### ERC2771Context 合约

业务合约要支持代付交易的话,需要继承ERC2771Context合约, 会自动判断接收到的交易是直接支付还是代付, 代付时会将msg.sender重写替换为原始客户端Client而非ERC2771Forwarder合约的地址, msg.data重写为原始交易data



```solidity
import ”@openzeppelin/contracts/metatx/ERC2771Context.sol";

contract MyContract is ERC2771Context{
	...
}
```



另外要设置ERC2771Forwarder合约地址, 只允许指定合约调用

```solidity
constructor(address trustedForwarder) {
    _trustedForwarder = trustedForwarder;
}
```



## OpenGSN

该框架也实现了元交易规范 ERC2771, 功能要比openzeppelin实现更完善

#### 角色:

**客户端-Client**

客户端也就是各种Dapp，是GSN架构的最上层。客户端负责发起对原交易进行签名，并将签名后的原交易发送到中继服务器中。

前端可以直接引入@opengsn/provider包来集成GSN



**中继服务器-RealServer**

中继服务器主要用来处理用户的元交易请求，主要的功能包括：

- 通过调用中继仓库（RelayHub）合约，判断待付人（Paymaster）是否允许为该笔交易支付手续费，并且有足够的以太币
- 中继服务器将交易发送到链上, 对于中继服务器，多个客户端可以使用一个，也可以一个客户端对应一个。



**待付人-Paymaster**

待付人为Gas的实际支付者。付款人是一个智能合约，该合约根据具体业务实现用户的待付费用扣除逻辑, 并实现交易过滤器决定了可以为哪些业务交易支付费用。 

```solidity
import "@opengsn/contracts/src/BasePaymaster.sol";

contract TokenPaymaster is BasePaymaster {
    constructor(address forwarder) {
        trustedForwarder = forwarder;
    }
    ...
}
```



需要实现以下两个方法:

在中继调用前会调用待代付人合约的preRelayedCall函数来验证时候进行后续中继调用

常用的验证方式包括：

- 白名单
- 令牌认证
- 对特定方法放行
- 以代币形式向用户收费（可能由您发行）
- 链下委托授权

```solidity
function preRelayedCall(
    GsnTypes.RelayRequest relayRequest,
    bytes approvalData,
    uint256 maxPossibleGas
)
external
returns (
    bytes memory context,
    bool rejectOnRecipientRevert
);
```



在中继调用后会调用待代付人合约的postRelayedCall函数,提供交易成本的准确估计,可以实现向用户收取token费用

```solidity
function postRelayedCall(
    bytes context,
    bool success,
    bytes32 preRetVal,
    uint256 gasUseWithoutPost,
    GsnTypes.RelayData calldata relayData
) external;
```





**中继仓库-RelayHub**

中继仓库本身是一份智能合约，提供的功能包括：

- 维护一份中继服务器列表，供客户端查询
- 提供`RelayHub.balances[recipient]`方法，供中继服务器在支付Gas前检查代付人已存入足够的ETH 
- 中继仓库合约可以自行部署，也可以直接使用GSN提供的。自行部署的RelayHub无法共享已存在的中继器。 以太坊主网上的RelayHub合约地址：[0xD216153c06E857cD7f72665E0aF1d7D82172F494](https://link.zhihu.com/?target=https%3A//cn.etherscan.com/address/0xD216153c06E857cD7f72665E0aF1d7D82172F494)



**可信转发器-Trusted Forwarder**

可信转发器用来验证发送者签名和Nonce值，RelayHub通过可信转发器将元交易转发到Dapp合约中。



**中继接收合约-RelayRecipient**

每个支持GSN的应用合约都需要继承RelayRecipient，在继承RelayRecipient合约后，会自动判断接收到的交易是直接支付还是代付, 代付时会将msg.sender重写替换为原始客户端Client而非可信转发器合约的地址。并提供与RelayHub通信的接口从而可被 GSN 调用。在部署Dapp合约时，需要初始化RelayHub的地址。 

```solidity
import "@opengsn/contracts/src/BaseRelayRecipient.sol";

contract MyContract is BaseRelayRecipient {
    ...
}
```



开发者和用户可以直接使用openGSN提供的中继仓库, 中继服务器和可信转发器, 不需要了解或信任

开发者只需实现代付人Paymaster合约和中继接收合约RelayRecipient, 将其注册到中继仓库即可



元交易流程

1、客户端Client到中继仓库RelayHub查询可用的中继服务器列表 

2、客户端Client对原交易签名后发送到中继服务器RelayServer

3、中继服务器RelayServer到RelayHub中验证待付人Paymaster是否有足够的ETH用于支付Gas，并且代付人是否允许该交易, 验证通过后锁定代付人账户中相应的钱

4、中继服务器RelayServer对元交易进行签名并发送给RelayHub合约(这一步中继服务器支付了gas)

6、RelayHub合约调用内部的可信转发器Trusted Forwarder，对交易的签名和nonce进行校验,

7、可信转发器校验通过后调用接收者合约RelayRecipient

8、接收者合约RelayRecipient会将msg.sender重写替换为原始客户端Client, 供后续合约业务逻辑使用

9、对代付人扣款(可以额外收取一定的服务费)并支付给中继服务器RelayServer

10、代付人对用户进行扣款



使用ERC20代币支付Gas费流程

![image-20240418134843886](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240418134843886.png)



代付人合约多种基础实现:

https://github.com/opengsn/gsn/tree/master/packages/paymasters/contracts


![image-20240418142121535](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240418142121535.png)

OpenGSN文档

https://docs.opengsn.org/javascript-client/tutorial.html



| EIP-4337账户抽象                                            | OpenGSN-元交易标准                                  |
| ----------------------------------------------------------- | --------------------------------------------------- |
| 交易上链由以太坊矿工完成,去中心程度高                       | 交易上链由中继服务器完成,中心化程度高               |
| 业务合约不用改动                                            | 业务合约必须要继承BaseRelayRecipient合约            |
| 必须要为每个用户创建一个智能合约钱包且需要使用4337 兼容钱包 | 用户不需要修改, MetaMask 等现有钱包可以继续正常使用 |

![image-20240419161014698](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240419161014698.png)

https://github.com/eth-infinitism/account-abstraction