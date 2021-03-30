# crypto解析



目录结构

```
crypto/
├── blake2b              // blake2b算法
├── bls12381             // bls12381算法
├── bn256                // bn256算法
├── crypto.go
├── crypto_test.go
├── ecies 
├── secp256k1            // 使用了比特币的C语言库libsecp256k1, 使用cgo调用c语言库
├── signature_cgo.go     // 签名 cgo
├── signature_nocgo.go   // 非cgo, 使用了btcsuite/btcd库
├── signature_test.go
└── signify
```


