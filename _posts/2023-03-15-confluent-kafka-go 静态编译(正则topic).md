---
layout:     post
title:      confluent-kafka-go 静态编译
subtitle:   kafka消费
date:       2023-03-15
author:     kuba
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - kafka
    - Go编译
    - goland
    - confluent
    - CGO
    - 静态编译
---

# confluent-kafka-go 静态编译

--
目前golang社区推荐的kafka客户端并不多，我的业务场景中期望在消费组订阅topic时能够通过指定正则规则，能够自动订阅后期新增的topic,测试了几种客户端，最终选择`confluent-kafka-go`,
在编译打包时，我需要将我的服务打成docker镜像，`confluent-kafka-go`存在cgo问题，就不得不解决下静态编译问题了。

---

## 初识
>[confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go) is Confluent's Golang client for Apache Kafka and the Confluent Platform.   

### 案例
先抛出一个简单的消费案例，我希望通过订阅一组相同规则的topic。

```go
package main

import (
	"fmt"
	"time"

	"github.com/childe/gohangout/codec"
	"github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {

	c, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers": "xxx:9092",
		"group.id":          "group-1",
		"security.protocol": "SASL_PLAINTEXT",
		"auto.commit.interval.ms": 100,
		"sasl.mechanism":          "PLAIN",
		"sasl.username":           "user",
		"sasl.password":           "pwd",
	})

	if err != nil {
		panic(err)
	}

	c.SubscribeTopics([]string{"^demo.group.*_log"}, nil)

	// A signal handler or similar could be used to set this to false to break the loop.
	run := true

	for run {
		msg, err := c.ReadMessage(time.Second)
		if err == nil {
			fmt.Printf("Message on %s: %s\n", *msg.TopicPartition.Topic, string(msg.Value))
			decoder := codec.NewDecoder("json")
			val := decoder.Decode(msg.Value)
			fmt.Println(val)
		} else if !err.(kafka.Error).IsTimeout() {
			// The client will automatically try to recover from all errors.
			// Timeout is not considered an error because it is raised by
			// ReadMessage in absence of messages.
			fmt.Printf("Consumer error: %v (%v)\n", err, msg)
		}
	}

	c.Close()
}
```
### 1.尝试关闭CGO编译
由于confluent-kafka-go包基于kafka c库而实现，所以我们没法关闭CGO，如果关闭CGO，将遇到下面编译问题：
```cmd
# CGO_ENABLED=0 go build
#main
./producer.go:15:42: undefined: kafka.ConfigMap
./producer.go:17:29: undefined: kafka.ConfigValue
./producer.go:50:18: undefined: kafka.NewProducer
./producer.go:85:22: undefined: kafka.Message
./producer.go:86:28: undefined: kafka.TopicPartition
./producer.go:86:75: undefined: kafka.PartitionAny
```

### 2.检查动态链接
默认情况依赖confluent-kafka-go包的Go程序会采用动态链接，通过ldd查看编译后的程序结果如下(on CentOS)
```cmd
# make build
# ldd main
linux-vdso.so.1 =&gt;  (0x00007ffcf87ec000)
libm.so.6 =&gt; /lib64/libm.so.6 (0x00007f473d014000)
libdl.so.2 =&gt; /lib64/libdl.so.2 (0x00007f473ce10000)
libpthread.so.0 =&gt; /lib64/libpthread.so.0 (0x00007f473cbf4000)
librt.so.1 =&gt; /lib64/librt.so.1 (0x00007f473c9ec000)
libc.so.6 =&gt; /lib64/libc.so.6 (0x00007f473c61e000)
/lib64/ld-linux-x86-64.so.2 (0x00007f473d316000)
```

### 3.尝试编译
confluent-kafka-go包官方目前确认还不支持静态编译,不过我们可以尝试下：
```cmd
go build  -o consumer-static -ldflags '-linkmode "external" -extldflags "-static"' ./cmd/
```

如果出现报错：
```cmd
/root/.bin/go1.18beta2/pkg/tool/linux_amd64/link: running gcc failed: exit status 1
/usr/bin/ld: 找不到 -lm
/usr/bin/ld: 找不到 -ldl
/usr/bin/ld: 找不到 -lpthread
/usr/bin/ld: 找不到 -lrt
/usr/bin/ld: 找不到 -lpthread
/usr/bin/ld: 找不到 -lc
collect2: 错误：ld 返回 1
```

不慌，手动安装下依赖：
```cmd
yum install glibc-static
```

### 4.重试第三步的编译命令：
```cmd
go build  -o consumer-static -ldflags '-linkmode "external" -extldflags "-static"' ./cmd/
```

编译日志：
```cmd
# xxx/demo/cmd/server
/tmp/go-link-2166829897/000020.o: In function `_goboringcrypto_DLOPEN_OPENSSL':
/usr/lib/golang/src/crypto/internal/boring/goopenssl.h:59: warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2166829897/000062.o: In function `mygetgrouplist':
/usr/lib/golang/src/os/user/getgrouplist_unix.go:18: warning: Using 'getgrouplist' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2166829897/000061.o: In function `mygetgrgid_r':
/usr/lib/golang/src/os/user/cgo_lookup_unix.go:40: warning: Using 'getgrgid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2166829897/000061.o: In function `mygetgrnam_r':
/usr/lib/golang/src/os/user/cgo_lookup_unix.go:45: warning: Using 'getgrnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2166829897/000061.o: In function `mygetpwnam_r':
/usr/lib/golang/src/os/user/cgo_lookup_unix.go:35: warning: Using 'getpwnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2166829897/000061.o: In function `mygetpwuid_r':
/usr/lib/golang/src/os/user/cgo_lookup_unix.go:30: warning: Using 'getpwuid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/tmp/go-link-2166829897/000004.o: In function `_cgo_3c1cec0c9a4e_C2func_getaddrinfo':
/tmp/go-build/cgo-gcc-prolog:58: warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/root/go/pkg/mod/github.com/confluentinc/confluent-kafka-go/v2@v2.0.2/kafka/librdkafka_vendor/librdkafka_glibc_linux_amd64.a(libcrypto-lib-bio_sock.o): In function `BIO_gethostbyname':
(.text+0x51): warning: Using 'gethostbyname' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/root/go/pkg/mod/github.com/confluentinc/confluent-kafka-go/v2@v2.0.2/kafka/librdkafka_vendor/librdkafka_glibc_linux_amd64.a(libcurl_la-hostip4.o): In function `Curl_ipv4_resolve_r':
(.text+0x5e): warning: Using 'gethostbyname_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```

有warn日志，不过编译成功了：
```cmd
#ldd consumer-static
not a dynamic executable
```

### 5.服务打docker镜像是可以正常运行的。

## 番外

如出现报错：
```cmd
unrecognized relocation (0x2a) in section `.text'
```

### ld版本升级
```gcmd
# 下载 2.26.1 版本对应源码压缩包
wget http://ftp.gnu.org/gnu/binutils/binutils-2.26.1.tar.gz

# 解压缩到当前路径
tar xvf binutils-2.26.1.tar.gz 

# 切换工作目录
cd ./binutils-2.26.1

# 通过 configure 生成 makefile 文件，以及设置 make install 时的安装路径
./configure --prefix=/root/binutils-2.26.1/build

#make -j
#make install

# 配置路径到系统环境变量中
#vi /etc/profile
添加一行：
export PATH=/root/binutils-2.26.1/build/bin:$PATH

#source /etc/profile


# ld -v
GNU ld (GNU Binutils) 2.26.1
```


成功！  
好好学习，天天向上！