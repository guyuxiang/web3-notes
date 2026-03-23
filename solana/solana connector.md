# solana connector设计

目标是：**稳定监听链上事件、可靠处理交易、可水平扩展、可观测、可回放、能抗 Solana 分叉/回滚/RPC 抖动**。

Solana 的确认语义是 `processed → confirmed → finalized`；官方建议在依赖后续处理时优先用 `confirmed` 平衡速度与回滚安全，而高价值场景再等 `finalized`。另外，WebSocket 订阅很多场景默认是 `finalized`，`blockSubscribe` 还是一个不稳定接口；提交交易时也要特别处理 blockhash 过期与重试逻辑。



覆盖 4 类能力：

1. **链上数据接入**
   - 监听转账、合约调用、Program 日志、账户变更
   - 拉取交易详情、账户状态、余额、token 持仓
   - 支持历史补数和断点续跑
2. **业务语义抽象**
   - 把原始链上数据转成业务事件
      例如：
   - `DepositDetected`
   - `DepositConfirmed`
3. **交易发送**
   - 构造、签名、发送交易
   - 跟踪状态
   - 处理超时、重试、blockhash 过期、重复广播
4. **生产保障**
   - 高可用
   - 幂等
   - 可观测
   - 回滚补偿
   - 对账与重放



## 1）订阅和查询分离

不要只靠 WebSocket。

WebSocket 很适合“快速发现事件”，但**不适合单独作为最终真相来源**。更稳的做法是：

- **订阅流**：负责尽快发现新签名、新日志、新账户变更
- **HTTP/RPC 拉取**：收到签名后再主动 `getTransaction` / `getSignatureStatuses` / `getAccountInfo`
- **数据库状态机**：决定这笔交易是否进入业务最终态

也就是：

> **订阅负责发现，查询负责确认，数据库负责定责。**



## 2）链上状态要做成“分层确认”

Solana 会有分叉和短回滚，所以不要一看到事件就直接记为成功。

建议定义内部状态：

```
SEEN        -> 通过订阅发现
FETCHED     -> 已拉到交易详情
PROCESSED   -> 链上已处理
CONFIRMED   -> 已确认，可进入大多数业务流程
FINALIZED   -> 最终确认，适合高价值入账
DROPPED     -> 被分叉丢弃 / 查询不到 / 过期失效
REVERTED    -> 之前见过，后续确认失败或被替代
```

## 3）必须支持“先订阅、后补拉、再对账”

生产事故里最常见的不是“代码不会解析交易”，而是：

- WebSocket 断线
- RPC 节点短暂缺块
- 服务重启期间漏消息
- 某些 slot 的交易详情暂时查不到
- provider 局部故障

所以必须有三层保障：

### 第一层：实时订阅

发现最新事件。

### 第二层：定时补拉

按地址、slot、signature 范围补拉。

### 第三层：对账任务

以数据库中的“未终态记录”为输入，反查链上最终状态，修正遗漏。



你会用到：

- `logsSubscribe`
- `accountSubscribe`
- `signatureSubscribe`
- `getTransaction`
- `getSignatureStatuses`
- `getSignaturesForAddress`

官方有 `logsSubscribe`、WebSocket 订阅方法以及 `getSignaturesForAddress` 文档。





## 1. ingress-listener：事件接入层

职责：

- 管理多个 RPC/WS 连接
- 建立订阅
- 心跳检测
- 自动重连

### 关键点

- 每个订阅单独 goroutine
- 带 backoff 重连
- 重连后触发补拉任务
- 不做重业务逻辑，避免阻塞订阅线程

## 2. fetcher：链上详情拉取层

职责：

- 根据 signature 拉交易详情
- 根据 account 拉账户信息
- 根据地址做历史签名扫描
- 失败重试和 provider failover

### 建议

建立 **多 RPC provider 池**：

```
primary:   Helius / Triton / QuickNode / 自建RPC
secondary: 另一家商业RPC
fallback:  公共RPC（仅应急，不建议生产主用）
```

### 规则

- 读请求可多 provider 熔断切换
- 发交易与查交易最好尽量同 provider 域内优先



## 3. parser：协议解析层

职责：

- 解析原始交易
- 抽取：
  - signature
  - slot
  - signer
  - fee payer
  - account keys
  - instructions
  - inner instructions
  - token balances
  - logs
  - err

### 再往上做业务解码

例如：

- SOL 转账
- SPL Token 转账
- ATA 创建
- 特定 Program 调用
- DEX swap
- NFT mint
- 你自己关注的合约事件

建议做成接口化：

```
type Decoder interface {
    ProgramID() string
    Decode(tx *EnrichedTransaction) ([]DomainEvent, error)
}
```

这样以后接 Jupiter、Raydium、你自己的 Program，都能插件式扩展。





## 4. processor：状态机和业务处理层

职责：

- 维护每笔交易/事件生命周期
- 幂等入库
- 状态推进
- 分叉回滚处理
- 发出业务事件

### 核心表设计建议

#### `chain_transactions`

```
id
chain = solana
signature
slot
block_time
observed_commitment
final_commitment
status               -- seen/fetched/confirmed/finalized/reverted
err_code
fee_payer
raw_tx_json
created_at
updated_at
unique(signature)
```

#### `chain_events`

```
id
signature
event_type
program_id
account
amount
mint
owner
biz_key
status
payload_json
created_at
updated_at
unique(biz_key)
```

这里的 `biz_key` 用来做业务幂等，例如：

```
solana:deposit:{wallet}:{mint}:{signature}:{index}
```





## 5. broadcaster：交易发送层

职责：

- 构造交易
- 申请 recent blockhash
- 签名
- 发送
- 轮询确认
- 失败重试
- 过期后重新签名再发

官方明确提醒：在重新签名前，必须确认原交易 blockhash 已过期，否则可能造成重复风险；`sendTransaction` 也支持 `maxRetries`，并建议开启 preflight。

### 非常关键的原则

**不要因为“暂时查不到确认”就立刻重签重发。**

要先判断：

- 交易是否还可能在网络中传播
- blockhash 是否过期
- 原签名是否已被链接受

否则容易造成双发。





## 6. reconciler：对账修复层

这是生产服务的救命模块。

职责：

- 扫描“长时间未终态”的交易
- 重新查链
- 扫描监控地址最近 N 条签名
- 补入漏单
- 修复由于分叉导致的脏状态

### 定时任务建议

- 每 30 秒：扫最近 5 分钟内 `SEEN/FETCHED/PROCESSED`
- 每 5 分钟：扫最近 2 小时监控地址签名
- 每 1 小时：扫最近 24 小时异常交易
- 每天：做一次全量抽样对账



## 监听充值地址入账

这是最常见的。

### 推荐流程

1. 对“充值地址集合”做维护
2. 通过以下方式组合监听：
   - `logsSubscribe`：快速发现相关交易
   - `getSignaturesForAddress`：补历史
   - `getTransaction`：取详情
3. 解析 token transfer / SOL transfer
4. 校验：
   - 目标地址是否属于我方
   - mint 是否正确
   - token program 是否正确
   - 金额是否满足精度要求
5. 按确认级别推进入账状态

### 为什么不建议只用 accountSubscribe

因为账户变化很多时候只告诉你“变了”，但业务要知道：

- 谁转的
- 哪笔交易
- 是否成功
- 是主转账还是 inner instruction

最终还是要回到交易级解析。





## 场景 2：监听某个 Program 的业务事件

例如你监听自家 Solana Program。

### 推荐流程

1. 对 ProgramID 做 `logsSubscribe`
2. 根据日志/指令过滤业务交易
3. 拉完整交易
4. 解析 instruction data / inner instructions / accounts
5. 映射成业务事件





# 失败场景与处理

## 场景 A：WebSocket 断线

处理：

- 自动重连
- 从上次 checkpoint 开始补拉
- 对最近 N 分钟地址做签名回扫

## 场景 B：RPC 查不到交易

处理：

- 延迟重试
- 切换 provider
- 超时后标记 `PENDING_RECHECK`

## 场景 C：交易一直没确认

处理：

- 查 signature status
- 判断 blockhash 是否过期
- 过期前不要贸然重签
- 过期后再重新构造

## 场景 D：收到重复事件

处理：

- 直接走 DB 唯一键幂等

## 场景 E：出现短回滚

处理：

- 状态回退
- 撤销未最终确认的业务副作用





# 监控与报警

没有可观测，就不算生产级。

## Metrics

至少打这些：

### 订阅层

- ws_connected
- ws_reconnect_total
- ws_message_lag_ms
- ws_message_rate

### RPC 层

- rpc_request_total
- rpc_error_total
- rpc_latency_ms
- rpc_429_total
- provider_switch_total

### 处理层

- tx_seen_total
- tx_confirmed_total
- tx_finalized_total
- tx_reverted_total
- parser_error_total

### 补偿层

- reconcile_backlog
- reconcile_fixed_total
- unmatched_events_total

### 交易发送层

- tx_submit_total
- tx_submit_fail_total
- tx_expired_total
- tx_confirm_timeout_total

## 报警

- WS 全断
- 某 provider 错误率 > 20%
- 充值监控 5 分钟无事件但链上活跃正常
- reconcile backlog 持续增长
- 某个 program 解析错误飙升
