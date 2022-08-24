# google-cloud-kms-importing

## 概述

根据 Google Cloud 文档[1]的描述，将密钥导入 Google Cloud KMS，主要包括以下步骤。

- 格式化密钥
- 封装
- 导入
- 验证

## 格式化密钥

这里主要讨论下如何将助记词生成的非对称密钥，格式化成可以导入的格式。

1. 生成助记词

   ```
   candy maple cake sugar pudding cream honey rich smooth crumble sweet treat
   ```

2. 生成私钥

   ```
   0xc87509a1c067bbde78beb793e6fa76530b6382a4c0241e5e4a9ec0a0f44dc0d3
   ```

3. 使用`asn1`将私钥从16进制字符串转换为`pem`格式

   ```js
   const asn1 = require('asn1.js');
   const crypto = require('crypto');
   
   const ECPrivateKeyASN = asn1.define('ECPrivateKey', function encode() {
     this.seq().obj(
       this.key('version').int(),
       this.key('privateKey').octstr(),
       this.key('parameters').explicit(0).objid().optional(),
       this.key('publicKey').explicit(1).bitstr().optional(),
     );
   });
   
   const toPEM = (privateKeyHex) => {
     const ecdh = crypto.createECDH('secp256k1');
     const keypair = ecdh.setPrivateKey(privateKeyHex, 'hex');
     const privateKey = keypair.getPrivateKey();
     const publicKey = keypair.getPublicKey();
     const privateKeyObject = {
       version: 1,
       privateKey,
       publicKey: { data: publicKey },
       parameters: [1, 3, 132, 0, 10],
     };
     return ECPrivateKeyASN.encode(privateKeyObject, 'pem', { label: 'EC PRIVATE KEY' });
   };
   ```

   可以得到`pem`格式密钥

   ```
   -----BEGIN EC PRIVATE KEY-----
   MHQCAQEEIMh1CaHAZ7veeL63k+b6dlMLY4KkwCQeXkqewKD0TcDToAcGBSuBBAAK
   oUQDQgAEr4C5DSUUXaKMWDNZvrR7IXlrL+GiPBUR5EPnpk39sn10NMOA8KpMUA4i
   CqGp0GhRSx/01QGeYk57oe/oKzQKWQ==
   -----END EC PRIVATE KEY-----
   ```



4. 使用`openssl`命令将私钥从`pem`格式转换为`der`格式

   ```bash
   openssl pkcs8 -topk8 -nocrypt -inform PEM -outform DER \
       -in /path/to/asymmetric-key-pem \
       -out /path/to/formatted-key
   ```


## 封装

封装的方式共有两种。

### 使用 Google Cloud CLI 自动封装密钥

使用 Google Cloud CLI 自动封装密钥需要在本地准备好密钥的源文件，在本地安装 Google Cloud CLI 和 Pyca 加密库。Google Cloud KMS 官方推荐这种封装方法。

使用`gcloud`命令可以实现自动封装和导入。这时只需要提供未封装的密钥文件。

### 手动封装密钥

在某些情况下，由于合规或者监管要求，可能必须手动封装密钥。这种情况下，需要重新编译 OpenSSL 以添加对“使用填充的 AES 密钥封装”的支持。

## 导入

1. 创建目标密钥环
2. 创建目标密钥
3. 创建导入作业
4. 检查导入作业的状态
5. 自动封装和导入密钥
6. 导入手动封装的密钥

## 验证

1. 验证密钥材料是否相同
2. 验证对 Cloud HSM 密钥的证明

## References

[1]: https://cloud.google.com/kms/docs/importing-a-key	"将密钥导入到 Cloud KMS 中"
