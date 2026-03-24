# Uniswap V3

## 背景

当前市场上的常值函数做市商大多存在资金利用率不高的问题。在Uniswap v1/v2使用的恒定乘积做市商公式中，对于给定价格，池子中仅部分资金参与做市。这显得十分低效，特别是当代币总是在特定价格附近交易时。

> 注：以稳定币为例，USDC/USDT的波动范围极小，而根据v2的公式，流动性提供者实际上会将资金分布在价格区间(0, ∞)，即使这些价格几乎永远也无法使用到。因此在Uniswap v1/v2版本，资金利用效率较低，同时也导致交易滑点相对较高。

在此之前，Curve和YieldSpace等一些产品尝试解决这个资金利用率问题，他们通过建立池子，并使用不同的函数描述代币之间的关系。这要求池子里的所有流动性提供者都遵守同一个公式，而如果他们希望在不同的价格区间提供流动性，将导致流动性分裂。



## 特性

Uniswap v3，一种新的自动做市商（AMM），它给予流动性提供者对资金被使用的价格区间更多控制权，并降低流动性分裂和gas消耗等问题的影响。该设计不依赖任何基于代币价格行为的共同假设。Uniswap v3仍然基于之前版本的常值函数曲线（即x⋅y=k），但提供许多重要的新特性：

- *集中流动性*：流动性提供者（LP）将被赋予在**任意价格区间**集中流动性的能力。这将提高池子的资金利用率，并允许LP估算他们认可的价格曲线，同时又与池子里剩余资金一起提供高效聚合的流动性。

- *灵活的手续费*：交易手续费将不再限定在0.30%。相反，手续费等级在每个池子初始化时设置，每一个交易对包含多个等级（池子）。默认支持手续费等级为0.05%，0.30%和1%。可以通过UNI治理增加新的手续费等级。

  > 注：UNI[第9号提案](https://app.uniswap.org/#/vote/2/9?chain=mainnet)申请引入新的手续费等级：0.01%，该提案已生效。0.01%的手续费适用于稳定币交易场景，使交易的滑点更小，这让Uniswap可以在稳定币交易领域直面Curve等市场龙头的竞争。

- *协议手续费治理*：UNI治理可以灵活设置协议手续费对交易手续费的分成占比。

- *改进的价格预言机*：Uniswap v3为用户提供了一种方式查询近期累计价格，从而避免了在计算TWAP（时间加权平均价格）的时间段开头和结尾手动记录累计价格。

- *流动性预言机*：合约提供了一种时间加权平均流动性的预言机。



## Concentrated Liquidity 集中流动性

允许LP将他们的流动性集中到更小的价格区间（而非(0,∞)）。我们将集中到一个有限区间的流动性称为“头寸”。

一个头寸只需要维持足够的代币余额以支持该区间的交易即可。

一个头寸只需要持有足够的代币X以支持价格移动到其上限，因为当价格向上方移动时需要消耗X代币。同样，只需要持有足够的代币Y以支持价格移动到下限。图1描述了在价格区间[pa,pb]的头寸与当前价格pc∈[pa,pb]的关系。xreal与yreal代表头寸的真实代币余额。

当价格离开头寸区间时，该头寸的流动性将不再活跃，同时无法获得手续费。在该价格点上，流动性将完全只由一种代币组成，因为另一种代币都被耗尽。如果价格重新进入区间，流动性将再次变得活跃。

> 注：从图1可知，Uniswap常值函数池子的价格移动是以池子中两种代币余额的此消彼长来实现的，当价格（过高或过低）离开头寸区间，意味着其中一种代币被完全替换为另一种代币，因此此时区间中仅剩余一种代币。

<img src="https://i.imgur.com/6XyH2px.jpg" alt="img" style="zoom: 80%;" />

流动性数量可以用L衡量，其等价于
$$
\sqrt{k}
$$
头寸的真实代币余额可以用以下曲线表示：
$$
(x + \frac{L}{\sqrt(p_b)})(y + L \sqrt{p_a}) = L^2 \tag{2.2}
$$
推导过程

https://hackmd.io/@adshao/B1OQXbVf9#2-Concentrated-Liquidity-%E9%9B%86%E4%B8%AD%E6%B5%81%E5%8A%A8%E6%80%A7

<img src="https://i.imgur.com/cBGb3ra.jpg" alt="img" style="zoom:80%;" />



由图2可知，v3区间流动性曲线（图中real reserves曲线）实际上是将v2流动性曲线（图中virtual reserves曲线）通过坐标平移而来。

只要流动性提供者觉得合适，他们可以自由地创建任意数量的头寸，每个头寸拥有自己的价格区间。通过这种方式，LP可以模拟价格空间中任意有分布需求的流动性（图3列举了部分例子）。此外，这种方式可以让市场决定流动性应该分配在什么地方。理智的LP们可以通过在当前价格附近的狭窄区间集中流动性来减少资金成本，并且通过添加或移除代币来移动价格，以使他们的流动性始终保持活跃。



![img](https://i.imgur.com/yZqtzrF.jpg)



![Introducing ticks in Uniswap V3 | By RareSkills – RareSkills](https://rareskills.io/wp-content/uploads/2025/01/tickTickTickTick.jpg)

### 2.1 Range Orders 区间订单

在极小区间的头寸看起来非常像限价单，如果价格穿越区间，头寸将由完全为一种资产变成另一种资产（以及累计手续费）。区间订单与传统限价单有两点不同：

- 一个头寸的最小区间是有限制的。当价格正好位于头寸之内时，该限价单可能只有部分成交。
- 当头寸被穿越后，需要手动取回。否则，当价格再次回到区间时，头寸将自动反向交易。

> 注：如果价格反复穿越一个区间订单，头寸中的资产持仓将自动变化，从一种资产完全变成另一种资产，再反向变化，循环反复。而CEX的限价单在完全成交后，即使后期价格恢复，已成交的订单也不会回滚。
>
> 因此，如果需要实现像传统交易所一样的限价单效果，当价格穿越限价区间后，流动性提供者需要手动执行取回操作，才能完全获得另一种代币。或者可以使用第三方应用提供的自动取回功能，比如[Gelato](https://www.gelato.network/)可以支持使用Uniswap v3的区间订单实现传统限价单的效果。



### 3 Architectural Changes 架构变动

### 3.1 Multiple Pools Per Pair 多池交易对

在Uniswap v1和v2，每个交易对对应一个独立的流动性池子，并针对所有交易统一收取0.30%的手续费。虽然历史数据表明默认的手续费等级对于大部分代币都是合理的，但对于部分池子可能太高了（比如稳定币池子），而对于另一部分池子又太低了（比如高波动性或者冷门代币）。

Uniswap v3为每个交易对引入了多个池子，允许分别设置不同的交易手续费。

因此一个代币对可能会有多个池子。对应不同的手续费。

所有池子都使用相同的工厂合约创建。默认允许创建三个手续费等级：0.05%，0.30%和1%。可以通过UNI治理添加更多手续费等级。

> 注：目前已经通过投票新增了一个0.01%的手续费等级。

### 3.2 Non-Fungible Liquidity 不可互换的流动性

之前版本的手续费收入会被作为流动性持续存入池子。这意味着即使没有主动存入，池子流动性也会随着时间而增长，并且可以复利地获取手续费收入。

在Uniswap v3，由于每个头寸的价格区间都不一样，因此v3的流动性不再像v2一样分布在所有价格区间，也就是说，v2流动性是可互换的，因此可以使用ERC-20代币表示。而v3流动性实际上是一个NFT（不可互换代币），使用[ERC-721](https://eips.ethereum.org/EIPS/eip-721)表示。

由于自定义流动性的特性，现在手续费以独立的代币被池子收集并持有，而不是自动复投为池子的流动性。

因此v3的池子合约没有实现ERC-20标准。任何人都可以在periphery创建一种ERC-20代币合约，以便让流动性头寸变得更可互换，但这需要额外的逻辑来处理手续费收入的分发或再投资。

#### 3.3 协议手续费

与Uniswap v2相同，Uniswap v3也有可以被UNI治理打开的协议手续费。在Uniswap v3，UNI治理可以更灵活地设置协议获取的交易手续费比例，可以将协议手续费设置为
$$
\frac{1}{N}
$$
的交易手续费或者0，其中，4≤N≤10。该参数可以基于每个池子设置。

> 注：Uniswap v2只能基于全局设置协议手续费，而Uniswap v3可以基于每个池子设置。

UNI治理可以添加额外的交易手续费等级。**当添加一个手续费等级时，可以同时定义其对应的tickSpacing参数**



**关于tick和tick spacing的概念**

简单而言，每个tick（点）对应一个价格，为了聚合不同头寸的流动性，价格空间被划分为一个个可被初始化的tick，只有能被tickSpacing整除的tick才允许初始化；在tick内的交易机制与v2一样，当该tick的流动性被消耗以后，价格将进入下一个tick，并重复上述交易过程。因此tickSpacing越小意味着流动性越连续，交易滑点越小，但同时也带来了更大的gas消耗。

因此，每个手续费等级的tickSpacing是一个权衡值，但总体而言，越高的手续费等级，其tickSpacing越大。因为手续费越高，代表交易对的波动性越大，交易者能够承受的滑点也越大。

我们可以通过[factory合约](https://etherscan.io/address/0x1f98431c8ad98523631ae4a59f267346ea31f984#readContract)查看链上手续费配置：feeAmountTickSpacing，目前支持的feeAmount和tickSpacing分别为：`{100: 1, 500: 10, 3000: 60, 10000: 200}`。

初始的手续费等级和tickSpacing分别为

0.05%（tickSpacing为10，两个初始化tick之间约为0.10%），

0.30%（tickSpacing为60，两个初始化tick之间约为0.60%），

1%（tickSpacing为200，两个初始化tick之间约为2%）。



### 6 Implementing Concentrated Liquidity 实现集中流动性

为了实现自定义流动性供应，可能的价格空间被离散的点（tick）划分。流动性提供者可以在任意两个（无需是临近的）tick定义的区间提供流动性。tickSpacing理论最小为1，也就是1个基点，也就是每两个相邻的价格差0.01%，那么从概念上，每当价格p等于1.0001的整数次方时就存在1个tick（点）。

我们使用整数i表示tick（点），使得该点的价格可以表示为：
$$
p(i) = 1.0001^i \tag{6.1}
$$
根据定义，两个相邻的tick之间的价格移动精度为0.01%（1个基点）。

交易对池子实际上使用开根号价格price来跟踪tick（点）。可将上述等式转换为等价的开根号价格形式：
$$
\sqrt{p}(i) = \sqrt{1.0001}^i = 1.0001^{\frac{i}{2}} \tag{6.2}
$$
举个例子，

\sqrt{p}(0)（tick 0的开根号价格）等于1

\sqrt{p}(1)等于
$$
\sqrt{1.0001} \approx 1.00005
$$
\sqrt{p}(−1)等于
$$
\frac{1}{\sqrt{1.0001}} \approx 0.99995
$$


当流动性加入一个区间，如果其中一个或全部tick都没有被已存在的头寸用作边界点，该tick将被初始化。



实际上不是每个tick都能被初始化

交易对池子在初始化时有一个参数tickSpacing，只有那些序号能够被tickSpacing整除的tick才能被初始化。比如，如果tickSpacing是2，则只有偶数tick (…-4, -2, 0, 2, 4…)能被初始化。小的tickSpacing允许更严格和更精确的区间，但可能导致每次交易消耗更多gas（因为每次交易穿越一个初始化的tick时，都需要给操作方带来gas消耗）。

任何时候价格穿越一个初始化的tick时，虚拟流动性将被加入或者移除。穿越一个初始化的tick所带来的gas消耗是固定的，与在该tick添加或移除虚拟流动性的头寸数量无关。

为了确保当价格穿越tick时，能够添加和移除正确数量的流动性；同时也为了确保当头寸在价格区间内时，能够正确获取对应比例的手续费收入，交易对池子需要一些记账工作。交易对合约使用存储变量来分别记录全局（每个池子）级别、每个tick级别和每个头寸级别的状态。

### 6.2 Global State 全局状态

合约的全局状态包括7个与交换和流动性供应相关的存储变量。（它也有其他一些存储变量用于预言机，如第5节描述。）



![img](https://i.imgur.com/GXC3N2R.jpg)

**流动性liquidity**

当前活跃流动性，表示：当前 tick 区间内，可用于 swap 的总流动性

 注意：

- 不是池子“总流动性”
- 是 **“当前价格所在区间” 的流动性**

价格跨 tick → liquidity 会增加或减少



**开根号价格sqrtPrice**

这两个值可根据虚拟余额计算如下：
$$
L = \sqrt{xy} \tag{6.3}
$$

$$
\sqrt{P} = \sqrt{\frac{y}{x}} \tag{6.4}
$$

反过来，两种代币的虚拟余额也可以使用这两个值计算得出：
$$
x = \frac{L}{\sqrt{P}} \tag{6.5}
$$

$$
y = L \cdot \sqrt{P} \tag{6.6}
$$

使用L和P（而不是x和y）计算比较方便，因为一个时刻只有其中一个值会变化。当在一个tick内交易时，只有价格（即P）发生变化；当穿越一个tick或者铸造/销毁流动性时，只有流动性（即L）发生变化。这避免了在记录虚拟余额时可能遇到的舍入误差问题。

因为
$$
y_1 - y_0 = L \cdot (\sqrt{P_1} - \sqrt{P_0})
$$
所以
$$
L = \frac{y_1 - y_0}{\sqrt{P_1} - \sqrt{P_0}} = \frac{\Delta{Y}}{\Delta{\sqrt{P}}}
$$
因此该公式方便交易时的计算



**全局状态记录当前tick序号为tick(i)**，一个表示当前tick的有符号整数（更准确地说，是低于当前价格的最接近的tick）。

因为在任意时刻，你需要能够基于当前的开根号价格sqrtPrice计算出对应的tick。在任意时间点，以下等式总是成立：
$$
i_c = \lfloor \log_{\sqrt{1.0001}} \sqrt{P} \rfloor \tag{6.8}
$$


**feeGrowthGlobal0 (fg,0)和feeGrowthGlobal1 (fg,1)**

每单位流动性，累计获得的手续费（全局）

也就是每单位流动性，从池子创建到当前时刻，累计产生的手续费



**protocolFees0 (fp,0)和protocolFees1 (fp,1)**

该变量以无符号uint128类型表示。累计协议手续费可以通过UNI治理领取，通过调用collectProtocol方法。



### Global State 与 swap 的关系（执行流程）

以一次 `swap()` 为例：

### swap 过程中会修改哪些 Global State？

1️⃣ 根据 `slot0.sqrtPriceX96` 确定当前价格
 2️⃣ 在当前 `liquidity` 下计算可 swap 数量
 3️⃣ 若价格触及 tick 边界：

- 更新 `tick`
- 更新 `liquidity`
   4️⃣ 累加：
- `feeGrowthGlobal0X128 / 1X128`
   5️⃣ 更新：
- `slot0.sqrtPriceX96`
- `slot0.tick`
   6️⃣ 写入新的 observation（TWAP）



### 6.2.2 Fees 手续费

每个交易对池子初始化时会设置一个不可修改的手续费，表示交易者需要支付的手续费，以百分之一基点为一个单位（0.0001%）。

> 注：默认的手续费值为500，3000，10000，分别表示的手续费为：500 x 0.0001% = 0.05%, 3000 x 0.0001% = 0.30%, 1000 x 0.0001% = 1%。

另一个变量为协议手续费ϕ，初始时设置为0，但是可以通过UNI治理修改。该数字表示交易者支付手续费的部分比例将分给协议，而不是流动性提供者。ϕ只允许被设置为以下几个合法值：0, 1/4, 1/5, 1/6, 1/7, 1/8, 1/9 或者 1/10。

> 注：协议手续费开关无法在创建交易对的时候自动打开，只能由UNI治理针对具体池子单独执行手续费设置，并且可以针对不同池子分别设置协议手续费。





### 6.2.3 Swapping Within a Single Tick 在一个Tick内交易

对于那些无法使价格变化超过一个tick（点）的小额交易，该合约像一个 
$$
x \cdot y = k
$$
池子一样工作。

**计算手续费**

假设 γ 是交易手续费，比如0.003，yin 是传入的token1代币数量。

首先，feeGrowthGlobal1和protocolFees1将增加：
$$
\Delta{f_{g,1}} = y_{in} \cdot \gamma \cdot (1 - \phi) \tag{6.9}
$$

$$
\Delta{f_{p,1}} = y_{in} \cdot \gamma \cdot \phi \tag{6.10}
$$

剩余的手续费分给流动性提供者，即扣除协议手续费后的交易手续费

**计算出交易得到token0的代币数量：**

因为
$$
x_1 - x_0 = L \cdot (\frac{1}{\sqrt{P_1}} - \frac{1}{\sqrt{P_0}})
$$
所以
$$
\Delta{x} = L \cdot \Delta{\frac{1}{\sqrt{P}}}
$$
根据
$$
y_1 - y_0 = L \cdot ({\sqrt{P_1}} - {\sqrt{P_0}})
$$

$$
L = \frac{y_1 - y_0}{\sqrt{P_1} - \sqrt{P_0}} = \frac{\Delta{Y}}{\Delta{\sqrt{P}}}
$$

$$
\Delta{\sqrt{P}} = \frac{\Delta{y}}{L}
$$

$$
{\Delta{y}} = {L} \cdot {\Delta{\sqrt{P}}}
$$

就可以计算出交易得到token0的代币数量



![image-20260302152412469](C:\Users\顾宇翔\AppData\Roaming\Typora\typora-user-images\image-20260302152412469.png)





### 6.2.4 Swapping output a Single Tick 跳出一个Tick的交易

如果计算后的P进入下一个初始化的tick，合约将完成当前tick（仅占一部分交易），再继续进入下一个tick完成剩余的交易

![img](https://i.imgur.com/4oIYwDE.jpg)



1. **确定方向**：根据当前价格与目标价格的关系，确定是向上（买入）还是向下（卖出）。
2. **获取 Tick 队列**：从当前 `activeTick` 开始，获取向目标价格方向排列的所有包含流动性变化的 Tick。
3. **进入循环计算**：

- **Step A**: 找到下一个有流动性变化的 Tick（`nextTick`）。
- **Step B**: 计算从当前价格推到 `nextTick` 价格所需的 `amountIn`。
- **Step C**:
  - 检查现有 amountRemaining 是否足够
  - 如果amountRemaining < amountIn，价格在当前 tick 内移动,sqrtPriceX96 更新,不改变 liquidity
  - 如果amountRemaining >`amountIn` 更新当前价格为 `nextTick` 的价格，更新 liquidity
  - 继续 swap 剩余 amount。



**具体过程：**

1. **开始swap**

   合约只读取 Global State：

- `slot0.sqrtPriceX96`：当前价格
- `slot0.tick`：当前 tick
- `liquidity`：当前区间的 **active liquidity**

同时确定：

- 方向：`zeroForOne`（价格向下）或反之（价格向上）
- 目标：`amountSpecified`（exact in / exact out）



2. **开始while** 

   通过 `tickBitmap.nextInitializedTickWithinOneWord()`：

   找到 **最近的、已初始化的 tick**，它用位运算，在 O(1) 级别定位“最近的已初始化 tick”，避免遍历所有 tick。

   得到：

   - `nextTick`
   - `sqrtPriceNextTick`

   **bitmap** 原理：

   Uniswap v3 把 tick 是否被初始化，用 **bitmap** 存起来：

   - **1 bit = 1 个 tick 是否 initialized**
   - 每 **256 个 tick** 组成一个 **word（uint256）**
   - 用映射保存：`mapping(int16 => uint256) tickBitmap`

   **nextInitializedTickWithinOneWord时，**

   1. 使用当前 tick找到所在的word，按照方向向左或向右，使用位运算&判断是否存在本 word 内有没有已初始化的tick

   	2. 如果有，则使用BitMath的一堆位运算/移位（有点像二分定位）找到最近的1的位置，也就找到下一次tick了

   

3. **在“当前区间内”推进价格**

  ```solidity
  function computeSwapStep(
      uint160 sqrtRatioCurrentX96,  // 当前价格
      uint160 sqrtRatioTargetX96,  // 目标价格
      uint128 liquidity,           // 当前 Tick 区间内的有效流动性
      int256 amountRemaining,     // 当前还剩多少 swap 量没处理
      uint24 feePips 				// 手续费比例
  )
  ```

  算出四个关键量：

  ```
  return (
      sqrtRatioNextX96, // 下一个tick的价格
      amountIn, // 本 step 实际消耗的 input
      amountOut, // 本 step 实际给出的 output
      feeAmount // 本 step 收取的手续费
  );
  ```

  

  有两种结果：

​	A. 没碰到 tick → 这轮就结束

​	B. 正好推到 tick 边界 → 需要跨 tick



4. **更新 Global State**

无论是否跨 tick，这些都会更新：

- `sqrtPriceX96 = sqrtPriceNext`
- `feeGrowthGlobal{0,1}` += 手续费 / liquidity
- `amountRemaining` 扣减
- `amountCalculated` 累加



5. **Tick.cross()**

​	① 翻转手续费刻度（feeGrowthOutside）

​	② 更新 active liquidity（liquidityNet）

​		更新当前 tick



6. **继续 while，直到 swap 完成**

​	



### 6.2.4 Initialized Tick Bitmap 初始化Tick的位图

如果一个tick没有被用作流动性区间的边界点（即如果该tick没有被初始化），那么在交易过程中可以跳过这个tick。

为了更高效寻找下一个已初始化的tick，合约使用一个位图tickBitmap记录已初始化的tick。如果tick已被初始化，位图中对应于该tick序号的位置设置为1，否则为0。

当tick被一个新头寸用作边界点，并且该tick没有被任何其他流动性使用，那么它将被初始化，位图中对应的比特位置为1。当该点关联的流动性都被移除时，已初始化的tick将重新变成未初始化，位图中对应的比特位置为0。

### 6.3 Tick-Indexed State Tick索引状态

为了记录每个tick被穿越时需要添加和移除的净流动性，以及在大于和小于该tick时所挣取的手续费，合约需要额外保存每个tick相关的信息。

![img](https://i.imgur.com/nackvVF.jpg)

### liquidityNet

`liquidityNet` 表示：当价格“向上穿过该 tick”时，池子的 active liquidity 应该净增加（或减少）多少，也就是需要加入和移除的总流动性数量

Tick只需要记录一个有符号整数：因为 `liquidityNet` 定义的是：

**“向上穿越 tick 时的变化量”**

在区间[lowerTick, upperTick)

lowerTick：`+ΔL`

upperTick：`-ΔL`

需要 **正负号**

当 swap 推动价格 **向上**，并且触及一个 tick：

```
liquidity = liquidity + tick.liquidityNet;
```

当价格 **向下**，则：

```
liquidity = liquidity - tick.liquidityNet;
```



该值无需在每次价格穿越tick时更新（只需在使用该tick作为边界点的头寸更新时才更新）。

![Uniswap V3 Deep Dive: Visualizing Ticks and Liquidity ...](https://miro.medium.com/v2/resize%3Afit%3A1400/0%2A3HOr_AXC4jP5pUE8.jpeg)

### liquidityGross

`liquidityGross` 表示：在某一个 tick 上，所有 LP 仓位在这个 tick 处“挂着”的流动性绝对值之和

- 只关心“有多少流动性引用了这个 tick”
- 不带方向的
- 不直接参与 swap 计算

**1️⃣ LP mint（增加流动性）**

当 LP 创建一个区间 `[lowerTick, upperTick)`，流动性为 `ΔL`：

- 在 `lowerTick`：

```
liquidityGross += ΔL
```

- 在 `upperTick`：

```
liquidityGross += ΔL
```

📌 **注意：两个 tick 都是“加”**

**2️⃣ LP burn（减少流动性）**

- 在 `lowerTick`：

```
liquidityGross -= ΔL
```

- 在 `upperTick`：

```
liquidityGross -= ΔL
```



liquidityGross是为了判断 tick 是否“还活着

当liquidityGross == 0意味着：

- 没有任何 LP 再使用这个 tick
- 这个 tick 可以被 **安全删除 / 清空**，以此决定是否更新tick位图。



### feeGrowthOutside{0, 1}

**区间内手续费 = 全局累计 − 区间外累计**

这是一种 **时间维度（Global）× 空间维度（Tick）解耦的前缀和思想**。

feeGrowthGlobal记录从池子创建到现在，**每单位流动性**累计产生了多少手续费

价格在 tick 轴上来回移动

LP 只在自己的区间内吃手续费



feeGrowthOutside{0, 1}用于记录“在这个 tick 之外”某个时刻总共累计多少手续费。

需特别注意，该值随着当前tick ic变化后会改变表示的方向。fo需要在每次tick穿越时被更新，**可以理解为被穿过时进行一次快照，记录在我这条价格边界外，手续费的累计，也就是手续费不会再变化的outside区间的累计手续费**，

这样后续可以被feeGrowthGlobal减掉outside的手续费，计算出最新的指定区间的累计手续费

根据当前价格是否在区间内，你可以使用一个公式计算每份流动性在tick i之上（fa）和之下（fb）获取的手续费（根据当前tick序号ic是否大于等于i）：
$$
f_a(i) = \begin{cases} f_g - f_o(i) & \text{$i_c \geq i$}\\ f_o(i) & \text{$i_c < i$} \end{cases} \tag{6.17}
$$

$$
f_b(i) = \begin{cases} f_o(i) & \text{$i_c \geq i$}\\ f_g - f_o(i) & \text{$i_c < i$}\end{cases} \tag{6.18}
$$



注： 首先回顾一下每个变量的含义，fg是（每个流动性）全局累计手续费；fo(i)是在指定tick i之外（每个流动性）累计的手续费，



当 ic<i 时，
$$
\underbrace{\overbrace{i_c, ..., i - 1}^{f_b(i) = f_g - f_o(i)}, i, \overbrace{i + 1, ...}^{f_a(i)=f_o(i)}}_{f_g}
$$
当 ic≥i 时，
$$
\underbrace{\overbrace{..., i - 1}^{f_b(i) = f_o(i)}, i, \overbrace{i + 1, ..., i_c}^{f_a(i)=f_g - f_o(i)}}_{f_g}
$$
因为
$$
\underbrace{\overbrace{..., i_l - 1}^{f_b(i_l)}, \overbrace{i_l, i_l + 1, ..., i_u - 1, i_u}^{f_r}, \overbrace{i_u + 1, ...}^{f_a(i_u)}}_{f_g}
$$
我们可以使用上述函数计算任意两个tick（低点tick il和高点tick iu）区间内，每个流动性累计的全部手续费fr：
$$
f_r = f_g - f_b(i_l) - f_a(i_u) \tag{6.19}
$$


当tick i的fo初始化时，它的初始值被设置成：
$$
f_o := \begin{cases} f_g & \text{$i_c \geq i$}\\ 0 & \text{$i_c < i$} \end{cases} \tag{6.21}
$$
因为不同tick的f0值可以在不同时刻初始化，因此比较他们的f0是无意义

**所有的头寸只需要知道从上一次边界被价格交互后，计算区间内的g值增长即可。**

### 6.3.1 Crossing a Tick 穿越一个Tick

当在初始化的tick之间交易时，Uniswap v3可以像k常值函数一样工作。但是，当交易穿越一个已初始化的tick时，合约需要添加或移除流动性，以确保没有流动性提供者会破产。这意味着ΔL是从tick中提取，并应用到全局L中。

**为了记录在价格区间内时，该tick作为边界点的手续费收入（和持续时间）**，合约需要更新tick的状态。feeGrowthOutside{0, 1}和secondsOutside被更新到反映当前值，当与该tick关联的交易方向改变时，按照下述公式更新：
$$
f_o := f_g - f_o \tag{6.26}
$$

$$
t_o := t - t_o \tag{6.27}
$$



### 6.4 Position-Indexed State 头寸索引状态

合约记录一个映射表，

从**用户地址**，**头寸低点（左边界，一个tick序号，int24类型）和高点（右边界，一个tick序号，int24类型）**到**具体头寸信息**的映射关系。每个头寸信息记录三个值：

![img](https://i.imgur.com/M33yNaC.jpg)

`liquidity`：
 👉 **这个 LP 头寸在区间内提供的流动性规模**，也就是该 LP 在这个 [lowerTick, upperTick) 区间内当前“生效中的流动性数量”

它是用户 **Position 级别**，与 Global `liquidity`（当前价格区间的总流动性）不同

**`mint / burn` 本质上只做三件事：**
 1️⃣ 根据价格区间与当前价格，**计算能增加/减少的 liquidity ΔL**
 2️⃣ **更新 Position.liquidity（±ΔL）**
 3️⃣ **在两个 tick 上登记边界（liquidityGross / liquidityNet）**
 👉 只有当“当前价格在区间内”时，才会同步影响 **Global `liquidity`**



### liquidity 计算

对一个给定区间 `[PL, PU)`：

- **由 token0 决定的 liquidity：**

​	因为
$$
\Delta{x} = L \cdot \Delta{\frac{1}{\sqrt{P}}}
$$

$$
L0 = \frac{amount0 \cdot \sqrt{PL} \cdot \sqrt{PU}}{\sqrt{PU} - \sqrt{PL}}
$$



- **由 token1 决定的 liquidity：**

  因为
  $$
  L = \frac{y_1 - y_0}{\sqrt{P_1} - \sqrt{P_0}} = \frac{\Delta{Y}}{\Delta{\sqrt{P}}}
  $$
  

$$
L1 = \frac{amount1}{\sqrt{PU} - \sqrt{PL}}
$$

👉 **真正可增加的 liquidity：**
$$
ΔL = \min(L_0, L_1)
$$
（因为两边资产必须“配平”）

------

## 三、mint：三种价格位置，三种计算路径

### 情况 A：当前价格 **≤ lowerTick**（价格在区间左侧）

- 只需要 **token0**
- liquidity 由 `amount0` 决定：

$$
ΔL = \frac{amount0 \cdot \sqrt{PL} \cdot \sqrt{PU}} {\sqrt{PU} - \sqrt{PL}}
$$

📌 不需要 token1
 📌 **不会影响 Global liquidity**

------

### 情况 B：当前价格 **≥ upperTick**（价格在区间右侧）

- 只需要 **token1**
- liquidity 由 `amount1` 决定：

$$
ΔL = \frac{amount1} {\sqrt{PU} - \sqrt{PL}}ΔL
$$

📌 不需要 token0
 📌 **不会影响 Global liquidity**

------

### 情况 C：当前价格 **在区间内**

- 同时需要 token0 + token1
- 分别算 `L0 / L1`，取最小值：

$$
ΔL = \min(L_0, L_1)
$$

📌 **会立刻增加 Global liquidity**





`feeGrowthInside1Last`：
 👉 **上一次结算时，该区间内“每单位流动性”的 token1 手续费累计值**，它是一个 **checkpoint（快照）**



### 第一次添加流动性

① Pool 被部署（createPool）

```
factory.createPool(token0, token1, fee)
```

② 第一次 initialize —— 价格的“创世时刻”

```
pool.initialize(sqrtPriceX96)
```

③ 第一次 mint

```
pool.mint(
  owner,
  tickLower,
  tickUpper,
  liquidity,
  data
)
```




### 6.4.1 setPosition 设置头寸

setPosition方法允许流动性提供者更新他们的头寸。

setPosition的两个参数：lowerTick和upperTick，与调用者msg.sender一起组成了头寸的key

该方法接受一个额外参数：liquidityDelta，用于指定用户希望添加或移除（负值）的虚拟流动性。

**工作流程：**

1. 该方法计算头寸的未领取手续费（fu）（分别以两种代币表示），头寸中以feeGrowthInside{0, 1}Last保存已领取的手续费，

表示这个 Position 上一次结算时，区间内「每单位流动性」已经累计到的手续费

**计算手续费：**
$$
ΔfeeGrowthInside1 = feeGrowthInside1Now - feeGrowthInside1Last
$$

$$
fees1 = liquidity * ΔfeeGrowthInside1 / Q128
$$

2. 计算liquidityDelta加到头寸的liquidity（流动性），在tick区间低点，它同时将liquidityDelta加到liquidityNet（注：tick从左到右，表示加入流动性）；而在头寸的高点，则从liquidityNet减去liquidityDelta（注：tick从右到左，表示移除流动性）。如果池子当前价格在头寸区间内，合约也会将liquidity加到全局的globalLiquidity。

3. 根据销毁或铸造的流动性数量，池子将代币从用户转出（如果liquidityDelta是负值，则将代币转给用户）。



### mint (创建头寸)

功能：

- 添加新的流动性位置
- 消耗指定数量的 token0 / token1
- 返回：
  - NFT tokenId
  - 实际提供的流动性
  - 消耗的两个代币数量

过程简述：

```
addLiquidity → 计算流动性 → mint ERC721 NFT → 记录头寸信息
```

头寸信息包括 liquidity、手续费累计点快照等。

### increaseLiquidity (增加流动性)

和 mint 类似，但针对已有 NFT：

- 根据传入 token0/1 增加头寸流动性
- 同时结算此前未领取的手续费
- 更新 head position info（包含 tokensOwed 与 feeGrowthInside last）

###  decreaseLiquidity (减少流动性)

- 从头寸中减少部分 liquidity
- 调用核心合约 burn
- 增加待领代币数量（包括手续费 + 释放的 token0/1）
- 仍需要通过 collect 去领取实际 tokensOwed 余额



### 5 Oracle Upgrades 预言机升级

Uniswap v2引入了时间加权平均价格（TWAP）预言机功能，Uniswap v3的TWAP包括三个重要改动。

1. 其中最重要的改动是Uniswap v3无需预言机用户在外部记录历史累计价格。Uniswap v2要求用户在需要计算TWAP的区间的开始和结束阶段分别记录累计价格。**Uniswap v3将累计检查点放到core合约，允许外部合约直接计算最近一段时间的链上TWAP，无需额外保存累计价格。**

2.  另一个改动是Uniswap v3不再使用累计价格之和计算算术平均数TWAP，**而是通过记录log价格之和计算几何平均数TWAP**。几何平均数相比算术平均数，受极端值的影响更小，并且无需为每种代币记录单独的累计价格，因为一个代币的几何平均数价格是另一个的倒数。

3. 除了价格累计数外，**Uniswap v3还增加了一个流动性累计数**，每秒累计

$$
\frac{1}{L}
$$
（即流动性倒数）。累计流动性对于那些基于Uniswap v3实现流动性挖矿的外部合约很有用。它也可以被其他合约用于判断一个交易对的哪个池子具有最可信的TWAP。





### 5.1 Oracle Observations 预言机观测

与Uniswap v2类似，Uniswap v3在每个区块开始记录累计价格，乘以自上一个区块到现在的时间（秒数）。

Uniswap v2的池子仅保存累计价格的最新值，该值由最近一个发生交易的区块更新。当在Uniswap v2计算平均价格时，需要由外部调用者负责提供提供累计价格的历史数据。如果有很多外部用户，每个用户都需要独立维护记录累计价格历史值的方法，或者使用一个共享方法减少成本。另外，无法保证每个有交互的区块都能影响累计价格。

**在Uniswap v3，池子保存累计价格的一系列历史（如5.3节所述，也包括累计流动性）**。在每个区块与池子第一次交互时，合约会自动记录累计价格，**并且循环地使用新值覆盖数组中的最旧值，类似于一个环形缓冲区**。虽然初始时数组仅分配一个检查点的空间，但是任何人都能够初始化额外的存储槽来扩展该数组，最多可达65,536个检查点。

任何扩展该交易对检查点的人调用交易对的合约需要支付一次性的gas消耗来为数组初始化额外的存储槽。

交易对池子不仅向用户提供历史观测数据数组，还封装了一个便利函数用于在观测点周期内寻找任意时间点的累计价格。



### 5.2 Geometric Mean Price Oracle 几何平均数价格预言机

Uniswap v3是通过记录当前tick的累计和为底层来计算价格
$$
\log_{1.0001}{P}
$$


**即以1.0001为底的价格P对数**，而不是记录累计价格P。

任意时间点的累计数等价于该合约截止当前时间每秒对数价格之和：
$$
a_t = \sum^{t}_{i=1} \log_{1.0001}(P_i) \tag{5.1}
$$
因为Uniswap v3使用int24（24位有符号整数）表示tick，假设当前tick为i，对应的价格为P1；下一个最近的tick为i+1，对应的价格P2=P1⋅1.0001，其相对P1的价格变化精度为：
$$
\frac{P_2 - P_1}{P_1} = \frac{P_1 \cdot 1.0001 - P_1}{P_1} = 1.0001 - 1 = 0.0001 = 0.01\%
$$
任意时间段t1到t2的几何平均价格（时间加权平均价格）为：
$$
P_{t_1,t_2} = \left(\prod^{t_2}_{i=t_1} P_i \right)^{\frac{1}{t_2 - t_1}} \tag{5.2}
$$


为了计算这个值，你可以分别查看t1和t2时刻的累计tick价格，将后者减去前者，并除以时间差（秒数），最后计算1.0001的x次方得出时间加权几何平均价格：
$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{\sum^{t_2}_{i=t_1} \log_{1.0001}(P_i)}{t_2 - t_1} \tag{5.3}
$$

$$
\log_{1.0001}(P_{t_1,t_2}) = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1} \tag{5.4}
$$

$$
P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2 - t_1}} \tag{5.5}
$$

### 5.3 Liquidity Oracle 流动性预言机

**Uniswap v3 的「流动性累计数」本质上是一种“时间加权的流动性积分”**：

> 用来回答「在某个价格区间里，过去一段时间内，真实参与定价的流动性有多大」。

在 **Uniswap v3** 中，每个 Pool 都会维护一个状态：

```
liquidityCumulative
```

概念公式可以理解成：

```
liquidityCumulative(t)
= ∫ liquidity(t) dt
```

也就是：

> **当前活跃流动性 × 经过的时间，不断累加**

⚠️ 注意关键词：**活跃流动性（active liquidity）**



链上合约可以使用该累计数，以使他们的预言机更健壮（比如用于评估哪个手续费等级的池子更适合被作为预言机数据源）。



# Uniswap V4

### **Hooks**

通过 Hooks，可以自定义与流动性池、交易、费用、LP 头寸的各种交互。Hooks 的本质是实现可定制化的流动性池。开发者可通过部署 Hooks（钩子）来实现满足自身需求场景 DEX，底层则是利用 Uniswap V4 的大流动性池。

也就是说，Uniswap V4 变成了一个基础平台，基于 V4 开发者可以开发出各种新功能，比如限价订单（之前是 CEX 较之于 DEX 的特色）、基于波动率或其他条件的动态手续费用（解锁了更高的激励可能性）、TWAMM（时间加权平均做市商，可以实现更大额资产的更平滑的交易）、将超出范围的流动性存入借贷协议、定制链上预言机、LP 费用自动复合到 LP 头寸等等。随着开发者了解的深入，会根据自身和社区需求开发出更多的钩子案例出来。



Uniswap 基于自身需求的可定制化，可以激发开发者推出新型的流动性池，满足各种交易场景，让流动性跟社区、跟项目自身的发展有更深的绑定。

· **onSwap**：在交换发生时被调用，可以用于实现自定义逻辑，例如记录交易信息、执行特定的操作或修改交易费用等。

· **onMint**：在流动性提供者向池子增加流动性时被调用，可以用于自定义逻辑，例如记录流动性提供的相关信息或执行特定的操作。

· **onBurn**：在流动性提供者从池子中撤回流动性时被调用，可以用于自定义逻辑，例如记录流动性提供的相关信息或执行特定的操作。

![img](https://upload.techflowpost.com/upload/images/20230620/2023062021450856895044.png)

Uniswap Labs展示了以下一系列可能性，揭示了产品的独特特点，包括：

· 基于时间加权平均的做市商 ( TWAMM )

· 基于波动性或其他数值的动态手续费

· 链上限价单

· 范围外的流动性存入借贷协议

· 自订义的链上 Oracle，如 geomean oracles

· 自动复投 LP 手续费到 LP 头寸

· 内置的 MEV (矿工可提取价值) 利润分配给 LP





### **Singleton**

原先是 factoy/pool 模式，每个流动性池一个合约，属于多代币多合约的架构，

现在引入一个大合约架构（singleton，「单例」合约），所有流动性池处于单例智能合约中。

原来的架构在创建流动性池和进行多池的交换时成本较高，V4 中的单例合约架构则减少了 gas 成本，无须在不同池的不同合约之间转移。这种架构根据 Uniswap 团队的估计，可以将创建流动性池的 gas 成本降低 99%。

![（来自 Uniswap 官网）](https://upload.techflowpost.com/upload/images/20230614/2023061414510578963677.jpg)

### **Flash accounting**

**快速记账系统作为Singleton的补充。**在V4中，该系统不再在每次兑换结束时进行资产的转移进出流动性池，而是只在净余额上进行转移。这种设计使得系统更高效，在Uniswap V4中能够提供额外的Gas节省。

### **Native ETH**

之前的版本里，用户实际上是在和WETH交易，ETH并不是Token Contract而WETH是 Token Contract, 对Uniswap来说ERC20 contract更容易集成，因此每次用户Swap需要额外打包一次ETH,将ETH变成WETH，这一步引发了gas浪费。**V4恢复了对原生ETH的支持，进一步节省了Gas开销。**



# Uniswap X

UniswapX 是一款聚合交易协议，允许用户聚合不同的流动性来源以匹配出最佳价格的交易。



UniswapX 本质上是一个基于**荷兰式拍卖**的非托管交易协议。

协议允许第三方 Filler 执行交易（做 taker），Filler 可以是链上链下的流动性提供者，比如做市商、MEV 搜索者、DEX 等。 

Filler 间的竞争通过荷兰拍来实现，即一种参数化荷兰式订单起始价格的方式。

订单的交换率会从初始汇率逐渐降低到更小的金额（荷兰拍卖方式），直到 Filler 有利可图地填充该订单。

荷兰拍的起始价格通过 RFQ，一个链下的询价系统，对一些 Filler 进行投票（为了将订单路由到链上流动性池，做市商将受到激励使用私人交易中继）。同时为了激励 Filler 网络提供最优惠的价格，UniswapX 允许订单指定一个 Filler，在短暂的时间内独享填充订单的权利，之后荷兰式拍卖开始，任何 Filler 都可以执行订单。

UniswapX，旨在通过将路由复杂性外包给开放的第三方构建者网络来解决这个问题，他们被称为fillers（填单者），这些三方实体利用链上流动性（例如跨所有 Uniswap 版本的去中心化交易池或他们自己的私人持仓）互相竞争以填充用户发起的代币互换交易。



RFQ+ 荷兰拍这种模式 Cowswap 的 Coincident of Wants 很早就已经实现，1inch Fusion 在去年也实现了集成专业做市商链下订单匹配的功能，UniswapX 选择集成专业做市商的方式结合后期 V4 组合性，令市场的多元化选择更多。

![img](https://upload.techflowpost.com/upload/images/20230725/2023072513595381482726.png)





### 无Gas交易

在 UniswapX 的使用中，交易者首先通过签名授权给 Permits，以提供转移代币的权利。这个过程需要支付一定的 gas 代币费用

接下来交易者签署一份链下订单，明确一些交易参数包括输入的代币种类和数量，输出的代币种类和数量等，并授权给 Reactor Contract（用来结算的相关合约）用于花费代币。

该订单由填单者（fillers）提交到链上，并代表交易者支付Gas费以完成交易。由于交易者不需要支付 Gas，因此他们甚至不需要持有区块链的原生代币（如 ETH、MATIC）即可进行交易。

链下签署交易，对散户友好，Filler 会在优化 gas 费和优化实际交换之间进行综合计算，利用这种复杂性进行计算，以产生最佳结果。

在这个过程中，交易提交给 Reactor Contract 由 Filler 支付 gas，如果交易失败，gas 损失将由 Filler 承担。



### 跨链

UniswapX 协议可以扩展支持跨链交易，其中交易者可以在源链上交易他们持有的资产，以获取目标链上的所需资产。链下签署的订单，不仅解决了池的复杂性问题，还解决了桥接的复杂性问题。复杂性都被相同的服务提供商、相同的提交者解决。

UniswapX 跨链实现以下功能：

(1) **快速交换** - 只要两个区块链之间存在消息传递桥梁，UniswapX 可以在任意两个链之间提供快速的资产交换；

(2) **简化操作** - 交换和桥接被合并为一个单一的操作，消除了用户直接与桥梁交互、维护各链上的 gas 代币或等待结算延迟的需求；

(3) **快速退出** -UniswapX 可以实现从二层链到其母链的几乎即时退出；

(4) **本地资产交换** - 交易者可以指定在目标链上接收本地或规范化的资产，而不是桥接的资产。例如，在主网上的 ETH 可以直接与 Avalanche 链上的 AVAX 交换；

(5) **最小化被动桥风险** - 交易者在交换本地资产时不承担任何与桥接相关的风险，而 Filler 仅在通过桥接在链之间重新平衡时承担桥风险。



### MEV

MEV（Maximal Extractable Value）是指在交易过程中，矿工或其他交易者通过优先处理交易、重新排序交易顺序或选择性地包含或排除交易，从中获得的最大可提取价值。MEV 是由区块链的交易序列性质和共识机制引起的现象。目前来看，受攻击最严重的合约是 Uniswap V3 和 Uniswap V2



UniswapX引入无需准入的 Filler 网络，Filler 选择各类 Reactor 进行结算，通过拍卖一个 batch 完成交易，mempool 保密等多种方式在某种程度上实现对用户的 MEV 保护，用户成为了 MEV 收益的分享者。

MEV 收益内置化，一部分补贴给 swapper（以更低成交价的方式），一部分被 Filler 获得，利润还给用户；

