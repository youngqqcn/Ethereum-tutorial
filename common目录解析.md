


```
common/
├── big.go
├── bitutil   位操作相关
│   ├── bitutil.go
│   ├── bitutil_test.go
│   ├── compress.go
│   └── compress_test.go
├── bytes.go
├── bytes_test.go
├── compiler   // solidity 和 vyper 编译器相关
│   ├── helpers.go
│   ├── solidity.go
│   ├── solidity_test.go
│   ├── test_bad.v.py
│   ├── test.v.py
│   ├── vyper.go
│   └── vyper_test.go
├── debug.go
├── fdlimit   // 系统fd limit相关
│   ├── fdlimit_bsd.go
│   ├── fdlimit_darwin.go
│   ├── fdlimit_test.go
│   ├── fdlimit_unix.go
│   └── fdlimit_windows.go
├── format.go
├── hexutil  // 十六进制处理
│   ├── hexutil.go
│   ├── hexutil_test.go
│   ├── json_example_test.go
│   ├── json.go
│   └── json_test.go
├── math   // 大数
│   ├── big.go
│   ├── big_test.go
│   ├── integer.go
│   └── integer_test.go
├── mclock  // 定时器
│   ├── mclock.go
│   ├── simclock.go
│   └── simclock_test.go
├── path.go   // 路径
├── prque   // 优先级队列
│   ├── lazyqueue.go
│   ├── lazyqueue_test.go
│   ├── prque.go
│   ├── prque_test.go
│   ├── sstack.go
│   └── sstack_test.go
├── size.go
├── size_test.go
├── test_utils.go
├── types.go  // 一些类型定义, 如Address, Hash, 以及一些工具函数
└── types_test.go

```