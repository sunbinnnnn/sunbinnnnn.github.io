---
title: Go并发-并发原语踩坑
date: 2022-07-12 10:38:23
tags:
- 并发
- Golang
---

## RWMutex 的 3 个踩坑点

### **1：不可复制**

不能复制的原因和互斥锁一样。一旦读写锁被使用，它的字段就会记录它当前的一些状态。这个时候你去复制这把锁，就会把它的状态也给复制过来。但是，原来的锁在释放的时候，并不会修改你复制出来的这个读写锁，这就会导致复制出来的读写锁的状态不对，可能永远无法释放锁。

### **2：重入导致死锁**

读写锁因为重入（或递归调用）导致死锁（deadlock）的情况更多。

死锁三种情况：

- 第一种情况：读写锁内部基于互斥锁实现对 writer 的并发访问，而互斥锁本身是有重入问题的，所以，writer 重入调用 Lock 的时候，就会出现死锁的现象。
- 第二种情况：我们知道，有活跃 reader 的时候，writer 会等待，如果我们在 reader 的读操作时调用 writer 的写操作（它会调用 Lock 方法），那么，这个 reader 和 writer 就会形成互相依赖的死锁状态。Reader 想等待 writer 完成后再释放锁，而 writer 需要这个 reader 释放锁之后，才能不阻塞地继续执行。这是一个读写锁常见的死锁场景。
- 第三种情况：当一个 writer 请求锁的时候，如果已经有一些活跃的 reader，它会等待这些活跃的 reader 完成，才有可能获取到锁，但是，如果之后活跃的 reader 再依赖新的 reader 的话，这些新的 reader 就会等待 writer 释放锁之后才能继续执行，这就形成了一个环形依赖： writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer。

```go
func main() {
    var mu sync.RWMutex

    // writer,稍微等待，然后制造一个调用Lock的场景
    go func() {
        time.Sleep(200 * time.Millisecond)
        mu.Lock()
        fmt.Println("Lock")
        time.Sleep(100 * time.Millisecond)
        mu.Unlock()
        fmt.Println("Unlock")
    }()

    go func() {
        factorial(&mu, 10) // 计算10的阶乘, 10!
    }()
    
    select {}
}

// 递归调用计算阶乘
func factorial(m *sync.RWMutex, n int) int {
    if n < 1 { // 阶乘退出条件 
        return 0
    }
    fmt.Println("RLock")
    m.RLock()
    defer func() {
        fmt.Println("RUnlock")
        m.RUnlock()
    }()
    time.Sleep(100 * time.Millisecond)
    return factorial(m, n-1) * n // 递归调用
}
```

factoria 方法是一个递归计算阶乘的方法，我们用它来模拟 reader。为了更容易地制造出死锁场景，在这里加上了 sleep 的调用，延缓逻辑的执行。这个方法会调用读锁（第 27 行），在第 33 行递归地调用此方法，每次调用都会产生一次读锁的调用，所以可以不断地产生读锁的调用，而且必须等到新请求的读锁释放，这个读锁才能释放。

同时，我们使用另一个 goroutine 去调用 Lock 方法，来实现 writer，这个 writer 会等待 200 毫秒后才会调用 Lock，这样在调用 Lock 的时候，factoria 方法还在执行中不断调用 RLock。这两个 goroutine 互相持有锁并等待，谁也不会退让一步，满足了“writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer”的死锁条件，所以就导致了死锁的产生。

### **3：释放未加锁的 RWMutex**

和互斥锁一样，Lock 和 Unlock 的调用总是成对出现的，RLock 和 RUnlock 的调用也必须成对出现。Lock 和 RLock 多余的调用会导致锁没有被释放，可能会出现死锁，而 Unlock 和 RUnlock 多余的调用会导致 panic。在生产环境中出现 panic 是大忌，你总不希望半夜爬起来处理生产环境程序崩溃的问题吧？所以，在使用读写锁的时候，一定要注意，不遗漏不多余。
