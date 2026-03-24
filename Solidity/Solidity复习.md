# Solidity复习

库（Library）不能有Fallback函数和payable关键字

，也不能定义storage变量，但在合约里调用库里的函数时，可以把合约里的storage变量作为引用传给库合约



变量分为固定大小的变量和动态变量



Address类型地址的长度是8字节 160位



引用类型：Bytes、String、Array、Struct



Bytes:动态大小字节数组

string:动态大小UTF-8编码字符串



声明动态数组

动态数组 不指定长度。

uint[] public numbers;

特点：

- 初始长度为 `0`
- 可以随时增加元素
- 可以通过 `length` 查看长度

添加元素（push）

获取数组长度 numbers.length;

读取指定元素 numbers[index]

删除数组元素：Solidity 不能直接删除某个 index 并自动移动元素

方法1 delete
delete numbers[1];


效果：

[10,20,30]

delete index 1

[10,0,30]


长度 不变。

方法2 pop（删除最后一个）
numbers.pop();

方法3 swap + pop（最常用）

用于 删除指定 index。

function remove(uint index) public {
    numbers[index] = numbers[numbers.length - 1];
    numbers.pop();
}



创建 memory 动态数组

在函数内部：

uint[] memory arr = new uint[](size);

注意：

memory数组不能 push

必须指定长度



越界
numbers[5]

如果不存在会 revert。





mapping 默认值:

**mapping 不会判断 key 是否存在**。返回零值



mapping 删除

删除某个 key：

delete balances[user];



# mapping 无法遍历（重要）

mapping **不能循环**。





mapping + array（最常见结构）

用于遍历，比如
mapping 存余额
array 记录用户列表



# 嵌套 mapping

mapping 可以嵌套。



# mapping gas优势

相比数组：

```
mapping 写入 gas 更低
```

原因：

- 不需要扩展数组
- 不需要维护 length



### 错误1 mapping memory

❌

```
mapping(address => uint) memory balances;
```

mapping **只能 storage**。



栈限制

函数里变量的定义是放在栈上，因为栈限制，太多会报错栈溢出

解决方法是：

拆小函数:把一大段逻辑拆成多个内部函数，让每个函数只处理一部分变量。

把变量封装到结构体或数组里

减少返回值数量



启用 IR 编译优化

via_ir = true
optimizer = true
optimizer_runs = 200



# memory vs calldata

### calldata

只能用于 **external函数参数**。

特点：

- 不可修改
- 最省gas
- 直接从交易输入读取

### memory

函数内部变量。

特点：

- 可以修改
- 存在内存中
- gas 比 calldata 稍高





# 修饰符



internal: 只能内部访问

注意调用internal函数或变量时，不能加前缀this

前缀this表示通过外部方式访问

在状态变量里，internal是默认修饰符

函数要求必须显性设置修饰符，不然编译会报错



external:只能外部访问，不能内部访问

不可以f()，但可以this.f()

external并不会把



Public:既可以外部也可以内部

对于public类型变量，自动创建同名访问器



private:只在当前合约可访问，继承的合约不可访问



constant:用于 **状态变量**

表示 **这个变量是常量，部署后不可修改**。

**不会占用 storage**

编译时赋值，直接写入合约代码中



immutable

在constructor函数中赋值



view

函数无法改变链上的状态变量、不能调用事件、不能转账，不能创建合约、不能调用其他会改变状态的函数

该修饰词还会被sdk判断是通过call请求调用



pure

函数既不能读取状态，也不能修改状态。

只能依赖：

- 函数参数
- 局部变量



Payable:从交易调用者接受ether，但并不要求必须发送 ETH。

如果函数 **没有 `payable`**，发送 ETH 调用该函数时交易会 **revert**。

# msg.value

`msg.value` 表示：

> **本次交易附带的 ETH 数量**

单位是 **wei**。



查看合约余额

合约余额：

address(this).balance



call 转账（推荐）

现代 Solidity 推荐：

(bool success,) = payable(msg.sender).call{value:1 ether}("");
require(success);



如果你**直接向合约地址转账且不调用任何函数**，EVM 会尝试触发合约的 **`receive()` 或 `fallback()`** 函数。这两个函数带有payable修饰符

如果既没有 receive() 也没有 payable fallback()，转账会失败



# 正确使用`assert()`, `require()`, `revert()`

**assert** 和 **require**函数可以用于参数校验，如果不通过则抛出异常。**assert**函数应仅用于检查内部错误和检查不变量。**require**函数更适合用于确保条件满足，如输入或合约状态变量被满足，也可以验证调用外部合约的返回值。



# Extermal vs Public

造成燃料消耗差别的原因在于:在Public函数中， Solidity 马上把数组参数复
制到 memory，而 Extermal函数则直接从calldata读取，从calldaaa读取相比内存的分配比较
便宜。
Public 函数为什么要把所有的参数复制到 memory呢？这是因为 Public 函数可能被Inter-
nal函数调用，这个过程相对于 External的调用而言，是在完全不同的过程里发生的。 Inter-
nal调用通过JUMP指令实现，数组参数是通过指向内存的指针来传递。因而,当编译器在
为 Intermnal函数产生指令时,函数是通过内存来访问参数。对于 Exteral 的数，编译器不允
许Internal调用，所以它允许从 calldata来直接读取参数，而没有复制这个步骤。
总结如下:
1）因为Public函数不知道调用者是Extermal或者 Internal, Public函数都会像处理Inter-
nal 函数一样将参数复制到 memory，而这个操作非常昂贵。
2）如果可以确信函数只能被外部调用，请使用Exteral修饰符。





# 自定义修饰符

可被继承

函数使用了自定义修饰符，那么函数的代码会在修饰器里的 _ 中被填充执行，

_ 之后还可以有自定义修饰器的代码，比如重入锁：

```
locked = true;

_;

locked = false;
```



# 数据位置

storage:全局变量，永久存储

memory:函数内部内存存储，函数执行完销毁

calldatea:函数参数，放在交易里，不可修改



引用类型必须指定位置，否则编译报错



函数里声明storage 可以引用合约的 storage。



函数里声明memory 可以 **复制合约的storage数据**。

calldata 可以被复制到 memory：



# storage 布局

storage 是按 **slot** 存储：slot0、slot1、slot2

slot → 32 bytes (256 bits)

每个变量默认占用 **一个 slot（32 bytes）**。

# 小变量打包（packing）

如果变量小于 32 bytes，Solidity 会 **打包到同一个 slot**。

Solidity **按声明顺序打包**。



# struct 的 storage

struct 也按 slot 存储。

如果struct内的变量小，可能就一个slot就能存放要给strct

如果内部结构多，则按照slot顺序放下放下一个slot



# array storage

数组的 **长度** 存在 slot。

slot0 → arr.length

真正数据在：

```
keccak256(slot0)
```

例如：

```
arr[0] → keccak256(0)
arr[1] → keccak256(0) + 1
arr[2] → keccak256(0) + 2
```





# mapping storage

mapping 不按顺序存储。

mapping 使用：

```
keccak256(key,  slot)
```



# 继承 storage

继承的 storage **按继承顺序排列**。

示例：

```
contract A {
    uint a;
}

contract B {
    uint b;
}

contract C is A, B {
    uint c;
}
```

布局：

```
slot0 → a
slot1 → b
slot2 → c
```

storage layout：

```
1. 最 base 的合约
2. 按继承声明顺序
3. 最后是当前合约
```

Base1 → Base2 → ... → Child



# 4. 事件

最多可以有三个参数设置为indexed，实际事件签名也是一个topic总共 **4个**。

如果数组（包括string、bytes）被标记indexed,会被进行keccak-256哈希后的值作为topic

事件签名也是一个topic

```
log {
  address: 合约地址
  topics: [
    event_signature_hash,
    indexed_param1,
    indexed_param2
  ]
  data: 非 indexed 参数
}
```

事件日志其实存储在 **区块的 logs 中**，而不是 storage。会便宜些

设置索引后运行通过rpc查询过滤

示例：topics 是数组：

```
{
  "jsonrpc":"2.0",
  "method":"eth_getLogs",
  "params":[
    {
      "fromBlock":"0xA00000",
      "toBlock":"latest",
      "address":"0xTokenAddress",
      "topics":[
        "0xddf252ad..."
      ]
    }
  ],
  "id":1
}
```



# 函数返回值



使用模拟执行时可以获取返回值，但是提交交易则无法得到返回值

因为sendTransaction rpc方法的响应是交易哈希，因此推荐方案使用event





# 多重继承

但 Solidity 使用的是 **C3 Linearization（线性化继承顺序）** 来解决继承冲突。

Solidity 的继承解析顺序是：从右到左



# 函数冲突（override）

如果多个父合约 **有同名函数**、修饰器、事件，子合约必须 **override**。

指定具体调用



# super 调用

可以调用父合约实现：

super.f();

`super` 会按照 **继承线性顺序调用**。也就是从右到左



菱形继承

例如：

```
A
├── B
├── C
└── D
```

如果：

```
contract D is B, C
```

顺序：

```
D → C → B → A
```



构造函数继承

```
contract B is A {

    constructor() A(10) {}

}
```



# this

当前合约对象，

可以显式转化为address：address(this)

this = 当前合约地址 + 外部调用接口

this.foo() 是 external call，当需要调用本合约的external函数时，内部调用走不通就是使用this.f()



场景：

1. 需要 msg.sender 变成合约地址 ，比如权限控制只允许当前合约调用

2. 触发 try/catch

try/catch 只能捕获 external call。

在构造函数中无法调用this，因为合约对象还没生成



# 特殊单位

1 ether

1 wei

1 Gwei

...

1 second

1 minute

1hour

...



# 内链汇编

> 在 Solidity 代码中直接写 **EVM 汇编代码**，用于实现更底层、更高效或 Solidity 无法直接实现的逻辑。

Solidity 使用的汇编语言叫 **Yul**。

```
assembly {
    // Yul代码
}
```

通常只有在以下情况才使用：

1️⃣ **gas 优化**
 2️⃣ **访问底层 EVM 指令**
 3️⃣ **处理 memory / storage 布局**
 4️⃣ **编写 proxy / delegatecall**
 5️⃣ **ABI 编码解码**



:=   赋值
add(a,b)  EVM opcode

sload(slot) 读取 storage。

sstore(slot, value)  写 storage



mload	读取 memory
mstore	写入 memory



Solidity 有个 **free memory pointer**：

```
0x40
```

读取：

```
assembly {
    let ptr := mload(0x40)
}
```

意思：

```
当前可用 memory 地址
```



# call 外部合约

汇编调用：

```
assembly {

    let success := call(
        gas(),
        target,
        value,
        input,
        inputSize,
        output,
        outputSize
    )

}
```

参数：

| 参数   | 含义     |
| ------ | -------- |
| gas    | gas      |
| target | 地址     |
| value  | ETH      |
| input  | calldata |
| output | 返回值   |



读取calldata

assembly {

    let x := calldataload(4)

}
解释：

前4字节 → function selector
之后 → 参数



返回数据

assembly {

    mstore(0x00, 123)
    
    return(0x00, 32)

}
意思：

返回 memory[0x00] 的 32 bytes



# 内链汇编场景

1. **读写特定slot，例如 EIP-1967 proxy：**

```
bytes32 constant IMPLEMENTATION_SLOT =
0x360894a13ba1a321...

assembly {
    sstore(IMPLEMENTATION_SLOT, impl)
}
```



2. **省 gas、直接操作 memory/calldata、处理底层 call、做数学和字节解析。**



3. **当合约低级调用另一个合约失败时，希望把对方返回的错误原样抛出来，而不是只写一个固定报错。**





# calldata 解析，尤其是 path 解码

Uniswap V3 有个非常经典的设计：**把 swap path 编码成 bytes**。

比如多跳路径，不是传数组结构体，而是传一个紧凑的 `bytes path`。

这样做比传数组更省 gas。



# FullMath / 位运算优化

Uniswap V3 最有名的一块是数学库，尤其是高精度乘除法。

因为 AMM、价格、tick、liquidity 这些计算，经常需要：

- 超过 256 位中间精度
- 避免精度损失
- 控制 rounding

普通 Solidity 在这些地方会很吃力，所以会用汇编和位操作。





# 合约间调用

call：执行其他合约代码

delegatecall：执行其他合约代码，但是用调用合约的上下文空间





## calldata

#### 带 `calldata` 参数的函数只能是 `external` 函数，因此不能被内部调用（internal call）。Solidity 内部函数调用不会产生新的 calldata



#### 外部调用（external call）

EVM 会执行：

```
CALL
```

调用流程：

```
foo()
   │
   └── CALL
         │
         └── 新的 EVM call frame
               │
               └── 新 calldata
```

此时：

```
入参
```

会被 **ABI encode 成 calldata**：

```
selector + args
```

存放在：

```
新调用的 calldata 区域
```



### call frame（调用帧）

每一次 external call 都会创建一个 **新的 EVM call frame**。

结构：

```
Call Frame
│
├─ Stack
├─ Memory
├─ Calldata
├─ ReturnData
└─ Gas
```

所以：

```
foo() calldata
```

和

```
bar() calldata
```

是 **两个完全独立的 calldata 区域**。





### delegatecall 的情况

如果是：

```
delegatecall
```

例如 proxy：

```
Proxy
   └─ delegatecall
         Implementation
```

此时：

```
calldata 不变
```

Implementation 读取的：

```
还是 proxy 的 calldata
```

这也是 proxy 能工作的原因。





### 获取合约间调用的返回值

1. **强类型接口调用**

编译器帮你做了这些事：

- ABI encode 函数选择器和参数
- 发起 external call
- 拿到 returndata
- 按返回类型自动 decode





1. **低级调用 `call / staticcall / delegatecall`**

那么返回值不是自动解码的，而是拿到：

- `success`
- `bytes memory data`

然后你自己 `abi.decode`



如果 `success == false`，`data` 里通常放的是：

- revert reason
- panic code
- custom error 编码

你可以直接回抛：

```
if (!success) {
    assembly {
        revert(add(data, 32), mload(data))
    }
}
```



1. **`try/catch` 调用**

如果是强类型 external 调用，可以用 `try/catch`。



# ABI 的主要用途

在合约开发里 ABI 主要用于 4 类事情：

1. **编码函数调用数据（call data）**
2. **解码返回值**
3. **构造低级调用**
4. **编码事件或哈希数据**

常见函数：

```
abi.encode
abi.encodePacked
abi.encodeWithSelector
abi.encodeWithSignature
abi.decode
```



# abi.encode（最标准编码）

`abi.encode` 按照 **标准 ABI 编码**生成 bytes。

特点：

- 每个参数 **32字节对齐**
- 动态类型会有 **offset**

适合：

- ABI 标准编码
- 函数调用参数
- call data

# abi.encodePacked（紧凑编码）

`encodePacked` 是 **紧凑编码**。

特点：

```
不做 32 字节填充
```

适合：

- 哈希计算
- EIP-712
- 签名



# abi.encodeWithSignature

用于 **构造函数调用数据**。

```
bytes memory data =
    abi.encodeWithSignature(
        "transfer(address,uint256)",
        to,
        amount
    );
```

生成结构：

```
4 bytes selector
+ 参数ABI编码
```

可以直接用于 `call`：

```
(bool success,) = token.call(data);
```



# abi.encodeWithSelector

比 signature 更高效。

```
bytes memory data =
    abi.encodeWithSelector(
        IERC20.transfer.selector,
        to,
        amount
    );
```

结构：

```
selector + ABI编码参数
```

推荐用于：

```
低级调用
Router
Proxy
```



# abi.decode（解码数据）

用于把 `bytes` 解析回 Solidity 类型。

常见场景：

- `call` 返回值
- 跨合约通信
- calldata解析





# ABI 在 calldata 中的结构

函数调用 data：

```
| 4 bytes selector |
| arg1 |
| arg2 |
| arg3 |
```



# ABI 编码动态类型

动态类型：

```
string
bytes
array
```

编码包含：

```
offset  = 从整个 ABI 编码数据起始位置到“动态数据区”的字节偏移量。
length
data
```







## CREATE 地址计算公式

合约地址由：

```
keccak256( RLP(sender, nonce) )
```

地址可预测，但必须知道 nonce



## CREATE2 地址计算公式

CREATE2 的地址公式：

```
keccak256(
  0xff ++ sender ++ salt ++ keccak256(bytecode)
)
```

CREATE2 **与 nonce 无关**。

只取决于：

```
deployer
salt
init_code
```

更方便的预测地址



**`init code` 指的是部署合约时执行的“初始化代码”（constructor code）**，而不是最终部署到链上的合约运行代码。

在 Solidity 编译后，一个合约实际上有两段代码：

```
Contract bytecode
│
├─ init code
│
└─ runtime code
```

区别：

| 类型         | 作用                 |
| ------------ | -------------------- |
| init code    | 部署时执行           |
| runtime code | 部署完成后存储在链上 |

init code
│
├─ constructor 执行
│
├─ codecopy(runtime_code)
│
└─ return(runtime_code)



因此相同的salt、相同的runtime code，但是不同的constructor初始化参数，使用create2时会有不同的地址



### CREATE2 失败情况

CREATE2 会失败：

1️⃣ 地址已有合约

```
EXTCODESIZE > 0
```

2️⃣ init code revert

3️⃣ gas 不够







# 设计模式

**工厂合约模式**

用来创建同类型的多个子合约

可能会存储所有子合约地址，方便检索



## 批量处理交易

在一个合约中调用多个函数，同时保留 msg.sender 和 msg.value 等环境变量





## 使用 Merkle 树减少存储

在 Airdrop、白名单（Whitelist）等场景中，使用 Merkle 树可以大幅减少链上存储成本。

使用 Merkle 树，只需在链上存储一个 32 字节的根哈希，用户提交证明，链上*验证 Merkle 证明*

```
    function claim(
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external {
        require(!claimed[msg.sender], "Already claimed");

        // 验证 Merkle 证明
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount));
        require(
            MerkleProof.verify(merkleProof, merkleRoot, leaf),
            "Invalid proof"
        );

        claimed[msg.sender] = true;
        // 执行 airdrop 逻辑...
    }
```



### ECDSA 签名方案

```
    function claim(
        uint256 amount,
        bytes calldata signature
    ) external {
        require(!claimed[msg.sender], "Already claimed");

        // 构建消息哈希
        bytes32 messageHash = keccak256(abi.encodePacked(msg.sender, amount));
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();

        // 验证签名
        address recoveredSigner = ethSignedMessageHash.recover(signature);
        require(recoveredSigner == signer, "Invalid signature");

        claimed[msg.sender] = true;
        // 执行 airdrop 逻辑...
    }
}
```





## 可升级合约

这些模式都围绕一个核心机制：
 在 Ethereum Virtual Machine 中通过 `delegatecall` 让 Proxy 使用 Implementation 的代码，但使用 Proxy 的 storage。

1. **Transparent Proxy**   升级逻辑在 **Proxy 合约里**。为了方式函数selectot冲突，Transparent Proxy 每次调用：检查 msg.sender 是否 admin，因此 gas 稍高。
2. **UUPS Proxy**      把升级逻辑从 Proxy 移到 Implementation。
3. **Minimal Proxy**（最小代理）是一种非常轻量的代理合约模式，用来 **极低成本部署大量合约实例**。只做一件事：把所有调用 `delegatecall` 到目标逻辑合约，合约不可升级
4. **Beacon Proxy**

Beacon Proxy 适用于：

```
大量 Proxy 共享同一 Implementation
```

------

## 架构

```
User
 │
 ▼
Proxy
 │
 ▼
Beacon
 │
 ▼
Implementation
```

## 升级流程

升级只需要：

```
update Beacon implementation
```

所有 Proxy 自动升级。









# Minimal Proxy bytecode

EIP-1167 定义了一个极小的代理 bytecode：

```
363d3d373d3d3d363d73<implementation_address>5af43d82803e903d91602b57fd5bf3
```

大小：

```
45 bytes
```

非常小。

这段 bytecode 的作用就是：

```
1 复制 calldata
2 delegatecall 到 implementation
3 返回结果
```