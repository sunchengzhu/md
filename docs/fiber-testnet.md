# fiber测试网公共节点使用手册

## 部署节点

1. 下载fnn

   以macOS (Apple silicon)版本为例

   ```bash
   mkdir tmp && cd tmp
   wget https://github.com/nervosnetwork/fiber/releases/download/v0.4.0/fnn_v0.4.0-x86_64-darwin-portable.tar.gz
   tar xzvf fnn_v0.4.0-x86_64-darwin-portable.tar.gz
   ```

   

2. 把账户私钥导出到fiber节点的ckb目录下

   后续这个[ckb-cli](https://github.com/nervosnetwork/ckb-cli)账户会为本地节点和测试网公共节点建立channel付费。

   ```bash
   wget https://github.com/nervosnetwork/ckb-cli/releases/download/v1.12.0/ckb-cli_v1.12.0_aarch64-apple-darwin.zip
   unzip ckb-cli_v1.12.0_aarch64-apple-darwin.zip
   cp ckb-cli_v1.12.0_aarch64-apple-darwin/ckb-cli .
   ```

   创建本地节点nodeA目录

   ```bash
   mkdir -p testnet-fnn/nodeA/ckb
   ./ckb-cli account new
   ./ckb-cli account export  --lock-arg 0xcc015401df73a3287d8b2b19f0cc23572ac8b14d --extended-privkey-path exported-key
   head -n 1 ./exported-key > testnet-fnn/nodeA/ckb/key
   chmod 600 testnet-fnn/nodeA/ckb/key
   # check nodeA key
   ./ckb-cli util key-info  --privkey-path testnet-fnn/nodeA/ckb/key
   ```




3. 复制config.yml

   ```bash
   cp config/testnet/config.yml testnet-fnn/nodeA
   ```
   
   
   
4. 通过faucet给nodeA节点的地址充值10000ckb和100RUSD。

   RUSD的faucet没有办法直接填地址领，所以可以先连接joyid这样的钱包领100RUSD，再通过https://testnet.joyid.dev转账给nodeB节点的地址。

   - ckb: https://faucet.nervos.org

   - RUSD: https://testnet0815.stablepp.xyz/faucet

     

5. 删掉不再用的文件

   ```bash
   rm -f ckb-cli ckb-cli_v1.12.0_aarch64-apple-darwin.zip fnn_v0.4.0-x86_64-darwin-portable.tar.gz exported-key fnn-migrate
   rm -rf ckb-cli_v1.12.0_aarch64-apple-darwin config
   ```



6. 启动节点

   ```bash
   RUST_LOG=info ./fnn -c testnet-fnn/nodeA/config.yml -d testnet-fnn/nodeA > testnet-fnn/nodeA/a.log 2>&1 &
   ```
   



## 和公共节点1建立ckb channel

1. 确认node1的node info

   ```bash
   curl -s --location 'http://18.162.235.225:8227' --header 'Content-Type: application/json' --data '{
       "id": 1,
       "jsonrpc": "2.0",
       "method": "node_info",
       "params": []
   }' | jq '.result.addresses[0]'
   ```

   ```json
   "/ip4/18.162.235.225/tcp/8119/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
   ```



2. 建立nodeA和node1的网络连接

   ```bash
   curl --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 2,
       "jsonrpc": "2.0",
       "method": "connect_peer",
       "params": [
           {
               "address": "/ip4/18.162.235.225/tcp/8119/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","result":null,"id":2}
   ```



3. 建立一个500ckb的channel: nodeA (500ckb) ⟺ node1 (0)

   _node1配置的自动接受最小资金是500ckb，所以请传入500ckb及以上的funding_amount_

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 3,
       "jsonrpc": "2.0",
       "method": "open_channel",
       "params": [
           {
               "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
               "funding_amount": "0xba43b7400",
               "public": true
           }
       ]
   }'
   ```
      ```json
   {"jsonrpc":"2.0","result":{"temporary_channel_id":"0xd2acd24156e9373db5d699c2adf3b7c3e443acfd7f8b53591987c9d2afd6cfef"},"id":3}
      ```




4. 查询nodeA和node1之间channels

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 4,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
           }
       ]
   }' | jq
   ```

   等到state_name变为CHANNEL_READY
   
   **注意：channel刚变为CHANNEL_READY状态，向其发送send_payment，仍可能报`error: Failed to build route`，可以等待一段时间后再重试。**
   
   ```json
   {
     "jsonrpc": "2.0",
     "result": {
       "channels": [
         {
           "channel_id": "0x4614318335bc5aff00ad08babae123e7c42a4f1d8955b19ad3596ed357e8678e",
           "is_public": true,
           "channel_outpoint": "0x0886c0d579d0cc2a402caca7657ecce148fde16dacbb0aa68bd3d6e0aa951b7c00000000",
           "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
           "funding_udt_type_script": null,
           "state": {
             "state_name": "CHANNEL_READY",
             "state_flags": []
           },
           "local_balance": "0xa32aef600",
           "offered_tlc_balance": "0x0",
           "remote_balance": "0x460913c00",
           "received_tlc_balance": "0x0",
           "latest_commitment_transaction_hash": "0x4eff1258f0f2ec301cde845f1c856be48e07d8cf210706dd1e7d85eae4269ee7",
           "created_at": "0x19584157367",
           "enabled": true,
           "tlc_expiry_delta": "0x5265c00",
           "tlc_fee_proportional_millionths": "0x3e8"
         }
       ]
     },
     "id": 4
   }
   ```



5. 调用node2的new_invoice接口生成一个invoice

   amount为0x5f5e100(100000000)，也就是1个ckb。

   ```bash
   # Generate a 32-byte random number and represent it in hexadecimal
   payment_preimage="0x$(openssl rand -hex 32)"
   echo $payment_preimage
   ```
   
   ```bash
   0x848bf468e2b15ba56eba675a8e2dbe9db69dfaaaaf6ddcf784593137bf44a66a
   ```
   
   ```bash
   curl -s --location 'http://18.163.221.211:8227' --header 'Content-Type: application/json' --data '{
       "id": 5,
       "jsonrpc": "2.0",
       "method": "new_invoice",
       "params": [
           {
               "amount": "0x5f5e100",
               "currency": "Fibt",
               "description": "test invoice generated by node2",
               "expiry": "0xe10",
               "final_cltv": "0x28",
               "payment_preimage": "0x848bf468e2b15ba56eba675a8e2dbe9db69dfaaaaf6ddcf784593137bf44a66a",
               "hash_algorithm": "sha256"
           }
       ]
   }' | jq -c '.result'
   ```
   
   ```json
   {"invoice_address":"fibt1000000001peseucdphcxgfw0pnm6vhpylev6mvyglew29wvy79vdyzln38vjc4fs2tty7c0j02fg827c0l06e3d6jpkqlxnwdu02wps5j8mtrhygeuuaarf5a5g742uscs9c22s3684q6u8m5wfn2s4rwv4rrjk2qv3ghn26e797yva2l874pdgsduwxr5tkryhke9dex34u3w40yeukedk03jtcmsh042g86vdfs4gqavypcyv08q2g3a42fq2uxwzjd38m5g09qqafh9frlwmfrv8v46nz4mzhu0let7dcv0vfyetmfx3wrnm90l6ygsahs6z4a2xrmjc65dnh8srjh00f30j8p3njgfxzx2zluwne5hnpgqdnvf7f","invoice":{"currency":"Fibt","amount":"0x5f5e100","signature":"1d09170509031f0e1b09030c070c151a1302151b02171c0f1f190b1e0d180c0f0c0904190b1b0906110e03131b050f1f1a0408101d17101a02151d0a06031b12181a140d131707100312170f0f09110f12070111131208090602060a021f1c0e1319141713010800","data":{"timestamp":"0x1958447cede","payment_hash":"0x668e7d2f15b8becba2bd933026134428e0816cb65f96a7689794ff26cb0f6272","attrs":[{"Description":"test invoice generated by node2"},{"ExpiryTime":{"secs":3600,"nanos":0}},{"HashAlgorithm":"sha256"},{"PayeePublicKey":"0291a6576bd5a94bd74b27080a48340875338fff9f6d6361fe6b8db8d0d1912fcc"}]}}}
   ```
   
   记录下返回的invoice_address
   
   
   
6. 让nodeA付款前先查询一下各channel的local_balance和remote_balance

   nodeA ⟺ node1

   ```json
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 6,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
           }
       ]
   }' | jq -c '.result.channels[] | {local_balance, remote_balance}'
   ```

   ```json
   {"local_balance":"0xa32aef600","remote_balance":"0x460913c00"}
   ```

   node1 ⟺ node2

   ```bash
   curl -s --location 'http://18.162.235.225:8227' --header 'Content-Type: application/json' --data '{
       "id": 6,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmbKyzq9qUmymW2Gi8Zq7kKVpPiNA1XUJ6uMvsUC4F3p89"
           }
       ]
   }' | jq -c '.result.channels[] | {local_balance, remote_balance}'
   ```

   ```json
   {"local_balance":"0x1748184d3b","remote_balance":"0x5e9ac5"}
   {"local_balance":"0x1748630df7","remote_balance":"0x13da09"}
   {"local_balance":"0xa38b9d","remote_balance":"0x1747d35c63"}
   {"local_balance":"0xc505f","remote_balance":"0x17486a97a1"}
   {"local_balance":"0x45a9b5cf3","remote_balance":"0x916e2dc010d"}
   {"local_balance":"0x504f21d045c","remote_balance":"0x4164b5a59a4"}
   {"local_balance":"0x916dce625e9","remote_balance":"0x460913817"}
   {"local_balance":"0x48ab5ace976","remote_balance":"0x49087ca748a"}
   {"local_balance":"0x48ab5acd200","remote_balance":"0x49087ca8c00"}
   ```



7. 向nodeA发送send_payment，实现nodeA向node2付款。

   传入之前记录的invoice_address

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 7,
       "jsonrpc": "2.0",
       "method": "send_payment",
       "params": [
           {
               "invoice": "fibt1000000001peseucdphcxgfw0pnm6vhpylev6mvyglew29wvy79vdyzln38vjc4fs2tty7c0j02fg827c0l06e3d6jpkqlxnwdu02wps5j8mtrhygeuuaarf5a5g742uscs9c22s3684q6u8m5wfn2s4rwv4rrjk2qv3ghn26e797yva2l874pdgsduwxr5tkryhke9dex34u3w40yeukedk03jtcmsh042g86vdfs4gqavypcyv08q2g3a42fq2uxwzjd38m5g09qqafh9frlwmfrv8v46nz4mzhu0let7dcv0vfyetmfx3wrnm90l6ygsahs6z4a2xrmjc65dnh8srjh00f30j8p3njgfxzx2zluwne5hnpgqdnvf7f"
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","result":{"payment_hash":"0x668e7d2f15b8becba2bd933026134428e0816cb65f96a7689794ff26cb0f6272","status":"Created","created_at":"0x195844e10b1","last_updated_at":"0x195844e10b1","failed_error":null,"fee":"0x186a0"},"id":7}
   ```



8. 再次查询各channel的local_balance和remote_balance

   nodeA ⟺ node1

   从`{"local_balance":"0xa32aef600","remote_balance":"0x460913c00"}`变为`{"local_balance":"0xa2cb78e60","remote_balance":"0x46688a3a0"}`

   node1 ⟺ node2

   其余未变，`{"local_balance":"0x916dce625e9","remote_balance":"0x460913817"}`变为`{"local_balance":"0x916d6f044e9","remote_balance":"0x466871917"}`

   也就是说，付款前后channels的金额完成了下面的变化：

   - 付款前

     nodeA (43800000000) ⟺ node1 (18800000000)

     node1 (9993800001001) ⟺ node2 (18799998999)

   - 付款后

     nodeA (43699900000) ⟺ node1 (18900100000)

     node1 (9993700001001) ⟺ node2 (18899998999)

   nodeA资金变化：43699900000 - 43800000000 = -100100000

   node1资金变化：9993700001001 + 18900100000 - 9993800001001 - 18800000000 = 100000 

   node2资金变化：18899998999 - 18799998999 = 100000000 

   **综上可以看出成功完成了nodeA → node1 → node2金额为100000000的ckb转账，中间节点node1获得了100000手续费。**



9. 关闭这个nodeA和node1的channel

   传入channel_id和接收通道余额的地址

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 9,
       "jsonrpc": "2.0",
       "method": "shutdown_channel",
       "params": [
           {
               "channel_id": "0x4614318335bc5aff00ad08babae123e7c42a4f1d8955b19ad3596ed357e8678e",
               "close_script": {
                   "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
                   "hash_type": "type",
                   "args": "0xcc015401df73a3287d8b2b19f0cc23572ac8b14d"
               },
               "fee_rate": "0x3FC"
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","result":null,"id":9}
   ```

   可以在ckb explorer上看到nodeA的地址新增了一笔+498.99899462CKB的交易，也就是说在fiber节点上的ckb流转的最终结果会在shutdown_channel关闭channel后在链上体现。

   

## 和公共节点1建立udt channel

1. 建立nodeA和node1的网络连接

   

2. 建立一个100RUSD的channel: nodeA (100RUSD) ⟺ node1 (0)

   _node1配置的自动接受最小资金是100RUSD，所以请传入100RUSD及以上的funding_amount_

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 2,
       "jsonrpc": "2.0",
       "method": "open_channel",
       "params": [
           {
               "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
               "funding_amount": "0x2540be400",
               "public": true,
               "funding_udt_type_script": {
                   "code_hash": "0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a",
                   "hash_type": "type",
                   "args": "0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b"
               }
           }
       ]
   }'
   ```

   ```json
   {"jsonrpc":"2.0","result":{"temporary_channel_id":"0x9fe5bfa58c2b14c731bf006b20a337c285194ea5cb5b11c7164d54fcb405a3da"},"id":2}
   ```

   

3. 查询nodeA和node1之间channels

   ```bash
   curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
       "id": 3,
       "jsonrpc": "2.0",
       "method": "list_channels",
       "params": [
           {
               "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
           }
       ]
   }' | jq
   ```

   ```json
   {
     "jsonrpc": "2.0",
     "result": {
       "channels": [
         {
           "channel_id": "0x2d05d9515edc8447084584eaa08de0ae9644b64d4b6abf78907ec687b243fda9",
           "is_public": true,
           "channel_outpoint": "0x40041969869ef7ad90fbb7de989fbd823a6a4935d9aa14e1613379e13bfebf4f00000000",
           "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
           "funding_udt_type_script": {
             "code_hash": "0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a",
             "hash_type": "type",
             "args": "0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b"
           },
           "state": {
             "state_name": "CHANNEL_READY",
             "state_flags": []
           },
           "local_balance": "0x2540be400",
           "offered_tlc_balance": "0x0",
           "remote_balance": "0x0",
           "received_tlc_balance": "0x0",
           "latest_commitment_transaction_hash": "0x774b9b5ef3083f4c1f8883773d8984bbbd4c01eb69c33af5dfad2ca968d18fe6",
           "created_at": "0x19584beed41",
           "enabled": true,
           "tlc_expiry_delta": "0x5265c00",
           "tlc_fee_proportional_millionths": "0x3e8"
         }
       ]
     },
     "id": 3
   }
   ```




4. 调用node2的new_invoice接口生成一个invoice

   amount为0x5f5e100(100000000)，也就是1个RUSD。

   这边还是要用的唯一的payment_preimage，可以用`echo "0x$(openssl rand -hex 32)"`生成。

   ```bash
   curl -s --location 'http://18.163.221.211:8227' --header 'Content-Type: application/json' --data '{
       "id": 4,
       "jsonrpc": "2.0",
       "method": "new_invoice",
       "params": [
           {
               "amount": "0x5f5e100",
               "currency": "Fibt",
               "description": "test invoice generated by node2",
               "expiry": "0xe10",
               "final_cltv": "0x28",
               "payment_preimage": "0x894fcc54fa2af0f73eabb2b2d75c67f6520f6cff87ce8100fb020cb9fe5308a5",
               "hash_algorithm": "sha256",
               "udt_type_script": {
                   "code_hash": "0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a",
                   "hash_type": "type",
                   "args": "0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b"
               }
           }
       ]
   }' | jq -c '.result'
   ```

   ```json
   {"invoice_address":"fibt1000000001px88ja42xcmczxzat8lhtucspgzzm9jy2kntteezhh5x70hvkce8dnreerucdlxx0wgdg7xzxp2q83uf4sesdqkqvwffzmtadqsfgst2hhazly2lhg7944anzr3tgg60hwks6h4d3r4k8ur40lazj7lnezjumk5drajd97eut2qxgq4qzjnr5v6nwt6f3zq6nkpfxrre3z4x596qt7d54qxasydr37p96wjnccyfy299d6snvzp66n7drtyjrl85eaf9dxxe2czn87zhap7yumvn7hk2j23th0glxdhpnzty66peshqeuhl7fgm9valjzv7v8ukhdy5y8lh2r55ghjrhxx6zfm3p4rxe3ydmvf855clksvu060de8upcc5s5rzakwa6mf0vmud5mal3kwlxmsf0vtsmcgku9tk9q7qkh8lnh08q2geyfwef53dptyyg47j28c228zfcl04k7n90vpla8dys34qsdfzdysqcz5z6e8qqty53ay","invoice":{"currency":"Fibt","amount":"0x5f5e100","signature":"1b1c0d141b1d1f11160e1f061b10090f0c0b101b1808161c050b1605001e001617071f13170f07000a081904090e190914110d010b040408151e120a07180a0a070209181f0f15161e13050f0c011f1d070d0410111500100d09020d041000180214021a19070000","data":{"timestamp":"0x19584c35498","payment_hash":"0x65840b00cd7f01caacfcfa8836f27f5055254b8e12ab21e4894f2aa0fe696c0d","attrs":[{"Description":"test invoice generated by node2"},{"ExpiryTime":{"secs":3600,"nanos":0}},{"UdtScript":"0x550000001000000030000000310000001142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a0120000000878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b"},{"HashAlgorithm":"sha256"},{"PayeePublicKey":"0291a6576bd5a94bd74b27080a48340875338fff9f6d6361fe6b8db8d0d1912fcc"}]}}}
   ```

   

   5. 让nodeA付款前先查询一下各channel的local_balance和remote_balance

      nodeA ⟺ node1

      ```bash
      curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
          "id": 5,
          "jsonrpc": "2.0",
          "method": "list_channels",
          "params": [
              {
                  "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
              }
          ]
      }' | jq -c '.result.channels[] | {local_balance, remote_balance}'
      ```

      ```json
      {"local_balance":"0x2540be400","remote_balance":"0x0"}
      ```

      node1 ⟺ node2

      只看包含funding_udt_type_script的channel

      ```bash
      curl -s --location 'http://18.162.235.225:8227' --header 'Content-Type: application/json' --data '{
          "id": 5,
          "jsonrpc": "2.0",
          "method": "list_channels",
          "params": [
              {
                  "peer_id": "QmbKyzq9qUmymW2Gi8Zq7kKVpPiNA1XUJ6uMvsUC4F3p89"
              }
          ]
      }' | jq -c '.result.channels[] | select(.funding_udt_type_script != null) | {local_balance, remote_balance}'
      ```

      ```json
      {"local_balance":"0x174203e7bb","remote_balance":"0x6730045"}
      {"local_balance":"0x1748630df7","remote_balance":"0x13da09"}
      {"local_balance":"0xa38b9d","remote_balance":"0x1747d35c63"}
      {"local_balance":"0xc505f","remote_balance":"0x17486a97a1"}
      ```

   

   6. 向nodeA发送send_payment，实现nodeA向node2付款。

      ```bash
      curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
          "id": 6,
          "jsonrpc": "2.0",
          "method": "send_payment",
          "params": [
              {
                  "invoice": "fibt1000000001px88ja42xcmczxzat8lhtucspgzzm9jy2kntteezhh5x70hvkce8dnreerucdlxx0wgdg7xzxp2q83uf4sesdqkqvwffzmtadqsfgst2hhazly2lhg7944anzr3tgg60hwks6h4d3r4k8ur40lazj7lnezjumk5drajd97eut2qxgq4qzjnr5v6nwt6f3zq6nkpfxrre3z4x596qt7d54qxasydr37p96wjnccyfy299d6snvzp66n7drtyjrl85eaf9dxxe2czn87zhap7yumvn7hk2j23th0glxdhpnzty66peshqeuhl7fgm9valjzv7v8ukhdy5y8lh2r55ghjrhxx6zfm3p4rxe3ydmvf855clksvu060de8upcc5s5rzakwa6mf0vmud5mal3kwlxmsf0vtsmcgku9tk9q7qkh8lnh08q2geyfwef53dptyyg47j28c228zfcl04k7n90vpla8dys34qsdfzdysqcz5z6e8qqty53ay"
              }
          ]
      }'
      ```

      ```json
      {"jsonrpc":"2.0","result":{"payment_hash":"0x65840b00cd7f01caacfcfa8836f27f5055254b8e12ab21e4894f2aa0fe696c0d","status":"Created","created_at":"0x19584d6f1b8","last_updated_at":"0x19584d6f1b8","failed_error":null,"fee":"0x186a0"},"id":6}
      ```

      

   7. 再次查询各channel的local_balance和remote_balance

      nodeA ⟺ node1

      从`{"local_balance":"0x2540be400","remote_balance":"0x0"}`变为`{"local_balance":"0x24e147c60","remote_balance":"0x5f767a0"}`

      node1 ⟺ node2

      其余未变，`{"local_balance":"0x174203e7bb","remote_balance":"0x6730045"}`变为`{"local_balance":"0x173c0e06bb","remote_balance":"0xc68e145"}`

      也就是说，付款前后channels的金额完成了下面的变化：

      - 付款前

        nodeA (10000000000) ⟺ node1 (0)

        node1 (99891799995) ⟺ node2 (108200005)

      - 付款后

        nodeA (9899900000) ⟺ node1 (100100000)

        node1 (99791799995) ⟺ node2 (208200005)

      nodeA资金变化：9899900000 - 10000000000 = -100100000 

      node1资金变化：99791799995 + 100100000 - 99891799995 = 100000 

      node2资金变化：208200005 - 108200005 = 100000000 

      **综上可以看出成功完成了nodeA → node1 → node2金额为100000000的udt转账，中间节点node1获得了100000手续费。**

      

   8. 关闭这个nodeA和node1的channel

      ```bash
      curl -s --location 'http://127.0.0.1:8227' --header 'Content-Type: application/json' --data '{
          "id": 8,
          "jsonrpc": "2.0",
          "method": "shutdown_channel",
          "params": [
              {
                  "channel_id": "0x2d05d9515edc8447084584eaa08de0ae9644b64d4b6abf78907ec687b243fda9",
                  "close_script": {
                      "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
                      "hash_type": "type",
                      "args": "0xcc015401df73a3287d8b2b19f0cc23572ac8b14d"
                  },
                  "fee_rate": "0x3FC"
              }
          ]
      }'
      ```

      ```json
      {"jsonrpc":"2.0","result":null,"id":8}
      ```

      可以在ckb explorer上看到nodeA的地址新增了一笔+98.999RUSD的交易，也就是说在fiber节点上的udt流转的最终结果会在shutdown_channel关闭channel后在链上体现。

      

   

   

   

   
