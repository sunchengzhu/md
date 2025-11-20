# 新版ckb浏览器测试分析

## Block

### 能通过rpc接口查到的字段

直接用ckb-cli rpc get_block_by_number --number Block number即可对比字段的正确性即可，例如
`ckb-cli rpc get_block_by_number --number 19179003 --output-format json | jq '.header.number'`，下面括号里面的是jq后面的参数。

#### get_block_by_number

- Block Height (.header.number)
  - 点击左边箭头，跳转到第N-1个区块
  - 点击由边箭头，跳转到第N+1个区块
- Transactions (.transactions | length)
- Cycles (.cycles | add)
  - 需要用到--with-cycles参数
  - 只有Cellbase交易的话，Cycles是空
- Proposal Transactions (.proposals | length)
- Miner Reward (.transactions[0].outputs[0].capacity)
  - 需要加11个区块号查询，参考：https://docs.nervos.org/docs/mining/rewards
  - N+11个区块没挖出来是pending状态
  - N+11个区块挖出来之后可以点击跳转
- Nonce (.header.nonce)
- Uncle Count (.uncles | length)
  - 例如19188553
- Miner (.transactions[0].outputs[0].lock)
  - 需要加11个区块号查询


- Miner Message (.transactions[0].witnesses)
- Epoch (.header.epoch)
- Epoch Start Number (.header.epoch)
  - block height - index
- Block Index (.header.epoch)
- Timestamp (.header.timestamp)

#### get_blockchain_info

- Difficulty (.difficulty)
  - 只能查到实时的

### 能通过ckb日志查到的字段

- Size

### Block里面的Transactions

- Transactions不重复不遗漏

### 创世区块

- Miner Reward是0
- Cycles是null
- _Miner应该是空吧_



## Transaction

### Cellbase

#### 能通过rpc接口查到的字段

`ckb-cli rpc get_transaction --hash`

- Block Height (.tx_status.block_number)

  点击可跳转

- Transaction Fee | Fee Rate是0
- Timestamp用的是block的timestamp
- Status = 最新区块 - Block Height
- Cycles是空

##### Reward Info

Cellbase交易是分配给打包第N-11个区块的矿工的奖励，所以`ckb-cli rpc get_block_economic_state`要查询的都是第N-11个区块。

- Cellbase for Block

  看能不能跳转到第N-11个区块

- Base Reward (.miner_reward.primary)

- Secondary Reward (.miner_reward.secondary)
- Commit Reward (.miner_reward.committed)
- Proposal Reward (.miner_reward.proposal)
- primary + secondary + committed + proposal = .transactions[0].outputs[0].capacity
- 第0个区块 (创世块)
  - 老的浏览器是把第一个交易当成了cellbase交易并做了特殊处理
- 前1-11个区块
  - cellbase应该是空

### rpc查不到的字段

- Size
  - 考虑用rust-sdk统计

### Parameters和Raw

- CellDep、HeaderDeps、Witnesses不重复不遗漏

### 转账类型

- 普通地址（也就是SECP256K1/blake160）转账ckb
  - Detail是CKB Capacity
  - Output的Cell可以看出是否是Unspent
- joyid转账CKB
- 多签地址转账ckb
  - Detail能体现出是多签
  - 覆盖新旧两种多签地址
  - 带since值的多签地址，参考：https://testnet.explorer.nervos.org/transaction/0x9a3fc80ef00fae32a6cc75432c55b49ee5d1a9ccaf49f4d7aec278e732ac0058
  - since能看到具体是哪个Epoch解锁
- 转账xudt
  - Capacity/Amount的单位是xudt的Name
  - Detail名字是xudt的Name，点击能跳转到xudt的详情页面
- 部署合约
  - output的Cell Detail能体现出是Deployment，参考：https://testnet.explorer.nervos.org/en/transaction/0x592217b4ad54a812780bc4a23e7d5c742c968b512f8d951c7ce072e35fb6becc

- Nervos DAO Deposit
- Nervos DAO Withdraw
- Nervos DAO Withdraw带since
- 转账DOB，这块不太熟，后面再具体想，参考：https://explorer.nervos.org/transaction/0x45f5da199710171ed479aebc6c26df223273555a7a60facb275048922db23d14

## Cell



