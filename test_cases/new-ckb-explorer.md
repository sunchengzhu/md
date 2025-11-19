# 新版ckb浏览器测试分析

## Block

### 通过rpc接口查到的字段

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

### 通过ckb日志查到的字段

- Size

