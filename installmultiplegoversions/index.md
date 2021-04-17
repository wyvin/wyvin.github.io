# 安装多个Go版本

系统：`MacOS`
原有的版本为：`1.13.6`
增加额外版本：`1.16.3`

## 设置代理

直接下载会遇到网络问题，需要设置一下代理
```bash
⇒  export GO111MODULE=on
⇒  export GOPROXY=https://goproxy.cn
```

## 下载

```bash
⇒  go get golang.org/dl/go1.16.3
go: finding golang.org/dl latest
go: downloading golang.org/dl v0.0.0-20210401214017-5e9de8bfb3b7
go: extracting golang.org/dl v0.0.0-20210401214017-5e9de8bfb3b7

⇒  go1.16.3 download
Success. You may now run 'go1.16.3'
```

## 验证
```bash
⇒  go1.16.3 version
go version go1.16.3 darwin/amd64

⇒  go1.16.3 env GOROOT
../sdk/go1.16.3
```
`1.16.3`版本作为sdk添加到go命令中

## 使用
再做个版本差异的验证

在`1.15`版本的特性中：将较小的整数值转换为接口值的过程中不再会引起内存分配
```go
import (
	"testing"
)

func Convert(val int) []interface{} {
	var slice = make([]interface{}, 100)
	for i := 0; i < 100; i++ {
		slice[i] = val
	}

	return slice
}

func BenchmarkConvert(b *testing.B) {
	for i := 0; i < b.N; i++ {
		result := Convert(254)
		_ = result
	}
}
```

用`1.13`运行的结果：内存分配的次数是101次
```bash
⇒  go test -bench . -benchmem
goos: darwin
goarch: amd64
BenchmarkConvert-4  559560    2174 ns/op    2592 B/op       101 allocs/op
PASS
ok      2.434s
```

用`1.16`运行的结果：内存分配的次数是0次
```bash
⇒  go1.16.3 test -bench . -benchmem
goos: darwin
goarch: amd64
BenchmarkConvert-4  4120009     259.9 ns/op     0 B/op      0 allocs/op
PASS
ok      2.000s
```

## 卸载

* 删除`$GOPATH/bin/go1.16.3`
* 删除`/sdk/go1.16.3`
