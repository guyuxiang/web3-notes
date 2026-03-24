# chainalysis接入调研

### **接入点**

- **交易上链前/中**：在用户发起交易前中调用Chainalysis 风险评估接口

  - 链下API调用：

    检索基于地址的实体（例如个人钱包或合约地址）的风险评估
    https://docs.chainalysis.com/api/address-screening/#introduction
    检索基于交易的风险评估
    https://docs.chainalysis.com/api/kyt/#introduction

    检索加密货币钱包地址是否已被列入制裁名单

    https://public.chainalysis.com/docs/index.html#introduction

  - 链上合约检查：
    Chainalysis 合约提供检查方法，调用合约输入地址返回是否在制裁名单上

    https://go.chainalysis.com/chainalysis-oracle-docs.html

  - 预言机方式：

    合约集成 Chainlink Functions ，交易发起时合约通知Oracle节点自动对 Chainalysis 进行 API 调用并将数据返回到合约，如何数据不符合监管要求，交易将无法继续执行
    https://github.com/smartcontractkit/functions-chainalysis

- **实时交易监控**：帮助企业实时检测高风险活动模式

  - 后端服务通过 API 持续监控历史区块链交易及其相关交易对手的风险信息

    https://docs.chainalysis.com/api/kyt/#introduction

  - 结合 Chainalysis 提供的预警数据，可以设置自动报警阈值、警报级别或人工审核。

    请求API后，若报警数据超过设置阈值，自动发送报警

    https://docs.chainalysis.com/api/kyt/#deposit-addresses-retrieve-deposit-addresses

- **数据查询：**查询历史交易数据、资金流向、链上关系图谱等
  - 交易监控用户界面：[https://kyt.chainalysis.com](https://kyt.chainalysis.com/)
  - Chainalysis 图表：[https://reactor.chainalysis.com](https://reactor.chainalysis.com/)
  - 警报的历史记录：https://docs.chainalysis.com/api/kyt/#alerts-get-all-alerts
- **报告生成：**生成符合监管要求的合规报告

​	未找到文档，需要联系Chainalysis 



API 技术集成指南：
https://docs.chainalysis.com/api/kyt/tech-solutions-guide/#api-technical-integration-guide
https://docs.chainalysis.com/api/kyt/guides/#developer-portal