---
title: Go语言底层剖析
mathjax: true
category: 
- Golang
---

本文主要介绍slice channel 和map等golang中比较重要的数据结构底层

# Channel
## channel基本概念

在Go语言中，channel是一种类型，它可以用来在协程之间传递数据。channel类型的定义如下：

```golang
 chan T
```

构造：
```go
ch1 :=make(chan int)//无缓冲channel
ch2 :=make(chan int,10)//有缓冲channel
```

读写：
```go
val :=<- ch
<-ch
val,ok :=<-ch

var data int
ch <- data
```
关闭：
```go
close(ch)
```
select 是 Go 中的一个控制结构。select 语句类似于 switch 语句，但是select会随机执行一个可运行的case。如果没有case可运行，它将阻塞，直到有case可运行。

select是Golang在语言层面提供的多路IO复用的机制，其可以检测多个channel是否ready(即是否可读或可写)，使用起来非常方便。
```go
select {
    case communication clause:
        statement(s);
    case communication clause:
        statement(s);
    default:
        statement(s);
}
```

### 特性
Go语言中的channel具有以下几个特性：

***线程安全***  
channel是线程安全的，多个协程可以同时读写一个channel，而不会发生数据竞争的问题。这是因为Go语言中的channel内部实现了锁机制，保证了多个协程之间对channel的访问是安全的。

***阻塞式发送和接收*** 
当一个协程向一个channel发送数据时，如果channel已经满了，发送操作会被阻塞，直到有其他协程从channel中取走了数据。同样地，当一个协程从一个channel中接收数据时，如果channel中没有数据可供接收，接收操作会被阻塞，直到有其他协程向channel中发送了数据。这种阻塞式的机制可以保证协程之间的同步和通信。

***顺序性***  
通过channel发送的数据是按照发送的顺序进行排列的。也就是说，如果协程A先向channel中发送了数据x，而协程B再向channel中发送了数据y，那么从channel中接收数据时，先接收到的一定是x，后接收到的一定是y。

***可以关闭***  
通过关闭channel可以通知其他协程这个channel已经不再使用了。关闭一个channel之后，其他协程仍然可以从中接收数据，但是不能再向其中发送数据了。关闭channel的操作可以避免内存泄漏等问题。

***缓冲区大小***  
channel可以带有一个缓冲区，用于存储一定量的数据。如果缓冲区已经满了，发送操作会被阻塞，直到有其他协程从channel中取走了数据；如果缓冲区已经空了，接收操作会被阻塞，直到有其他协程向channel中发送了数据。缓冲区的大小可以在创建channel时指定，例如：

## channel底层数据结构
chan的实现在runtime/chan.go下，是一个hchan的结构体：
```go
type hchan struct {
  qcount   uint           // chan中的数据个数
  dataqsiz uint           // chan中的元素容量
  buf      unsafe.Pointer // chan中的元素队列（环形数组）
  elemsize uint16 //chan元素类型的大小
  closed   uint32 // 标识channel是否关闭
  elemtype *_type // 数据 元素类型
  sendx    uint   // 写入元素的 index
  recvx    uint   // 读取元素的 index
  recvq    waitq  // 阻塞的读协程队列
  sendq    waitq  // 阻塞的写协程队列
  
  lock mutex  // 锁 
}
```
