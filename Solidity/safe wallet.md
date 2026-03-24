## Gonosis Safe

Gnosis Safe多签钱包是以太坊最流行的多签钱包，Safe 广泛应用于个人资产管理、DAO 治理、企业财务等领域，管理近千亿美元资产，合约经过审计和实战测试，支持多链（以太坊，BSC，Polygon等）， 已融资1亿美金



例如：设置为 2-of-3 的 Safe 钱包由三个账户共同拥有，需要至少两个账户的签名才能执行交易。

假设 Alice、Bob 和 Charlie 是该钱包的共有者。Alice 提出向另一地址转账 1 ETH 的提案，并已完成自己的签名，交易提案添加自己的签名。Bob 或 Charlie 在审查提案无误后，可以在交易提案中添加他的签名。一旦签名达到所需数量，任何人均可执行该交易提案，发送到Safe智能合约钱包，但执行该交易的人需要支付 gas 费用。



Safe 不仅支持原生代币（如ETH），还支持各种 ERC20 代币和 ERC721 NFT。用户可以在 Safe 钱包界面直接查看和管理这些资产。

Safe 提供了与多个 DeFi 协议的原生集成，例如：

- Uniswap：直接在 Safe 界面进行代币交换。
- Aave：管理借贷头寸。

![image-20240527203231600](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240527203231600.png)

### 使用场景：

企业资金管理

对于企业来说，Safe 提供了一个理想的资金管理解决方案：

- 多人授权：防止单点故障和内部欺诈。
- 交易透明：所有交易都可以在链上追踪和审计。
- 灵活控制：可以为不同级别的支出设置不同的审批流程。

DAO 治理

在去中心化自治组织（DAO）中，Safe 可以作为国库管理工具：

- 提案执行：通过多签机制执行已通过的提案。
- 资金分配：根据社区决策管理和分配资金。
- 权限管理：可以为不同的工作组设置不同的权限。

个人资产保护

对于持有大量加密资产的个人，Safe 提供了额外的安全层：

- 防止私钥丢失：即使丢失一个私钥，也不会影响资金安全。
- 继承计划：可以设置时间锁和继承人，确保资产可以被合法继承。
- 分散风险：可以将资产分散到多个设备或地理位置。



### Safe合约架构

Gonosis Safe采用模块化架构, 各模块可插拔的, 因此可以根据具有业务需要使用相应模块

1. **工厂合约（GnosisSafeProxy）**：用于创建一个Safe代理合约
2. **代理合约（GnosisSafeProxy）**：每个 Safe 钱包实例都是一个代理合约，指向主合约的实现通过委托调用（delegatecall），SafeProxy合约存储了所有者、阈值和实现地址。
3. **主合约（GnosisSafe）**：负责管理safe钱包的核心逻辑，相当于逻辑合约，无状态，包括所有者管理、交易执行等，并定义了集成插件、Hook、函数处理器和签名验证器。
4. **模块管理（ModuleManager）**：可插拔的功能扩展，如支付流、恢复机制、会话密钥、hook、社交恢复等。
5. **防护模块（GuardManager）**：用于增加额外的安全检查（制定特定地址才能执行、重入保护、允许/禁止 delegatecall 特定地址等）。

![image-20240527203327427](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20240527203327427.png) 



### 多签流程:

1. 第一个签名者提出交易
   1. 创建交易：定义金额、目的地和任何其他数据
   2. 在提议之前对交易进行链下签名
   3. 将交易和签名提交给safe交易服务
2. 第二位签名者确认交易
   1. 从 Safe 服务获取待处理交易
   2. 执行交易的链下签名
   3. 将签名提交给服务
3. 任何人执行交易
   1. 在此示例中，第一个签名者执行交易
   2. 任何人都可以从 Safe 服务获取待处理的交易
   3. 执行交易的账户支付 gas 费

其中交易服务器可以使用官方服务器也可以自己部署

该服务使多位签名者协同完成多签，通过服务提供的RESTFul API接口将签名发送到服务以收集它们，获取有关的交易信息（如读取交易历史记录、待处理交易、启用的模块和保护等）以及其他功能。



### 具体签名类型

Safe 支持不同类型的签名, 各签名者在链下进行各自的签名,

- ECDSA 签名

  该签名让用户可以直接使用小狐狸钱包进行签名

  对于EOA账户, 其在链下使用标准的ECDSA椭圆曲线加密算法对EIP-712定义的消息进行签名以批准交易, 签名结果为65字节的

  ```
  {32-bytes r}{32-bytes s}{1-byte v}
  ```

  在链上使用自带的recover函数即可进行签名恢复验证

  

- EIP-1271签名

  该签名让用户可以使用自定义实现的签名算法

  IERC1271是一个提供自定义验签名过程的合约对外的接口标准, 传参为待验证消息的hash和签名, 签名的验证过程可以自定义实现, 如果验签通过返回0x1626ba7e，即bytes4(keccak256("isValidSignature(bytes32,bytes)")

  ```solidity
  interface IERC1271 {
      function isValidSignature(bytes32 hash, bytes memory signature) external view returns (bytes4 magicValue);
  }
  ```

  

  signature字段在GnosisSafe的编码规则中为

  ```
  {32-bytes signature verifier}{32-bytes data position}{1-byte signature type}{32-bytes signature length}{bytes signature data}
  ```

  verifier: 实现 EIP-1271 接口的合约的地址

  position: 相对于签名数据开头的偏移量,用于指定签名数据

  type: 0, 用于标记为EIP-1271签名类型

  

  length: 指定签名长度

  data: 签名数据

  

  safe合约通过外部调用实现IERC1271的合约的`isValidSignature`方法对签名进行验证

  ```solidity
  ISignatureValidator(addr).isValidSignature(data,sig) == EIP1271_MAGIC_VALUE
  ```

  

- 预签名

  预签名是一种链上签名的解决方案, 其解决的是智能合约没有私钥，但要对一笔交易进行签名的问题, 签名和验签的过程都是在链上进行的

  某个合约调用另一个合约进行

  相对其他两种签名, 多了一步, 因此叫预签名

  在多签钱包合约中维护一个全局变量
  
  ```solidity
  mapping(address => mapping(bytes32 => uint256)) public approvedHashes;
  ```

  签名过程实际类似授权的过程, 要进行签名的合约调用钱包approveHash函数, 传入签名数据的hash, 该函数会判断调用者是否属于多签账户中的一个, 是的话将账户和签名数据hash的映射存储起来, 完成签名
  
  ```solidity
  function approveHash(bytes32 hashToApprove) external {
  	require(owners[msg.sender] != 0, "GnosisSafe/approveHash msg.sender is not an owner");
  	approveHashes[msg.sender][hashToApprove] = uint256(1);
  }
  ```

  验证签名时, 向mapping查询是否有此映射, 来确认某个账户对一笔交易哈希是否已经预先签名
  
  ```solidity
  if (v == 1) {
  	address currentOwner = address(uint160(uint256(r)));
  	require(approvedHashes[currentOwner][dataHash] != 0 || msg.sender == currentOwner);
  }
  ```

  GnosisSafe的预签名编码规则:
  
  ```
  {32-bytes hash validator}{32-bytes ignored}{1-byte signature type}
  ```

  validator: 预签名的账户地址

  ignored: 占位符填充0, 该字段未使用
  
  type: 1, 用于标记该签名为预签名



最后

多个签名者在使用各自的签名类型对交易签名后, 执行者将所有签名将合并为一个签名

```
"0x" + 
"cf06921ff2AD4c99dF77dcF741d6ce51061dE64300000000000000000000000000000000000000000000000000000000000000c300" + // EIP-1271签名
"Ccd5c9a1C54cE1521877Cc93124cb3513C2f7684000000000000000000000000000000000000000000000000000000000000000001" + // 预签名
"bde0b9f486b1960454e326375d0b1680243e031fd4fb3f070d9a3ef9871ccfd57d1a653cffb6321f889169f08e548684e005f2b0c3a6c06fba4c4a68f5e006241c" + // ECDSA签名
"00000000000000000000000000000000080000000000000000000000000000deadbeef"// EIP-1271的签名数据部分, 长度和签名数据
```

传输到智能合约钱包合约进行验证和执行。

验证过程:

执行一个for循环分别验证签名组合中的每一个签名，因为三种签名的编码格式都是{32-bytes}{32-bytes}{1-byte}, 

因此在每一个循环内部从sigatures里拿到对应的`r,s,v`值作为相应位置的变量进行后续验证





### Safe使用流程

可以直接使用官方提供js/ts脚本来进行账户创建，另外脚本还支持账户管理, 交易提出和签名, 交易执行等功能

https://github.com/safe-global/safe-core-sdk



创建一个Safe多签钱包：

```typescript
const { SafeFactory } = require('@safe-global/protocol-kit');

const safeFactory = await SafeFactory.init({
  provider: '<provider url>', // 替换为实际的 RPC url
  signer: '<sender private key>', // 私钥，用于部署多签钱包合约，该私钥对应的账户不一定是多签的签名者
})

// SafeAccountConfig结构体中包含了多个签名方以及多签阈值
const safeAccountConfig = {
  owners: [
    '<signer address 1>', // 替换签名者地址
    '<signer address 2>', // 替换签名者地址
    '<signer address 3>', // 替换签名者地址
  ],
  threshold: 2, 
}

// 部署多签钱包合约
const safeWallet = await safeFactory.deploySafe({ safeAccountConfig });

// 查看部署的钱包
const safeAddress = await safeWallet.getAddress();
console.log('Safe 钱包已部署');
console.log(`https://app.safe.global/sep:${safeAddress}`);
```

该过程实际是调用GnosisSafeProxyFactory工厂合约创建了一个新的GnosisSafeProxy钱包代理合约，

初始化钱包参数，存储在该代理合约上, 并将该代理合约指向一个存在的GnosisSafe实现合约，

对GnosisSafeProxy钱包代理合约的所有函数调用都通过`delegatecall`的方式远程调用`GnosisSafe`逻辑合约。



读取多签钱包相关信息

```javascript
  const Safe = require('@safe-global/protocol-kit').default;

  const safeWallet = await Safe.init({
      provider: '<provider url>', // 替换为实际的 RPC url
      safeAddress: '<safeAddress>', //替换为实际的多签钱包地址
  })

  const threshold = await safeWallet.getThreshold();
  const owners = await safeWallet.getOwners();

  console.log(`Threshold: ${threshold}`); // 阈值（最小签名数量）
  console.log(`Owners: ${owners.join(', ')}`); // 签名者
```



发起一笔 ETH 转账多签交易（前提是该多签钱包有足够的 ETH，提前充值好）

第一位签名者签名

```javascript
const Safe = require('@safe-global/protocol-kit').default;
const SafeApiKit = require('@safe-global/api-kit').default;

const safeWallet = await Safe.init({
    provider:  '<provider url>', // 替换为实际的 RPC url
    signer: '<signerPrivateKey>', // 替换为多签钱包签名者1的私钥
    safeAddress: '<safeAddress>', //替换为实际的多签钱包地址
})

const apiKit = new SafeApiKit({
  chainId: await safeWallet.getChainId(),
})

// 转账 0.01 ether
const safeTransactionData = {
  to: '<receiver>', // 替换为收款人地址
  data: '0x',
  value: ethers.parseUnits('0.01', 'ether').toString()
}


// 创建多签交易
const safeTransaction = await safeWallet.createTransaction({ 'transactions': [safeTransactionData] });

// 获取这笔交易的哈希值
const safeTxHash = await safeWallet.getTransactionHash(safeTransaction);

// 对交易哈希值进行签名
const senderSignature = await safeWallet.signHash(safeTxHash);

// 提交交易到 Safe 服务（这样，其他签名者可以在 Safe 网站中看到待签名的交易）
await apiKit.proposeTransaction({
  'safeAddress': '<safe address>', // 替换为实际的多签钱包地址
  'safeTransactionData': safeTransaction.data,
  'safeTxHash': safeTxHash,
  'senderAddress': new Wallet(signerPrivateKey).address,
  'senderSignature': senderSignature.data,
})
```

第二位签名者签名交易

```javascript
const Safe = require('@safe-global/protocol-kit').default;
const SafeApiKit = require('@safe-global/api-kit').default;

const safeWallet = await Safe.init({
    provider:  '<provider url>', // 替换为实际的 RPC url
    signer: '<signerPrivateKey>', // 替换为多签钱包签名者2的私钥
    safeAddress: '<safeAddress>', //替换为实际的多签钱包地址
})

const apiKit = new SafeApiKit({
  chainId: await safeWallet.getChainId(),
})

const transaction = (await apiKit.getPendingTransactions(safeAddress)).results[0];

if (!transaction) return;

const safeTxHash = transaction.safeTxHash;

const signature = await safeWallet.signHash(safeTxHash);
const response = await apiKit.confirmTransaction(safeTxHash, signature.data);
console.log('已确认交易:\n', response);
```

执行交易

```javascript
const Safe = require('@safe-global/protocol-kit').default;
const SafeApiKit = require('@safe-global/api-kit').default;

const safeWallet = await Safe.init({
    provider:  '<provider url>', // 替换为实际的 RPC url
    signer: '<signerPrivateKey>', // 替换执行交易人的私钥
    safeAddress: '<safeAddress>', //替换为实际的多签钱包地址
})

const apiKit = new SafeApiKit({
  chainId: await safeWallet.getChainId(),
})

const transaction = (await apiKit.getPendingTransactions(safeAddress)).results[0];

if (transaction.confirmations.length !== transaction.confirmationsRequired) {
  console.log('未达到签名确认数');
  return;
}

const safeTransaction = await apiKit.getTransaction(transaction.safeTxHash);
const executeTxResponse = await safeWallet.executeTransaction(safeTransaction);
const receipt = await executeTxResponse.transactionResponse?.wait();

console.log('交易已执行，hash：', receipt.hash);
```