---
title: Go_Redis
date: 2023-07-02 19:18:05
tags:
- golang
- redis
mathjax: true
category: 
- golang
- redis
---

# Go_Redis入门

之前是已经学习完了redis的基本用法，现在开始学习go中的一款redis框架用于操作redis。在使用之前当然得先准备好redis环境，这里就不多做赘述。

本文大多数内容参自go-redis的官方文档

## 安装

Go 社区中目前有很多成熟的 redis client 库，比如[https://github.com/gomodule/redigo 和https://github.com/go-redis/redis，可以自行选择适合自己的库。

```go
go get github.com/go-redis/redis/v8
```

## 连接

### 普通连接

goredis使用redis.NewClient函数连接Redis服务器

```go
func main() {
	red := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,
	})
	fmt.Println(red)
	fmt.Println("test_______GoRedis")
}
```

redis.Options是redis配置文件，其定义如下：

```golang
type Options struct {
	// The network type, either tcp or unix.
	// Default is tcp.
	Network string
	// host:port address.
	Addr string

	// Dialer creates new network connection and has priority over
	// Network and Addr options.
	Dialer func(ctx context.Context, network, addr string) (net.Conn, error)

	// Hook that is called when new connection is established.
	OnConnect func(ctx context.Context, cn *Conn) error

	// Use the specified Username to authenticate the current connection
	// with one of the connections defined in the ACL list when connecting
	// to a Redis 6.0 instance, or greater, that is using the Redis ACL system.
	Username string
	// Optional password. Must match the password specified in the
	// requirepass server configuration option (if connecting to a Redis 5.0 instance, or lower),
	// or the User Password when connecting to a Redis 6.0 instance, or greater,
	// that is using the Redis ACL system.
	Password string

	// Database to be selected after connecting to the server.
	DB int

	// Maximum number of retries before giving up.
	// Default is 3 retries; -1 (not 0) disables retries.
	MaxRetries int
	// Minimum backoff between each retry.
	// Default is 8 milliseconds; -1 disables backoff.
	MinRetryBackoff time.Duration
	// Maximum backoff between each retry.
	// Default is 512 milliseconds; -1 disables backoff.
	MaxRetryBackoff time.Duration

	// Dial timeout for establishing new connections.
	// Default is 5 seconds.
	DialTimeout time.Duration
	// Timeout for socket reads. If reached, commands will fail
	// with a timeout instead of blocking. Use value -1 for no timeout and 0 for default.
	// Default is 3 seconds.
	ReadTimeout time.Duration
	// Timeout for socket writes. If reached, commands will fail
	// with a timeout instead of blocking.
	// Default is ReadTimeout.
	WriteTimeout time.Duration

	// Type of connection pool.
	// true for FIFO pool, false for LIFO pool.
	// Note that fifo has higher overhead compared to lifo.
	PoolFIFO bool
	// Maximum number of socket connections.
	// Default is 10 connections per every available CPU as reported by runtime.GOMAXPROCS.
	PoolSize int
	// Minimum number of idle connections which is useful when establishing
	// new connection is slow.
	MinIdleConns int
	// Connection age at which client retires (closes) the connection.
	// Default is to not close aged connections.
	MaxConnAge time.Duration
	// Amount of time client waits for connection if all connections
	// are busy before returning an error.
	// Default is ReadTimeout + 1 second.
	PoolTimeout time.Duration
	// Amount of time after which client closes idle connections.
	// Should be less than server's timeout.
	// Default is 5 minutes. -1 disables idle timeout check.
	IdleTimeout time.Duration
	// Frequency of idle checks made by idle connections reaper.
	// Default is 1 minute. -1 disables idle connections reaper,
	// but idle connections are still discarded by the client
	// if IdleTimeout is set.
	IdleCheckFrequency time.Duration

	// Enables read only queries on slave nodes.
	readOnly bool

	// TLS Config to use. When set TLS will be negotiated.
	TLSConfig *tls.Config
	// Limiter interface used to implemented circuit breaker or rate limiter.
	Limiter Limiter
}
```

也可以使用redis.ParseURL解析url参数来获取opt，再以此连接redis，其基本格式为：

```go
"redis://<user>:<pass>@localhost:6379/<db>"
```

```go
func main() {
	opt,err:=redis.ParseURL("redis://:@localhost:6379/0")
	if err!=nil{
		log.Fatal(err)
	}
	rdb := redis.NewClient(opt)
	fmt.Println(rdb)
}
```

另外还有TLS，SSH方式，但是我计网基础不行，理解不了，后续我把计算机网络相关知识搞了再来整整吧。

## Context 上下文

go-redis 支持 Context，你可以使用它控制 [超时](https://redis.uptrace.dev/zh/guide/go-redis-debugging.html#timeouts) 或者传递一些数据, 也可以 [监控](https://redis.uptrace.dev/zh/guide/go-redis-monitoring.html) go-redis 性能。

```go
ctx := context.Background()
```

## 执行 Redis 命令

执行 Redis 命令:

```go
val, err := rdb.Get(ctx, "key").Result()
fmt.Println(val)
```

你也可以分别访问值和错误：

```go
get := rdb.Get(ctx, "key")
fmt.Println(get.Val(), get.Err())
```

```go
func main() {
	ctx := context.Background()
	opt, err := redis.ParseURL("redis://:@localhost:6379/0")
	if err != nil {
		log.Fatal(err)
	}
    
	rdb := redis.NewClient(opt)
	val, err := rdb.Get(ctx, "k1").Result()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(rdb)
	fmt.Println(val)
}

//output:
//Redis<localhost:6379 db:0>
//v1
```

## 不支持的命令

可以使用 `Do()` 方法执行尚不支持或者任意命令:

```go
val, err := rdb.Do(ctx, "get", "key").Result()
if err != nil {
	if err == redis.Nil {
		fmt.Println("key does not exists")
		return
	}
	panic(err)
}
fmt.Println(val.(string))
```

`Do()` 方法返回 [Cmd在新窗口打开](https://pkg.go.dev/github.com/redis/go-redis/v9#Cmd) 类型，你可以使用它获取你想要的类型：

```go
val, err := rdb.Do(ctx, "get", "key").Text()
fmt.Println(val, err)
```

方法列表:

```go
s, err := cmd.Text()
flag, err := cmd.Bool()

num, err := cmd.Int()
num, err := cmd.Int64()
num, err := cmd.Uint64()
num, err := cmd.Float32()
num, err := cmd.Float64()

ss, err := cmd.StringSlice()
ns, err := cmd.Int64Slice()
ns, err := cmd.Uint64Slice()
fs, err := cmd.Float32Slice()
fs, err := cmd.Float64Slice()
bs, err := cmd.BoolSlice()
```

例如想要使用redis的HGET命令就可以这么传

```go
	ctx1 := context.Background()
	val1, err := rdb.Do(ctx1, "HGET", "redisHash", "f1").Text()
	if err != nil {
		log.Fatal(err)
	}
//output:v1
```

## redis.Nil

`redis.Nil` 是一种特殊的错误，严格意义上来说它并不是错误，而是代表一种状态，例如你使用 Get 命令获取 key 的值，当 key 不存在时，返回 `redis.Nil`。在其他比如 `BLPOP` 、 `ZSCORE` 也有类似的响应，你需要区分错误：

```go
val, err := rdb.Get(ctx, "key").Result()
switch {
case err == redis.Nil:
	fmt.Println("key不存在")
case err != nil:
	fmt.Println("错误", err)
case val == "":
	fmt.Println("值是空字符串")
}
```

## Conn

redis.Conn 是从连接池中取出的单个连接，除非你有特殊的需要，否则尽量不要使用它。你可以使用它向 redis 发送任何数据并读取 redis 的响应，当你使用完毕时，应该把它返回给 go-redis，否则连接池会永远丢失一个连接。

```go
cn := rdb.Conn(ctx)
defer cn.Close()

if err := cn.ClientSetName(ctx, "myclient").Err(); err != nil {
	panic(err)
}

name, err := cn.ClientGetName(ctx).Result()
if err != nil {
	panic(err)
}
fmt.Println("client name", name)
```