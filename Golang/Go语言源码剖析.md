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

### 构造

```go
ch1 :=make(chan int)//无缓冲channel
ch2 :=make(chan int,10)//有缓冲channel
```

### 读写

```go
val :=<- ch
<-ch
val,ok :=<-ch

var data int
ch <- data
```
### 关闭

```go
close(ch)
```
***

### Select

select里的case后面并不带判断条件，而是一个信道的操作，不同于switch里的case，对于从其它语言转过来的开发者来说有些需要特别注意的地方。

golang 的 select 就是监听 IO 操作，当 IO 操作发生时，触发相应的动作每个case语句里必须是一个IO操作，确切的说，应该是一个面向channel的IO操作。

```go
select {
    case <-ch1:
        // 如果从 ch1 信道成功接收数据，则执行该分支代码
    case ch2 <- 1:
        // 如果成功向 ch2 信道成功发送数据，则执行该分支代码
    default:
        // 如果上面都没有成功，则进入 default 分支处理流程
}
```

如果 `ch1` 或者 `ch2` 信道都阻塞的话，就会立即进入 `default` 分支，并不会阻塞。但是如果没有 `default` 语句，则会阻塞直到某个信道操作成功为止

#### Select特性

* select语句只能用于信道的读写操作

* xxxxxxxxxx type hchan struct {  qcount   uint           // chan中的数据个数  dataqsiz uint           // chan中的元素容量  buf      unsafe.Pointer // chan中的元素队列（环形数组）  elemsize uint16 //chan元素类型的大小  closed   uint32 // 标识channel是否关闭  elemtype *_type // 数据 元素类型  sendx    uint   // 写入元素的 index  recvx    uint   // 读取元素的 index  recvq    waitq  // 阻塞的读协程队列  sendq    waitq  // 阻塞的写协程队列    lock mutex  // 锁 }go

* 对于case条件语句中，如果存在信道值为nil的读写操作，则该分支将被忽略，可以理解为从select语句中删除了这个case语句

* 如果有超时条件语句，判断逻辑为如果在这个时间段内一直没有满足条件的case,则执行这个超时case。如果此段时间内出现了可操作的case,则直接执行这个case。一般用超时语句代替了default语句

* 对于空的select{}，会引起死锁

* 对于for中的select{}, 也有可能会引起cpu占用过高的问题

***

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
waitq的底层实现
```golang
//阻塞的协程队列
type waitq struct{
    first *sudog//队列头部
    last *sudog//队列尾部
}
```
sudog的底层  
sudog本质上是对goroutinue的一个再封装
```golang
//用于包装协程的节点
type sudog struct{
    g *g //协程

    next *sudog//队列中的下一个节点
    prev *sudog//队列中的前一个节点
    elem unsafe.Pointer //data element(may point to stack)

    inSelect bool//是否处于select多路复用的模式下

    c *hchan//标识与当前sudog交互的chan
}

```
## 创建channel
当我们在代码里面通过make创建一个channel时，实际调用的是***runtime.makechan(t *chantype, size int)***,

channel的类型有三种：  
1. 无缓冲类型的channel
2. 有缓冲struct类型的channel
3. 有缓冲pointer类型的channel

makechan()实现：
```go
func makechan(t *chantype, size int) *hchan {
  elem := t.elem
  // 判断 元素类型的大小
  if elem.size >= 1<<16 {
    throw("makechan: invalid channel element type")
  }
  // 判断对齐限制
  if hchanSize%maxAlign != 0 || elem.align > maxAlign {
    throw("makechan: bad alignment")
  }
  // 判断 size非负 和 是否大于 maxAlloc限制
  // 缓冲区大小
  mem, overflow := math.MulUintptr(elem.size, uintptr(size))
  if overflow || mem > maxAlloc-hchanSize || size < 0 {
    panic(plainError("makechan: size out of range"))
  }
  var c *hchan
  switch {
  case mem == 0: // 无缓冲区，即 make没设置大小
    c = (*hchan)(mallocgc(hchanSize, nil, true)) 
    c.buf = c.raceaddr()
  case elem.ptrdata == 0:  // 数据类型不包含指针
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
  default:  // 如果包含指针
    // Elements contain pointers.
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
  }
  c.elemsize = uint16(elem.size)
  c.elemtype = elem
  c.dataqsiz = uint(size)
  if debugChan {
    print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
  }
  return c
}

```

## 写流程
### 异常情况处理
此处chansend源码已省略。
```go
func chansend1(c *hchan,elem unsafe.Pointer){
    chansend(c,elem,true,getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {//未初始化的channel，抛出异常死锁
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

    ...

	lock(&c.lock)

	if c.closed != 0 {//往已关闭的channel中写数据
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	...
}

```
1. 对于未初始化的chan，写操作引发死锁
2. 对于已关闭的chan，写操作引发panic

### 写时存在阻塞读协程
存在阻塞读协程，显然表明channel底层的环形数组为空
```go

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	...

	lock(&c.lock)

	...

	if sg := c.recvq.dequeue(); sg != nil {

		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

    ...
}

```
1. 加锁保证并发安全
2. 从阻塞读协程队列中取出一个goroutinue的封装对象sudog
3. 在send方法中，会给予memmove方法，直接将元素拷贝交给sudog对应的goroutinue
4. 在send的方法中会完成解锁动作  

### 写时无阻塞读协程但环形缓冲区仍然有空间
如果写时无阻塞读协程但是环形缓冲区仍然有空间，那么则将对应的数据放入缓冲区即可。
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...

	lock(&c.lock)

	...

	if c.qcount < c.dataqsiz {
	
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++//写数据对应的指针，标明新添加的数据应该写入环形数组的哪一个凹槽之中
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}//环形
		c.qcount++//元素数量
		unlock(&c.lock)//解锁
		return true
	}

	...
}

```
1. 加锁
2. 将当前元素添加到环形缓冲区sendx对应的位置
3. sendx++
4. qcount++
5. 解锁返回true

### 写时无阻塞读协程且环形缓冲区没有空间
将当前的写goroutinue添加到对应的阻塞的写协程队列
``` go

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	
    ...

	lock(&c.lock)

    ...


	gp := getg()//返回当前goroutinue的一个引用
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)//包装节点并且添加到写协程阻塞队列中

	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    //阻塞微停


	KeepAlive(ep)

	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}

	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}

```
1. 加锁
2. 构造封装当前goroutinue的sudog对象
3.  完成指针指向，建立sudog，goroutinue，channel之间的指向关系
4. 把sudog添加到当前的channel的阻塞写协程队列之中
5. park（微停阻塞）当前协程
6. 倘若协程从park中被唤醒，则回收sudog（sudog能被唤醒，则对应的goroutinue一定已经被读协程队列取走）
7. 解锁返回true

### 写总流程
![channel](./Go语言底层剖析/channel写流程 .png)

## 读流程

### 异常情况处理

读空channel
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    if c == nil {
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
     ...
}
```
1. park 挂起，引起死锁；

channel 已关闭且内部无元素
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  
    lock(&c.lock)

    if c.closed != 0 {
        if c.qcount == 0 {
            unlock(&c.lock)
            if ep != nil {
                typedmemclr(c.elemtype, ep)
            }
            return true, false
        }
        // The channel has been closed, but the channel's buffer have data.
    } 

     ...
}

```
直接解锁返回即可

***

### 读时有阻塞的写协程
有阻塞的写协程则channel一定满了
``` go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   
    lock(&c.lock)

    if sg := c.sendq.dequeue(); sg != nil {
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
     }
      ...
}
```
1. 加锁；
2. 从阻塞写协程队列中获取到一个写协程；
3. 倘若 channel 无缓冲区，则直接读取写协程元素，并唤醒写协程；
4. 倘若 channel 有缓冲区，则读取缓冲区头部元素，并将写协程元素写入缓冲区尾部后唤醒写写成；
5. 解锁，返回.

***

### 读时无阻塞写协程且缓冲区有元素
 ```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
     ...
    lock(&c.lock)
     ...
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }
    ...
}
 ```

1. 加锁
2. 获取到recvx对应位置的元素
3. recvx++
4. qcount--
5. 解锁返回true

***

### 读时无阻塞写协程且缓冲区无元素

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
   lock(&c.lock)
    ...
    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    gp.waiting = mysg
    mysg.g = gp
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg)
    atomic.Store8(&gp.parkingOnChan, 1)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

    gp.waiting = nil
    success := mysg.success
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, success
}
```

1. 加锁
2. 构造封装当前的gorountinue的sudog对象
3. 完成指针指向，建立sudog，goroutinue，channel之间的指向关系
4. 把sudog添加到当前的阻塞读协程队列之中
5. park（微停阻塞）当前的协程
6. 倘若协程从park中被唤醒，则回收sudog（因为该协程被唤醒，那么就一定代表有元素写入了环形队列中）
7. 解锁返回

### 读总流程

![channel](./Go语言底层剖析/channel读流程.png)

## 阻塞模式和非阻塞模式

在上述源码分析流程中，均是以阻塞模式为主线进行讲述，忽略非阻塞模式的有关处理逻辑. 此处阐明两个问题：

-  非阻塞模式下，流程逻辑有何区别？
-  何时会进入非阻塞模式？

### 非阻塞模式逻辑区别

非阻塞模式下，读/写 channel 方法通过一个 bool 型的响应参数，用以标识是否读取/写入成功.

- 所有需要使得当前 goroutine 被挂起的操作，在非阻塞模式下都会返回 false；
-  所有使得当前 goroutine 会进入死锁的操作，在非阻塞模式下都会返回 false；
- 所有能立即完成读取/写入操作的条件下，非阻塞模式下会返回 true.

例如如下代码，如果即将使得当前goroutinue被挂起park，切当前为非阻塞模式，那么就会直接return false

```go
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
```

### 何时进入非阻塞模式

默认情况下，读/写 channel 都是阻塞模式，只有在 select 语句组成的多路复用分支中，与 channel 的交互会变成非阻塞模式：

```go
select {
    case <-ch1:
        // 如果从 ch1 信道成功接收数据，则执行该分支代码
    case ch2 <- 1:
        // 如果成功向 ch2 信道成功发送数据，则执行该分支代码
    default:
        // 如果上面都没有成功，则进入 default 分支处理流程
}
```

### 非阻塞模式代码一览

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    return chansend(c, elem, false, getcallerpc())
}

func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
    return chanrecv(c, elem, false)
}
```

在 select 语句包裹的多路复用分支中，读和写 channel 操作会被汇编为 selectnbrecv 和 selectnbsend 方法，底层同样复用 chanrecv 和 chansend 方法，但此时由于第三个入参 block 被设置为 false，导致后续会走进非阻塞的处理分支.

## 两种读channel协议

读取 channel 时，可以根据第二个 bool 型的返回值用以判断当前 channel 是否已处于关闭状态：

```go
ch := make(chan int, 2)
got1 := <- ch
got2,ok := <- ch
```

实现上述功能的原因是，两种格式下，读 channel 操作会被汇编成不同的方法：

```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}
```

如果对某一个chan写入一个0值，然后又要读取这个chan，此时，无论这个channel是否关闭，都能读取到0，如果想要知道channel是被关了，还是真实的有这笔数据，我们需要进行如下处理：

```go
ch<-0

if val,ok:=<-ch;ok{
    //open
}else{
    //close
}
```

## 关闭

closechan执行流程

![channel](./Go语言底层剖析/channel关闭.png)

closechan源码：

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

