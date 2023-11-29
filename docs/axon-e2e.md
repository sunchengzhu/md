# Axon e2e 本地测试步骤

## 前置步骤

1. 本地启动axon节点

   略

2. 安装依赖

   ```bash
   cd axon/tests/e2e
   yarn install
   ```

## 步骤

1. 启动http-server

   有了http-server后可以通过html网页向metamask插件传入参数调用接口
   ```bash
   echo '\n' | yarn http-server
   ```

   可以得到以下输出：
    ```bash
    http-server version: 14.1.1
    
    http-server settings: 
    CORS: disabled
    Cache: 3600 seconds
    Connection Timeout: 120 seconds
    Directory Listings: visible
    AutoIndex: visible
    Serve GZIP Files: false
    Serve Brotli Files: false
    Default File Extension: none
    
    Available on:
      http://127.0.0.1:8080
      http://192.168.31.204:8080
      http://192.168.31.2:8080
      http://198.18.0.1:8080
    Hit CTRL-C to stop the server
    ```

   注意：端口必须是8080，否则请执行下面的命令重置。
   
   ```
   yarn run posttest
   ```

2. 测试

   新开一个终端窗口

   ```
   # 执行所有用例
   yarn test
   # 执行单个用例
   jest src/eth_getCode.test.js
   ```

3. Metamask手动操作

   执行上面的测试命令后，会开启一个测试版chrome页面以及弹出metamask插件，请手动勾选"I agree"并点击"Import an existing wallet"按钮，后面测试步骤才能自动执行。

   ![image-20231129231859596](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20231129231859596.png)



4. Debug

   测试过程一闪而过非常快，如果需要在某个用例执行后sleep个几秒观察metamask的返回值，可以参考[pr 1593](https://github.com/axonweb3/axon/pull/1593/files)。
   
   dappeteer测试框架在测试结束后会自动关闭整个测试版chrome页面，如果不希望关掉（比如你想用结合chrome开发者工具仔细研究）也可以用http-server+本地chrome，即启动http-server后直接通过类似`http://127.0.0.1:8080/eth_getCode.html`这样的url直接测试，当前请求的区块链网络就是你本地Metamask选择的网络。
