# DID-Web5

## 合约

### mint

#### 参数校验

- **没有 local_id**

  _那只需要看 DidWeb5DataV1 另一个字段 document_

    - **output - document**
        - Output Cell 的 document 字段是非法的 CBOR 格式，创建失败，报错：InvalidCbor
        - Output Cell 的 document 字段是合法的 CBOR 格式，创建成功
        - Output Cell 不存在，创建失败，报错：MoleculeError::OutOfBound(0, 0)
        - Output Cell 很短，填一两个字节，创建失败，报错：ReaderError



- **有 local_id**

    - **output - local_id**
        - local_id 开头不是 "did:plc:"，创建失败，报错：InvalidDidFormat
        - 随便写个 Identifier，base32 解码失败，创建失败，报错：InvalidDidFormat

    - **witness - operation history**

      文档详见：[The DID Method Powered by CKB (RFC) Pre-2.1](https://www.notion.so/cryptape/The-DID-Method-Powered-by-CKB-RFC-Pre-2-1-2088f0d3781e8023a400c798ef7996f9) 的「Did-web5-ts 必须执行以下验证」

        - operation history 长度+1 ≠ signingKeys 长度，创建失败，报错：InvalidHistory
        - history 长度 = 0，创建失败，报错：InvalidHistory
        - history[0] 的 prev 不为 null，创建失败，报错：NotGenesisOperation
        - history[0] 里的 Identifier 和 local_id 的 Identifier 不一致，创建失败，报错：DidMismatched
        - 构建两条 operation，history[1] 的 prev 和 history[0] 的 CID 不一致，创建失败，报错：InvalidPrev
        - 构建一个 rotationKeys 字段类型不符的 operation，应为数组写成字符串，创建失败，报错：RotationKeysDecodeError
        - 构建一个字段缺失的 operation，例如 type 缺失，创建失败，报错：InvalidOperation
        - 构建一个 operation，去掉 sig 字段的 operation 内容用 rotationKeys[signing_key_index] 公钥签名的结果与这个 operation 的 sig 不一致，也就是验签失败，创建失败，报错：VerifySignatureFailed
        - 构建一个 operation，用的 rotationKeys[signing_key_index] 公钥既不是 secp256k1 公钥也不是 secp256r1（Nistp256）公钥，创建失败，报错：InvalidKey
        - 构建一个 sig 是 "=" 结尾（老的 base64）的 operation，创建失败，报错：InvalidSignaturePadding
        - 构建一个随便写 sig 的 operation，创建失败，报错：InvalidSignature

    - **witness - final sig**
        - 本次上链创建 did:web5 DID 操作的交易选的签名私钥是之前 Operation 里的某个 rotationKeys，验签失败，创建失败，报错（没有特定的 error 标志）
        - 最后一个 Operation 里的每个 rotationKeys 都可以签名这笔交易，验签成功，交易成功

    - **witness - signing_keys**
        - witness 里的 signing_keys 索引数组的值如果超过 rotationKeys 长度，例如只有两个 rotationKeys，传了 3，创建失败，报错：InvalidKeyIndex

    - **legacy 兼容性**
        - 用 plcRotationKeys 字段名而不是 rotationKeys，其他字段保持一致，创建成功
---

### 合法性验证

- 另一个 DID Document Cell 作为输入，创建失败，报错
- 输出是两个 DID Document Cell，创建失败，报错
- 输出是一个 DID Document Cell，创建成功
