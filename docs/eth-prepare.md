# 以太坊兼容链性能测试1—准备工作

## 项目简介

项目地址：https://github.com/sunchengzhu/eth-prepare

相关文章：[以太坊兼容链性能测试2—性能数据统计](https://github.com/sunchengzhu/md/blob/main/docs/eth-performance.md)、[以太坊兼容链性能测试3—开展测试](https://github.com/sunchengzhu/md/blob/main/docs/eth-jmeter.md)

### 功能

1. 快速给同一助记词生成的大量账户充值，保证后续转账、调用合约函数等发送交易的测试场景需要发起账户有足够的资金
2. 部署包含简单函数的合约以及uniswap合约，获取合约地址以及调用函数的data

### 设计

先给从助记词第1个账户一个个转给第一批账户，如间隔500个，即给助记词第500、1000、1500...个账户转账。再在多进程中由这第一批账户转给其间隔段的账户，如第500个账户会转给第500～999个账户，这样就能完成所有账户的充值。

为什么要设置这个第一批账户呢？

如果所有转账都由第1个账户发起，那么在不同的进程中可能会取到同样的nonce值，导致交易失败。

为什么要给这么多账户充值？例子中配置了1w个账户

1. 真实场景一般涉及到较多账户状态的变更，不会是一两个账户不断执行交易，所以为了尽可能接近真实场景，我们在准备阶段给较多账户进行充值。
2. 避免节点处理同一账户不同nonce值的交易，例如我用了较少的账户数压测，某个账户的nonce为5，但节点先收到了nonce为6、7的交易就没办法处理，因为交易是有顺序的，必须等收到nonce为5的交易处理完成之后才能处理nonce为6、7的交易，所以这样测试并不能反应节点真正的性能。用较多账户就不会出现此问题，因为我循环用这1w个账户发交易，nonce为5的交易发完要隔很久才发nonce为6的交易，发nonce为6的交易的时候nonce为5的交易早就处理完了。

## 1.资金准备

### 使用步骤（以[fantom测试网](https://testnet.ftmscan.com)为例）

1. 安装依赖包

   ```bash
   npm install
   ```

2. 编译合约

   ```bash
   npx hardhat compile
   ```

3. 在项目根目录下新建.env文件，配置环境变量，可以参考如下配置：

   从助记词`MNEMONIC`生成的第一个账户分发FTM，给助记词生成的`ACCOUNTSNUM`个账户每个都充值`DEPOSITAMOUNT`个FTM，充值之后检查这`ACCOUNTSNUM`个账户是否都有至少`MINAMOUNT`个FTM。

   ```
   MNEMONIC='strike gather blush lens excite ridge flock random empty remember text universe'
   COUNT=500
   ACCOUNTSNUM=10000
   DEPOSITAMOUNT=0.01
   MINAMOUNT=0.001
   ```

4. 部署批量转账合约并[将合约地址加到配置中](https://github.com/sunchengzhu/eth-prepare/blob/895ba1bc5ee4bb824bf72ffdbca720baf1894d6e/test/distribute.js#L18-L20)

   如果给一个个账户转FTM的话非常低效，通过[合约](https://github.com/sunchengzhu/eth-prepare/blob/895ba1bc5ee4bb824bf72ffdbca720baf1894d6e/contracts/BatchTransfer.sol)批量转账好点，考虑到交易体大小有限制，一次交易[最多给50个账户转](https://github.com/sunchengzhu/eth-prepare/blob/895ba1bc5ee4bb824bf72ffdbca720baf1894d6e/test/distribute.js#L56)。

   ```bash
   # 这里最好把COUNT临时改成较小的值比如5（第5步记得改回来），不然hardhat生成COUNT数的sigenrs这步会比较慢，影响case执行速度。
   npx hardhat test --grep "deploy BatchTransfer" --network fantom_testnet
   ```

5. 给第一批账户转入FTM

   从助记词第一个账户给间隔`COUNT`的账户转入`DEPOSITAMOUNT`×`COUNT`×1.1个FTM。之所以是`DEPOSITAMOUNT`×`COUNT`×1.1个FTM而不是`DEPOSITAMOUNT`×`COUNT`个FTM，是为了让第一批账户有钱支付手续费。

   ```bash
   npx hardhat test --grep "recharge" --network fantom_testnet
   ```

6. 多进程分发FTM（默认4个进程）

   从第一批账户通过批量转账合约给所有账户充值

   ```bash
   bash run.sh deposit fantom_testnet
   ```

7. 多进程检查账户余额

   ```bash
   bash run.sh afterDeposit fantom_testnet
   ```

## 2.合约数据准备

*这里最好把COUNT改成较小的值比如5，不然hardhat生成COUNT数的sigenrs这步会比较慢，影响case执行速度。* 

### eth_call

以往我在测试联盟链的时候会用和[这个](https://github.com/sunchengzhu/eth-prepare/blob/895ba1bc5ee4bb824bf72ffdbca720baf1894d6e/contracts/SimpleStorage.sol)类似的合约进行压测，因为联盟链比较多的是存证场景，在链上存东西，这里我主要是部署一个有简单函数的合约，后续调用`eth_call`接口压测。

1. 部署合约

   ```bash
   npx hardhat test --grep "deploy SimpleStorage" --network fantom_testnet
   ```

2. 保存合约地址和`data`，后面压测要用

   ```
   simpleStorage address: 0x6Fc1E7631D24b6173d975a892ba5563E8C04CB9f
   tx data: 0x20965255
   ```

3. 用`etc_call`调用一下合约的`getValue`函数

   ```bash
   curl https://rpc.testnet.fantom.network \
     -X POST \
     -H "Content-Type: application/json" \
     -d '{"jsonrpc":"2.0","method":"eth_call","params":[{"from":"0x045346bE7C2915F96c9261BEE90d3FF9005a6c2b","to":"0x6Fc1E7631D24b6173d975a892ba5563E8C04CB9f","data":"0x20965255"}, "latest"],"id":1}'
   ```

### uniswap

看[etherscan gastracker](https://etherscan.io/gastracker)上的排名，现在在公链上最多的场景还是token转换。所以这里我们也部署uniswap V2合约进行压测，我们选取`swapExactETHForTokens`函数进行压测，用账户的资金（1000wei）换token，这样比较简单，因为我们对每个账户已经进行了充值都有资金，只需要一开始给这个token增加流动性即可。如果要压测`swapExactTokensForETH`函数的话，每个账户都需要充值该token以及执行`approve`授权uniswap路由合约操作token。

1. 部署合约并执行一次`swapExactTokensForETH`获取`data`

   ```bash
   npx hardhat test --grep "deployAndSwap" --network fantom_testnet
   ```

2. 保存uniswap路由合约地址和data，后面压测要用

   ```
   uniAddress: '0x7C7087d81c5f4Bd7EA30A5e13095414395DfD4F1'
   data: 0x7ff36ab50000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000008000000000000000000000000079026e949ba3ef5c854186244d1597a369bc326d00000000000000000000000000000000000000000000000000005af3107a3fff0000000000000000000000000000000000000000000000000000000000000002000000000000000000000000a6465996d9b1c6e82a65d4503d07ee1f68ed3a34000000000000000000000000a37614c751f37cbc54c5223254e8695024fa36c7
   ```

   

### 已核验合约

[simpleStorage](https://testnet.ftmscan.com/address/0x6fc1e7631d24b6173d975a892ba5563e8c04cb9f#code)

[UniswapV2Router02](https://testnet.ftmscan.com/address/0x7C7087d81c5f4Bd7EA30A5e13095414395DfD4F1#code)

[ERC20](https://testnet.ftmscan.com/token/0xA37614c751F37cBc54C5223254e8695024fA36c7#code)

[WETH9](https://testnet.ftmscan.com/address/0xa6465996d9b1c6e82a65d4503d07ee1f68ed3a34#code)
