# Solana Cli

```
solana --version
```

### 设置网络（默认 mainnet-beta）

开发建议先用 **devnet**：

```
solana config set --url devnet
```

可选网络：

```
mainnet-beta
devnet
testnet
localhost
```

查看当前配置：

```
solana config get
```

### 生成本地钱包 Keypair

```
solana-keygen new
```

默认路径：

```
~/.config/solana/id.json
```

查看你的地址：

```
solana address
```

### Devnet 领空投（测试用）

```
solana airdrop 2
```

查余额：

```
solana balance
```

指定 keypair：

```
solana balance --keypair xxx.json
solana address --keypair xxx.json
```

### 转账 SOL

```
solana transfer <接收地址> 1
```

### 查询账户信息（调试神器）

```
solana account <地址>
```

输出包括：

- lamports
- owner
- executable
- rentEpoch
- data（base64）

 **调试 PDA / Program Account 时必用**



### 开启本地测试链

```
surfpool start
```



### [为 Solana 开发搭建 AI 工具](https://solana.com/docs/intro/installation/dependencies#set-up-ai-tooling-for-solana-development)

本节详细介绍可用于加速 Solana 开发的可选 AI 工具设置。

| 工具     | 描述                                                         | 关联                        |
| -------- | ------------------------------------------------------------ | --------------------------- |
| MCP      | 您可以使用光标连接到 MCP 服务器，以改进 Solana AI 辅助开发。 | https://mcp.solana.com/     |
| LLMs.txt | 可用于在 Solana 文档上训练 LLM 的 LLM 优化文档。             | https://solana.com/llms.txt |

### [关闭程序](https://www.anchor-lang.com/docs/quickstart/local#close-the-program)

要收回分配给程序帐户的 SOL，您可以关闭您的 Solana 程序。

要关闭程序，请使用`solana program close <PROGRAM_ID>`命令。例如：

```
solana program close 3ynNB373Q3VAzKp7m4x238po36hjAGFXFJB4ybN2iTyg --bypass-warning
```

请注意，一旦程序关闭，程序 ID 就不能再用于部署新程序。



## Token / SPL 相关

> 前提：安装 `spl-token`

```
cargo install spl-token-cli
```

------

### 创建 Token

```
spl-token create-token
```

------

### 创建 Token Account（ATA）

```
spl-token create-account <token_mint>
```

------

### mint / transfer Token

```
spl-token mint <mint> 100
spl-token transfer <mint> 10 <to_address>
```

查看余额：

```
spl-token balance <mint>
spl-token accounts
```





