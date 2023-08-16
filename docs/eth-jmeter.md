# 以太坊兼容链性能测试3—开展测试

## 项目简介

项目地址：https://github.com/sunchengzhu/eth-jmeter

相关文章：[以太坊兼容链性能测试1—准备工作](https://github.com/sunchengzhu/md/blob/main/docs/eth-prepare.md)、[以太坊兼容链性能测试2—性能数据统计](https://github.com/sunchengzhu/md/blob/main/docs/eth-performance.md)

### 功能

1. 使用jmeter开展rpc接口的压测，包含`eth_getBalance`、`eth_getBlockByNumber`、`eth_call`、`eth_sendRawTransaction`，其他接口有需要可以根据现有的代码改造。
1. 根据助记词生成指定数量的账户循环发送交易和查询余额，主要用[jmeter的Concurrency Thread Group插件](https://jmeter-plugins.org/wiki/ConcurrencyThreadGroup/)控制测试时间和线程数。

### 设计

主要通过两个workflow开展压测，[query.yml](https://github.com/sunchengzhu/eth-jmeter/blob/main/.github/workflows/query.yml)是非交易接口`eth_getBalance`、`eth_getBlockByNumber`、`eth_call`的压测，而[tx.yml](https://github.com/sunchengzhu/eth-jmeter/blob/main/.github/workflows/tx.yml)则是原生转账和uniswap `swapExactTokensForETH`交易接口`eth_sendRawTransaction`两个场景的压测。tx.yml会多一步触发[eth_performace的ethStats.yml](https://github.com/sunchengzhu/eth-performance/blob/main/.github/workflows/ethStats.yml)，统计交易性能数据。query.yml的接口是实时返回的，可以直接在客户端这边统计性能数据，我设置了[5s秒打印一次summary](https://github.com/sunchengzhu/eth-jmeter/blob/af87f984ddc3b43f3b70e1978731a2af9f959573/pom.xml#L68)，summary数据格式如下：

```
[INFO] summary +   4099 in 00:00:06 =  683.9/s Avg:   176 Min:   160 Max:   498 Err:     0 (0.00%) Active: 120 Started: 120 Finished: 0 
```

query.yml如果有error能直接在控制台打印出来，并体现在summary的Err数中：

<img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816215300025.png" alt="image-20230816215300025" style="zoom:80%;" />  

## 使用tx.yml

1. fork到自己的仓库（需要fork项目到自己的仓库以使用Actions）

2. 设置GitHub个人访问令牌，跟[性能数据统计](https://github.com/sunchengzhu/md/blob/main/docs/eth-performance.md)中的步骤一样，这里不再赘述。

3. tx.yml新增fantom_testnet环境使用[准备工作](https://github.com/sunchengzhu/md/blob/main/docs/eth-prepare.md)中得到的数据，这边[已经配置完了](https://github.com/sunchengzhu/eth-jmeter/blob/af87f984ddc3b43f3b70e1978731a2af9f959573/.github/workflows/tx.yml#L48-L53)。

4. 选择fantom_testnet环境，输入助记词`strike gather blush lens excite ridge flock random empty remember text universe`（这个助记词的账户有钱）和你自己的Gist ID触发workflow

5. 可以在eth-performance的ethStats中看到交易性能数据

6. 以上用了默认的jmeter线程配置，可以再建一个test分支，通过`mvn jmeter:gui`加载并修改jmx文件不断调整线程数，达到链的性能瓶颈。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816221940890.png" alt="image-20230816221940890" style="zoom:80%;" />  

   

## 使用query.yml

1. tx.yml新增fantom_testnet环境使用准备工作中得到的数据，这边[已经配置完了](https://github.com/sunchengzhu/eth-jmeter/blob/af87f984ddc3b43f3b70e1978731a2af9f959573/.github/workflows/query.yml#L46-L48)。
2. 选择fantom_testnet环境，只有getBalance.jmx需要用到助记词，别的不需要直接点Run workflow就行。
