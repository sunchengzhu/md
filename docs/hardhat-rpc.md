> 通过分析hardhat在部署合约、调用合约函数过程中用到的rpc方法、方法参数、方法返回值的作用和意义，了解部署合约、调用合约函数的整个详细流程。



通过hardhat部署合约、调用合约函数的简单实现：[learnHardhat](https://github.com/sunchengzhu/learnHardhat)

## hardhat如何与以太坊网络交互？

hardhat项目中使用hardhat-ethers插件将一个ethers对象添加到hardhat运行时的环境，该对象具有ethers.js相同的API，通过API可以构造参数向以太坊客户端发送JSON-RPC请求，实现与以太坊网络的交互。

## 0. 准备工作

### 打印JSON-RPC日志

hardhat包本身没有打印JSON-RPC日志的功能，所以我们需要给hardhat包打补丁，实现JSON-RPC日志的打印。

- 打印请求、返回、签名前的交易体

详见：[补丁代码](https://github.com/sunchengzhu/learnHardhat/blob/main/patches/hardhat%2B2.12.2.patch)、[npm包打补丁教程](https://juejin.cn/post/6962554654643191815)

- 设置hardhat记录每个请求

hardhat.config.js中设置`loggingEnabled: true`

### 编译合约

编译合约得到artifacts/contracts/Learn.sol/Learn.json，其中包含合约字节码`bytecode` 和`abi` json。合约字节码是solidity合约被编译成的可被EVM执行的16进制字符。ABI是合约接口的说明，定义与合约进行交互数据编码规则。

## 1. 部署合约

部署合约就是在链上存储编译得到的合约字节码，关联合约地址供我们调用。合约地址根据创建者的地址和nonce值计算，所以同一个创建者每次部署都会返回新的地址。

### 合约代码

[Learn.sol](https://github.com/sunchengzhu/learnHardhat/blob/main/contracts/Learn.sol)

### hardhat代码

```jsx
//根据合约名称，创建合约工厂类实例
const Learn = await ethers.getContractFactory("Learn");
//部署合约
const contract = await Learn.deploy();
//获取回执
await contract.deployed();
```

### 相应的rpc日志

排除掉eth_chainId的rpc日志：[deploy.log](https://zhu-1304641378.cos.ap-shanghai.myqcloud.com/log/deploy.log) &nbsp;&nbsp;&nbsp;&nbsp; 原始的rpc日志：[original-deploy.log](https://zhu-1304641378.cos.ap-shanghai.myqcloud.com/log/original-deploy.log)

ethers.js处于安全考虑，[每次使用provider都会调用eth_chainId](https://github.com/ethers-io/ethers.js/issues/901)，而ethers.js访问区块链数据的API都需要通过provider，如getBlockNumber、getGasPrice，所以在rpc日志中会看到大量的eth_chainId调用，下面给出的rpc日志排除掉了eth_chainId接口，方便我们理解。

### 日志解读

1. eth_blockNumber

   发送交易前查一次区块高度，作为startBlock。如果需要在6个区块后确认合约部署，则使用`await contract.deployTransaction.wait(6)` 替换掉`await contract.deployed()` ，最新区块和startBlock相减就能知道是否达到6个区块。

   *相关源码：node_modules/@ethersproject/providers/src.ts/base-provider.ts*

2. eth_estimateGas

   预估交易的gas，参数中from为部署合约的外部账户（创建者），默认取hardhat.config.js中配置的accounts中的第一个，data为合约字节码。

3. eth_getBlockByNumber

   根据结果中有无baseFeePerGas判断是否支持eip1559，如果存在baseFeePerGas，则会调用eth_feeHistory，用于计算出交易体所需的maxFeePerGas和maxPriorityFeePerGas。

   hardhat调用eth_feeHistory的逻辑存在[问题](https://github.com/NomicFoundation/hardhat/issues/3395)，所以暂时还看不到eth_feeHistory的日志。eth_feeHistory调用失败之后会使用非eip1559节点的逻辑，即构造交易体不使用maxFeePerGas和maxPriorityFeePerGas参数。<u>[问题](https://github.com/NomicFoundation/hardhat/issues/3395)已被修复，所以支持eip1559的节点，如hardhat本地节点、goerli会使用eth_feeHistory，而不是eth_gasPrice。</u>

   *相关源码：node_modules/hardhat/src/internal/core/providers/gas-providers.ts*

4. eth_gasPrice

   查询当前gas的价格

5. eth_getTransactionCount

   获取创建者的交易次数，用于提供交易体中的nonce值，参数用了pending，统计了该账户之前执行与正在执行的交易次数，避免nonce值冲突。例子中结果是`0x1f`(31)，现在要构造第32笔交易，因为nonce值从0开始，所以nonce值应取31。

6. eth_sendRawTransaction

   发送创建者签名后的交易，交易体中的gas、gasPrice、nonce分别是eth_estimateGas、eth_gasPrice、eth_getTransactionCount的返回结果，data为合约字节码。

7. eth_getTransactionByHash

   根据eth_sendRawTransactio返回的交易hash查询交易在链上的执行情况，如果返回结果为null（比如交易还没进入交易池）则会一直查，直到查到为止，查到之后就开始查回执。返回结果中的v r s为交易的签名数据。

8. eth_getTransactionReceipt

   查询交易回执，交易执行完成之后可以查到交易的回执，如果查不到的话（比如例子中的情况）会调用eth_blockNumber看有没有出新块，如果出了新块则再次查回执，查到回执的话，轮询结束，如果没有出新块或没有查到回执则继续调用eth_blockNumber。返回结果中的status表示交易是否成功，`0x1`为成功，`0x0`为失败。

## 2.1 调用合约函数-pure函数

### hardhat代码

```jsx
//attach从已经部署的合约和现有实例（重用相同的ABI和Signer）创建一个新的合约实例
learn = await Learn.attach(learnAddress);
//add是pure函数
const result = await learn.add(1, 2);
```

### 相应的rpc日志

```jsx
jsonRpcRequest: {
  jsonrpc: '2.0',
  method: 'eth_call',
  params: [
    {
      from: '0x7752dcd7c6ce4aed048c028021d635cbec6c001d',
      to: '0xb03e3f89dde1bcb25991a12dab94389e128606d5',
      data: '0xbb4e3f4d00000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000002'
    },
    'latest'
  ],
  id: 4
}
jsonRpcResponse: {
  jsonrpc: '2.0',
  id: 4,
  result: '0x0000000000000000000000000000000000000000000000000000000000000003'
}
```

### 日志解读

因为pure函数不会引起合约状态的变更，所以只需要通过eth_call调用而不需要发送交易。

1. eth_call

   data中包含函数选择器和函数的参数，前4个字节bb4e3f4d是函数选择器，指定了要调用的函数，后面000…01和000…2是函数的两个参数。

   **函数选择器**：函数签名中只包含函数名和参数类型，没有参数名和空格。以`add(uint8 a, uint8 b)`为例，其函数签名是`add(uint8, uint8)`。函数选择器是函数签名（Function Signature）进行Keccak-256(sha3)运算后，左起的前四个字节，即`bytes4(sha3(“add(uint8,uint8)”)) = 0xbb4e3f4d`。

   **函数参数**：函数参数的编解码需要结合ABI描述信息的内容，根据ABI描述信息中接口的类型列表对参数进行编码。`enc(uint8(1),uint8(2)) = "00000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000002"`

   可以通过合约中的getCallData函数直接得到调用add函数的data

    ```solidity
    function getCallData(uint8 a, uint8 b) public pure returns (bytes memory) {
        return abi.encodeWithSelector(this.add.selector, a, b);
    }
    ```


## 2.2 调用合约函数-发送交易

### hardhat代码

```jsx
const randomNum = Math.floor(Math.random() * 1000000);
//修改合约里的一个状态变量
const tx = await learn.setValue(randomNum);
const receipt = await tx.wait();
//查询该状态变量
const value = await learn.getValue();
//该状态变量成功被修改
expect(value).to.be.equal(randomNum);
```

### 相应的rpc日志

[setValue.log](https://zhu-1304641378.cos.ap-shanghai.myqcloud.com/log/setValue.log)

### 日志解读

发送交易的过程与部署流程类似，交易体中的data与调用pure函数中的data类似，这边不再赘述。

## 2.3 调用合约函数-指定eth_call

### hardhat代码

```jsx
const randomNum = Math.floor(Math.random() * 1000000);
//通过callStatic指定使用eth_call调用
const result = await learn.callStatic.setValue(randomNum);
const value = await learn.getValue();
//该状态变量未被修改
expect(value).to.be.not.equal(randomNum);
```

### 相应的rpc日志

```jsx
jsonRpcRequest: {
  jsonrpc: '2.0',
  method: 'eth_call',
  params: [
    {
      from: '0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266',
      to: '0xcf7ed3acca5a467e9e704c703e8d87f634fb0fc9',
      data: '0x55241077000000000000000000000000000000000000000000000000000000000003778e'
    },
    'latest'
  ],
  id: 4
}
jsonRpcResponse: {
  jsonrpc: '2.0',
  id: 4,
  result: '0x000000000000000000000000000000000000000000000000000000000003778e'
}
jsonRpcRequest: { jsonrpc: '2.0', method: 'eth_chainId', params: [], id: 5 }
jsonRpcResponse: { jsonrpc: '2.0', id: 5, result: '0x7a69' }
jsonRpcRequest: {
  jsonrpc: '2.0',
  method: 'eth_call',
  params: [
    {
      from: '0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266',
      to: '0xcf7ed3acca5a467e9e704c703e8d87f634fb0fc9',
      data: '0x20965255'
    },
    'latest'
  ],
  id: 6
}
jsonRpcResponse: {
  jsonrpc: '2.0',
  id: 6,
  result: '0x000000000000000000000000000000000000000000000000000000000000ffbc'
}
```

### 日志解读

callStatic是个只读操作，它模拟完成事务中会发生什么，但在完成时丢弃所有状态更改。例子中调用的函数即便需要修改状态变量，但通过eth_call调用的话，最后状态变量还是未被修改。
