# Web3 钱包架构设计

## 一、设计目标

**模块化**: 各个功能模块职责清晰，易于维护和扩展

**多钱包支持**: 支持各种钱包提供者（自研LinklogisProvider、MetaMask、WalletConnect等）

**多账户：**支持管理多个账户的连接使用

**多签名:** 支持多种签名方式，EIP1155、EIP4337、多签KMS等签名机制

**多链支持**: 通过适配器模式支持不同的区块链

**多环境：**适用于浏览器、移动端或 Node.js 等不同运行环境

**类型安全**: 使用TypeScript提供完整的类型定义

**事件驱动**: 通过事件系统实现响应式更新

**错误处理**: 统一的错误处理机制

**可扩展**: 支持插件系统，方便添加新功能

**安全性**: 密钥管理和存储加密



┌────────────────────────┐
│   应用接口层  │ ← 钱包 UI / App 使用的 SDK 接口，事件系统，错误处理
└────────────────────────┘
         			 ↓
┌────────────────────────┐
│   核心服务层│ ← 账户管理、交易管理、签名服务、密钥管理服务等
└────────────────────────┘
         			 ↓
┌────────────────────────┐
│  基础服务层│ ← Provider 、多链适配、存储、网络通信等
└────────────────────────┘



## 应用层 

#### **1. 钱包SDK 主入口**

- 账户管理：connect、getAccount
- 网络切换：switchNetwork
- 交易发送：sendTransaction
- 事件监听：on/off

#### **2. 事件系统**

- 账户状态事件：accountConnect/disconnect、accountChanged
- 网络事件：chainChanged
- 交易生命周期事件事件：transactionSent、transactionConfirmed

#### **3. 错误处理**

- ErrorCode 转换和处理



## 核心服务层 (Modules)    		

#### 1. 账户管理模块 `AccountManager`

- 支持账户连接/断开
- 支持多种账户来源：
- - 助记词EOA
  - 合约钱包
  - 硬件钱包（Ledger、Trezor）
  - 社交登录（WebAuthn、OAuth）
  - 企业级账户（KMS、HSM）
- 支持账户恢复
- 支持Session Key
- 账户状态监听
- 支持 EIP-7702 升级（EOA升级为智能合约账户）

#### 2. 网络模块 `NetworkManager`

- 管理多链网络（EVM链、非 EVM，如 Aptos、Solana、XRPL）
- 切换网络、断网重连
- 自定义 RPC provider（支持 ethers.js、viem、web3.js 等）

#### 3. 交易模块 `TxManager`

- 构建交易（转账 / 合约调用）、批量交易
- 模拟执行（estimateGas）
- 签名与发送交易、发送并等待交易结果
- 支持 ERC-4337 打包交易（UserOperation）、payment使用
- 支持 Permit2 授权

#### 4. 签名模块 `Signer`

- 支持多种签名方式（EOA、ERC-4337）：signTransaction
- 支持离线签名（ EIP-712、TypedData ） ：signTypedData
- 支持分片多签
- 支持其他签名方式：助记词 / 硬件钱包 / 社交登录/ HSM

#### 5. 密钥管理服务

- generateMnemonic
- deriveAccount
- encryptPrivateKey / decryptPrivateKey

#### 6. 插件系统 `PluginManager` 

- 用于扩展功能，如 ENS、ZKP 认证、RWA 插件、EIP-4361 (SIWE)、DID等
- 插件生命周期管理（init、destroy、hook）

#### 7. 跨链与网络扩展

- WalletConnect V2

- CAIP-2 (Chain Agnostic Improvement Proposals): 链 ID 标准化（`eip155:1`）

  CAIP-10: Account ID 标准（支持链+地址组合）



## 基础服务层

1. #### 链适配器（`ChainAdapter`）

   统一封装链特定逻辑（getBalance、sendTransaction、estimateGas、getTransactionReceipt）

   支持链的特定方法扩展

2. #### 钱包提供者providre（`ProviderInterface`）

   自研LinklogisProvider

   其他Provider，如MetaMaskProvider、WalletConnect 

3. #### 存储适配器

​	浏览器（localStorage / IndexedDB），get、set、remove、clear

​	其他安全存储实现（加密）





## LinklogisProvider设计

所属：基础服务层

职责：连接节点、签名交易、提供链上交互能力

标准遵循：

- **EIP-1193**：定义了与钱包交互的最小通用接口
- **EIP-5792**：AA 与 Bundler 支持的智能合约钱包扩展接口

建议实现方式：

- 基于 `ethers.providers.JsonRpcProvider` 扩展
- 内置 signer 签名管理（支持本地/远程）

可配合 wagmi、Web3Modal 、RainbowKit 等钱包对接，推荐研究下Wagmi

Provider 继承拓展自 ethers.js Provider/Signer，然后将Provider 导入到开源钱包中使用





请求中的FlowNo字段问题

request中具体方法的参数未符合标准



密码unlock机制



打开密码输入的交互ui组件能否成为provider的一部分，供其他应用wallet调用我们provider时弹出相应的ui交互？

鉴权逻辑和交互UI组件能否放入provider的connect方法中，调用connect是弹出交互登录？

getGasPrice需要针对不同网络做特殊处理

现有method和参数不规范，比如sendSignedTransaction改成sendRawTransaction，

没有特殊定制的方法可以直接fall到etherprovider的方法中

checkBalance通过estimategas的报错进行触发，而不是在estimategas后去调用

请求中的contractFunctionAbi相关没必须要在provider中，而是放在钱包中

provider去除请求中的signFlowNo概念，而是放在钱包中

pollingTransactionStatus轮询结果是原生方法？或者订阅

permit方法放在钱包层

使用viem

多签设计，多签是钱包的能力，而不是provider的能力





EIP-1193 Provider — request({ method })

├─ ✅ 基本钱包方法（EIP-1193 / JSON-RPC）
│   ├─ eth_accounts
│   ├─ eth_requestAccounts
│   ├─ eth_chainId
│   ├─ eth_sendTransaction
│   ├─ eth_call / estimateGas / getBalance / getTransactionCount / getCode
│
├─ 🟩 签名方法
│   ├─ personal_sign
│   ├─ eth_signTypedData / v3 / v4
│
├─ 🟨 wallet_* 扩展
│   ├─ wallet_switchEthereumChain
│   ├─ wallet_addEthereumChain
│   ├─ wallet_watchAsset
│
├─ 🟧 ERC-4337（可选）
│   ├─ eth_sendUserOperation
│   ├─ eth_estimateUserOperationGas
│
└─ 🔔 Events (.on)
    ├─ accountsChanged
    ├─ chainChanged
    └─ disconnect