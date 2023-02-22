# ERC-4337账户抽象实践

> 目前在网上搜索ERC-4337基本只能搜到科普介绍性质的文章，缺乏动手实践指导的文章，本文希望能补上这部分的空白。

很多同学在学习ERC-4337的时候会去了解[eip-4337中提到](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4337.md?plain=1#L913-L915)的[account-abstraction项目](https://github.com/eth-infinitism/account-abstraction)，但其中给的单测例子用的是[simulateValidation](https://discord.com/channels/892780451570270219/892780453940056066/1045219792174465104)，并没有发送真正的交易，所以我参考[erc-4337-examples](https://github.com/stackup-wallet/erc-4337-examples)实现了抽象账户（AA）简单的几种交易。

已在goerli测试网上实践，项目地址：https://github.com/sunchengzhu/erc-4337-examples

## 前置步骤（已执行）

### 合约部署

首先我们需要部署account-abstraction中所需的合约，EntryPoint合约不需要我们自己部署，从Infinitism的discord上可以找到[最新的EntryPoint地址](https://discord.com/channels/892780451570270219/892780453940056066/1061786895161491456)，其他合约需要我们自己部署。为了让合约可被核验，需要利用[multisol](https://github.com/paulrberg/multisol)把主合约及其依赖合约放到同级目录下，命令如下：

```bash
#以VerifyingPaymaster.sol为例
cd account-abstraction
multisol contracts/samples/VerifyingPaymaster.sol
#执行后得到multisol-verifyingpaymaster目录
```

然后利用remixd让remix访问本地的multisol-verifyingpaymaster目录，在remix上连接metamask传入EntryPoint的合约地址和verifyingSigner的账户地址部署[VerifyingPaymaster](https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/samples/VerifyingPaymaster.sol)。

```bash
remixd -s multisol-verifyingpaymaster --remix-ide https://remix.ethereum.org
```

VerifyingPaymaster代付gas费的交易必须由verifyingSigner签名原始UserOperation，否则[验签不通过](https://github.com/eth-infinitism/account-abstraction/blob/f3b5f795515ad8a7a7bf447575d6554854b820da/contracts/samples/VerifyingPaymaster.sol#L90-L93)。所以我们要不掌握这个verifyingSigner直接签名原始的UserOperation，要不有个paymaster服务可以让verifyingSigner签名我们原始的UserOperation，我的方案为了简单[选了前者](https://github.com/sunchengzhu/erc-4337-examples/blob/54d50747ad4bde887652160be5f479564210e2d5/src/getMyPaymaster.ts#L53)，stackup收费的[paymaster api](https://docs.stackup.sh/docs/api/paymaster/introduction#stackup-paymaster-api)则是后者。

### 合约核验

Compiler Type选择`Solidity (Multi-Part files)`，上传multisol-verifyingpaymaster目录下所有的sol文件核验即可。

### 质押、充值eth

想要让paymaster可用还需要为其质押（addStake）、充值（depositTo）eth，参考[verifying_paymaster.test.ts](https://github.com/eth-infinitism/account-abstraction/blob/develop/test/verifying_paymaster.test.ts)，因为我们的合约已经被核验过了，所以可以直接在etherscan上连接metamask调用这两个函数，可以通过[addStake交易日志](https://goerli.etherscan.io/tx/0x2a24872030e552acd97f6873e73ad22181edb889e4c86083c63cad362c442e6d#eventlog)和[depositTo交易日志](https://goerli.etherscan.io/tx/0x35cad8d13b2edcc12a7a907e5532a3ec89e6b66710f9eb4ba104ac7ef5522d1e#eventlog)了解调用详情，可以通过向[EntryPoint的read方法](https://goerli.etherscan.io/address/0x0F46c65C17AA6b4102046935F33301f0510B163A#readContract)deposits和getDepositInfo传入paymaster的地址查询质押和充值的余额。

### 合约地址

其他合约也跟VerifyingPaymaster.sol一样部署，所需的合约如下：

[EntryPoint](https://goerli.etherscan.io/address/0x0F46c65C17AA6b4102046935F33301f0510B163A)

[VerifyingPaymaster](https://goerli.etherscan.io/address/0xE0165B20422B0dC3802085D34013bA0E2a83f640)

[TestToken](https://goerli.etherscan.io/token/0x61a89342f52d9f31626b56b64a83579e5c368f4c)（erc20合约，任何账户都可以通过它mint任意token）
[SimpleAccountFactory](https://goerli.etherscan.io/address/0xd9743aBf3031BD1B0b9B64a53307468677b4051B)（SimpleAccount的工厂合约）

[sender](https://goerli.etherscan.io/address/0x4Ed6e8753EE82D10952f4D720b30E8d2BCA09565)（向[SimpleAccountFactory中的createAccount](https://goerli.etherscan.io/address/0xd9743aBf3031BD1B0b9B64a53307468677b4051B#writeContract)传入[signingKey](https://github.com/sunchengzhu/erc-4337-examples/blob/54d50747ad4bde887652160be5f479564210e2d5/config.json#L4)对应的账户地址创建出来的一个SimpleAccount实例，即抽象账户，为UserOperation中的sender)

## Bundler搭建

我们需要一个bundler将UserOperations捆绑并创建一个EntryPoint.handleOps() 交易，可以使用stackup的[免费bundler实例](https://docs.stackup.sh/docs/guides/quickstart#3-initialize-your-configuration)，也可以在[本地搭建bundler](https://github.com/stackup-wallet/stackup-bundler)，这边更推荐在本地搭建bundler，方便查看日志。

**需要注意的是**ERC4337_BUNDLER_PRIVATE_KEY对应的bundler账户[必须持有足够支付gas fee的eth](https://discord.com/channels/874596133148696576/942772249662996520/1049685662305091584)，ERC4337_BUNDLER_ETH_CLIENT_URL对应的节点[必须支持`debug_traceCall`](https://github.com/eth-infinitism/bundler/blob/6b23f7d7cf92eef97f715a23ea30a5ba8773dab5/README.md?plain=1#L12)。
alchemy、infura、quicknode等[主流的节点提供商都不支持`debug_traceCall`](https://discord.com/channels/874596133148696576/942772249662996520/1066236623949418657)，而geth则需要full node才可以使用`debug_traceCall`，[snap和light均不支持](https://miaoguoge.xyz/geth-snap-rpc/)，这对本地机器资源要求较高。 所以我在[chainlist](https://chainlist.org/chain/5)上试了几个goerli的rpc server，下面这个看起来是可用的，测试命令如下：

```bash
curl https://goerli.blockpi.network/v1/rpc/public  \
-X POST \
-H "Content-Type: application/json" \
--data '{"method":"debug_traceCall","params":[{"from":null,"to":"0x6b175474e89094c44da98b954eedeac495271d0f","data":"0x70a082310000000000000000000000006E0d01A76C3Cf4288372a29124A26D4353EE51BE"}, "latest"],"id":1,"jsonrpc":"2.0"}'
```

## 使用命令

可以用erc-4337-examples中的[配置](https://github.com/sunchengzhu/erc-4337-examples/blob/main/config.json)直接跑

### 获取simpleAccount账户地址

```bash
yarn run simpleAccount address
```

### eth转账

```bash
yarn run simpleAccount transfer --to <address> --amount <eth>
```
例子：
```bash
yarn run simpleAccount transfer --to 0x413978328AA912d3fc63929d356d353F6e854Ee1 --amount 0.001
```

### erc20转账
```bash
yarn run simpleAccount erc20Transfer --token <address> --to <address> --amount <decimal>
```
例子：
```bash
yarn run simpleAccount erc20Transfer --token 0x61a89342F52d9F31626B56b64A83579E5c368f4c --to 0x413978328AA912d3fc63929d356d353F6e854Ee1 --amount 0.1
```
### 使用Paymaster

附加 `--withPaymaster`即可

```bash
yarn run simpleAccount:erc20Transfer --withPaymaster ...
```
例子：
```bash
yarn run simpleAccount erc20Transfer --token 0x61a89342F52d9F31626B56b64A83579E5c368f4c --to 0x413978328AA912d3fc63929d356d353F6e854Ee1 --amount 0.1 --withPaymaster
```

## 交易分析

1. [例子中的eth转账交易](https://goerli.etherscan.io/tx/0xc01b3537400468763f01e32357c6792f55366643cb099db94e731a872e3cfe2a)中from账户为ERC4337_BUNDLER_PRIVATE_KEY对应的[bundler账户](https://goerli.etherscan.io/address/0xf6286e20f6bdc4bdcff7cb5a8a397bda017fcb84)，to为EntryPoint合约，执行的函数是handleOps，0.026864612848902924 ETH从sender转到EntryPoint，EntryPoint转给目标地址0.001 ETH（命令行传入的值），0.015580453241024552 ETH从EntryPoint补给bundler账户（因为这笔交易花掉的gas费扣到了bundler账户身上）。

2. [例子中的erc20转账交易](https://goerli.etherscan.io/tx/0x4071ac2224535a45c6923e422ace694f84778eb50dbc88d1e592a74c84b576c9)中与eth转账交易类似，只不过EntryPoint发起的不再是eth转账而是erc20转账。
3. [例子中的用了paymaster的erc20转账](https://goerli.etherscan.io/tx/0xb7fa4ba386dbd79e7d567f6b027d9ddda204dcedee06b8c490fddec6b50b33ce)中只有EntryPoint补给bundler账户gas费这一笔互动，因为只需要扣paymaster存在EntryPoint里面的eth即可。

## 原生支持gasless交易的链推荐

如果开发者有为用户代付gas费的需求话**强烈推荐使用[godwoken团队的gasless feature](https://docs.godwoken.io/gasless-feature)**，可以直接处理UserOperation而不需要搭建bundler，UserOperation也能[在区块链浏览器直观展示](https://v1.testnet.gwscan.com/tx/0xc2e2c0141231ae0d544956ef977d8ca328d44134c431c57f71dcb47f71a86fcd)。

