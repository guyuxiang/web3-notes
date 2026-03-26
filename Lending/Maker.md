## Maker - 现为 Sky Protocol

Maker 是最古老的 DeFi 借贷协议。它通过将 30 多种受支持的代币锁定在智能合约中来超额抵押贷款，以铸造 DAI，一种与美元挂钩的去中心化稳定币。除了作为借贷协议外，Maker 还充当稳定币发行者（DAI）。

2017 年 12 月 19 日，Maker 最初以单抵押 DAI (SAI) 开始。它使用以太币 (ETH) 作为唯一的抵押品进行铸造。2019 年 11 月 18 日，Maker 将 SAI 升级为多抵押 DAI (DAI)，可以使用多种不同的代币作为抵押品进行铸造。

Maker 现在甚至接受中心化稳定币 USDC，以帮助管理 DAI 的价格稳定。



![MakerDAO 中的借贷流程](https://img.learnblockchain.cn/pics/20231012183627.png)

**借贷流程**：

每个被批准作为抵押资产的代币都有一个单独的金库合约。MakerDAO 中的金库（Treasury）功能由[Join合约](https://github.com/makerdao/dss/blob/master/src/join.sol)管理。

账单（Accounting） 在 [vat.sol 合约](https://github.com/makerdao/dss/blob/master/src/vat.sol)内处理。当抵押品进入或退出系统时，Join 会更新此[合约](https://github.com/makerdao/dss/blob/fa4f6630afb0624d04a003e920b0d71a00331d98/src/join.sol#L111)。如果用户借款，他们会直接与 [vat.sol 合约](https://github.com/makerdao/dss/blob/fa4f6630afb0624d04a003e920b0d71a00331d98/src/vat.sol#L143)进行交互。此操作会更新用户的债务余额，并允许他们在 DAI 中铸造 DAI。

偿还时，用户在 DAI Join 合约 中燃烧 DAI。然后，此过程会更新 Vat，使用户能够结算借贷。

此外， vat.sol合约充当[风险管理](https://github.com/makerdao/dss/blob/fa4f6630afb0624d04a003e920b0d71a00331d98/src/vat.sol#L160)引擎。它维持全部借贷限额，设定每个用户的最低阈值，并监督抵押比率。当用户的债务或抵押品余额发生变化时，vat.sol 合约会评估利率和现货（spot）。



**MakerDAO 特点：**

- 每个资产都有自己合约。
- 账单功能集中在单个合约中，该合约还记录和执行风险参数，包括抵押检查
- 与其他应用程序不同，预言机来更新合约，监督抵押
- 价格和利率预言机使用不同的接口
- 利率源自外部
- 要借款，用户必须与多个合约交互



**现实世界资产（RWA）集成**：

Maker 在弥合与传统金融的差距方面取得了重要进展。2021 年 4 月 21 日，该公司成功执行了其首笔 MakerDAO 贷款，以房屋作为抵押的 18.1 万美元贷款，有效地创建了首批基于区块链的抵押贷款之一。

截至 2025 年，RWA（现实世界资产）已成为 MakerDAO 战略的重要组成部分，RWA 抵押品约占其储备的 14%，并产生约 10.9% 的总收入。这标志着 DeFi 与传统金融融合的重要进展。



**当前市场地位（2025年）**：

截至 2025 年底，MakerDAO 的 TVL 约为 60 亿美元，DAI 供应量约为 53-84 亿美元。DAI 依然是最大的去中心化稳定币之一，在 DeFi 生态系统中得到了广泛采用。

2023年， MakerDAO 推出了 Spark Protocol，这是一个基于 Aave V3 代码的借贷协议，进一步扩展了其在 DeFi 借贷领域的影响力。

2024 年 8 月，MakerDAO 宣布重塑品牌为"Sky"，这是其"Endgame Plan"（终局计划）的一部分，旨在提高去中心化程度、改善可访问性并适应不断变化的监管环境。

