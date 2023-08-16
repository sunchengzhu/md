# 以太坊兼容链性能测试2—性能数据统计

## 项目简介

项目地址：https://github.com/sunchengzhu/eth-performance

### 功能

1. 压测过程中实时统计区块交易数、出块间隔、tps等数据
2. 压测结束之后统计区块交易成功率，若存在失败交易则会打印交易hash

### 设计

统计维度是一个个块，调用`eth_blockNumber`不断查询，本地记录出块时间戳，每出一个块我们要统计和上一个块之间的时间间隔，再通过`eth_getBlockTransactionCountByNumber`得到新块的交易数量，交易数量除以时间间隔就是tps。

统计交易成功率的是先通过`eth_getBlockByNumber`获取区块所有的交易hash，再通过多线程执行`eth_getTransactionReceipt`统计每笔交易的状态，然后计算当前块的成功率。[`eth_getBlockReceipts`的代码还没合](https://github.com/ethereum/go-ethereum/pull/27702)，还没成为没在[以太坊标准接口](https://ethereum.github.io/execution-apis/api-documentation/)里面，所以只能先用目前这种笨办法。

不断监控commands.txt文件的变化执行不同的函数，直到执行完`successRate`会结束ethStats服务。

为什么要用`eth_blockNumber`不断查询，而不是用区块时间戳？

1. Layer2链的区块时间戳有不同解决机制并不是真正的出块时间，可以看[这篇文章](https://mirror.xyz/msfew.eth/XxP3h9n67mvQRivwJRM-XC-vX8zoUYI1A0O7bELMuPQ)，即便是以太坊矿工也可以微调几秒的区块时间戳。
2. 区块时间戳往往是秒级别的，对于出块时间较短的链，误差会比较大。

为什么要用websocket连接而不是http？

1. 避免频繁调用`eth_blockNumber`占用带宽资源从而影响真实性能表现
2. http请求延时较大，容易导致本地记录的出块时间戳不太准确从而影响tps统计。之前想过用多线程发http请求减小误差，效果甚微，因为无法控制不同线程请求发送间隔，而websocket可以做到每100ms发一次请求快速得到响应，这样出块间隔误差最多是200ms，这在可接受范围内。

## 使用jar包（手动统计）

1. 下载jar包

   ```bash
   wget https://github.com/sunchengzhu/eth-performance/releases/download/v1.0.0/ethStats.jar
   ```

2. 启动服务

   ```bash
   # 传入webSocket url
   java -jar ethStats.jar $webSocketUrl
   # 不传入参数的话默认连以太坊主网
   java -jar ethStats.jar
   ```

3. 测试ethStats服务是否可用

   打开另一个终端窗口，cd到ethStats.jar的目录。

   ```bash
   # 执行下面的命令，如果服务端成功打印区块高度了，说明服务可用
   echo "printBlockNumber" >> commands.txt
   ```

4. 统计tps

   ```bash
   echo "tps" >> commands.txt
   ```

5. 启动性能测试

   执行你自己的性能测试任务

6. 性能测试结束后停止统计tps

   ```bash
   echo "stopTps" >> commands.txt
   ```

7. 统计交易成功率并结束ethStats服务

   ```bash
   echo "successRate" >> commands.txt
   ```

## 使用Github Actions（自动统计）

1. fork到自己的仓库（需要fork项目到自己的仓库以使用Actions）

2. 新建gist

   之前是监控commands.txt，在另一个终端中往commands.txt中追加数据，现在放到Actions中的两个workflow中，就无法共享数据了，只能通过gist作为媒介，一个workflow写到gist，一个workflow不断发查询请求监控gist。因为要写gist，gist又只能由本人修改，所以无法复用我的，只能新建一个。

   进入[Github Gist](https://gist.github.com)输入commands.txt，点击`Create secret gist`创建gist。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816162317737.png" alt="image-20230816162317737" style="zoom:80%;"/> 

   保存gist id，后面的步骤要用。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816162736927.png" alt="image-20230816162736927" style="zoom:50%;"/>  

   

   

3. 设置GitHub个人访问令牌

   要通过个人访问令牌获得权限通过curl命令去修改gist和触发名下的其他的workflow

   进入[tokens设置](https://github.com/settings/tokens)，点击`Generate new token(classic)`勾选workflow和gist创建token。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816164328799.png" alt="image-20230816164328799" style="zoom:50%;" />    

   保存ghp_开头的token，后面的步骤要用。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/%E6%88%AA%E5%9B%BE.png" alt="截图" style="zoom:50%;" />   

4. 运用GitHub个人访问令牌到项目中

   打开你fork的eth-performance项目，在Settings→Secrets and variables→Actions→New repository secrect中设置GH_TOKEN，Secret就填ghp_开头的token。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816172032582.png" alt="image-20230816172032582" style="zoom:40%;" />    

   

5. 将[ethStats_demo.yml](https://github.com/sunchengzhu/eth-performance/blob/main/.github/workflows/ethStats_demo.yml)中的Gist_ID替换成你自己的

6. 用workflow：EthStats Demo测试一把

   可以不填第一个，默认统计以太坊的性能数据，统计时间可以用默认的两分钟。

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816173422886.png" alt="image-20230816173422886" style="zoom:50%;" /> 

   控制台应该长下面这样

   <img src="https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816173731418.png" alt="image-20230816173731418" style="zoom:80%;" /> 

7. 另外，性能数据还会以csv的格式保存下来，可以点击files下载

   ![image-20230816174324717](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816174324717.png) 

   ![](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816174423125.png) 

   ![image-20230816174506577](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20230816174506577.png) 

