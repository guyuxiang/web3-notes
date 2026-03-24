# Gas优化

1. **建立基线** - 在优化前先测量当前性能
2. **使用多种工具** - 结合 gas report、snapshot 和 gasleft()
3. **版本控制** - 提交 `.gas-snapshot` 文件
4. **关注关键路径** - 重点优化高频调用的函数
5. **验证优化效果** - 优化后必须再次测量
6. 

✅ **提交到版本控制** - 将 `.gas-snapshot` 文件提交到 Git
✅ **在 PR 中审查** - 关注 gas 消耗的变化
✅ **设置告警阈值** - 显著的 gas 增加应该引起注意
✅ **为关键函数编写专门测试** - 确保核心功能的 gas 效率



## Gas 成本基础

### EVM 操作码 Gas 成本（精选）

| 操作          | Gas 成本      | 说明                             |
| ------------- | ------------- | -------------------------------- |
| `ADD/SUB`     | 3             | 算术运算                         |
| `MUL/DIV`     | 5             | 乘除运算                         |
| `SLOAD`       | 100-2100      | 读取存储（首次100，后续2100）    |
| `SSTORE`      | 20,000-22,100 | 写入存储（新值20000，更新22100） |
| `BALANCE`     | 700           | 查询余额                         |
| `CALL`        | 至少 700      | 外部调用                         |
| `CREATE`      | 32,000        | 创建合约                         |
| `CREATE2`     | 32,000+       | 带盐的合约创建                   |
| `EXTCODESIZE` | 700           | 获取合约代码大小                 |

成本**较高**的操作包括：

- 读写存储在合约存储中的状态变量
- 外部函数调用
- 创建合约
- 循环操作



## viaIR 优化

`viaIR` 是 **Solidity 编译器的一种编译优化路径**，意思是：
 **先把 Solidity 编译成 IR（Intermediate Representation，中间表示），再从 IR 生成 EVM Bytecode**，而不是直接从 Solidity AST 生成 bytecode。

简单说就是：

```
Solidity源码
   ↓
AST
   ↓
IR (Yul Intermediate Representation)
   ↓
优化
   ↓
EVM Bytecode
```

而传统流程是：

```
Solidity源码
   ↓
AST
   ↓
直接生成 EVM Bytecode
```

### 一、什么是 IR（Intermediate Representation）

Solidity 的 IR 其实就是 **Yul**。

Yul 是一种 **低级中间语言**，接近 EVM，但比汇编更结构化。

### 三、viaIR 的作用（为什么要用）

主要是为了 **更强的优化能力**。

Solidity 团队现在把优化重点都放在 IR 上。

主要优势：

### 1 更好的 Gas 优化

很多情况下能减少 gas。

例如：

```
20% ~ 30%
```

特别是：

- 循环
- 内存操作
- 复杂函数
- struct
- ABI编码

------

### 2 解决 Stack too deep

Solidity 经典报错：

```
Stack too deep
```

EVM stack 只有：

```
16 slots
```

viaIR 可以自动：

- 变量复用
- spill到memory

所以很多 `stack too deep` 会自动消失。

------

### 3 更好的函数内联

IR optimizer 会自动：

- inline
- dead code elimination
- constant folding

例如：

```
uint a = 1 + 2;
```

编译后直接变：

```
3
```

------

### 4 更好的 ABI 编码优化

比如：

```
abi.encode
abi.decode
```

gas 会更低。







# 技巧

## 1.  尽可能避免频繁从非零到零的写入

当存储变量从零变为非零时，用户必须支付总共22,100 gas（20,000 gas 用于从零到非零的写入，2,100 gas 用于冷存储访问）。

这就是为什么 Openzeppelin 的重入保护使用1和2来注册函数的活动状态，而不是0和1。将存储变量从非零更改为非零只需花费5,000 gas。



## 2.  尽量减少存储的使用，缓存存储变量

缓存存储变量：仅写入和读取存储变量一次



## 3. 变量打包

将相关变量打包到同一个槽位中可以通过最小化昂贵的存储相关操作来减少 gas 成本。

**手动打包是最高效的**

我们通过位移操作将两个 uint80 值存储在一个变量（uint160）中。这样只使用一个存储槽位，在单个事务中存储或读取各个值时更便宜。



## 4. 打包结构体

像打包相关状态变量一样，打包结构体成员可以帮助节省 gas 。（需要注意的是，在 Solidity 中，结构体成员按顺序存储在合约的存储中，从它们初始化的槽位位置开始）。





## 5. 使用固定大小变量替代动态变量

## 保持字符串长度小于32字节

如果数据可以控制在32字节内，建议使用bytes32数据类型替代bytes或strings。一般来说，固定大小的变量比可变大小的变量消耗的Gas更少。如果字节长度可以限制，尽量选择从bytes1到bytes32的最小长度。

在 Solidity 中，字符串是可变长度的动态数据类型，意味着它们的长度可以根据需要进行更改和增长。

如果长度为32字节或更长，它们定义的槽位中存储的是字符串长度 * 2 + 1，而实际数据存储在其它位置（该槽位的 keccak 哈希值）。

然而，如果字符串长度小于32字节，长度 * 2 存储在其存储槽位的最低有效字节中，并且字符串的实际数据从定义它的槽位的最高有效字节开始存储。

**字符串示例（小于32字节）**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract StringStorage1 {
    // Uses only one slot
    // slot 0: 0x(len * 2)00...(hex"hello")
    // Has smaller gas cost due to size.
    string public exampleString = "hello";

    function getString() public view returns (string memory) {
        return exampleString;
    }
}
```

**字符串示例（大于32字节）**

```solidity
contract StringStorage2 {
    // Length is more than 32 bytes. 
    // Slot 0: 0x00...(length*2+1).
    // keccak256(0x00): stores hex representation of "hello"
    // Has increased gas cost due to size.
    string public exampleString = "This is a string that is slightly over 32 bytes!";

    function getStringLongerThan32bytes() public view returns (string memory) {
        return exampleString;
    }
}
```





## 6. 使用不可变的或常量

在 Solidity 中，不打算更新的变量应该是常量或不可变的。

这是因为常量和不可变值直接嵌入到它们所定义的合约的字节码中，不使用存储空间。





## 7. 使用瞬时存储

瞬时存储（Transient Storage）是 2024年3月 Cancun 升级引入的新型存储方式。它使用两个新操作码：

- `TSTORE`：写入瞬时数据（100 gas）
- `TLOAD`：读取瞬时数据（100 gas）

**关键特性：**

- 数据仅在单个交易期间存在
- 交易结束后自动清零
- [Gas](https://learnblockchain.cn/tags/Gas?map=EVM) 成本仅 100 gas（相比 SSTORE 的 22,100 gas，节省 **99.5%**）

瞬时存储特别适合需要在**单个交易内**共享状态的场景：

1. 重入锁（Reentrancy Guard）
2. 闪电贷状态管理
3. 批量操作的中间状态
4. 跨合约调用的瞬时标记

**使用瞬时存储（Solidity 0.8.24+）**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract TransientReentrancyGuard {
    // 使用 transient 关键字声明瞬时存储变量
    bool private transient _locked;

    modifier nonReentrant() {
        require(!_locked, "ReentrancyGuard: reentrant call");
        _locked = true;   // TSTORE: 100 gas
        _;
        _locked = false;  // TSTORE: 100 gas
    }

    function withdraw() public nonReentrant {
        // 业务逻辑
    }
}
// 总成本：仅 200 gas（节省 96%+）
```





## 8. 使用映射而不是数组

使用映射而不是数组以避免长度检查,当存储你希望按特定顺序组织并使用固定键/索引检索的项目列表或组时，通常使用数组数据结构是常见的做法。

在底层，当你读取数组的索引值时，Solidity 会添加字节码来检查你是否正在读取有效的索引（即索引严格小于数组的长度），否则会回滚并显示恐慌错误（具体为 Panic(0x32)）。这样可以防止读取未分配或更糟糕的已分配存储/内存位置。

由于映射的方式是（简单的键=>值对），不需要进行这样的检查，我们可以直接从存储槽中读取。重要的是要注意，当以这种方式使用映射时，你的代码应确保不要读取超出规范数组索引的位置。



## Calldata 优化

**calldata 是交易输入数据所在的独立只读区域**。
 它 **不在 storage、memory、stack 中的任何一个**，而是 **EVM 专门的一块只读数据区**。



记住这个原则：如果函数参数是只读的，应优先使用calldata而非memory。这样可以避免从函数calldata到memory的不必要复制操作。

## Calldata （通常）比Memory更便宜

直接从 calldata 中加载函数输入或数据比从内存中加载更便宜。这是因为从 calldata 访问数据涉及的操作和 gas 成本较少。因此，建议仅在函数需要修改数据时使用内存（calldata 无法修改）。

## Cancun 升级后的 Calldata 优化策略

在 2024 Cancun 升级后，EIP-4844 引入的 Blob 交易，Blob 使用独立的 gas 计价机制, 并且费用大幅降低，L2 现在主要使用 Blob 交易发布数据到 L1，Blob 不区分零字节和非零字节。
因此在 L2 网络：Calldata 优化重要性降低。



## 使用内联汇编代码

手动计算 mapping / struct slot

#### 直接存储访问

```solidity
// ❌ 标准Solidity
function getValue(uint256 slot) public view returns (uint256) {
    return storageArray[slot];  // 边界检查 + SLOAD
}

// ✅ 内联汇编（无边界检查）
function getValueUnsafe(uint256 slot) public view returns (uint256 value) {
    assembly {
        value := sload(slot)  // 直接SLOAD，无边界检查
    }
    // 警告：需确保slot有效，否则可能读取任意存储
}

// ✅ 安全的内联汇编
function getValueSafe(uint256 slot) public view returns (uint256) {
    require(slot < storageArray.length, "Invalid slot");
    uint256 value;
    assembly {
        value := sload(add(storageArray.slot, slot))
    }
    return value;
}
```







## 9. 在数组上使用 unsafeAccess

使用 unsafeAccess 在数组上避免冗余的长度检查.

使用映射来避免 [Solidity](https://learnblockchain.cn/course/93) 在读取数组时进行的长度检查（同时仍然使用数组）的另一种方法是使用 Openzeppelin 的 [Arrays.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Arrays.sol) 库中的 unsafeAccess 函数。这使开发人员可以直接访问数组中任意给定索引的值，同时跳过长度溢出检查。但是，仅在确保传递给函数的索引不会超过传递的数组的长度时才使用此方法。



## 10. 使用布尔位图

在使用大量布尔值时，使用位图而不是布尔值。
一个常见的模式，特别是在空投中，是在领取空投或 NFT 时将地址标记为“已使用”。

然而，由于只需要一个位来存储这些信息，而每个存储槽是 256 位，这意味着可以使用一个存储槽存储 256 个标志/布尔值。



# 部署时节省 gas

## 1. 关于部署gas

**CREATE 和 CREATE2 合约部署成本：**

```undefined
部署成本 = 基础成本 + 代码存储成本 + 初始化成本
```

基础成本：
CREATE: 32,000 Gas
CREATE2: 32,000 Gas（相同）

代码存储成本：
200 Gas/字节

示例：
合约字节码大小：10,000 字节

成本计算：

- 基础：32,000 Gas
- 代码存储：10,000 × 200 = 2,000,000 Gas
- 构造函数执行：变量（例如 500,000 [Gas](https://learnblockchain.cn/tags/Gas?map=EVM)）
  总计：约 2,532,000 [Gas](https://learnblockchain.cn/tags/Gas?map=EVM)



## 2. 大型合约优化

根据 EIP-170 合约大小限制最大为：24,576 字节 (24 KB)
超出限制的解决方案：

### 将合约分开

1. 将合约分开应该永远你的首要方法。如何把合约分成多个小合约

2. Diamond Pattern（钻石模式）

   拆分功能到多个 Facet

   通过 Diamond Proxy 聚合

3. 使用库（Library）



### runs 的核心含义

在 Solidity 编译器里，`optimizer.runs` 是 **优化器的一个重要参数**，它告诉编译器：

> **预计这个合约里的函数会被执行多少次**

编译器会根据这个“运行次数预期”在 **部署成本（deployment gas）** 和 **运行成本（runtime gas）** 之间做权衡。

------

# 

```
optimizer: {
  enabled: true,
  runs: 200
}
```

这里的 `runs = 200` 意思是：

> 编译器假设这个函数平均会被调用 **200 次**

于是优化策略会偏向 **降低运行 gas**，而不是降低部署 gas。







## 10. 使用自定义错误

[自定义错误](https://learnblockchain.cn/article/22557?course_id=93#revert() - 灵活的错误处理)（通常）比 require 语句更小，由于自定义错误的处理方式，它们比使用字符串的 require 语句更便宜。Solidity 只存储错误签名的前4个字节，并且只返回这4个字节。这意味着在回滚时，只需要在内存中存储4个字节。而对于 require 语句中的字符串消息，[Solidity](https://learnblockchain.cn/course/93) 必须至少存储（在内存中）并回滚64个字节。



## 11. 使用 SSTORE2、SSTORE3

### SSTORE2

它使用合约的字节码来写入和存储数据，写入数据创建合约，读取数据从合约字节码中计算读取

### SSTORE3

SSTORE3 实现了这样一个设计，即新部署的地址与我们提供的数据无关。能够通过仅提供盐值（可以少于 20 字节）高效地计算出数据的指针地址



#### 12. 批量存储操作

```solidity
// ❌ 多次单独存储
function updateMultiple() public {
    userData.name = "Alice";
    userData.age = 30;
    userData.balance = 1000;
    // 3次SSTORE = 60,000+ Gas
}

// ✅ 使用结构体一次性更新
struct UserData {
    string name;
    uint256 age;
    uint256 balance;
}

UserData private userData;

function updateUser(UserData memory newData) public {
    userData = newData;  // 1次SSTORE（如果结构体打包）
}
```

## 



## 12. 倒数计数

从 n 倒数到零，而不是从零到 n 进行计数
当将存储变量设置为零时，会获得退款，因此如果存储变量的最终状态为零，则计数所花费的净 gas 将更少。



## 13. 时间戳和区块号

存储中的时间戳和区块编号不需要是 uint256
一个大小为 uint48 的时间戳可以工作数百万年。一个区块编号每 12 秒递增一次。这应该让你对合理的数字大小有所了解。