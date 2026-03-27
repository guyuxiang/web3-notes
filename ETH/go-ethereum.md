# go-ethereum

以太坊主要包含了以太坊启动模块,p2p模块,数据库模块,以太坊网络同步协议模块,MPT树模块,挖矿模块,以太坊区块链数据结构,缓存,交易池模块,PRC模块

account/ 实现了一个高等级的**以太坊账户管理**
cmd/ ethereum 相关的 **Command-line 程序**。该目录下的每个子目录都包含一个可运行的 main.go。
   |── clef/ Ethereum 官方推出的 Account 管理程序。
   |── geth/ Geth 的本体
core/   以太坊核心模块，包括核心数据结构，**statedb，EVM 等算法实现**
   |── rawdb/ db 相关函数的高层封装（在 ethdb 和更底层的 **leveldb 之上的封装**）
   |── state/
       ├──statedb.go  **StateDB** 结构用于存储所有的与 Merkle trie 相关的存储，包括一些循环 state 结构  
   |── types/  包括 Block 在内的**以太坊核心数据结构**
      |── block.go  以太坊 block
      |── bloom9.go  一个 Bloom Filter 的实现
      |── transaction.go 以太坊 transaction 的数据结构与实现
      |── transaction_signing.go 用于对 transaction 进行签名的函数的实现
      |── receipt.go  以太坊收据的实现，用于说明以太坊交易的结果
   |── vm/      **evm虚拟机**
   |── genesis.go     **创世区块**相关的函数，在每个 geth 初始化的都需要调用这个模块
   |── tx_pool.go     Ethereum **Transaction Pool** 的实现
eth/  以太坊网络协议
   |── downloader 		主要用于和**网络同步**，包含了传统同步方式和快速同步方式
   |── fetcher			主要用于基于**块通知的同步**，接收到当我们接收到NewBlockHashesMsg消息得时候，我们只收到了很多Block的hash值。 需要通过hash值来同步区块。
   |── filter			提供基于RPC的过滤功能，包括实时数据的同步(PendingTx)，和历史的日志查询(Log filter)
consensus/
   |── consensus.go   共识相关的参数设定，包括 Block Reward 的数量
ethdb/    Ethereum 本地存储的相关实现，包括 leveldb 的调用
   |── leveldb/   Go-Ethereum 使用的与 Bitcoin Core version 一样的 **Leveldb** 作为本机存储用的数据库
event/
   |── event.go   发布订阅,**内部处理实时事件** 
miner/
   |── miner.go   矿工模块的实现。
   |── worker.go  真正的 block generation 的实现实现，包括打包 transaction，计算合法的 Block
p2p/     Ethereum 的 **P2P 模块**
   |── params    Ethereum 的一些参数的配置，例如：bootnode 的 enode 地址
   |── bootnodes.go  bootnode 的 enode 地址 like: aws 的一些节点，azure 的一些节点，Ethereum Foundation 的节点和 Rinkeby 测试网的节点
rlp/     **RLP 的 Encode 与 Decode** 的相关
node/    一个典型的node就是**一个p2p的节点**
rpc/     Ethereum **RPC 客户端**的实现
trie/    Ethereum 中至关重要的数据结构 **Merkle Patrica Trie(MPT) 的实现**
   |── committer.go    Trie 向 Memory Database 提交数据的工具函数。
   |── database.go     **Memory Database，是 Trie 数据和 Disk Database 提交的中间层。**同时还实现了 Trie 剪枝的功能。

   |── node.go         MPT 中的节点的定义以及相关的函数。
   |── secure_trie.go  基于 Trie 的封装的 **Trie 结构**。与 trie 中的函数功能相同，不过 secure_trie 中的 key 是经过 hashKey() 函数 hash 过的，无法通过路径获得原始的 key 值
   |── stack_trie.go   Block 中使用的 Transaction/Receipt Trie 的实现
   |── trie.go         **MPT 具体功能的函数实现**

--------------------------------------------------------------------------------------------------------------------------------------------------------------
## 程序启动过程

geth是一个go语言命令行程序
**init过程:**
1.初始化封装**命令行解析**的过程,能从用户输入命令解析出相应的命令参数,并记录在上下文中用于后续读取,还会执行特定的函数,使用的是gopkg.in/urfave/cli第三方库
2.设定程序启动执行的函数geth(),如果用户没有输入其他的子命令的情况下直接**运行geth节点**，就会默认启动一个全节点模式的节点
**geth()执行过程**
1.prepare() 
	1.它主要用于设置一些节点初始化需要的配置,比如**启动网络模式(主网,测试网),节点类型(全节点,轻节点)**
	2.**启动指标收集器**
2.makeFullNode()
	1.根据读取的配置文件和命令行参数,创建一个 Node 类型的实例stack,主要**配置其节点基础设置(如RPC,数据库,P2P),账户,eth网络协议(交易池,共识算法,gas,缓存,同步方式),检测指标**
	Node 是 Geth 生命周期中最顶级的实例，它的开启和关闭与 Geth 的启动和关闭直接对应。
	2.**打开数据库**
	3.**装载创世区块**。 根据节点条件判断是从数据库里面读取，还是从默认配置文件读取，还是从自定义配置文件读取，或者是从代码里面获取默认值。并返回区块链的config和创世块的hash。
	4.**创建Etherum struct**为eth 是负责提供更为具体的以太坊的功能性服务, 比如管理 Blockchain，共识算法等,
		根据配置文件决定启动为轻节点还是全节点,区别为是否启动mining矿工模块
		eventMux可以认为是一个全局的事件多路复用器，

​		accountManager认为是一个全局的账户管理器。
​		engine创建共识引擎。
​		etherbase 配置此Etherum的主账号地址。

​		初始化bloomRequests 通道和bloom过滤器。
​	5.**创建BlockChain,**也就是eth的区块链
​	6.**初始化eth 区块链的交易池**，存储本地生产的和P2P网络同步过来的交易。
​	7.**初始化以太坊协议处理器**，维护了 backend 中同步/请求数据的实例,比如`downloader.Downloader`，`fetcher.TxFetcher`
​	8.**初始化矿工**
​	10.**初始化RPC服务**
​	11.最后注册eth服务到node的生命周期管理列表,随着node.Start()的执行而启动ethereum.Start()。
**3.startNode()**
​	1.**启动P2P服务,启动RPC服务,启动注册到lifecycle生命周期管理的服务**,启动协程监听关闭Node信号来执行stack.Close()
​	2.启动RPC服务
​	2.**解锁账户**；
​	3.开启钱包事件监听；
​	4.创建一个客户端与本地geth交互
​	5.启动挖矿

在`geth()`函数的最后，函数通过执行`stack.Wait()`，使得主线程进入了阻塞状态，其他的功能模块的服务被分散到其他的子协程中进行维护。
Node 中维护了节点运行所需要的后端的实例和服务

---------------------------------------------------------------------------------------------------------------------------------
## p2p

P2P网络中的每个节点都可以既是客户端也是服务端，因此不适合使用HTTP协议进行节点之间的通信，且一般都是直接使用Socket进行网络编程
P2P主要存在四种不同的网络模型，代表着P2P技术的四个发展阶段

**集中式**，即存在一个中心节点，它保存了其他所有节点的索引信息,实现简单,但缺点也很明显，由于中心节点需要存储所有节点的路由信息，当节点规模扩展时，就很容易出现性能瓶颈；而且也存在单点故障问题。

**分布式**，在P2P节点之间建立**随机网络**。在新加入节点与P2P网络中的某个节点之间随机建立连接通道，从而形成一个随机拓扑结构。消息进行随机的传递,新节点加入该网络时随机选择一个已经存在的节点并建立邻居关系。还需要进行全网广播，让整个网络知道该节点的存在。全网广播的步骤是，该节点首先向邻居节点广播，邻居节点收到广播消息后，再继续向自己的邻居节点广播，以此类推，从而将消息广播到整个网络。这种广播方法也称为**泛洪机制**。分布式结构不存在集中式结构的单点性能瓶颈问题和单点故障问题，具有较好的可扩展性。(Gossip BTC,cosmos)

泛洪机制的问题主要是可控性差，具体讲包括两个较大的问题：一个问题是容易形成泛洪循环，比如节点A发出的消息经过节点B到节点C，节点C再广播到节点A，这就形成了一个循环；另一个问题是响应消息风暴，比如节点A想请求的资源被很多节点所拥有，那么在很短时间内，会出现大量节点同时向节点A发送响应消息，这就可能会让节点A瞬间瘫痪。
消除泛洪循环问题的方法可以借鉴IP网络路由协议中有关泛洪广播的控制，一种方法是对每个查询**消息设置TTL值**，泛洪消息每被转发一次，TTL值减1，当节点接受的TTL为0时，不再转发消息，这样可以避免查询消息在网络中产生死循环。还可以为泛洪消息设置唯一的标志，对接收到的重复消息不进行转发以防止消息产生死循环。解决响应消息风暴问题，一般会在数据链路层进行网络分段，减少消息跨段广播。

**混合式**，网络中存在多个超级节点组成的分布式网络，消息的传递发生在超级节点中。一个新的普通节点加入，先选择一个超级节点进行通信，该超级节点再推送其他超级节点列表,这种结构的泛洪广播只是发生在超级节点之间，因此可以避免大规模泛洪问题。在实际应用中，混合式结构是相对灵活且比较有效的组网架构，实现难度也相对较小(**DPOS**)

**结构化网络**
结构化网络则将所有节点按照某种结构进行有序组织，比如形成一个环状网络或树状网络。结构化网络在具体实现上，普遍**基于分布式哈希表算法**
结构化网络在可拓展性,可靠性,可维护性,节点发现效率最优



#### **以太坊是基于Kademlia(KAD)算法实现结构化网络**

但不存储值,只存储节点
Kademlia是一种分布式哈希表（DHT）技术，Kademlia**使用异或XOR运算计算节点之间的距离**(这里的距离**不是地理空间的距离，而是路由的跳数**)，从而建立DHT拓扑结构。这种算法可以极大地提高路由的查询速度。哈希表中存储的是节点

**（1）标识**
在Kademlia算法网络中，**每一个节点都使用哈希算法生成一个 节点 ID 来唯一标识自己**,由于哈希函数的特点，key的分布是高度随机的，因此也是高度离散的，任何两个key都不会非常临近
**（2）节点距离**
在Kademlia网络中，任意两个节点间的距离是通过对两个节点的ID进行异或（XOR）运算得出来的,因此距离本节点不同距离会有多个不同的节点

**distance(A, B) = A XOR B**

对于异或操作，拥有类似于几何距离的某些特性,XOR 的结果越小，表示距离越近。

假设两个节点 ID（简化成 8 bit）：

```
A = 10111001
B = 10010011
```

做 XOR：

```
A = 10111001
B = 10010011
--------------
XOR=00101010
```

👉 distance = `00101010`（二进制） = 42（十进制）

（⊕表示XOR）：❏ A ⊕ A = 0，反身性，自身距离为零。
❏ A ⊕ B = B ⊕ A, XOR符合交换律，具备对称性。
❏ A ⊕ B ⊕ C = A ⊕ (B ⊕ C) = (A ⊕ B) ⊕ C，结合律。
❏ (A ⊕ B) > 0，不同的两个key之间的距离必大于零。
❏ (A ⊕ B) + (B ⊕ C) >= (A ⊕ C)，三角不等式。
对于任意给定的节点和距离，总能找到另一个存在的节点，使两个节点的距离正好等于给定的距离
**（3）路由表**
分布式哈希表中的一个**桶来保存与当前节点距离计算结果在某个范围内的所有节点列表**,每一个桶中维护了一个链表记录了距离自己n的节点信息,根据距离不同有多个桶构成了节点的路由表

```
[0~2^1)   -> bucket0
[2^1~2^2) -> bucket1
...
[2^159~2^160)
```

链上节点信息是按照节点**发现的时间顺序排列**的，最先发现的节点保存到链表的头部，最后发现的节点保存在链表的尾部。
每条链上最多只能保存K个节点的信息，因此叫K桶,这是为了在节点查询的时候能够不断地向目标节点靠近，从达到快速收敛的效果。
**以太坊K桶按照距离进行排序，共256个K桶，每个K桶包含16个节点,第i个k桶代表当前节点（本机）距离为i+1的网络节点集合**
K可以看做一个系统参数，能够对节点的运行性能进行调节，K值能够保证系统的稳定性。
若k桶已经满了，则把k-bucket分裂为两个大小相同的新k-bucket

基于此，一个完整的网络空间可以被表示根据节点ID构造成一颗二叉树,二叉树的每一个叶子节点代表一个Kademlia网络上的节点。从根节点到叶子节点的路径等于节点ID
**（4）通信信议**
Kademlia protocol使用了**UDP协议**来进行网络通信。
网络传输了4种数据包(UDP协议是基于报文的协议。传输的是一个一个数据包，分别是**ping,pong,findnode和neighbors**。
**❏ Ping**：用于节点探测，判断节点是否存活。ping包里包含unix时间戳用于超时校验,如果数据包内的时间戳过期可能会导致无法处理。
         签名PIng包还要**带上hash**并且让对方主机回复的Pong包要带上之前Ping包的hash才可以通过校验,为了防止udp反射放大攻击造成ETH的发现节点池会不断的被堆满,
**❏ Pong**:Ping响应。**包中必须ping的hash,**忽略未经请求的不含ping包hash的pong包,如果在12小时内未与发送方进行任何通信，则除了发送pong之外还应发送ping以进行端点验证
**❏ Findnode**：向其他节点**查询与目标节点距离接近的节点**信息用于**节点发现**。**包含目标节点id,**为了抵抗流量放大攻击，只有经过端点验证的FindNode发送者才能被邻近节点回复。
**❏ Neighbours**:Findnode响应。包含一系列节点的ip和端口信息



**udp信息加密和安全问题**

discover协议因为没有承载什么敏感数据，所以数据是以明文传输，但是**为了确保数据的完整性和不被篡改，所以在数据包的包头加上了数字签名。**

一个 UDP 包结构大致是：

```
[ hash || signature || packet-type || packet-data ]
```

signature = sign(packet-type || packet-data, 节点privateKey)

NodeID 作为 public key 验签名



#### **Discovery v5**

v4（无状态模型）

v5（引入 session）：握手 → 建立 session → 加密通信

引入**❏ ENR节点记录请求**:请求获取节点信息
**❏ ENR节点记录响应**:节点记录包含了节点的网络信息如ip端口和键值对自定义信息



路由能力（Kademlia 扩展）： **topic-based discovery**

- 找 validator
- 找某个网络服务节点
- 找某个 shard 的节点



#### 以太坊节点发现机制

当节点是一个全新的、从未运行的节点是初始节点
**初始节点的节点发现过程:**
节点第一次启动时公钥哈希值作为本机节点id保存下来

1. 加载**配置文件**或代码中**硬编码**的已知长期稳定运行的节点种子,包含IP地址或DNS种子(DNS服务器会将该域名对应的IP地址返回)
2. 依次对每个种子节点发送ping探测消息
3. 当对端节点响应pong消息后,存储到分布式哈希表中的相应K桶中



**日常运行节点发现过程：**

节点之前运行过，节点数据库中保存着网络中其他节点的信息，此时节点发现可以依靠节点数据库中的节点获得P2P网络的信息，从而刷新自己的路由表
以太坊节点刷新机制，系统每隔7200ms,7.2s刷新一次K桶。
1.**随机生成**目标节点Id，计数器+1
2.**计算**目标节点id和本机id的距离，记为Dlt
3.根据距离找到路由表中该距离的K桶,**找出k桶中**距离目标节点id小于目标节点id和本机id的距离的α个节点,通常可设为3
4.向这些节点**发送FindNODE**命令，FindNODE命令**包含目标节点id**

5.这些节点收到FindNODE命令后，根据目标节点id计算出响应的桶，将**从K桶中找到**的K个节点使用Neighbours命令响应给本机节点,可能返回不足k个

6. 本机节点**收到Neighbour**s后，将收到的节点ping探测后**写入到K桶**中,如果k桶已满,则PING链表最前面的节点,ping通了，将旧节点挪到列表最底，并丢弃新节点,如果PING不通，删除旧节点，并将新节点加入列表
7. 若搜索次数不超过8次，刷新时间不超过600ms，**循环执行前面步骤**,选取α个尚未查询过的节点向其发送FindNode,当查询了所有k个最近节点，并获得其响应，查找过程终止

以太坊节点在发现邻居节点的8次循环中，所查找的节点均在距离上向随机生成的目标节点id收敛。
由于每次查询都是从最接近t的k-bucket中获取信息，这样的机制保证了每一次递归查询操作都能够获得距离减半的效果，从而保证了整个查询过程的收敛速度的算法复杂度为o(log N)。

节点在离开Kademlia网络的时候并不需要发布任何信息。因为Kademlia协议要求每个节点必须周期性探测路由表中的每一个节点，把下线的节点从路由表中删除。



#### Kademlia算法的优势

❏ 简单性。与其他DHT协议相比，Kademlia是实现和原理都特别简单的协议。例如，与CAN相比，CAN的拓扑结构是基于多维笛卡尔环面的，而Kademlia是基于二叉树的拓扑结构的。Kademlia除了拓扑结构很简单，它的距离算法也很简单，即使用节点ID的异或运算（XOR）。
❏ 灵活性。Kademlia的“K-bucket”可以根据使用场景来动态调整K值，而且对K值的调整完全不影响代码实现。
❏ 性能。Kademlia的路由算法天生就支持并发，而很多DHT协议（包括Chord）没有这种优势。公网上哪怕是同样两个节点之间的传输速率都可能时快时慢。由于Kademlia路由请求支持并发，发出请求的节点总是可以获得最快的那个peer的响应。时间复杂度可控 O(log N)
❏ 安全性。在Kademlia网络中，攻击者想在Kademlia网络中添加一个恶意节点来攻击Kademlia网络是十分困难的。

```go
func (t *UDPv4) loop() // 会启动一个协程循环监听, 等待一个reply

case p := <-t.addpending:  //增加一个pending 设置deadline, 比如ping消息发送时会把pending结构体发送给addpending

case r := <-t.gotreply:  //收到一个reply 寻找匹配的pending

func (t *UDPv4) readLoop(unhandled chan<- ReadPacket) // 处理udp业务 Ping Pong Findnode Neighbors
```



#### 建立加密连接

节点发现后使用RLPx协议进行tcp连接,协商密钥,建立加密连接
每一个节点会开启两个同样的端口，一个是UDP端口，用来节点发现，**一个是TCP端口，用来承载业务数据。**

RLPx协议就定义了TCP链接的加密过程:

1. 链接的两方生成生成随机的私钥，通过随机的私钥得到公钥。
2.  然后双方交换各自的公钥， 这样双方都可以通过自己随机的私钥和对方的公钥来生成一个**同样的共享密钥,**
3. 双方使用共享密钥加密hello消息验证是否能正确加解密,如果正常说明**协商完成**,
4. 后续的通讯使用这个共享密钥作为对称加密算法的密钥。 

握手的过程
RLPx连接基于TCP通信，**并且每次通信都会生成随机的临时密钥用于加密和验证**。生成临时密钥的过程被称作“握手” (handshake)

建立TCP连接-->两方都生成随机的私钥-->密钥协商交换各自的公钥（auth、auth-ack）-->双方通过自己随机的私钥和对方的公钥导出相同的共享密钥-->发送使用共享密钥加密后的hello消息验证是否能正确加解密（协议协商）-->双方验证通过则握手创建完成-->后续的通讯使用这个共享密钥作为对称加密算法的密钥。

RLPx传输协议的前向安全性
这样来说。如果有一天一方的私钥被泄露，也只会影响泄露之后的消息的安全性，对于之前的通讯是安全的(因为通讯的密钥是随机生成的，用完后就消失了)。

RLPx握手被认为是易破解的，因为aes-secret和mac-secret被重复用于读取和写入
。RLPx连接的两端从相同的密钥，nonce和IV生成两个CTR流。如果攻击者知道一个明文，他们就可以用重用的密钥流破解未知明文。



#### 节点连接管理

❏ 如果连接节点数量过多，将会占用节点资源,因此，节点在与其他节点建立P2P连接时，并不需要与全部的相邻节点都建立TCP连接。只要保证节点存在足够数量的连接节点,因为结构化P2P网络可以保证节点产生的交易或者区块可以被广播到网络中任意一个节点。比如默认最多可以同时与50个节点建立TCP连接。当TCP连接数超过50时，节点会主动关闭连接
❏ 如果节点连接数量过少，则节点获取网络中交易和区块信息将会有延迟。尤其对于挖矿节点，如果获取区块信息有延迟，则会严重影响矿机的挖矿效率和收益。一般要大于5个
❏ 使用**动态节点评分机制**节点根据收到的对等节点的数据包的合法性,减少恶意节点对网络的影响。恶意节点会发送大量的无效交易和区块信息，这些无效信息不仅占用了整个网络的资源，也占用了节点的资源，降低节点同步数据的效率。因给对等节点打分,达到一定分数节点会将恶意节点写入本地黑名单，从节点池中删除恶意节点信息，主动停止与恶意节点的TCP连接



#### P2P 网络核心技术：Gossip 协议

Gossip 是非结构化网络，过程是由种子节点发起，当一个种子节点有状态需要更新到网络中的其他节点时，它会随机的选择周围几个节点散播消息，收到消息的节点也会重复该过程，每次散播消息都选择尚未发送过的节点进行散播,收到消息的节点不再往发送的节点广播直至最终网络中所有的节点都收到了消息。这个过程可能需要一定的时间，由于不能保证某个时刻所有节点都收到消息，但是理论上最终所有节点都会收到消息，因此它是一个最终一致性协议。


而 Bitcoin,cosmos 则是使用了 Gossip 协议来传播交易和区块信息

比如 IPFS，Ethereum 等，都使用了 Kadmelia 算法，



#### 内外网穿透

在局域网运行一个区块链节点，在公网是发现和连接不了的，因此比特币和以太坊均使用了 **UPnP 协议**作为局域网穿透工具，UPnP通用即插即用协议是指,只要 NAT 设备（路由器）支持 UPnP，并开启。

那么，当我们的主机上的应用程序比如以太坊节点向 NAT 设备发出端口映射请求的时候，**NAT 设备就可以自动为主机分配端口并进行端口映射**。这样，我们的节点就能够像公网主机一样被网络中任何主机访问了。

![How to Set Up Port Forwarding - Even Behind CGNAT](https://pinggy.io/images/how_to_set_up_port_forwording_even_behind_cgnat/port_forwarding.webp)

因为一旦开启 UPnP，就意味着我们把自己的主机暴露在公网环境中，

任何主机都可以向我们的电脑发起连接,路由防火墙就会完全失效，我们的主机就很容易受到恶意的网络窥探，感染病毒或者恶意程序的几率也大大增加。

**UPnP工作流UPnP的工作过程分为5部分：**寻址（Addressing），发现（Discovery），描述（Description），控制（Control），事件（Eventing）。
我们可以通过发送HTTP请求查询局域网中UPnP设备,如果网络中存在UPnP设备，此设备会发送响应消息获取设备描述URL地址。通过此URL就可以找到根设备的描述信息，从根设备的描述信息中又可以得到设备的控制URL,通过控制URL就可以控制UPnP的行为与之交互,在局域网中成功地找到了一台支持UPnP的设备。
拿到设备的控制URL后就可以发送控制信息了，每一种控制都是根据HTTP请求发送的,其中，path为控制URL; host:port为目的主机地址；actionName为控制UPnP设备执行响应的指令。使用soap
简单对象访问协议操作节点



#### 节点状态机

无论是种子节点状态刷新还是节点对于对等节点的UDP数据包处理，最终都是交由节点状态机处理。节点状态机主要就是根据节点的响应数据包更新节点状态的
有已知,未知,待验证,验证中几种状态
通常在以下几种情况下，会发生节点状态的改变：
❏ refresh方法刷新节点状态时，请求的节点如果是unkown状态，则会将状态变为verifyinit。
❏ findnodeQuery方法向remote节点查询其他节点的信息时，remote节点状态必须是known，如果节点状态为unknown，会通过transition方法将节点状态变更为verifyinit。之后通过在节点间发送探测包，将节点状态慢慢转移为known。
❏ handleNeighboursPacket方法接收到remote节点发送的邻居节点，如果邻居节点的状态为unknown，则会通过transition方法将邻居节点状态转移为verifyinit。
❏ 当节点状态为known时，需要将节点加入到K桶中，如果节点对应的K桶满了，需要将节点变为contested节点。

当对等节点状态为known时，节点会将对等节点加入到路由表中。当对等节点状态为unknown时，节点会从路由表中删除对等节点信息

（2）当对等节点状态为verifyinit时
节点收到对等节点的ping包，节点会给对等节点响应一个pong包，并将对等节点状态设置为verifywait。
节点收到对等节点的pong包，节点将对等节点状态设置为remoteverifywait。
节点向对等节点发送ping包，如果节点在pongTimeout（1s）内没有收到对等节点发回的包，节点会将对等节点状态设置为unknown。
如果在超时时间内收到pong包,则将节点状态设置为unknow

setupDialScheduler
会启动一个协程loop处理节点连接任务,取消连接任务,添加节点,删除节点,删除过期连接

srv.setupListening()启动一个单独线程(listenLoop())去监听某个端口有无主动发来的tcp连接,这里的监听端口和UDP的端口是一样的。 默认都是30303
创建maxAcceptConns个槽位。 我们只同时处理这么多连接。 多了也不要。
检查是否在白名单内。 如果不在白名单里面。那么关闭连接。
SetupConn,这个函数执行握手协议，并尝试把连接创建位一个peer对象,在dialTask.Do里面也有调用
1.执行rlpx协商
2.当两次握手都已经完成了。 把连接对象发送给addpeer队列。 server.run从这个队列中取出连接,启动一个goroutine里面处理会处理这个连接,创建peer， 启动了两个goroutine线程。 一个是读取。一个是执行ping操作。

go srv.run()
维护peers：所有建立了连接的peer；
维护trusted：信任的节点，因为dial其他节点的时候都要进行验证，对信任的节点可以加速验证过程；被信任的节点有这样一个特性， 如果连接太多，那么其他节点会被拒绝掉。但是被信任的节点会被接收。
按照for循环，Server的addpeer通道传出conn对象的时候，也就是收到连接后,就会执行newPeer()，然后启动一个协程执行peer.Run()，从而实现两个节点之间的连接。
在p2p代码里面。 peer代表了一条创建好的网络链路。在一条链路上可能运行着多个协议。比如以太坊的协议(eth)。 Swarm的协议。 或者是Whisper的协议。

peer的启动，启动了两个goroutine线程。 一个是读取。一个是执行ping操作。
**readLoop协程**调用p.rw来读取一个Msg这个rw实际是之前提到的frameRLPx的对象，也就是分帧之后的对象。然后根据Msg的类型进行对应的处理，如果Msg的类型是内部运行的协议的类型。那么发送到对应协议的proto.in队列上面。protoRW有read和write方法。 可以看到读取和写入都是阻塞式的。

**pingLoop**。这个方法很简单。就是定时的发送pingMsg消息到对端。



## 以太坊同步协议

节点之前的连接建立后,可以交互以太坊协议消息
建立连接后，首先必须发送状态消息,状态消息包含了**区块链网络id**和**最高区块高度**的**区块hash**等,接收到节点的状态消息后，以太坊会话将被激活，继而可以发送其他任何类型的消息。

#### 主动区块链下载同步

同步有两种模式，**分别是fastSync和fullSync模式。** **根据区块高度差判断**是否使用快速同步还是全同步

**快速同步模式**：是一种**不需要执行交易**的步骤，而是直接下载区块头和全球状态树和收据的同步方式，区块头验证通过后**直接修改本地状态树**,用于使本地链快速的跟上规范链的更新速度。
**全量同步模式:** **验证所有**的交易并执行交易，在本地生成状态和收据。这种同步方式缓慢且消耗CPU和磁盘。

**区块同步流程:**
1.从k桶节点中**找出拥有最高td的节点**，计算高度差得到需要同步的区块高度,根据高度差判断是使用快速同步还是全量同步
2、每次向节点同步当前区块高度+1的**区块头**,直到同步到完所有区块头信息；
3、同步了区块头后，从其他节点**同步状态，收据和交易**,这里区块头、区块主体的下载和区块执行可能同时进行；
4、**到达（距离最高区块高度64）时，关闭fastSync，采用fullSync模式；**
5、同步完成后,广播自己的头区块，**允许接受广播的交易**

同步过来的区块要验证该区块是否是孤块。如果是孤块则存入孤块池，并继续请求当前高度父区块；如果不是孤块则添加到本地区块链中，且高度+1

启动节点同步过程

```
lifecycle.Start()-->(s *Ethereum) Start()-->s.handler.Start(maxPeers)
-->go h.chainSync.loop()
-->cs.handler.blockFetcher.Start()
-->cs.handler.txFetcher.Start()
-->cs.startSync(op)
-->cs.handler.doSync(op
```

startSync任务
1、前提是；
2、获取同步模式：fastSync或fullSync,根据区块高度判断是否使用快速同步还是全同步；
3、使用该同步模式进行一次同步；
4、同步完成后允许以太坊节点接受其他节点广播的交易；
5、将当前块广播出去。

h.downloader.Synchronise(op.peer.ID(), op.head, op.td, op.mode)
负责区块链同步的主要工作，在最开始的同步中从远端节点下载区块链信息

#### 被动根据区块通知同步Fetcher

收集其他节点通知它的信息：**NewBlockMsg新区块通知(**包含完整的区块信息，并发送给一小部分已连接的节点)或**NewBlockHashMsg新区块hash通知**(发送给其他全部节点，仅包含新区块的哈希值)

根据通知的消息，如果是完整区块，就可以传递给eth插入区块，
如果只有区块Hash，则需要向其他节点发起查询区块头请求获取区块头,收到响应后再发起查询区块体请求获取区块体，最后再传递给eth插入区块。这个过程中还要区分是下载同步模块还是块通知同步模块查询回来的区块响应进行过滤

blockFetcher.Start和txFetcher.Start任务
通知:

```go
var eth66 = map[uint64]msgHandler{
	NewBlockHashesMsg:             handleNewBlockhashes,
	NewBlockMsg:                   handleNewBlock,
    GetBlockHeadersMsg:            handleGetBlockHeaders66, 
    BlockHeadersMsg:               handleBlockHeaders66,

    GetBlockBodiesMsg:             handleGetBlockBodies66,
    BlockBodiesMsg:                handleBlockBodies66,

    GetNodeDataMsg:                handleGetNodeData66,
    NodeDataMsg:                   handleNodeData66,
    GetReceiptsMsg:                handleGetReceipts66,
    ReceiptsMsg:                   handleReceipts66,

    TransactionsMsg:               handleTransactions,
    NewPooledTransactionHashesMsg: handleNewPooledTransactionHashes,
    GetPooledTransactionsMsg:      handleGetPooledTransactions66,
    PooledTransactionsMsg:         handlePooledTransactions66,
}
```


handleMessage处理其他peer发送过来的通知消息,进行相应处理
etcher处理区块时将区块设定为四种状态：announced、fetching、fetched和completing。

节点收到区块hash后,Notify()方法将传入的参数拼成announce对象，然后send给f.notify通道。
fetcher的主循环loop()中会从该通道取得这个announce，进行处理。
1、确保节点没有DOS攻击；
2、检查高度是否可用；
3、检查是否已经下载，即已经在fetching或comoleting状态列表中的announce
4、添加到announced中，供后面程序调度。
5、当f.announced==1时，代表有需要处理的announce，则重设fetchTimer.C定时器，由loop处理进行后续操作。

loop()中收到定时器fetchTimer.C通知时
1、遍历announced中的每个hash的消息，构建request,发送的消息是GetBlockHeadersMsg；
2、把所有的request都发送出去，每个peer都有一个协程，并从peer获得应答（fetchHeader(hash)）；

远程节点收到getBlockHeaderMsg后调用pm.blockchain.GetHeaderByHash或者pm.blockchain.GetHeaderByNumber从数据库中取出Header，发送给原来请求的节点，调用SendBlockHeaders(headers)，发送BlockHeadersMsg。
BlockHeadersMsg是一个特殊的消息，因为这个消息不止是fetcher要用，downloader也要用。所以这里需要用一个fetcher.FilterHeaders()去把fetcher需要的区块头给过滤下来处理直到完成放到fetched或者completing列表中，剩下的结果直接交给downloader去下载。

```
case filter := <-f.headerFilter:处理过程
```

从f.headerFilter取出filter，然后取出过滤任务task
它把区块头分成3类,遍历所有的区块头，填到到对应的分类中
unknown的不是fetcher请求的，返回给发送者,给Download处理
complete放没有交易和uncle的区块，有头就够了，
incomplete存放还需要获取body的区块头,把incomplete的区块头获取状态移动到fetched状态，然后触发completeTimer定时器，以便去处理complete的区块。

```
case <-completeTimer.C:处理过程
```

即先转变状态从fetched状态到completing状态
遍历所有待获取body的announce然后构建request。这里实际上调用了RequestBodies函数，发出了一个GetBlockBodiesMsg，从远程节点获得一个BlockBodiesMsg。
响应BlockBodiesMsg的处理与headers类似，需要有一个FilterBodies来过滤出fetcher请求的bodies，剩余的交给downloader。

```
filter := <-f.bodyFilter:处理过程
```

1、fetcher要的区块，单独取出来存到blocks中，它不要的继续留在task中。
2、判断是不是fetcher请求的方法：如果交易列表和叔块列表计算出的hash值与区块头中的一样，并且消息来自请求的Peer，则就是fetcher请求的。
3、将blocks中的区块加入到queued。

#### 广播

交易广播,当接收到**交易池添加新交易**的事件后将该交易广播出去

```
go h.txBroadcastLoop()
```

该方法将针对每一个交易，准确的将它发送到所有没有该交易的节点。

区块广播,当挖矿成功或同步到新区块后,将区块广播给不含该区块的节点

```
go h.minedBroadcastLoop()
```

1、筛选p2p节点中不含当前区块的节点；
2、如果propagate为true，将区块block和总难度td发送给一部分节点，节点数为根号n；
3、如果propagate为false，将区块的hash发送给所有的节点。

-------------------------------------------------------------------------------------------------------------------------------
## 区块结构

block结构实际上就只有两个部分，**header和body**。其他的如hash、size、td等都是在接受和验证区块后产生的内容，实际上向全网公布的时候block就只有header和body（uncles和transactions）

#### **Header结构体**

**1）ParentHash：前一个区块的hash**
**2）UncleHash：叔区块hash，如果有多个叔区块就加到一起**
**3）Coinbase：矿工账户**
**8）Difficulty：难度值**
**9）Number：区块号**

**4）Root：StateDB中的“state Trie”的根节点的RLP哈希值。状态树根hash**
**5）TxHash：“tx Trie”的根节点的RLP哈希值。交易数根hash**
**6）ReceiptHash： "Receipt Trie”的根节点的RLP哈希值。收据树根hash**

**7）Bloom：Bloom过滤器(Filter)，用来快速判断一个日志对象是否存在于一组已知的Log集合中。**
**10）GasLimit：区块内所有Gas消耗的理论上限。该数值在区块创建时设置，与父区块的GasUsed有关。**
**11）GasUsed：区块内所有Transaction执行时所实际消耗的Gas总和。**
**12）Time：时间戳**
**13）Extra：额外信息**
**14）Nonce：pow产生的数值，也可以用于验证矿工的工作**

**状态树根**
状态树是所有**账户信息**构成,该区块只修改的一部分账户信息,但需要重新计算出全球状态树的树根
**交易树根**
交易树是该区块的**所有交易**构成MPT树,key是交易hash,value是具体交易值
**收据树**
收据树是该区块**交易执行完的产生的收据**构成MPT树,key是收据hash,value是交易收据

**收据包括**
**1.交易执行状态**(值是0或1),如果gas费不够（也有可能是因为其他原因），整个交易可能会执行失败回退
**2.CumulativeGasUsed** 区块中已执行的交易累计消耗的Gas，包含当前交易。
**3.logs日志,**当前交易执行所产生的智能合约事件列表。一条日志包含主题和数据,一条日志可以有4个主题。主题总容量受限，最好不要存储大量数据。通常来说，想作为搜索关键字的数据，可以存储为topic。其他数据存储到data里
比如将智能合约中token转账的函数,发送者地址,接收者地址存入日志中的主题
这样可以快速查询某个交易合约今天的交易记录
过去一个小时，某用户发送token记录
上周，某用户接收token记录
**4.logs Bloom过滤器**:用于快速检测某主题的事件是否存在于日志中,将日志中的主题信息映射到位图中,可以通过布隆过滤器快速查询“可能存在”和“一定不存在”
在区块头中有一个总的Bloom过滤器，是区块中所有交易收据的Bloom过滤器的并集。



**收据的作用**
1.投分析或其他需求，要想追踪合约相关信息变更历史，需要到日志里查找
2.智能合约没法直接返回结果给前端（或者后端）。将事件写到日志里，这样dApps可以监听日志，以便获得通知来展示他们。
3.需要跨链时,交易会记录到日志。网关通过日志将会获取数据

智能合约如果想记录日志，代码里可以使用emit，这样就可以将收据记录到日志记录中

当我们需要查找过去某段时间某个智能合约相关的所有交易时，
第一步，在区块头中布隆过滤器查找是否存在相关的交易类型；
第二步，若存在，则在区块内部的所有收据里的布隆过滤器中查找。
若第一步中没有查到，则无需第二步了，直接查找下个区块。因此通过收据中的布隆过滤器可以快速排除掉无关的收据，提高了查询效率。

receiptTrie必须在Block的所有交易执行完成后才能生成；txTrie理论上只需tx数组transactions即可，不过依然被限制在所有交易执行完后才生成



**区块头、区块体、收据以RLP编码的形式储存在数据库中**

| key                           | value                                                        |
| ----------------------------- | ------------------------------------------------------------ |
| 'h' + 区块号 +区块hash        | header的RLP编码值                                            |
| 'b' + 区块号 +区块hash        | body的RLP编码值                                              |
| 'H' + 区块hash                | Header对应的区块号（规范链）                                 |
| 'r' + 区块号 +区块hash        | receipt的RLP编码值                                           |
| 'h' + 区块号 + 区块hash + 't' | 截止该区块的总难度值                                         |
| 'h' + 区块号 + 'n'            | 区块号对应的规范链上的区块的hash（规范链 ）                  |
| 'l' + hash                    | 【区块hash、区块号num、交易编号】的编码值，是交易查询入口（规范链） |

num(区块号)和hash(区块hash)是Block最为重要的两个属性：区块号用来确定Block在整个区块链中所处的位置，hash用来辨识惟一的区块对象。

#### body结构体

**1.交易数组**

**2.叔区块数组,**设计Uncles的目的是为了对挖出分叉区块的节点进行奖励,并提高区块链包含的工作量提高共识

区块block结构体数据结构

```
1.区块头
2.交易数组
3.叔区块数组
4.hash值,计算出的区块头hash,区块头本身并不保存该hash值,通过这个hash可以将区块和区块头一一对应,同步时先传输Header对象，验证通过后再传输Block对象，收到后还可以利用二者的成员哈希值做相互验证。
5.总难度,过去所有区块的难度之和
```

区块链blockchain数据结构

```
1）db：连接到底层数据储存，即leveldb；
2）hc：headerchain区块头链，由blockchain额外维护的另一条链，由于Header和Block的储存空间是有很大差别的，但同时Block的Hash值就是Header（RLP）的Hash值，所以维护一个headerchain可以用于快速延长链，验证通过后再下载blockchain，或者可以与blockchain进行相互验证；
3）genesisBlock：创始区块；
4）currentBlock：当前区块，blockchain中并不是储存链所有的block，而是通过currentBlock向前回溯直到genesisBlock，这样就构成了区块链。

5）bodyCache、bodyRLPCache、blockCache、futureBlocks：区块链中的缓存结构，用于加快区块链的读取和构建；
6）engine：是consensus模块中的接口，用来验证block的接口；
7）processor：执行区块链交易的接口，收到一个新的区块时，要对区块中的所有交易执行一遍，一方面是验证，一方面是更新worldState；
8）validator：验证数据有效性的接口
9）futureBlocks：收到的区块时间大于当前头区块时间15s而小于30s的区块，可作为当前节点待处理的区块。
10)badBlocks *lru.Cache // Bad block cache，来自DAO事件
```

**HeaderChain**是BlockChain额外维护的另一条链，其结构与后者有很大的相似性，包括要连接数据库、genesisHeader、currentHeader、currentHeaderHash以及缓存和engine。
不同之处在于，HeaderChain中由于不存在block的body部分，所以也就用不到processor和validator这两个接口,因此HeaderChain要比blockChain小很多

创建：NewBlockChain
增加、插入：InsertChain—>insertChain—>WriteBlockWithState
删除：Reset —> ResetWithGenesisBlock
查询：getBlockByNumber等

#### **初始化blockchain流程：NewBlockChain()**

1. 配置cacheConfig，创建各种lru缓存例如bodyCache,receiptsCache,txLookupCache,futureBlocks
2. 初始化bc := &BlockChain{}结构体
3.初始化State database
4.初始化区块和状态验证：NewBlockValidator()
5. 初始化状态处理器：NewStateProcessor()
6. 初始化区块头链：NewHeaderChain()
7. 查找创世区块
8. 加载最新的状态数据：bc.loadLastState() 就是要找到最新的区块头，然后设置currentBlock、currentHeader和currentFastBlock
9. 检查本地区块链上是否有bad block，如果有调用bc.SetHead回到硬分叉之前的区块头
10. go bc.update() 开启协程,定时处理future block

#### **BlockChain 区块插入和校验流程:InsertChain()**

主要功能：调用insertChain将一组区块挨个尝试插入数据库和规范链。
1、验证每个区块的header是否连续；
2、验证每个区块的body中的交易；
3、处理验证header和body所产生的错误；
4、如果验证通过，对待插入区块的交易状态进行验证，否则退出；
5、如果验证通过，调用WriteBlockWithState将第n个区块插入区块链（写入数据库），然后写入规范链，同时处理分叉问题。


具体流程
1.逐个检查要插入的一组区块的区块号是否连续以及hash链是否连续
2.解析区块信息到headers列表,bc.engine.VerifyHeaders(bc, headers, seals)异步检查区块的正确性
3.处理验证header和body所产生的错误:
  a. 待插入的区块已经是数据库中存在的,并且规范链头区块区块号大于该区块的区块号，直接忽略
  b. 待插入区块的时间戳大于当前时间15秒,是未来区块,就将它放入未来待处理区块缓存中
  c. 在数据库中找不到待插入区块的父区块但未来区块列表中能找到它的父区块,就将它放入未来待处理区块缓存中
  d. 待插入区块是数据库中的一个精简分支（精简分支区块就是一些没有state root的区块）,如果这个区块的总难度小于等于规范链的总难度值，只将它写入数据库，不上规范链,上分支
  e. 不属于上面四种错误，无法处理，那就直接return

查询过程
用区块hash查询数据库得到区块号后，用区块号从数据库中获取区块

（2）插入和删除
区块链与普通单向链表有一点非常明显的不同，就是Header的前向指针ParentHash是不能修改的，即当前区块的父区块是不能修改的。所以在插入的实现中，当决定写入一个新的Header到底层数据库时，从这个Header开始回溯，要保证它的parent以及parent的parent等，都已经写入数据库了。只有这样，才能确保从创世块（num为0）开始，直到当前新写入的区块，整个链式结构是完整的，没有中断或分叉。

删除的情形也类似，要从num最大的区块开始，逐步回溯。在BlockChain的操作里，删除一般是伴随着插入出现的，即当需要插入新区块

## 交易

❑Recipient：接收者地址。
❑Amount：交易转移的以太币数量，单位是wei。
❑Nonce：交易发送者交易数计数。
❑Price：交易的gas price。
❑GasLimit：本交易允许消耗的最大gas数量
❑Payload：交易可以携带的数据，在不同类型的交易中有不同的含义。
❑V, R, S：交易的签名数据。

这里并没有一个字段来指明交易的发送者，因为交易的发送者地址可以从签名恢复出公钥再得到地址。

计算交易hash
对(nonce, gasPrice, gasLimit, to, value, data, chainId, 0, 0)字段按顺序排列进行RLP编码；因涉及哈希运算，因此不能随意调整字段定义顺序
对上面的RLP编码值进行Keccak256计算hash

对上面的Keccak256值进行ECDSA私钥签名（即正向算法）；
对上面的ECDSA私钥签名(v、r、s)结果与交易消息再次进行RPL编码,就可以将交易广播出去了

交易的类型，在源码中交易只有一种数据结构，一般包括：**转账的交易，创建或者执行合约的交易**
转账交易是一种最简单的交易，这里转账是指从一个账户向另一个账户发送以太币。
发送转账交易的时候只需要指定交易的接收者、转币的数量，这里data是空的。

创建合约是将合约部署到区块链上，这也是通过发送交易来实现。
在创建合约的交易中，to字段要留空不填，data字段指定合约的二进制代码

执行合约的交易，就是调用合约中的方法，
需要将交易的to字段指定为要调用的合约的地址，通过data字段指定要调用的方法以及向该方法传递的参数。

任何节点会默认将接受到得合法交易及时发送给邻近节点。得益于P2P网络，一笔交易平均在6s内扩散到整个以太坊公链网络的各个节点中。

## 交易池

交易池中交易来源：**本地的交易和远端节点广播的交易**；
交易池中交易去向：被挖矿模块获取并验证，用于挖矿；挖矿成功后写进区块并被广播；

Miner取走交易是复制，交易池中的交易并不减少。直到交易被写进规范链后才从交易池删除
交易如果被写进分叉，交易池中的交易也不减少，等待重新打包。

#### 交易池

	type TxPool struct {
		config       TxPoolConfig
		chainconfig  *params.ChainConfig
		chain        blockChain
		gasPrice     *big.Int
		txFeed       event.Feed
	    chainHeadCh  chan ChainHeadEvent  // txpool订阅区块头的消息
	    chainHeadSub event.Subscription  //  区块头消息订阅器，通过它可以取消消息
	    signer       types.Signer    //  封装了事务签名处理，椭圆算法
	
	    currentState  *state.StateDB      // 当前头区块对应的状态
	    pendingState  *state.ManagedState // 假设pending列表最后一个交易执行完后应该的状态
	    currentMaxGas uint64              // Current gas limit for transaction caps
	
	    locals  *accountSet // Set of local transaction to exempt from eviction rules
	    journal *txJournal  // Journal of local transaction to back up to disk
	
	    pending map[common.Address]*txList   // 可执行交易队列*当前可以处理的交易，key为交易发起者地址，交易按nonce交易计数排序
	    queue   map[common.Address]*txList   // 非可执行交易队列*当前暂时不能处理的交易,交易按照nonce进行排序
	    beats   map[common.Address]time.Time // 每个账号最后一次交易时间
	    all     *txLookup                    // 全局交易列表
	    priced  *txPricedList                // 所有交易按价格排序
	}
交易池可以配置交易进入交易池的最低 gasPrice 要求
交易池中单个账户非可执行交易上限，默认是64笔

本地交易在本地磁盘存储。这样，本地交易不会丢失，重启节点时可以重新加载到交易池，实时广播出去
本地交易可优先于远程交易。对交易量的限制等操作，不影响本地的账户和交易

交易入池验证：validateTx
1、交易的Nonce不能小于此账户当前Nonce的交易；
2、确保交易签名的正确性；只有合法的签名才能成功解析出签名者。一旦解析失败拒绝此交易。
3、账户余额必须足够
4、交易固定gas消耗不能大于交易设置的gasLimit

5、不是本地交易或本节点白名单中没有包含这个交易地址的情况下，交易的gas不能小于txpool下限；
6、最新状态中交易发起方的余额不能小于交易gas总消耗；
7、交易转账值不能为负；
8、交易的size不能过大.
9.交易的gas值超过了当前规范链头区块的gas值；

在以太坊中一个message的gas的计算方式是被规定好的：
1、如果是合约创建且是家园版本，固定消耗为53000gas；
2、如果是普通交易，固定消耗是21000gas；
3、非0值数据消耗：固定消耗+非0值数量*68，以64位能表示的最大值为封顶，超过则报错；
4、0值数据消耗：固定消耗+0值数量*4，同样以64位能表示的最大值为封顶，超过则报错。

添加交易TxPool.add()
add()方法用于将本地或远端的交易加入到交易池，这个方法的基本逻辑是：
1）检查交易是否收到过，重复接受的交易直接丢弃；
2）验证交易合法性；
3）如果交易池满了（pending+queen），待插入的交易的手续费比交易池中任意一个都低，则直接丢弃；否则移除优先级最低交易 
4）如果待插入的交易序号在可执行交易队列pending列表中已经存在，且待插入的交易的手续费大于或等于原交易的110%，则替换原交易，
5）如果待插入的交易序号在可执行交易队列pending列表中没有或无法被替代，则直接放入非可执行交易队列queue列表。如果对应的序号已经有交易了，则如果新交易的手续费大于或等于原交易的110%，替换原交易；

并非所有进入交易池的交易均被广播，而是只有交易从非可执行状态变成可执行状态后才会发送信号
一是交易时用于替换已经存在的可执行交易时。
二是有新的一批交易从非可执行状态提升到可执行状态后

可执行交易队列pending和非可执行交易队列queue的存放逻辑:
1、pending待处理列表中有tx3、4、5，这意味着address A 发起的交易tx0、1、2都已经被插入规范链了。
2、queue排队等待中有tx7、tx8是因为该节点没有收到tx6的交易，所以tx7、8暂时不处理；
3、tx7、8等待处理的时间是有限的，如果超过30分钟（当前时间-beats）还没有收到tx6，则会将7、8抛弃；
4、如果收到一笔新交易，交易池先把该交易往pending中进行比对和替换，如果pending中有则替换，没有则放入queue
5、如果又收到另一笔tx4，交易池会对原有tx4进行替换，替换的条件是新交易的价值要超过原交易的110%。


添加交易TxPool.add()调用时机
1.命令行发送交易
EthApiBackend.SendTx
    ——> TxPool.AddLocal
        ——> TxPool.addTx
            ——> TxPool.add
2）交易池重新整理的过程中
TxPool.reset
    ——> TxPool.addTxLocked
        ——> TxPool.add
3）收到远程节点广播的交易时
AddRemotes
    ——> TxPool.addRemotes
        ——> TxPool.addTxs
            ——> TxPool.addTxLocked
                ——> TxPool.add


TxPool.runReorg整理交易池
触发时机:TxPool收到链更新事件。

流程：
1.重置TxPool.reset(),
   1）找到由于规范链更新而作废的交易，比如在新区块来到后，删除所有低于新nonce的交易。 再根据账户可用余额，来移除交易开销（amount+gasLimit*gasPrice）高于此余额的交易
   2）给交易池设置最新的世界状态；
   3）把旧链退回的交易重新放入交易池；
2.交易池升级,由于规范链更新，将queue里面部分交易升级移到pending里。
3.交易池降级,由于分叉引起的降级，将pending中部分交易移到queue里面；
4.控制交易池pending和queen的size

交易池升级：
TxPool.promoteTx
主要功能是将交易放入pending列表中。这个方法与add的不同之处在于，add是获得到的新交易插入queen，而promoteTx是将queue排队等待列表中的Txs放入pending待处理列表。

TxPool.promoteExecuables
promoteExecuables()的主要任务是把给定的账号地址列表中已经变得可以执行的交易从queue列表中插入到pending中，同时检查一些失效的交易，然后发送交易池更新事件。
1.首先从queue中取出要升级交易的账户的交易列表
2.过滤交易序号过低或交易余额不足的交易或gas过大的交易，然后从all交易池中删除
3.收集所有可以被加入pending列表的交易，依据是交易序号连续且递增
4.遍历交易使用TxPool.promoteTx填入pending

交易池降级：
交易降级有三种可能的情况：
1、分叉导致Account的Nonce值降低：假如原规范链上某账户的交易序号m都已经上链，但分叉后新规范链上交易序号m没有上链，这就导致在规范链上的记录的账户的Nonce降低（由m+1变成了m），这样交易m就必须要回滚到交易池，放到queue中；
2、分叉后出现间隙：间隙的出现通常是因为交易余额问题导致的。假如原规范链上交易m花费100，分叉后该账户又发出一个交易m花费200，这就导致该账户余额本来可以支付原来规范链上的某笔交易，但在新的规范链上可能就不够了。这个余额不足的交易如果是m+3，那么在m+2，m+4号交易之间就出现了空隙，这就导致从m+3开始往后所有的交易都要降级；
3、分叉导致pending最前一个交易的nonce值与状态的nonce值不等

TxPool.demoteUnexecutables
主要功能是从pending中移除无效的交易，同时将后续的交易都移动到queue。
1）丢弃nonce值过低的交易：list.Foward(nonce)
2）删除账户余额已经不足以支付交易的交易：list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
3）将暂时无效的交易移动到queue：

4）如果前面有间隙，将后面的交易移到queue中。



## RLP编码

RLP(Recursive Length Prefix) 递归长度前缀编码  是以太坊中最常使用的序列化格式方法。到处都在使用它，如区块、交易、账户、消息等等
一是长度前缀，数据编码后带有一个前缀，这个前缀是跟被编码数据的长度,并且当数据长度大时,前缀还会有长度的长度
二是递归，被编码的数据是递归的结构，编码算法是递归进行处理数据结构的；
编码时同级节点则通过字段名有序排列,确保了编码后的一致性，因为每笔交易过程中要进行Keccak256，如果不能保证编码后的一致性，会导致其Hash值不同

RLP编码的优点
1.使用了灵活的长度前缀来表示数据的实际长度，并且使用递归的方式能编码相当大的数据。可以通过前缀来判断数据的长度和类型
2.rlp编码后的体积小,相比于json或者protobuf编码序列化时引入了太多的冗余信息如字段名或字段编号导致编码结果体积比较大。

RLP编码只对3种数据类型编码：针对不同的类型有不同的编码方式,其他类型的数据需要转成这几种类型,原子数据类型（比如：字符串，整数型，浮点型）可以转成字节数组。struct、map 等类型可以转成列表
类型1：值在[0，127]之间的单个字节,对于值在[0, 127]之间的单个字节，其编码是其本身；
类型2：字节数组（元素数可为0）,如果byte数组的长度l<=55，短字节数组 编码的结果是128+字节数组长度作为前缀,再加上字节数组本身
                            如果数组长度大于55，长字节数组 编码结果第一个值是183（128+55）加数组长度的编码的长度，然后是数组长度本身的编码，最后是byte数组的编码。
类型3：列表（列表是以数组或列表为元素的数组，元素数不可为0）短列表 如果列表元素的编码总长度小于55，编码结果第一位是192加列表长度的编码，然后依次连接各个子列表的编码。
													   长列表 如果列表元素的编码总长度长度超过55，编码结果第一位是247(192 + 55)加列表长度的编码长度，然后是列表长度本身的编码，最后依次连接子列表的编码

RLP 解码流程
首先根据编码结果第一个字节f的大小,判断其后数据类型是单字节,短字节数组,长字节数组,短列表,长列表,再根据类型使用相应方法对数据进行解码
当接收或者解码经过RLP编码后的数据时，根据第1个字节就能推断数据的类型、大概长度和数据本身等信息。而其他的序列化方法，不能根据第1个字节获得如此多的信息量。

对比protobuf编码,protobuf编码也是一种二进制编码,
Protobuf 是一种紧密的消息结构, 编码后字段之间没有间隔, 每个字段头由两部分组成: 字段编号和编码类型,最后存储编码后的数据,解码时可根据该编码类型对数据进行解码
Protobuf 没有使用字段名称, 而仅仅采用了字段编号,发送端和接收端都需要用proto 文件定义的每个字段的编号,Protobuf 在通信数据中移除字段名称, 这可以大大降低消息的长度, 提高通信效率
编码类型:
Varint变长整型编码,每个字节的最高有效位作为标志位, 而剩余的 7 位以二进制补码的形式来存储数字值本身, 当最高有效位为 1 时, 代表其后还跟有字节, 当最高有效位为 0 时, 代表已经是该数字的最后的一个字节,
但由于它在每个字节的最高位额外采用了一个标志位来标记其后是否还跟有有效字节, 因此对于大的正数, 它会比使用普通的定长格式占用更多的空间
并且对于负数反而会占用更多的空间

对于负数使用Zigzag 编码,首先对负数做一次变换, 将其映射为一个正数, 变换以后便可以使用 Varints 编码



## MPT树 默克尔帕特里夏树

是默克尔树和帕特里夏压缩前缀树的混合

Merkle树又叫哈希树，是一种二叉树,最下面的叶节点包含存储数据和其哈希值，每个中间节点的hash值是它的两个子节点的哈希值组合进行hash计算得到，从叶子节点开始从下往上一直计算出根节点的hash值
merkle数
使用merkle树保证区块中的数据的不可篡改性,就可以通过算法来证明数据的正确性

通常来说，简单Merkle树构建之后就不再发生变化，然而采用账户模型的区块链系统需要记录并经常更新每个账户的状态，简单Merkle树并不适用于这种场景。因为每次数据更新之后，都需要重新计算Merkle树散列值，而这一操作的复杂度为。以太坊为账户状态的存储和更新设计了Merkle Patricia树（MPT）数据结构，采用账户模型的Cosmos-SDK则设计了Immutable AVL+（IAVL+）树数据结构

任意的Merkle树都可以构造元素的存在性证明，根据根据待证明节点和证明中包含的中间节点计算出根hash，然后比较区块头中包含的根hash,是轻客户端验证一笔交易合法性的基本过程
只有Merkle树的叶子节点对应的元素值有序排列时，才可以构造元素的非存在性证明。

如果区块体中的交易数不是2的整数次幂，则在Merkle树的逐层计算中会出现某一层的节点个数为奇数的情况。比特币中通过复制最右侧的节点来应对这种情况
另外再验证时会首先判断相邻两个节点是否具有相同的散列值，如果是的话，意味着该区块是利用安全隐患构建出来的问题区块

以太坊不同于比特币的 UXTO 模型，在账户模型中，账户存在多个属性（余额、代码、存储信息），属性（状态）需要经常更新。账户存放在叶子节点,账户地址作为路径,因此需要使用前缀树,
因为账户地址的分布比较稀疏,因此使用帕特里夏压缩前缀树,提高信息检索效率

以太坊的MPT树将帕特里夏树中的节点存储了其他节点的hash值,形成hash指针，就构建成了默克尔帕特里夏树，可以计算出树的根哈希值，保存在区块头中。

MPT树优点
①在执行插入、修改或者删除操作后能快速计算新的树根，而无需重新计算整个树。
②即使攻击者故意构造非常深的树，它的深度也是有限的。否则，攻击者可以通过特意构建足够深的树使得每次树更新变得极慢，从而执行拒绝服务攻击。
③树的根值仅取决于数据，而不取决于更新的顺序。以不同的顺序更新，甚至是从头重新计算树都不会改变树的根值。

MPT树中的节点包括：
❑叶子节点（leaf），表示为 [key, value]的一个键值对，其中key是key的一种特殊十六进制编码，value值的hash值的RLP编码。
❑分支节点（branch），是一个长度为17的列表，前16个元素对应着账户地址编码中的16个可能的十六进制字符作为中间节点，如果有一个路径编码在这个分支节点终止，最后一个元素存放该值，因此分支节点既可以是路径的终止也可以是路径的中间节点。
❑扩展节点（extension），key是压缩前缀，但是这里的value是其他节点的Hash值，这个Hash可以被用来查询数据库中的节点。也就是说通过Hash链接到其他节点。

拓展节点用于存储压缩的路径,减少树的深度,当一串路径编码中不会发生分叉时,可以将这一串路径编码存储在一个拓展节点中
分支节点用于存储在某个共同路径编码后出现分叉时,将这些分叉的编码共同存储在分支节点中,并指向相应的后续节点

❑叶子节点和分支节点可以保存RLP编码值，扩展节点保存指向节点的hash值；
❑有公共的key则根据公共前缀提取为一个扩展节点；这个扩展节点的key为所有数据在公共前缀之外拼接字节序列，value为子节点的Hash；
❑没有公共的key就使用分支节点进行分叉；

因为以太坊地址是16进制字符组成,因此每个地址字符可以存放在4bit中,两个地址字符可以存储在一个字节中,这样就可以减少空间浪费,因为每个节点存储的地址字符长度不同,奇数长度就会使用前面空闲的4bit位存放前缀,偶数长度则新增一个字节使用其中4bit位存放前缀
前缀是一个4比特的数值，用来区分扩展节点和叶子节点。因为扩展节点和叶子节点的数据结构类似，所以需要进行区分。

其中第三位为节点标记，如果是0则表示拓展节点，如果是1则表示叶子节点。最后一位为奇偶性，如果节点key路径编码长度为奇数，则为1，0为偶数,



## 以太坊采用多级存储

第一级缓存 stateDB.以 map 的形式存储账户,是账户状态的更新以及查询入口
第二级缓存以 MPT 的形式存储账户
第三级就是 LevelDB 上的持久化存储账户

以太坊的StateDB的生命周期是一个区块,一旦产生下一个区块，StateDB就会立刻清空。stateDB作为缓存保存账户信息的map,这些账户信息是从数据库中找到目标账户存入,避免重复从树中读取，提高效率,并且向StateDB的日志中存入反向操作日志，支持回滚状态

stateDB生命周期：
1.从交易池打包交易进行挖矿或或者收到区块广播时,执行交易前初始化stateDB，使用给定的树根从数据库中读取节点信息进行rlp解析构建状态树（不是完整的状态树），数据库中抽象掉状态树实现方面的细节，它就是把所有账户和合约存储槽堆放在一起，都由扁平的键值对来表示。
2.解析交易涉及的账户，向statusDB存的账户map中查询，如果没有则向数据库查询或着新建实例化一个账户对象添加到statusDB的账户map中，同时添加创建新账户反向操作到日志并日将志索引存入statusDB中，生成快照
3.查询账户余额等具体信息时，先从stateDB中获取账户，获取具体账户信息，比如获取合约账户的存储树，该树是懒加载，只有在第一次使用时，才加载树。因为不更改账户，因此不产生日志
4.当执行交易时，变更的账户记录在脏账户map中,产生账户变动的反向操作交易存入stateDB日志并保存日志索引，产生快照,此时所有的数据改动还仅仅存储在statusDB的map里
5.如果交易执行失败，通过快照编号获取日志索引，读取操作日志给定日志索引从后往前执行回滚日志进行回滚，移除恢复点后面的快照
6.执行完所有交易后，脏账户map中添加该将改动的账户，将被更新的账户写入状态树，计算状态树根，放在区块头中打包用于挖矿或者用于验证收到的广播区块
7.最后将状态树进行rlp编码写入数据库，清除变更日志、快照,statusDB的生命周期结束，从内存中删除




	
	type StateDB struct {
		db   Database //底层的数据库
		trie Trie     //trie
	
	    // 这个map相当于是stateDB的缓存，存放着活动的账户。如果有账户在这里找不到，则通过trie从数据库中找到目标账户，然后存到这个map里
	                         避免重复从树中读取，提高效率。所有获取过的数据，将会缓存在 map[common.Address*stateObject 中。
	    stateObjects      map[common.Address]*stateObject 
	    stateObjectsDirty map[common.Address]struct{} //脏账户,在StateDB中，账户已经被修改了，但是状态树中的值还没有修改
	
	    thash, bhash common.Hash
	    logs         map[common.Hash][]*types.Log
	
	    // Journal of state modifications. This is the backbone of
	    // Snapshot and RevertToSnapshot.
	    journal        *journal //即日志，它存的是账户变动的反向操作,这对以太坊实现快照功能以及回滚世界状态非常有用
	    validRevisions []revision //validRevisions是一个revision的切片，后者存的是日志（journal）的索引,通过revision的journalIndex可以索引到日志	切片的位置，回滚到某个revision就只要去查找当前journals切片中大于revision.journalIndex的那些日志，并执行，即可回滚到当前revision指定的世界状态。
	    nextRevisionId int
	}


账户信息包含账户地址,余额,交易计数,账户余额,账户存储树,账户合约代码
type stateObject struct {
	address  common.Address  // 账户 地址
	addrHash common.Hash // 账户地址的hash
	data     Account  // 账户数据,包括 账户的nonce交易计数值，每发起一笔交易确认后nonce值加1；
	                     账户余额；
	                     账户的storage存储树根；
	                     账户的code hash值
	db       *StateDB

	trie Trie // 存储树
	code Code // 合约代码
	
	dirtyCode bool // 标记stateObject.code被修改了
	suicided  bool // 标记上层调用了自杀命令
	deleted   bool // 标记账户已经从数据库中删除
}

--------------------------------------------------------------------------------------------------------------------------------------------
## levelDB

在做键值对插入时是不能保证顺序(即不按KEY排序)的，但是我们在查找时却希望是有序的，有顺序可以用二分进行查找而不是遍历。LevelDB最就是通过将插入时的无序演变为有序，最后让二分查找变为现实

采用了先写内存的方式再持久化来提高写效率,数据按key排序后写入内存表中,当一个memtable写满后会变成不可改变的只读表,后台再持久化到磁盘中，其对应的多个磁盘文件之间并不是有序的，这样造成在读操作时得对所有的磁盘文件进行遍历，严重影响了读效率。
所以，leveldb会定期聚合这次文件,将多个文件压缩成一个,这就形成新的一层文件存储,这样新的一层比之前一层但总文件数少,单个文件存储的多,并且新生成的多个文件是排好序的,每个文件标明了其存储范围
随着聚合的进行，磁盘文件在逻辑上被分成若干层。通过内存数据直接持久化出来的是level 0 层文件，后期整合出来的level i层文件。每一层文件总大小是下一层的10倍,因此可以存储很多数据
随着数据量的增加,层数不断增加,合并之后所有的旧文件都可以删掉，只留下新的
由于每个Level的磁盘数量有限,因此查询时间是常数级的

6个模块组成:
 MemTable(wTable)：内存数据结构，具体实现是 SkipList。 接受用户的读写请求，新的数据修改按用户定义的方法排序之后按序存储
 Immutable MemTable(rTable)：当 MemTable 的大小达到设定的阈值时，会变成 Immutable MemTable，只接受读操作，不再接受写操作，后续由后台线程 Flush 到磁盘上。
 SST Files：Sorted String Table Files，磁盘数据存储文件。分为 Level0 到 LevelN 多层，每一层包含多个 SST 文件，文件内数据有序。Level0 直接由 Immutable Memtable Flush 得到，其它每一层的数据由上一层进行 Compaction 得到。
 Manifest Files：Manifest 文件中记录 SST 文件在不同 Level 的分布，单个 SST 文件的最大、最小 key，以及其他一些 LevelDB 需要的元信息。由于 LevelDB 支持 snapshot，需要维护多版本，因此可能同时存在多个 Manifest 文件。
 Current File：由于 Manifest 文件可能存在多个，Current 记录的是当前的 Manifest 文件名。
 Log Files (WAL)：用于防止 MemTable 丢数据的日志文件。

LevelDB 的内存表中维护了 2 个跳跃列表，一个是只读的Immutable MemTable(rTable) ，一个是可修改的MemTable(wTable)
跳跃表实现了链表遍历的多条路径,解决依次遍历链表速度慢的问题,是用空间换时间

LevelDB中一次完整的数据插入过程:
1.数据插入LOG(防止突然的宕机，利于恢复现场)再插入内存MemTable
2.MemTable 的大小达到设定阈值的时候时内存MemTable会演变为内存Immutable MemTable，而内存Immutable MemTable的数据是不可修改，同时创建一个新的MemTable继续接API传来的数据。
3.接下来Immutable MemTable的数据将会由后台线程异步固化到SST中。SST的文件从level 0开始压缩到LevelTopN，每次Level升级其实也就是多个低Level的SST文件进行聚合操作。

而数据在SST中的文件内将会是有序的。这将便于查找。因此在LevelDB中，数据有可能存在MemTable--> Immutable MemTable --> SST之中，同时表明了数据流的流向过程。

读操作的流动方向和写操作类似：
 (1) 读 MemTable，如果存在，返回。
 (2) 读 Immutable MemTable，如果存在，返回。
 (3) 按顺序读 Level0 ~ Leveln，如果存在，返回。
 (4) 返回不存在。

LevelDB适合写多读少的场景，因为其采用的追加的形式，而且正常的DELETE需要先GET再DELETE，而LevelDB的DELETE只需要APPEND即可。
由于最新的数据在0层，最老的数据在 k 层，因此查询也是先查 0 层，如果没有要查的 key，再查 1层，依次逐层查。由于一次查询可能需要多次查找操作，因此读操作会稍微慢一些
在LevelDB中是没有数据删除的概念，最起码API是不可以主动删除数据，只能是追加一条KEY-DELETE记录，当查找时会发现KEY-DELETE了，视为KEY不存在。

ethdb:
使用第三方库github.com/syndtr/goleveldb/leveldb
封装之后的代码是支持多线程同时访问的，所以下面这些代码是不用使用锁来保护的
New()方法给定存储目录和配置参数,启动levelDB,并开启一个协程来收集levelDB的使用情况
每3秒钟获取一次leveldb内部的计数器，然后把他们公布到metrics子系统。 这是一个无限循环的方法, 直到quitChan收到了一个退出信号。
定义了一些数据库操作的接口

close()方法会使用通道通知收集器停止,并关闭levelDB



RPC
------------------------------------------------------------------------------------------------------------------------------

func (n *Node) openEndpoints() error
启动p2p节点
启动RPC服务:
1.注册用于IPC的API
2.启动IPC监听,用管道通信库npipe可以减去网卡的通信；直接基于内存的通信性能和内存的开销达到最低。net.Pipe创建一个内存中的同步、全双工网络连接。连接的两端都实现了Conn接口。一端的读取对应另一端的写入，直接将数据在两端之间作拷贝；没有内部缓冲。
通常用于不同区域的代码之间相互传递数据。不会占用本机端口
3.配置http服务
4.配置ws服务
5.启动http服务的TCP监听
6.启动ws服务TCP监听
websocket协议与服务器建立间接需要借助http协议，连接成功后使用websocket协议进行通信，客户端和服务器都可以主动向对方发送或接收数据。

	// 为httpServer定义了处理函数,在start()函数内监听http请求后会使用此函数处理serverHandler{c.server}.ServeHTTP(w, w.req)
	func (h *httpServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
		// check if ws request and serve if ws enabled
		ws := h.wsHandler.Load().(*rpcHandler)
		if ws != nil && isWebsocket(r) {
			if checkPath(r, h.wsConfig.prefix) {
				ws.ServeHTTP(w, r)
			}
			return
		}
		// if http-rpc is enabled, try to serve request
		rpc := h.httpHandler.Load().(*rpcHandler)
		if rpc != nil {
			// First try to route in the mux.
			// Requests to a path below root are handled by the mux,
			// which has all the handlers registered via Node.RegisterHandler.
			// These are made available when RPC is enabled.
			muxHandler, pattern := h.mux.Handler(r)
			if pattern != "" {
				muxHandler.ServeHTTP(w, r)
				return
			}
	     if checkPath(r, h.httpConfig.prefix) {
			rpc.ServeHTTP(w, r)
			return
		  }
		}
		w.WriteHeader(http.StatusNotFound)
	}


RPC客户端
func initClient(conn ServerCodec, idgen func() ID, services *serviceRegistry) *Client会创建然后是对象的初始化， 如果是HTTP连接的化，直接返回，否者就启动一个goroutine调用dispatch方法。 dispatch方法是整个client的指挥中心，通过上面提到的channel来和其他的goroutine来进行通信，获取信息，根据信息做出各种决策

对于http请求,使用call(),调用doRequest方法进行请求拿到回应。然后写入到resp队列	(op.resp <- &respmsg)。switch resp, err := op.wait(ctx);会读取响应信息
对于非http请求,使用send(),把op写入到requestOp这个队列，注意的是这个队列是没有缓冲区的,dispatch会收到请求消息

- 多线程串行发送请求到网络上的流程 首先发送requestOp请求到dispatch获取到锁， 然后把请求信息写入到网络，然后发送sendDone信息到dispatch解除锁。 通过requestOp和sendDone这两个channel以及dispatch代码的配合完成了串行的发送请求到网络上的功能。
- 读取返回信息然后返回给调用者的流程。 把请求信息发送到网络上之后， 内部的goroutine read会持续不断的从网络上读取信息。 read读取到返回信息之后，通过readResp队列发送给dispatch。 dispatch查找到对应的调用者，然后把返回信息写入调用者的resp队列中。完成返回信息的流程。
- 重连接流程。 重连接在外部调用者写入失败的情况下被外部调用者主动调用。 调用完成后发送新的连接给dispatch。 dispatch收到新的连接之后，会终止之前的连接，然后启动新的read goroutine来从新的连接上读取信息。
- 关闭流程。 调用者调用Close方法，Close方法会写入信息到close队列。 dispatch接收到close信息之后。 关闭didQuit队列，关闭连接，等待read goroutine停止。 所有等待在didQuit队列上面的客户端调用全部返回。



交易执行流程
--------------------------------------------------------------------------------------------------------------------------------------------

以太坊交易执行过程
miner从交易池中拿来的交易，交给worker对象。后者要调用commitTransaction在本地执行交易，生成收据receipt，更改世界状态，打包成挖矿的区块最后递交给engine进行挖矿

core.ApplyTransaction就是执行交易的入口

该函数的调用有两种情况：
1、是在将区块插入区块链前需要验证区块合法性
bc.insertChain ——> bc.processor.Process  ——> stateProcessor.Process ——> ApplyTransaction
2、是worker挖矿过程中执行交易时
Worker.commitTransaction ——> ApplyTransaction



1、将交易转换成可以调用执行函数的结构Message；
2、初始化一个EVM的执行环境；
3、执行交易
	1.先检查交易的nonce和当前状态nonce是否一致
	2.设置gas,gas池和发起者账户都要减去预支的gas
	3.发送方的nonce值+1,进行转账或合约调用或合约创建,
	4.将没用完的gas返回给发送者和gas池,将gas奖励给矿工
	5.将使用gas和evm执行错误返回
4.计算root改变stateDB世界状态，生成收据并返回

交易工作环境(StateTransition)的数据结构
type StateTransition struct {
	gp         *GasPool   // 区块工作环境中的gas剩余额度，就是header中的gasLimit,而header的gasLimit是通过父区块的gasUsed推算出来的
	msg        Message    // 交易转化的message
	gas        uint64     // 交易的gas余额，最开始等于initialGas，随着交易执行会递减,等于initialGas减去交易usedGas
	gasPrice   *big.Int   
	initialGas uint64     // 初始gas，等于交易的gasLimit
	value      *big.Int   // 交易转账额度
	data       []byte     // 交易的input，如果是合约创建，data就是合约代码
	state      vm.StateDB // 状态树
	evm        *vm.EVM    // evm对象
}


EVM 创建智能合约
evm.Create()
合约创建函数的调用时机，一是Worker执行交易的过程，交易如果是合约创建，则会在EVM执行交易时生成智能合约地址并部署智能合约；
                      二是通过opCreate指令，也即是智能合约内部创建新的智能合约。

创建一个合约所需要的参数：
caller：创建合约方地址；
合约地址（对发送者地址（caller.Address）和账户的nonce进行keccak256计算得到合约地址）； 
code：代码（input）  
gas：当前交易的剩余gas； 
value：转账额度；

流程
1、交易执行前的检查，如果evm递归深度大于1024，直接退出，判断交易发起方账户余额是否足够，否则直接退出
2、对当前StateDB进行快照，确保当前要创建的地址在世界状态中没有合约存在，如果存在，直接返回；给交易发送者的账户nonce加1

3、然后创建一个新账户，设置新账户为nonce为1；
4、进行转账，就是sender的账户减减（--），合约账户加加（++）。

5、创建一个待执行的合约对象填入caller地址 、合约地址、转账额和交易余额，合约代码，并执行run()函数将contract交给了evm解释器
	部署合约时使用contract.SetCodeOptionalHash将原始transaction中的Payload作为合约代码填入合约，transaction中的Payload就包含了程序自动添加的一段合约部署代码和合约函数引导代码，用来引导合约的部署和执行，因此包括三部分 合约部署代码 合约函数引导代码 和 合约代码
    调用合约时使用contract.SetCallCode将stateDB智能合约地址中储存的Code读取出来填入合约，因此并没有合约部署代码只有 合约函数引导代码 和 合约代码

6、处理返回值分别是ret（合约代码）和err，需要判断合约代码的长度以及err是否为nil，有错误则恢复之前的快照
7.计算本次合约创建消耗的gas，每字节200gas，如果交易gas余额足够，则成功部署合约，将合约代码设置到账户储存中
9.最后返回合约代码、合约地址、gas和错误

EVM 转账或调用合约
evm.Call()
参数
caller：转出方地址；
addr：转入方地址，如果是调用智能合约，那就是智能合约的地址；
input：调用函数的参数
gas：当前交易的剩余gas；
value：转账额度；

流程
1.交易执行前的检查，evm递归深度检查，发送方余额检查
2.生成快炒，判断世界状态中是否存在这个账号，如果不存在则创建账号
3.进行转账
4.从StateDB中查询该地址的合约代码，如果没有则完成转账，如果有则创建一个待执行的合约对象将读完出的合约代码填入，并执行
5.处理错误，如果有错误则使用快照恢复，最后返回合约代码，gas和错误

智能合约EVM的递归调用深度为1024，也就是指通过一个合约调用另一个合约，像这样的调用可以递归1024次。
为什么说是“递归”？因为从一个智能合约调用另一个智能合约，比如通过Call()方法，都要重新构建contract实例，然后执行run()。而run()的执行是通过EVMinterpreter.Run()进行的。而在EVMInterpreter结构体中又传入了*EVM的地址，然后执行了evm.depth++。所以实际上每一次调用都是在同一个EVM内进行的。

合约调用合约的方法有三种
1.call 如果外部账户A的某个操作通过Call方法调用B合约，而B合约又通过 Call方法 调用了C合约，那么最后实际上修改的是合约C账户的值；

2.CallCode 如果外部账户A的某个操作通过Call方法调用B合约，而B合约通过 CallCode方法 调用了C合约，那么B只是调用了C中的函数代码，而最终改变的还是合约B账户的值。

3.DelegateCall 其实跟CallCode方法的目的类似，都是只调用指定地址（合约C）的代码，而操作B的值。只不过它明确了CallerAddress是来自A，而不是B。所以这两种方法都可以用来实现动态库：即你调用我的函数和方法改动的都是你自己的数据）。

## EVM解释器

EVM解释器是面对Contract对象的，不论是Contract的创建还是调用，都会通过run()函数来调用Interpreter的Run()方法。该方法初始化执行过程中所需要的一些变量，然后进入堆栈操作的主循环