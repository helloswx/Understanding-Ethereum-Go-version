# Understanding-Ethereum-Go-version

- Title: Understanding Ethereum(Go version)｜理解以太坊(Go 版本源码剖析)
- Subject: Ethereum Source Code Analysis | 以太坊源码分析
- Author: Siyuan Han
- Go-Ethereum Version: v1.10.9
- Updated date: 2021-07

## Preface

### Background

从中本聪发表比特币白皮书至今，已经过了十几年。Blockchain依然当下技术社区最热点话题之一。Blockchain系统从专注于加密货币到如今的逐渐通用化，集成了包括数据库系统，分布式系统，点对点网络，编译原理，众包，密码学，经济学，货币金融学在内的等多个领域的知识。深入理解Blockchain技术最通常的方式是从State-of-the-arts的几个Blockchain Systems开始研究。不管是探究以加密货币导向（Crypto-based）的Bitcoin, 还是致力于实现通用框架（General-Purpose）的Ethereum的时候，文档往往是从high-level的层面来讲述Blockchain的基础概念，或者这些系统设计的思想。目前，技术社区有非常多的文档来讲述Blockchain System背后的数据结构和算法, 比如数据结构的设计实现: 梅克尔树 (Merkle Hash Tree)，帕特里夏树 (Patricia Tree)，DAG (Directed acyclic Graph); 共识算法的背后原理: BFT (Byzantine Fault Tolerance)， PoW (Proof-Of-Work); 以及类似双花 (Double-Spending)，DAO Attack (Decentralized autonomous organization) 等具体问题。

但是，了解各个组件的实现细节并不能保证从整体上理解系统的工作原理。某天，我忽然想到，究竟miner是怎么从网络中获取到transaction, 又是怎样从transaction pool中选取transaction，以及按照怎么的order把transaction打包进区块链中的呢？我尝试去搜索了一下，发现鲜有文章从系统工作流 (Workflow)的角度出发，对区块链系统中的具体的实现细节进行解析。与数据库系统(Database Management System)相似，Blockchain系统 同样是一个包含网络等，业务逻辑层，存储层的复杂数据管理系统。对它研究同样需要从系统的实现细节出发，从宏观到围观的了解每个执行逻辑的工作流，才能彻底理解和掌握这门技术的秘密。

笔者坚信，随着网络基础架构的不断完善，将带来的显著的带宽上升和通信延迟下降，同时存储以及计算技术的不断发展，将会让系统的软件的运行效率不断逼近硬件极限。在未来的是五到十年内，云端服务/去中心化系统的效率以及覆盖场景一定还会有很大的提升。未来技术世界一定是两极分化的。一极是以大云计算公司（i.e, Google，MS，Oracle，Snowflake，and Alibaba）为代表的中心化服务商。另一极就是以Blockchain技术作为核心的去中心化的世界。在这个世界中，Ethereum及其生态系统是当之无愧的领头羊。Ethereum 不光在Public Chain的层面取得了巨大的成功，而且Go-Ethereum作为其优秀的开源实现，已经被广泛的订制，来适应不同的私有/联盟场景(e.g., Quorum, Binance Smart Chain)。因此，要想真正掌握好区块链系统的原理，达到可以设计开发区块链系统的水平，研究好Ethereum的原理以及其设计思想是非常有必要。

本系列文章，作为我在博士期间学习/研究的记录，将会从Blockchain中业务的Workflow的视角出发，在源码的层面，来深度解析以太坊系统中各个模块的实现的细节，以及背后的蕴含的技术和设计思想。同时，在阅读源代码中发现的问题也可以及时提交Pr来贡献社区。Go-ethereum是以太坊协议的Go语言实现版本，目前由以太坊基金会维护。目前除了Go-ethereum之外，Ethereum还有C++, Python，Java, Rust等基于其他语言实现的版本。但相比于其他的社区版实现，go-ethereum的使用人数最多，开发人员最多，版本更新最频繁，issues的发现和处理都较快。运行也更更加的稳定。其他语言的Ethereum实现版本因为用户与开发人员的数量相对较少，更新频率相对较低，隐藏问题出现的可能性更高。因此我们选择从go-ethereum的代码出发，来理解Ethereum系统与网络的设计实现。

### 为什么要阅读区块链系统的源代码

1. 文档资料相对较少，且**内容浅尝辄止**。比如，*很多的科普文章都提到，在打包新的Block的时候，miner负责把a batch of transactions从transaction pool中打包到新的block中*。那么：
    - Miner会从Transaction Pool中选择哪些Transaction呢？
    - 被选择的Transaction又是以怎样的顺序(Order)被打包到区块中的呢？
    - 在执行Transaction的EVM是怎么计算gas used，从而限定Block中Transaction的数量的呢?
    - 剩余的gas又是怎么返还给Transaction Proposer的呢？
    - 在执行Transaction中是哪个模块，又是怎样去修改Contract中的持久化变量呢？
    - Contract中的持久化变量又是以什么样的形式存储的呢？
    - 当新的Block加入到Blockchain中时，World State又是何时怎样更新的呢？

2. 目前的Blockchain系统并没有像数据库系统(DBMS)那样统一实现的方法论，每个不同的系统中都集成了大量的细节。从源码的角度出发可以了解到很多容易被忽视的细节。简单的说，一个完整的区块链系统至少包含以下的模块: 
    - 密码学模块: 加解密，签名，安全hash，mining
    - 网络模块: P2P节点通信
    - 分布式共识模块: PoW, BFT
    - 智能合约解释器模块: Solidity编译语言，EVM解释器
    - 数据存储模块: 数据库，数据存储，index
    - Log日志模块

### Blockchain System (BCS) VS Database Management System (DBMS)

Blockchain 系统在设计层面借鉴了很多数据库系统中的设计逻辑。

- Blockchain系统同样也从Transaction作为基本的操作载核，包含一个Parser模块，Transaction Executor模块，和一个Storage 管理模块。

## Contents

### PART ONE - General Source Code Analysis: Basic Components

- [00_万物的起点从geth出发: Geth框架导引](CN/00_geth.md)
- [01_State-based 模型 & Account](CN/01_account.md)
- [02_Transaction是怎么被打包的: 一个Transaction的生老病死](CN/02_transaction.md)
- [03_从Block到Blockchain: 区块链数据结构的构建](CN/03_block_blockchain.md)
- [04_一个新节点是怎么加入网络并同步区块的](CN/04_p2p_net_node_sync.md)
- [05_一个网吧老板是怎么用闲置的电脑进行挖矿的](CN/05_mining_hash_gpu.md)

### PART TWO - General Source Code Analysis: Services

- [10_构建StateDB的实例](CN/10_tire_statedb.md)
- [11_Blockchain的数据是如何持久化的](CN/11_leveldb_in_practice.md)
- [12_Signer一个签名者的实现](CN/12_signer.md)
- [13_如何实现节点的RPC调用](CN/13_rpc.md)
- [14_如何实现节点的IPC调用](CN/14_ipc.md)

### PART THREE - Advanced Topics

- [20_结合BFT Consensus 解决拜占庭将军问题](CN/20_bft_consensus.md)
- [21_Plasma与 Zk Rollup](CN/21_zkp_Plasma.md)
- [22_ADS](CN/22_ads_merkel.md)
- [23_Bloom Filter](CN/23_bloom_filter.md)
- [24_图灵机和停机问题](CN/24_turing_halting.md)
- [25_Log-structured merge-tree in Ethereum](CN/25_lsm_tree.md)
- [26_Ethereum Transaction Concurrency](CN/26_txn_concurrency.md)

### PART FOUR - Ethereum in Practice

- [30_使用geth构建一个私有网络](CN/30_geth_private_network.md)
- [31_如何编写Solidity语言](CN/31_solidity_in_practice.md)
- [32_使用预言机(Oracle)构建随机化的DApp](CN/32_oracle.md)
- [33_Query On Ethereum Data](CN/33_query.md)

### PART FIVE - APPENDIX

- [40_FQA](#tips)
- [41_Ethereum System Tunning](CN/41_system_tunning.md)
- [42_go-ethereum的开发思想](CN/42_developer_view.md)
- [43_Metrics in Ethereum](CN/43_metrics.md)
- [44_Golang with Ethereum](CN/44_golang_ethereum.md)

-----------------------------------------------------------

## Conclusion

如何衡量对一个系统的理解程度

1. 掌握（Mastering）
    - 可以编写一个新的系统
2. 完全理解（Complete Understanding）
    - 完全理解系统的各项实现的细节，并能做出优化
    - 可以对现有的系统定制化到不同的应用场景
3. 理解（Understanding）
    - 熟练使用系统提供的API
    - 能对系统的部分模块进行重构
4. 简单了解（Brief understanding）
    - 了解系统设计的目标，了解系统的应用场景
    - 可以使用系统的部分的API

## Tips
<a name="tips"></a>

- 以太坊是基于State模型的区块链系统，miner在update new Block的时候，会直接修改自身的状态（添加区块奖励给自己）。所以与Bitcoin不同的是，Ethereum的区块中，并没有类似的Coinbase的transaction。
- 在core/transaction.go 中, transaction的的数据结构是有time.Time的参数的。但是在下面的newTransaction的function中只是使用Local的time.now()对Transaction.time进行初始化。
- 在core/transaction.go 的transaction 数据结构定义的时候, 在transaction.time 后面的注释写到（Time first seen locally (spam avoidance)）, Time 只是用于在本地首次看到的时间。
- uncle block中的transaction 不会被包括到主链上。
- go-ethereum有专用函数来控制每次transaction执行完，返还给用户的Gas的量。有根据EIP-3529，每次最多返还50%的gas.

## Reference

- [1] Ethereum Yellow Paper [(Paper Link)](https://ethereum.github.io/yellowpaper/paper.pdf)
- [2] Ethereum/Go-Ethereum [(link)](https://github.com/ethereum/go-ethereum)
- [3] Go-ethereum code analysis [(Link)](https://github.com/ZtesoftCS/go-ethereum-code-analysis) 
- [4] Ethereum Improvement Proposals [(link)](https://github.com/ethereum/EIPs)
- [5] Mastering Bitcoin(Second Edition)
- [6] Mastering Ethereum [(link)](https://github.com/ethereumbook/ethereumbook)

## Talks

- Succinct Proofs in Ethereum - Barry Whitehat, Ethereum Foundation [(Youtube)](https://www.youtube.com/watch?v=TtsDNneTDDY)
