# 16进制在线转换工具分享

> 以太坊中的数据以16进制表示，我在网上搜索在线进制转换工具，没找到纯粹的16进制和10进制相互转换的工具，往往还会展示其他的2进制、8进制，显得非常冗余，所以就写了个小工具方便我们平时看数据。

网址：https://web3zhu.cn

仓库：https://github.com/sunchengzhu/HexToDec

1. 其他在线进制转换工具会把0x开头的数据当作非法输入，这边兼容了0x开头的输入。

2. 另外，以太坊为了数据格式的一致性会往左侧补0以适应32字节的存储槽位（比如下面调用[getValue](https://sepolia.etherscan.io/address/0x9457e3892022c3c7c0641d9300978266b7c69ef5#readContract)的例子），这边也兼容了`0x000...00029a`这样的输入。

   ```shell
   curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_call","params":[{"to":"0x9457e3892022C3c7c0641d9300978266B7C69ef5","data":"0x20965255"},"latest"],"id":1}' https://eth-sepolia.g.alchemy.com/v2/${YOUR_ALCHEMY_API_KEY}
   ```

   ```json
   {"jsonrpc":"2.0","id":1,"result":"0x000000000000000000000000000000000000000000000000000000000000029a"}
   ```

