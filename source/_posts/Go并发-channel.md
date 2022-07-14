---
title: Go并发-channel
date: 2022-07-11 15:33:14
tags: 
- 并发
- Golang
typora-root-url: ..
---

Channel作为Golang内建的first-clas类型，可以说是每位Go开发者都会用到的技术，也是Go的核心之一，下面我们就围绕Channel来看看他的神奇之处。

<!--more-->

之前我们提到了Golang的一些并发原语：

[Go并发 - 并发原语Mutex、RWMutex](https://blog.neilcloud.net/2022/05/12/Go%E5%B9%B6%E5%8F%91/)

[Go并发-常见并发原语](https://blog.neilcloud.net/2022/07/11/Go%E5%B9%B6%E5%8F%91-%E5%B8%B8%E8%A7%81%E5%B9%B6%E5%8F%91%E5%8E%9F%E8%AF%AD/)

而这些使用Channel几乎都可以实现，但在合适的场景下使用正确的并发原语，仍然至关重要。

## Channel的应用场景



Channel的一个核心思想是：

> Don’t communicate by sharing memory, share memory by communicating.
>
> Go Proverbs by Rob Pike

可以看出在Golang的设计中，Channel主要被用于goroutine之间的通信，而实际上，Channel的扩展应用场景非常之多，典型的几种应用场景：

- **数据交流：**当作并发的 buffer 或者 queue，解决生产者 - 消费者问题。多个 goroutine 可以并发当作生产者（Producer）和消费者（Consumer）。
- **数据传递：**一个 goroutine 将数据交给另一个 goroutine，相当于把数据的拥有权 (引用) 托付出去。
- **信号通知：**一个 goroutine 可以将信号 (closing、closed、data ready 等) 传递给另一个或者另一组 goroutine 。
- **任务编排：**可以让一组 goroutine 按照一定的顺序并发或者串行的执行，这就是编排的功能。
- **锁：**利用 Channel 也可以实现互斥锁的机制。

## Channel的基本用法

Channel的使用频率一般比较高，Go开发者一般对此掌握得比较充分，本文基本用法部分只取一些典型用法，不做详细介绍。

**Channel的类型：**

```go
chan string          // 可以发送接收string
chan<- struct{}      // 只能发送struct{}, <-符号靠左
<-chan int           // 只能从chan接收int
```

**channel的初始化：**

首先，channel是可以不进行初始化的，未初始化的channel零值未nil，可以通过`make`关键字来初始化channel

```go
make(chan int, 9527) // 初始化元素为int类型，容量为9527的channel
make(chan int) // 初始化元素为int类型，容量为0的unbuffered chan
```

**接收数据：**

```go
x := <-ch // 把接收的一条数据赋值给变量x
foo(<-ch) // 把接收的一个的数据作为参数传给函数
<-ch // 丢弃接收的一条数据
```

接收数据时，**可以返回两个值**。

第一个值是返回的 chan 中的元素，

第二个值是 bool 类型，代表是否成功地从 chan 中读取到一个值，如果第二个参数是 false，chan 已经被 close 而且 chan 中没有缓存的数据，这个时候，第一个值是零值。所以，如果从 chan 读取到一个零值，可能是 sender 真正发送的零值，也可能是 closed 的并且没有缓存元素产生的零值。

**内建函数：**

内建函数close、cap、len等都可以操作channel。

close：关闭channel

cap：返回channel的容量

len：返回channel中缓存的还未被取走的元素数量

select case clause：

```go
func main() {
    var ch = make(chan int, 10)
    for i := 0; i < 10; i++ {
        select {
        case ch <- i:
        case v := <-ch:
            fmt.Println(v)
        }
    }
}
```

for-range：

```go
for v := range ch {
   fmt.Println(v)
}
```

清空chan：

```go
for range ch {
}
```

## Channel的内部实现

### **Channel类型的数据结构（[runtime.hchan](https://github.com/golang/go/blob/master/src/runtime/chan.go#L32)）**

<img src="/images/81304c1f1845d21c66195798b6ba48dd-165776877259212.jpg" alt="img" style="zoom:67%;" />

- **qcount：**代表 chan 中已经接收但还没被取走的元素的个数。内建函数 len 可以返回这个字段的值。
- **dataqsiz：**队列的大小。chan 使用一个循环队列来存放元素，循环队列很适合这种生产者 - 消费者的场景。
- **buf：**存放元素的循环队列的 buffer。
- **elemtype 和 elemsize：**chan 中元素的类型和 size。因为 chan 一旦声明，它的元素类型是固定的，即普通类型或者指针类型，所以元素大小也是固定的。
- **sendx：**处理发送数据的指针在 buf 中的位置。一旦接收了新的数据，指针就会加上 elemsize，移向下一个位置。buf 的总大小是 elemsize 的整数倍，而且 buf 是一个循环列表。
- **recvx：**处理接收请求时的指针在 buf 中的位置。一旦取出数据，此指针会移动到下一个位置。
- **recvq：**chan 是多生产者多消费者的模式，如果消费者因为没有数据可读而被阻塞了，就会被加入到 recvq 队列中。
- **sendq：**如果生产者因为 buf 满了而阻塞，会被加入到 sendq 队列中。

### **Channel的初始化**

`makechan`会根据 chan 的容量的大小和元素的类型不同，初始化不同的存储空间

```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem
  
        // 略去检查代码
        mem, overflow := math.MulUintptr(elem.size, uintptr(size))
        
    //
    var c *hchan
    switch {
    case mem == 0:
      // chan的size或者元素的size是0，不必创建buf
      c = (*hchan)(mallocgc(hchanSize, nil, true))
      c.buf = c.raceaddr()
    case elem.ptrdata == 0:
      // 元素不是指针，分配一块连续的内存给hchan数据结构和buf
      c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
            // hchan数据结构后面紧接着就是buf
      c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
      // 元素包含指针，那么单独分配buf
      c = new(hchan)
      c.buf = mallocgc(mem, elem, true)
    }
  
        // 元素大小、类型、容量都记录下来
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    lockInit(&c.lock, lockRankHchan)

    return c
  }
```

### **send**

Go 在编译发送数据给 chan 的时候，会把 send 语句转换成 chansend1 函数，chansend1 函数会调用 chansend。

```go
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
        // 第一部分
    if c == nil {
      if !block {
        return false
      }
      gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
      throw("unreachable")
    }
      ......
  }
```

最开始，第一部分是进行判断：如果 chan 是 nil 的话，就把调用者 goroutine park（阻塞休眠）， 调用者就永远被阻塞住了，所以，第 11 行是不可能执行到的代码。

```go
// 第二部分，如果chan没有被close,并且chan满了，直接返回
  if !block && c.closed == 0 && full(c) {
      return false
}
```

第二部分的逻辑是当你往一个已经满了的 chan 实例发送数据时，并且想不阻塞当前调用，那么这里的逻辑是直接返回。chansend1 方法在调用 chansend 的时候设置了阻塞参数，所以不会执行到第二部分的分支里。

```go
  // 第三部分，chan已经被close的情景
    lock(&c.lock) // 开始加锁
    if c.closed != 0 {
      unlock(&c.lock)
      panic(plainError("send on closed channel"))
  }
```

第三部分显示的是，如果 chan 已经被 close 了，再往里面发送数据的话会 panic。

```
      // 第四部分，从接收队列中出队一个等待的receiver
        if sg := c.recvq.dequeue(); sg != nil {
      // 
      send(c, sg, ep, func() { unlock(&c.lock) }, 3)
      return true
    }
```

第四部分，如果等待队列中有等待的 receiver，那么这段代码就把它从队列中弹出，然后直接把数据交给它（通过 memmove(dst, src, t.size)），而不需要放入到 buf 中，速度可以更快一些。

```go
     // 第五部分，buf还没满
      if c.qcount < c.dataqsiz {
      qp := chanbuf(c, c.sendx)
      if raceenabled {
        raceacquire(qp)
        racerelease(qp)
      }
      typedmemmove(c.elemtype, qp, ep)
      c.sendx++
      if c.sendx == c.dataqsiz {
        c.sendx = 0
      }
      c.qcount++
      unlock(&c.lock)
      return true
    }
```

第五部分说明当前没有 receiver，需要把数据放入到 buf 中，放入之后，就成功返回了。

```
      // 第六部分，buf满。
        // chansend1不会进入if块里，因为chansend1的block=true
        if !block {
      unlock(&c.lock)
      return false
    }
        ......
```

第六部分是处理 buf 满的情况。如果 buf 满了，发送者的 goroutine 就会加入到发送者的等待队列中，直到被唤醒。这个时候，数据或者被取走了，或者 chan 被 close 了。

### **recv**

在处理从 chan 中接收数据时，Go 会把代码转换成 chanrecv1 函数，如果要返回两个返回值，会转换成 chanrecv2，chanrecv1 函数和 chanrecv2 会调用 chanrecv。

```go
  func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
  }
  func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
  }

    func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
        // 第一部分，chan为nil
    if c == nil {
      if !block {
        return
      }
      gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
      throw("unreachable")
    }
```

chanrecv1 和 chanrecv2 传入的 block 参数的值是 true，都是阻塞方式，所以我们分析 chanrecv 的实现的时候，不考虑 block=false 的情况。

第一部分是 chan 为 nil 的情况。和 send 一样，从 nil chan 中接收（读取、获取）数据时，调用者会被永远阻塞。

```
  // 第二部分, block=false且c为空
    if !block && empty(c) {
      ......
    }
```

第二部分block=false，不再chan recv的考虑范畴。

```go
      // 加锁，返回时释放锁
      lock(&c.lock)
      // 第三部分，c已经被close,且chan为空empty
    if c.closed != 0 && c.qcount == 0 {
      unlock(&c.lock)
      if ep != nil {
        typedmemclr(c.elemtype, ep)
      }
      return true, false
    }
```

第三部分是 chan 已经被 close 的情况。如果 chan 已经被 close 了，并且队列中没有缓存的元素，那么返回 true、false（select case能接收，但没有数据（零值））。

```go
    // 第四部分，如果sendq队列中有等待发送的sender
    if sg := c.sendq.dequeue(); sg != nil {
      recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
      return true, true
    }
```

第四部分是处理 buf 满的情况。这个时候，如果是 unbuffer 的 chan，就直接将 sender 的数据复制给 receiver，否则就从队列头部读取一个值，并把这个 sender 的值加入到队列尾部。

```go
    // 第五部分, 没有等待的sender, buf中有数据
    if c.qcount > 0 {
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

    if !block {
      unlock(&c.lock)
      return false, false
    }

        // 第六部分， buf中没有元素，阻塞
        ......
```

第五部分是处理没有等待的 sender 的情况。这个是和 chansend 共用一把大锁，所以不会有并发的问题。如果 buf 有元素，就取出一个元素给 receiver。

第六部分是处理 buf 中没有元素的情况。如果没有元素，那么当前的 receiver 就会被阻塞，直到它从 sender 中接收了数据，或者是 chan 被 close，才返回。

### **close**

通过 close 函数，可以把 chan 关闭，编译器会替换成 closechan 方法的调用。

```go
func closechan(c *hchan) {
    if c == nil { // chan为nil, panic
      panic(plainError("close of nil channel"))
    }
  
    lock(&c.lock)
    if c.closed != 0 {// chan已经closed, panic
      unlock(&c.lock)
      panic(plainError("close of closed channel"))
    }

    c.closed = 1  

    var glist gList

    // 释放所有的reader
    for {
      sg := c.recvq.dequeue()
      ......
      gp := sg.g
      ......
      glist.push(gp)
    }
  
    // 释放所有的writer (它们会panic)
    for {
      sg := c.sendq.dequeue()
      ......
      gp := sg.g
      ......
      glist.push(gp)
    }
    unlock(&c.lock)
  
    for !glist.empty() {
      gp := glist.pop()
      gp.schedlink = 0
      goready(gp, 3)
    }
}
```

如果 chan 为 nil，close 会 panic；如果 chan 已经 closed，再次 close 也会 panic。否则的话，如果 chan 不为 nil，chan 也没有 closed，就把等待队列中的 sender（writer）和 receiver（reader）从队列中全部移除并唤醒。



Channel 并不是处理并发问题的“银弹”，有时候使用并发原语更简单，而且不容易出错。可以根据实际业务情况进行选择:

- **共享资源的并发访问使用传统并发原语；**
- **复杂的任务编排和消息传递使用 Channel；**
- **消息通知机制使用 Channel，除非只想 signal 一个 goroutine，才使用 Cond；**
- **简单等待所有任务的完成用 WaitGroup，也有 Channel 的推崇者用 Channel，都可以；**
- **需要和 Select 语句结合，使用 Channel；**
- **需要和超时配合时，使用 Channel 和 Context。**

### **不同状态的channel在recv、send和close操作下的情况**

![channel2](/images/channel2.png)
