# Quorum配置项说明

### data目录下包含需要的初始配置文件

├── address
├── genesis.json
├── geth
│   ├── LOCK
│   ├── chaindata
│   │   ├── 000001.log
│   │   ├── CURRENT
│   │   ├── LOCK
│   │   ├── LOG
│   │   └── MANIFEST-000000
│   └── lightchaindata
│       ├── 000001.log
│       ├── CURRENT
│       ├── LOCK
│       ├── LOG
│       └── MANIFEST-000000
├── keystore
│   ├── accountAddress
│   ├── accountKeystore
│   ├── accountPassword
│   └── accountPrivateKey
├── nodekey
├── nodekey.pub
└── static-nodes.json

### genesis.json 

```json
{
  "nonce": "0x0",
  "timestamp": "0x58ee40ba",        // 限制区块链启动时间, 设置大于时间戳1492009146后开启出块
  "extraData": "0xf83aa00000000000000000000000000000000000000000000000000000000000000000d59496bd7c7e805572b21cb5cbcbbbe2c73b3c27eefdc080c0",
  "gasLimit": "0xFFFFFF",           // 区块gas限制, 目前设置为最大
  "gasUsed": "0x0",
  "number": "0x0",
  "difficulty": "0x1",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "mixHash": "0x63746963616c2062797a616e74696e65206661756c7420746f6c6572616e6365",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "config": {
    "chainId": 1337,                         // 链id
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "muirglacierblock": 0,
    "berlinBlock": 0,
    "londonBlock": 0,
    "isQuorum": true,                         // 启用GoQuorum工作模式
    "maxCodeSizeConfig": [
      {
        "block": 0,
        "size": 64                           // 最大智能合约代码大小，最多可配置128kbyte, 目前设置为64kbyte
      }
    ],
    "txnSizeLimit": 64,                       // 交易大小限制，最多可配置128kbyte, 目前设置为64kbyte
    "qbft": {                                 // 网络使用QBFT并包含QBFT配置项
      "policy": 0,                            // 0:验证者轮流提出区块 1:单个验证者出区块，直到它离线或无法访问
      "epoch": 30000,                         // epoch周期区块数
      "ceil2Nby3Block": 0,
      "testQBFTBlock": 0,
      "blockperiodseconds": 1,                // 最短出块时间，以秒为单位, 目前设置为1秒
      "emptyblockperiodseconds": 60,          // 无交易空块的出块时间，以秒为单位, 目前设置为1分钟
      "requesttimeoutseconds": 5             // 每个共识轮次的超时时间，以秒为单位
    }
  },
  "alloc": {
    "0x12f2754773685d3fffab0e4fb551dff73dbe965f": {
      "balance": "1000000000000000000000000000"          // 初始化指定账户余额
    }
  }
}
```

### 启动参数

```shell
docker run -d -p 8545:8545 -p 8546:8546 -e PRIVATE_CONFIG=ignore -v /usr/src/GolandProjects/QBFT-Network/Node-0/data:/data quorumengineering/quorum:latest --datadir /data \
    --networkid 1337 \
    --nodiscover \
    --verbosity 5 \
    --syncmode full \
    --mine --miner.threads 1 --miner.gasprice 0 --emitcheckpoints \
    --http --http.addr 0.0.0.0 --http.port 8545 --http.corsdomain "*" --http.vhosts "*" \
    --ws --ws.addr 0.0.0.0 --ws.port 8546 --ws.origins "*" \
    --http.api admin,eth,debug,miner,net,txpool,personal,web3,istanbul \
    --ws.api admin,eth,debug,miner,net,txpool,personal,web3,istanbul \
    --unlock 12f2754773685d3fffab0e4fb551dff73dbe965f --allow-insecure-unlock --password /data/keystore/accountPassword \
    --port 30300


// --nodiscover 确保未手动添加您的人员无法发现我的节点
// --verbosity 日志等级0=silent, 1=error, 2=warn, 3=info, 4=debug, 5=detail (default: 3), 目前设置为最详细的5
// 环境变量PRIVATE_CONFIG设置为ignore, 不启动私人交易
// --mine --miner.threads 1 --miner.gasprice 0 --emitcheckpoints 因使用QBFT协议, gasprise设置为0, 配置文件中difficulty设置为0x1
```