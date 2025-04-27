# fiber本地性能测试

## 出现问题场景

### 正常场景

一直往A节点发send_payment和get_payment，实现A到C的转账。

![image-20250428030528151](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20250428030528151.png)

### 问题场景

多了一个B节点和C节点之间的channel，它是向C节点发送open_channel建立出来的，这样性能会有明显损耗。

![image-20250428030218317](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20250428030218317.png)



## 搭建本地测试网fiber节点

拉取仓库

```bash
git clone https://github.com/sunchengzhu/fiber-stability-test-nodes.git
cd fiber-stability-test-nodes/local
```

1. 先shutdown之前建的channel（如果有的话）

   ```bash 
   bash 1_shutdown_channel.sh
   ```

2. 编译fnn并配置节点目录

   在脚本后面传入develop或者find参数

   ```bash
   bash 2_prepare.sh find
   # 如果2_prepare.sh后面不加参数且能找到../fiber/target/release/fnn则会用原来的fnn重新配置节点目录
   bash 2_prepare.sh
   ```

3. 启动节点

   ```bash 
   bash 3_run.sh
   ```

4. 建立网络连接

   ```bash
   bash 4_connect_peer.sh
   ```

5. 建立channel

   建立nodeA ⟺ nodeB、nodeB ⟺ nodeC

   传入find参数则会建立nodeA ⟺ nodeB、nodeB ⟺ nodeC、nodeC ⟺ nodeB

   ```bash
   bash 5_open_channel.sh
   # 复现问题在请在脚本后面传入find参数
   bash 5_open_channel.sh find
   ```

6. 测试一下

   看看A能不能转给C

   ```bash
   bash 6_test_a_to_c.sh
   ```

7. 可以通过count_channels.sh看各channel上的ckb

   ```bash
   bash count_channels.sh
   ```

   

## 性能测试

拉取仓库

```bash
git clone https://github.com/sunchengzhu/fiber-jmeter-sample.git
cd fiber-jmeter-sample
```

默认线程数为4，如果有需要可以更改src/test/jmeter/local.jmx里面的`<stringProp name="ThreadGroup.num_threads">4</stringProp>`改变线程数。

开展测试

_需要有java和maven环境_

```bash
mvn package
rm -rf target/jmeter
mvn jmeter:jmeter@configuration2 -DjmeterTest=local.jmx
```

5min之后可以在target/jmeter下面找到reports目录

打开target/jmeter/reports/local/index.html就能看到测试报告

