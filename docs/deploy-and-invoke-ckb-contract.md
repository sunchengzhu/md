# 部署和调用ckb合约

> 逐步演示一下「通过ckb-cli部署一个最简单的ckb合约以及调用它」

## 前置步骤

### 编译ckb-cli

```bash
git clone https://github.com/nervosnetwork/ckb-cli.git
cd ckb-cli
make prod
cd ./target/release
# 看下当前ckb-cli版本
./ckb-cli --version
```

```yaml
ckb-cli 1.7.0 (149cd50 2024-03-05)
```

### 选择环境

这里以测试网为例，url来自[Public JSON RPC nodes](https://github.com/nervosnetwork/ckb/wiki/Public-JSON-RPC-nodes)。

```bash
export API_URL=https://testnet.ckbapp.dev
# 获取当前最高区块内容，简单测试一下ckb-cli是否可用
./ckb-cli rpc get_tip_header
```

```yaml
compact_target: 0x1d098b72
dao: 0xc2b7bc8bfd10f14bccfa2cb67afc27002758f09c57a42e06009af4358fe4e108
epoch: "0x70803d1002097 {number: 8343, index: 977, length: 1800}"
extra_hash: 0xf347c5962f84b369c0413453df368d0b6ac3abb7ac22bf7f8d65a11024fb5abb
hash: 0x4f17f265059ea85082255b23642377a045f3699f8fa7d215f30020945d9295e9
nonce: 0x62771bea6750dd77226b2b4fbde09d3e
number: 12589882
parent_hash: 0xb3fa0a893d28347b853af07f240461a06f940922023427c0a50b6d064c076458
proposals_hash: 0xf6156f588e7656ac71a48051b2269d83edd82f268454f4dc929a016ff78f5e2f
timestamp: "1710420984002 (2024-03-14 20:56:24.002 +08:00)"
transactions_root: 0x3f192204650d72e1656dea1b254c6e1d87e005ebf9337591c7c8ef8168648f16
version: 0
```

### 创建测试账户

```bash
# 直接进入华丽的交互模式：可以在CKB>后面输入命令，而不需要输入前面的./ckb-cli
./ckb-cli
# 设置Password新增账户，new个3次，新增3个账户
CKB> account new
# 列出所有账户
CKB> account list
# 后续命令都省略掉"CKB>"，方便复制
```

```yaml
- "#": 0
  address:
    mainnet: ckb1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqduhev6z4snqkm3k2cke0f234w5vcz39xg4szx97
    testnet: ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqduhev6z4snqkm3k2cke0f234w5vcz39xgmzff0x
  address(deprecated):
    mainnet: ckb1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vsvnane4
    testnet: ckt1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vs3krv4f
  has_ckb_pubkey_derivation_root_path: true
  lock_arg: 0xbcbe59a1561305b71b2b16cbd2a8d5d466051299
  lock_hash: 0xaadfdad01086dd90751bad403038511a9c784ec2d3fd39ea1ac9c5062dc52b8d
  source: Local File System
- "#": 1
  address:
    mainnet: ckb1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqgmsnp2z8rme5cqwejkfvwzm6t3su7r9cqgk7wc8
    testnet: ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqgmsnp2z8rme5cqwejkfvwzm6t3su7r9cqxy4pjl
  address(deprecated):
    mainnet: ckb1qyqphpxz5yw8hnfsqan9vjcu9h5hrpeuxtsqrgzpga
    testnet: ckt1qyqphpxz5yw8hnfsqan9vjcu9h5hrpeuxtsq7du7yp
  has_ckb_pubkey_derivation_root_path: true
  lock_arg: 0x1b84c2a11c7bcd300766564b1c2de971873c32e0
  lock_hash: 0xbd08e5a369cea0b62c635233b11f6c72f51af0c37004d95c10b8742c24b0edbc
  source: Local File System
- "#": 2
  address:
    mainnet: ckb1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqvtglslp73v00pg6606n8vdy40dgzspgegn6wkkx
    testnet: ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqvtglslp73v00pg6606n8vdy40dgzspgegag9eu7
  address(deprecated):
    mainnet: ckb1qyqgk3lp7razc77z345l4xwc6f276s9qz3jstvu8hj
    testnet: ckt1qyqgk3lp7razc77z345l4xwc6f276s9qz3jskfzcmw
  has_ckb_pubkey_derivation_root_path: true
  lock_arg: 0x8b47e1f0fa2c7bc28d69fa99d8d255ed40a01465
  lock_hash: 0xc7c3f41000b350c5a110e5738059782320f02b105f79469700e5de671a8c8a18
  source: Local File System
```

### 给测试账户充值

1. 通过[水龙头](https://faucet.nervos.org/)给address_0[充值30w ckb](https://pudge.explorer.nervos.org/zh/transaction/0x4ea605d6ffac5a5c54a606a248d461bb18dfd62119987bd0c9a2b02e8c6bd8ad)

2. 从address_0给address_1[转账1w ckb](https://pudge.explorer.nervos.org/zh/transaction/0x80ffb22cef042d94b02b95b29ef0148e859ffb997febb4dd84a863ccea5afc08)

   ```bash
   wallet transfer --to-address ckt1qyqphpxz5yw8hnfsqan9vjcu9h5hrpeuxtsq7du7yp --capacity 10000.0 --from-account ckt1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vs3krv4f
   ```

3. 查询address_0和address_1的余额

   ```bash
   # address_0
   wallet get-capacity --address ckt1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vs3krv4f
   total: 289999.99999536 (CKB)
   # address_1
   wallet get-capacity --address ckt1qyqphpxz5yw8hnfsqan9vjcu9h5hrpeuxtsq7du7yp
   total: 10000.0 (CKB)
   ```

## 部署合约

### 最简合约

1. 使用 C 写一个简单的 Demo

   simple.c

   ```c
   int main(int argc, char* argv[])
   {
     if (argc == 3) {
       return -2;
     } else if (argc == 5 ) {
      return -3;
     } else {
      return 0;
     }
   }
   ```

2. 编译合约

   ```bash
   # 当前在simple.c同级目录
   sudo docker run --rm -it -v `pwd`:/code nervos/ckb-riscv-gnu-toolchain:xenial bash
   root@9f4ed860a521:/# cd /code
   root@9f4ed860a521:/code# riscv64-unknown-elf-gcc -Os simple.c -o simple
   root@9f4ed860a521:/code# exit
   ```

   也可以直接下载[这个simple二进制](https://file-1304641378.cos.ap-shanghai.myqcloud.com/simple)

3. 把simple二进制放到ckb-cli二进制的同级目录下

### 部署合约

*构造deployment.toml发送部署合约交易*

1. 初始化deployment.toml文件

   ```bash
   deploy init-config --deployment-config deployment.toml
   ```

2. 修改deployment.toml

   ```bash
   # cd到ckb-cli二进制的同级目录
   vim deployment.toml
   ```

   **原始的deployment.toml模板**

   ```toml
   [[cells]]
   name = "my_cell"
   enable_type_id = true 
   location = { file = "build/release/my_cell" }
   
   # reference to on-chain cells, this config is referenced by dep_groups.cells
   [[cells]]
   name = "genesis_cell"
   enable_type_id = false
   location = { tx_hash = "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", index = 0 }
    
   # Dep group cells
   [[dep_groups]]
   name = "my_dep_group"
   cells = [
     "my_cell",
     "genesis_cell"
   ]
   
   # The lock script set to output cells
   [lock]
   code_hash = "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8"
   args = "0x0000000000000000000000000000000000000000"
   hash_type = "type"
   
   # For unlocking inputs with multisig lock script
   [multisig_config]
   sighash_addresses = [
     "ckt1qyq111111111111111111111111111111111111111",
     "ckt1qyq222222222222222222222222222222222222222",
     "ckt1qyq333333333333333333333333333333333333333",
   ]
   require_first_n = 1
   threshold = 2
   ```

   我们只需要配置一下本地的合约文件路径、空的dep_groups、以及发起账户的lock，例子中我们通过address_0发起部署合约的交易。

   **修改后的deployment.toml**

   ```toml
   [[cells]]
   name = "simple"
   enable_type_id = true
   location = { file = "./simple" }
   
   [[dep_groups]]
   name = "my_dep_group"
   cells = []
   
   [lock]
   code_hash = "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8"
   args = "0xbcbe59a1561305b71b2b16cbd2a8d5d466051299"
   hash_type = "type"
   ```


3. 根据deployment.toml创建info.json

   ```bash
   # 先创建一个migrations空目录
   mkdir migrations
   ```

   ```bash
   deploy gen-txs \
       --deployment-config ./deployment.toml \
       --migration-dir ./migrations \
       --from-address ckt1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vs3krv4f \
       --sign-now \
       --info-file info.json
   ```

   ```yaml
   ==== Cell transaction ====
   [cell] NewAdded , name: simple, old-capacity: 0.0, new-capacity: 6190.0
   > old total capacity: 0.0 (CKB) (removed items not included)
   > new total capacity: 6190.0 (CKB)
   [transaction fee]: 0.00006613
   ==== DepGroup transaction ====
   [dep_group] NewAdded , name: my_dep_group, old-capacity: 0.0, new-capacity: 65.0
   > old total capacity: 0.0 (CKB) (removed items not included)
   > new total capacity: 65.0 (CKB)
   [transaction fee]: 0.00000468
   Password: 
   status: success
   ```

4. 签名

   ```bash
   deploy sign-txs \
       --from-account ckt1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vs3krv4f \
       --add-signatures \
       --info-file info.json
   ```

   ```yaml
   Password: 
   cell_tx_signatures:
     0xbcbe59a1561305b71b2b16cbd2a8d5d466051299: 0x4c5e98c83243417b835d6b8d9822c05c03d490b776db0181981170c3458bfc063c9bb2956d1a9ed2f755c83152800aa8ddb8704727236c0e43d88315d6b7fb6401
   dep_group_tx_signatures:
     0xbcbe59a1561305b71b2b16cbd2a8d5d466051299: 0xc73aff4193fee93b3723f793f01a3be722b5c5e5961f5ed4a46618fa0ec67c2111eef9ee11a0d061e0fb14b2b7108ea3aff9e258c58af31a2228987cb3f0114900
   ```

5. 发送交易

   ```bash
   deploy apply-txs --migration-dir ./migrations --info-file info.json
   ```

   ```yaml
   > [send cell transaction]: 0x592217b4ad54a812780bc4a23e7d5c742c968b512f8d951c7ce072e35fb6becc
   > [send dep group transaction]: 0x43a7d66d01a3bc5acdacb796968faf12be3f4852f0e917ce1cadcd59dbcd37cb
   ```

   可以在[这笔交易](https://pudge.explorer.nervos.org/zh/transaction/0x592217b4ad54a812780bc4a23e7d5c742c968b512f8d951c7ce072e35fb6becc)里看到address_0新创建了一个包含type script的cell，说明合约已经部署成功。

   ![image-20240314230929623](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20240314230929623.png)

## 调用合约

*构造tx.json发送调用合约交易*

1. 初始化tx.json文件

   ```bash
   tx init --tx-file tx.json
   ```

   **原始的tx.json模板**

   ```yaml
   {
     "transaction": {
       "version": "0x0",
       "cell_deps": [],
       "header_deps": [],
       "inputs": [],
       "outputs": [],
       "outputs_data": [],
       "witnesses": []
     },
     "multisig_configs": {},
     "signatures": {}
   }
   ```

2. 把给address_1转账交易的交易hash放到input中

   index选address_1的cell

   ```bash
   tx add-input --tx-hash 0x80ffb22cef042d94b02b95b29ef0148e859ffb997febb4dd84a863ccea5afc08 --index 0 --tx-file tx.json
   ```

3. 把address_2放到output中

   目的是销毁掉address_1的cell，创建出address_2的新cell。创建过程会通过code_hash引用address_0 cell的合约script然后传入args执行生成新的cell，这个过程可以视作是对合约的一次调用。

   ```bash
   tx add-output --to-sighash-address ckt1qyqgk3lp7razc77z345l4xwc6f276s9qz3jskfzcmw --capacity 9999.123 --tx-file tx.json
   ```

   **命令修改后的tx.json**

   ```yaml
   {
     "transaction": {
       "version": "0x0",
       "cell_deps": [
         {
           "out_point": {
             "tx_hash": "0xf8de3bb47d055cdf460d93a2a6e1b05f7432f9777c8c474abf4eec1d4aee5d37",
             "index": "0x0"
           },
           "dep_type": "dep_group"
         }
       ],
       "header_deps": [],
       "inputs": [
         {
           "since": "0x0",
           "previous_output": {
             "tx_hash": "0x80ffb22cef042d94b02b95b29ef0148e859ffb997febb4dd84a863ccea5afc08",
             "index": "0x0"
           }
         }
       ],
       "outputs": [
         {
           "capacity": "0xe8cf6adde0",
           "lock": {
             "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
             "hash_type": "type",
             "args": "0x8b47e1f0fa2c7bc28d69fa99d8d255ed40a01465"
           },
           "type": null
         }
       ],
       "outputs_data": [
         "0x"
       ],
       "witnesses": []
     },
     "multisig_configs": {},
     "signatures": {}
   }
   ```

4. 查看交易详情

   ```bash
   tx info --tx-file tx.json
   ```

   ```yaml
   [input] ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqgmsnp2z8rme5cqwejkfvwzm6t3su7r9cqxy4pjl => 10000.0, (data-length: 0, type-script: none, lock-kind: sighash(secp))
   [output] ckt1qzda0cr08m85hc8jlnfp3zer7xulejywt49kt2rr0vthywaa50xwsqvtglslp73v00pg6606n8vdy40dgzspgegag9eu7 => 9999.123, (data-length: 0, type-script: none, lock-kind: sighash(secp))
   input_total: 10000.0 (CKB)
   output_total: 9999.123 (CKB)
   tx_fee: 0.877 (CKB)
   ```

   差值就是给矿工的tx_fee，ckb-cli这边的限制要小于1个ckb。

5. 手动修改output cell的type

   type的code_hash可以查看[账户](https://pudge.explorer.nervos.org/zh/address/ckt1qyqte0je59tpxpdhrv43dj7j4r2ages9z2vs3krv4f)的UTXO获取，在之前浏览器不展示type script hash的时候我们可以直接看migrations文件夹下json文件的type_id字段或者通过[这个函数](https://github.com/cryptape/ckb-py-integration-test/blob/305befb02f5acd9b37ff7f79d43b7bb01e45046b/framework/helper/contract.py#L65-L82)获取。

   ![](https://typora-1304641378.cos.ap-shanghai.myqcloud.com/images/image-20240315000706603.png)

   ```python
   from framework.helper.contract import get_ckb_contract_codehash
   
   
   def main():
       tx_hash = "0x592217b4ad54a812780bc4a23e7d5c742c968b512f8d951c7ce072e35fb6becc"
       tx_index = 0
       api_url = "https://testnet.ckbapp.dev"
   
       codehash = get_ckb_contract_codehash(tx_hash, tx_index, api_url=api_url)
       print("Type Code Hash:", codehash)
   
   
   if __name__ == "__main__":
       main()
   ```

   修改后的type

   ```json
   {
       "code_hash": "0x9b34af6134718d2845ebf7c6ca148be6e1034c20c38414d09541e6d708fb3a7b",
       "hash_type": "type",
       "args": "0x01"
   }
   ```

6. 手动添加cell_dep

   把部署合约的交易hash添加进cell_deps

   添加的cell_dep

   ```bash
   {
       "out_point": {
           "tx_hash": "0x592217b4ad54a812780bc4a23e7d5c742c968b512f8d951c7ce072e35fb6becc",
           "index": "0x0"
       },
       "dep_type": "code"
   }
   ```

7. 导出address_1的私钥

   ```bash
   account export --lock-arg 0x1b84c2a11c7bcd300766564b1c2de971873c32e0 --extended-privkey-path  ./address_1.demo
   ```

8. 构建签名

   ```bash
   tx sign-inputs --privkey-path ./address_1.demo --tx-file tx.json
   ```

   ```yaml
   - lock-arg: 0x1b84c2a11c7bcd300766564b1c2de971873c32e0
     signature: 0xfe5b3db9618d8ead16ffb414ac9aa5ffda03c2f4490962270322c28ffbcff7a36ed5f745ea426fdc3567089e04a97f9f4ed31136df3238199cf3fad8edd07e4400
   ```

9. 将上述签名添加到tx.json

   ```bash
   tx add-signature --lock-arg 0x1b84c2a11c7bcd300766564b1c2de971873c32e0 --signature 0xfe5b3db9618d8ead16ffb414ac9aa5ffda03c2f4490962270322c28ffbcff7a36ed5f745ea426fdc3567089e04a97f9f4ed31136df3238199cf3fad8edd07e4400 --tx-file tx.json
   ```

   **最终使用的tx.json**

   ```yaml
   {
     "transaction": {
       "version": "0x0",
       "cell_deps": [
         {
           "out_point": {
             "tx_hash": "0x592217b4ad54a812780bc4a23e7d5c742c968b512f8d951c7ce072e35fb6becc",
             "index": "0x0"
           },
           "dep_type": "code"
         },
         {
           "out_point": {
             "tx_hash": "0xf8de3bb47d055cdf460d93a2a6e1b05f7432f9777c8c474abf4eec1d4aee5d37",
             "index": "0x0"
           },
           "dep_type": "dep_group"
         }
       ],
       "header_deps": [],
       "inputs": [
         {
           "since": "0x0",
           "previous_output": {
             "tx_hash": "0x80ffb22cef042d94b02b95b29ef0148e859ffb997febb4dd84a863ccea5afc08",
             "index": "0x0"
           }
         }
       ],
       "outputs": [
         {
           "capacity": "0xe8cf6adde0",
           "lock": {
             "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
             "hash_type": "type",
             "args": "0x8b47e1f0fa2c7bc28d69fa99d8d255ed40a01465"
           },
           "type": {
             "code_hash": "0x9b34af6134718d2845ebf7c6ca148be6e1034c20c38414d09541e6d708fb3a7b",
             "hash_type": "type",
             "args": "0x01"
           }
         }
       ],
       "outputs_data": [
         "0x"
       ],
       "witnesses": []
     },
     "multisig_configs": {},
     "signatures": {
       "0x1b84c2a11c7bcd300766564b1c2de971873c32e0": [
         "0xfe5b3db9618d8ead16ffb414ac9aa5ffda03c2f4490962270322c28ffbcff7a36ed5f745ea426fdc3567089e04a97f9f4ed31136df3238199cf3fad8edd07e4400"
       ]
     }
   }
   ```

10. 发送交易

    ```bash
    tx send --tx-file tx.json
    ```

    返回交易hash：[0x82e66704a5868a738512761c97db1873e0c9fa971beb177f330112beb78199fe](https://pudge.explorer.nervos.org/zh/transaction/0x82e66704a5868a738512761c97db1873e0c9fa971beb177f330112beb78199fe)

11. 查询address_2的cell

    ```bash
    wallet get-live-cells --address ckt1qyqgk3lp7razc77z345l4xwc6f276s9qz3jskfzcmw
    ```

    ```yaml
    live_cells:
      - capacity: 9999.123 (CKB)
        data_bytes: 0
        index:
          output_index: 0
          tx_index: 1
        lock_hash: 0xc7c3f41000b350c5a110e5738059782320f02b105f79469700e5de671a8c8a18
        mature: true
        number: 12591647
        output_index: 0
        tx_hash: 0x82e66704a5868a738512761c97db1873e0c9fa971beb177f330112beb78199fe
        type_hashes:
          - 0x9b34af6134718d2845ebf7c6ca148be6e1034c20c38414d09541e6d708fb3a7b
          - 0x6e4f0a14d6ff27ab8bb8b2f5431fe01adc85b730b5cc6230a0f3650fb6f72702
    ```

    说明调用成功

    可以在[script引用的Cells](https://pudge.explorer.nervos.org/zh/script/0x9b34af6134718d2845ebf7c6ca148be6e1034c20c38414d09541e6d708fb3a7b/type/referring_cells)找到这笔交易

## 参考资料

1. [Deploy contracts](https://github.com/nervosnetwork/ckb-cli/wiki/Deploy-contracts)

2. [Handle Complex Transaction](https://github.com/nervosnetwork/ckb-cli/wiki/Handle-Complex-Transaction)

3. [最简合约](https://docs-xi-two.vercel.app/docs/docs/script/script-minimal)
