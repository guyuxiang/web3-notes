## @uniswap/universal-router-sdk  主要是做 swap / 组合操作执行（聚合 v2/v3/v4 等）



##  Universal Router 的统一入口：`execute(commands, inputs[, deadline])`

所有操作都走合约同一个函数：

- `execute(bytes commands, bytes[] inputs, uint256 deadline)`（或无 deadline 版本） 

其中：

- `commands`：一个字节数组（**每个 byte 就是一条“指令/opcode”**）
- `inputs[i]`：对应 `commands[i]` 的 ABI 编码参数（每条指令的参数结构不同） 

并且官方定位就写明：UniversalRouter 用来聚合 v2/v3/v4。 

------

## 2) “兼容 v2/v3/v4”靠的是：不同版本对应不同的 swap 指令

在 Universal Router 的命令集中，直接就有针对不同协议的 swap opcode（官方 Technical Reference 列得很清楚）： 

- **V3 swap 指令**
  - `0x00 – V3_SWAP_EXACT_IN`
  - `0x01 – V3_SWAP_EXACT_OUT`
- **V2 swap 指令**
  - `0x08 – V2_SWAP_EXACT_IN`
  - `0x09 – V2_SWAP_EXACT_OUT`
- **V4 swap 指令**
  - `0x10 – V4_SWAP`（以及一些 v4 相关的初始化/position manager call 等） 

也就是说：
 **同一个 execute() 里，你想走 v2 就塞 V2 opcode，想走 v3 就塞 V3 opcode，想走 v4 就塞 V4 opcode。**

------

## 3) @uniswap/universal-router-sdk 做了什么？

`@uniswap/universal-router-sdk` 的核心价值是：

- 提供 `Command` / `RoutePlanner` 之类的工具
- 帮你把 “我要做 v2 swap / v3 swap / v4 swap + 资金怎么进出 + 是否 wrap ETH + 是否 Permit2 授权”
   **编码成** `commands` 和 `inputs`

然后你把这两坨数据发给 UniversalRouter 合约的 `execute()` 就行。 

## 例如：

**使用 V4Planner 构建 Actions**

```
const v4Planner = new V4Planner();

v4Planner.addAction(Actions.SWAP_EXACT_IN_SINGLE, [CurrentConfig]); // 执行交换
v4Planner.addAction(Actions.SETTLE_ALL, [inputToken.address, CurrentConfig.amountIn]); // 结算输入
v4Planner.addAction(Actions.TAKE_ALL, [outputToken.address, CurrentConfig.amountOutMinimum]); // 提取输出
```

三个 Action 的作用：

| Action               | 作用                     |
| -------------------- | ------------------------ |
| SWAP_EXACT_IN_SINGLE | 在单个池中按输入精确交换 |
| SETTLE_ALL           | 结算并支付输入代币       |
| TAKE_ALL             | 提取输出代币             |

**通过 Universal Router 执行**

```
const routePlanner = new RoutePlanner();
routePlanner.addCommand(CommandType.V4_SWAP, [v4Planner.actions, v4Planner.params]);

const tx = await universalRouter.execute(
    routePlanner.commands,
    [encodedActions],
    deadline,
    txOptions
);
```



Universal Router 不只有 swap 指令，还有一堆“通用资金/授权/清算指令”，让不同 swap 版本可以像乐高一样拼：

- **Permit2 相关**：`PERMIT2_PERMIT`、`PERMIT2_TRANSFER_FROM`、batch 版本等
- **资金清算/转账**：`SWEEP`、`TRANSFER`、`PAY_PORTION`、`BALANCE_CHECK_ERC20`
- **ETH/WETH**：`WRAP_ETH`、`UNWRAP_WETH` 

这让你可以在一次 `execute()` 里做到类似：

> Permit2 拉钱 → v3 swap → sweep 多余 token → unwrap WETH → 付款到接收地址

不管 swap 走的是 v2/v3/v4，这套“资金管道”都统一。