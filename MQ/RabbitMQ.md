# RabbitMQ

### 1️⃣ RabbitMQ 是什么？

- 基于 **AMQP** 协议的消息中间件
- Erlang 实现（高并发 + 高可用）
- 用于系统解耦、异步处理、削峰填谷

------

### 2️⃣ 核心组件

```
Producer → Exchange → Queue → Consumer
```

#### 🔹 Producer（生产者）

发送消息的一方

#### 🔹 Exchange（交换机）

决定消息怎么路由

#### 🔹 Queue（队列）

存储消息

#### 🔹 Consumer（消费者）

消费消息

#### 🔹 Binding

Exchange 和 Queue 的绑定规则

# 🔀 二、Exchange 类型

### 1️⃣ Direct Exchange（精确匹配）

- routing key 完全匹配
- 👉 点对点 / 精确路由

------

### 2️⃣ Topic Exchange（通配符匹配）

- `*` 匹配一个词
- `#` 匹配多个词

👉 常用于：

- 日志系统
- 事件分发（Web3链上监听）

------

### 3️⃣ Fanout Exchange（广播）

- 不看 routing key
- 👉 全部队列都收到

------

### 4️⃣ Headers Exchange（少用）

- 根据 header 匹配

# 📦 三、消息可靠性

> **消息从生产 → RabbitMQ → 消费，全链路不能丢**

但现实中，任何一环都可能出问题：

| 阶段        | 可能问题               |
| ----------- | ---------------------- |
| 生产者 → MQ | 网络断开，消息没发成功 |
| MQ 内部     | 宕机丢数据             |
| MQ → 消费者 | 消费失败 / 重复消费    |

# RabbitMQ 的可靠性 = 三段式保证

```
生产者 → Broker(RabbitMQ) → 消费者
```

## 1️⃣ 生产者保证不丢消息

### ✔ Publisher Confirm模式（推荐）

```
生产者 → MQ → MQ返回ack/nack
```

👉 MQ 收到消息后会回一个确认

我会开启 Publisher Confirm 模式，确保消息成功进入 RabbitMQ，否则进行重试或记录失败。

## Return 机制（补坑）

如果：

- routing key 不匹配
- 没有队列接收

👉 消息会“丢”

------

### 解决：

```
mandatory = true
```

👉 MQ 会把消息 return 给生产者

## 2️⃣Broker 层保证（MQ 自己不丢数据）

##  问题

RabbitMQ 宕机：

👉 内存里的消息全没了

### ✔ 持久化

### 1️⃣ 队列持久化

```
queue durable = true
```

👉 队列不会丢

------

### 2️⃣ 消息持久化

```
deliveryMode = 2
```

👉 消息写磁盘

## 3️⃣ 消费者保证不丢

### ✔ 手动 ACK（必须掌握）

## 错误做法（新手常犯）

```
autoAck = true
```

👉 一收到就确认

## 正确做法：手动 ACK

```
autoAck = false
```

- ### 流程：

  ```
  1. 收到消息
  2. 业务处理
  3. 成功 → ack
  4. 失败 → nack / reject
  ```

------

## nack 处理策略

| 操作            | 含义         |
| --------------- | ------------ |
| requeue = true  | 重新入队     |
| requeue = false | 进入死信队列 |

## 为什么不能直接 requeue=true

因为：

- 会马上重新投递
- 如果是数据库故障/下游服务故障，会形成空转
- 消息会反复打爆消费者

所以 Web3 业务建议使用**延迟重试队列**，不要直接无脑 requeue。



  RabbitMQ 收到 ACK 后，通常会做这几件事：

    1. 把这条消息从当前 channel 的 unacked 集合中删除
    2. 释放这个 consumer 的一个 prefetch 配额
    3. 如果队列里还有后续消息，就可以继续派发下一条

# 四、消息重复 & 幂等

RabbitMQ **不保证 exactly once**

👉 会出现：

- 消息重复
- 消费多次

------

### 解决方案（必须会说）

#### ✔ 业务幂等

- 数据库唯一键
- 去重表
- 状态机

#### ✔ 消息 ID 去重

```
messageId / txHash（Web3）
```

#  五、延迟队列 & 死信队列

## 什么是死信？

消息变“异常”：

- 被 reject
- TTL 过期
- 队列满

👉 进入死信队列（DLQ）

## 作用

- 防止消息丢失
- 做失败重试
- 做异常排查

##  死信队列

人工排查或补偿任务处理。

## 1️⃣ TTL + DLX 实现延迟队列

流程：

```
消息 → TTL队列 → 过期 → 死信交换机 → 真正消费队列
```

------

## 2️⃣ 死信触发条件

- TTL 过期
- 队列满
- 消费失败（reject）

------

## 3️⃣ 应用场景

- 订单超时关闭
- 重试机制
- Web3 交易确认超时

# 完整可靠性链路

```
生产者：
  Confirm + Return

Broker：
  durable + persistent +（Quorum）

消费者：
  手动 ack + 幂等

兜底：
  死信队列
```