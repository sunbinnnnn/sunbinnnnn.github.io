---
title: Go并发 - 并发原语Mutex、RWMutex
date: 2022-05-12 15:11:36
tags: 
- 并发
- Golang
---



并发问题是在大型项目开发过程中不可绕过的问题，而Go以其对并发的性能优异而得名，那Golang对并发场景的优势以及对并发问题的解决原理是如何的呢？

<!--more-->

## 并发原语

原语一般是指内核提供给核外调用的过程或者函数成为原语（primitive），原语在执行过程中不允许中断,而并发原语一般是指原语的并发实现。

Golang作为主打并发编程的语言，在其设计中，也存在一系列的并发原语来解决并发编程中资源访问、线程交互等并发问题。

### Golang并发原语：

#### **Mutex**

Mutex即Go中最基本的互斥锁（排他锁）实现，在并发编程中，我们常常需要保障一组内存空间（资源）在同一时间有且仅有一个对象在访问或操作，这一组内存空间（资源）一般被称为**临界区**。

使用互斥锁可以限制临界区内只能由一个线程访问。

##### **Mutex（Locker）接口包含方法**

```go
type Locker interface {
    Lock()
    Unlock()
}
```

Lock：进入临界区时调用

Unlock：退出临界区时调用

##### **Mutex使用方法**

Mutex可以直接在代码中进行调用，而无需初始化
```go
package main

import (
        "fmt"
        "sync"
    )

func main() {
    // 互斥锁保护计数器
    var mu sync.Mutex
    // 计数器的值
    var count = 0
    
    // 辅助变量，用来确认所有的goroutine都完成
    var wg sync.WaitGroup
    wg.Add(10)

    // 启动10个gourontine
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 累加10万次
            for j := 0; j < 100000; j++ {
                mu.Lock()
                count++
                mu.Unlock()
            }
        }()
    }
    wg.Wait()
    fmt.Println(count)
}
```

很多情况下，Mutex会与其他的struct同时出现：

```go
type Counter struct {
    mu    sync.Mutex // 一般将Mutex放在需要控制的字段上面
    Count uint64
}

func (f *Counter) Bar() {
    f.mu.Lock()
    defer f.mu.Unlock()


    if f.count < 1000 {
        f.count += 3
        return
    }


    f.count++
    return
}
```

##### **Mutex内部实现**

虽然其他语言同样有关于Mutex的实现，这里我们只关注Go的Mutex实现。

Mutex的架构演进过程：

初版（完全依赖CAS） ->  给新人机会（新的goroutine也能有机会竞争锁） -> 多给些机会（新来的和被唤醒的有更多的机会竞争锁） -> 解决饥饿问题（解决竞争问题，不会让goroutine长久等待）

###### **初版**

完全依赖于CAS（compare-and-swap）。这里简单描述下CAS指令的过程，CAS指令将**给定的值**和**内存地址中的值**进行比较，如果是同一个值，就使用新值替换内存中地址中的值，**如果同时有其他线程已经修改了这个值，CAS返回失败，返回false**，不难看出，CAS是一个原子（atomic）操作。

初版实现：

```go

   // CAS操作，当时还没有抽象出atomic包
    func cas(val *int32, old, new int32) bool
    func semacquire(*int32)
    func semrelease(*int32)
    // 互斥锁的结构，包含两个字段
    type Mutex struct {
        key  int32 // 锁是否被持有的标识
        sema int32 // 信号量专用，用以阻塞/唤醒goroutine
    }
    
    // 保证成功在val上增加delta的值
    func xadd(val *int32, delta int32) (new int32) {
        for {
            v := *val
            if cas(val, v, v+delta) {
                return v + delta
            }
        }
        panic("unreached")
    }
    
    // 请求锁
    func (m *Mutex) Lock() {
        if xadd(&m.key, 1) == 1 { //标识加1，如果等于1，成功获取到锁
            return
        }
        semacquire(&m.sema) // 否则阻塞等待
    }
    
    func (m *Mutex) Unlock() {
        if xadd(&m.key, -1) == 0 { // 将标识减去1，如果等于0，则没有其它等待者
            return
        }
        semrelease(&m.sema) // 唤醒其它阻塞的goroutine
    }    
```

Mutex结构体包含两个字段：

**字段 key**：是一个 flag，用来标识这个排外锁是否被某个 goroutine 所持有，如果 key 大于等于 1，说明这个排外锁已经被持有；

**字段 sema**：是个信号量变量，用来控制等待 goroutine 的阻塞休眠和唤醒。

调用 Lock 请求锁的时候，通过 xadd 方法进行 CAS 操作（第 25行），xadd 方法通过循环执行 CAS 操作直到成功，保证对 key 加 1 的操作成功完成。如果比较幸运，锁没有被别的 goroutine 持有，那么，Lock 方法成功地将 key 设置为 1，这个 goroutine 就持有了这个锁；如果锁已经被别的 goroutine 持有了，那么，当前的 goroutine 会把 key 加 1，而且还会调用 semacquire 方法（第 28 行），使用信号量将自己休眠，等锁释放的时候，信号量会将它唤醒。

持有锁的 goroutine 调用 Unlock 释放锁时，它会将 key 减 1（第 32行）。如果当前没有其它等待这个锁的 goroutine，这个方法就返回了。但是，如果还有等待此锁的其它 goroutine，那么，它会调用 semrelease 方法（第 35 行），利用信号量唤醒等待锁的其它 goroutine 中的一个。

这里需要注意的一点是：

<u>**Unlock 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。这是因为，Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，Unlock 也不会对此进行检查。Mutex 的这个设计一直保持至今。**</u>

这种设计比较简单，但如果其他goroutine提前释放了自己的锁，在临界区的goroutine可能不知道自己的锁已经释放了，会带来data race问题。在使用Mutex要遵循<u>**谁申请，谁释放**</u>的原则。

###### **给新人机会**

Mutex发展的第二阶段，对Mutex进行了一次大的调整：

```go

   type Mutex struct {
        state int32
        sema  uint32
    }


    const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken
        mutexWaiterShift = iota
    )
```

这里Mutex的第一个字段从`key`改为了`state`。

state是一个复合字段，一个字段包含多个意义，这样可以通过尽可能少的内存来实现互斥锁。这个字段的第一位（最小的一位）来表示这个锁是否被持有，第二位代表是否有唤醒的 goroutine，剩余的位数代表的是等待此锁的 goroutine 数。所以，state 这一个字段被分成了三部分，代表三个数据：

mutexWaiters（阻塞等待的waiter数量） -- mutexWorken（唤醒标记） -- mutexLocked（持有锁的标记）

主要实现逻辑：

```go
func (m *Mutex) Lock() {
        // Fast path: 幸运case，能够直接获取到锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }

        awoke := false
        for {
            old := m.state
            new := old | mutexLocked // 新状态加锁
            if old&mutexLocked != 0 {
                new = old + 1<<mutexWaiterShift //等待者数量加一
            }
            if awoke {
                // goroutine是被唤醒的，
                // 新状态清除唤醒标志
                new &^= mutexWoken
            }
            if atomic.CompareAndSwapInt32(&m.state, old, new) {//设置新状态
                if old&mutexLocked == 0 { // 锁原状态未加锁
                    break
                }
                runtime.Semacquire(&m.sema) // 请求信号量
                awoke = true
            }
        }
    }
```

首先是通过 CAS 检测 state 字段中的标志（第 3 行），如果没有 goroutine 持有锁，也没有等待持有锁的 goroutine，那么，当前的 goroutine 就很幸运，可以直接获得锁。如果state不是零值，那就存在一定的竞争关系，此时当前goroutine会进行休眠，当锁被释放后被唤醒。

但当被唤醒时，并不能直接获取到锁，此时需要与waiter进行竞争（for 循环是不断尝试获取锁，如果获取不到，就通过 runtime.Semacquire(&m.sema) 休眠，休眠醒来之后 awoke 置为 true，尝试争抢锁。），**这点是与初版设计不一样的地方，这会给后来请求锁的goroutine一些机会，也让 CPU 中正在执行的 goroutine 有更多的机会获取到锁，在一定程度上提高了程序的性能。**

代码中的第 10 行将当前的 flag 设置为加锁状态，如果能成功地通过 CAS 把这个新值赋予 state（第 19 行和第 20 行），就代表抢夺锁的操作成功了。

由于涉及到对单个值的位操作，释放锁的逻辑也会相对复杂：

```go
func (m *Mutex) Unlock() {
        // Fast path: drop lock bit.
        new := atomic.AddInt32(&m.state, -mutexLocked) //去掉锁标志
        if (new+mutexLocked)&mutexLocked == 0 { //本来就没有加锁
            panic("sync: unlock of unlocked mutex")
        }
    
        old := new
        for {
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 { // 没有等待者，或者有唤醒的waiter，或者锁原来已加锁
                return
            }
            new = (old - 1<<mutexWaiterShift) | mutexWoken // 新状态，准备唤醒goroutine，并设置唤醒标志
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime.Semrelease(&m.sema)
                return
            }
            old = m.state
        }
    }
```

第 3 行是尝试将持有锁的标识设置为未加锁的状态，这是通过减 1 而不是将标志位置零的方式实现。

第 4 到 6 行还会检测原来锁的状态是否已经未加锁的状态，如果是 Unlock 一个未加锁的 Mutex 会直接 panic。不过，即使将加锁置为未加锁的状态，这个方法也不能直接返回，还需要一些额外的操作，因为还可能有一些等待这个锁的 goroutine（有时候我也把它们称之为 waiter）需要通过信号量的方式唤醒它们中的一个。

所以接下来的逻辑有两种情况。

- 第一种情况（无waiter竞争），如果没有其它的 waiter，说明对这个锁的竞争的 goroutine 只有一个，那就可以直接返回了；如果这个时候有唤醒的 goroutine，或者是又被别人加了锁，那么，无需我们操劳，其它 goroutine 自己干得都很好，当前的这个 goroutine 就可以放心返回了。
- 第二种情况（存在waiter），如果有等待者，并且没有唤醒的 waiter，那就需要唤醒一个等待的 waiter。在唤醒之前，需要将 waiter 数量减 1，并且将 mutexWoken 标志设置上，这样，Unlock 就可以返回了。

###### **多给些机会**

在这次的改动中，Golang工程师加入了[自旋](https://github.com/golang/go/blob/846dce9d05f19a1f53465e62a304dea21b99f910/src/runtime/proc.go#L5580)（spin，通过循环不断尝试）的特性。

如果信赖的goroutine或者是被唤醒的 goroutine 首次获取不到锁，它们就会通过自旋的方式，尝试检查锁是否被释放。在尝试一定的自旋次数后，再执行原来的逻辑。

```go
func (m *Mutex) Lock() {
        // Fast path: 幸运之路，正好获取到锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }

        awoke := false
        iter := 0
        for { // 不管是新来的请求锁的goroutine, 还是被唤醒的goroutine，都不断尝试请求锁
            old := m.state // 先保存当前锁的状态
            new := old | mutexLocked // 新状态设置加锁标志
            if old&mutexLocked != 0 { // 锁还没被释放
                if runtime_canSpin(iter) { // 还可以自旋
                    if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                        atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                        awoke = true
                    }
                    runtime_doSpin()
                    iter++
                    continue // 自旋，再次尝试请求锁
                }
                new = old + 1<<mutexWaiterShift
            }
            if awoke { // 唤醒状态
                if new&mutexWoken == 0 {
                    panic("sync: inconsistent mutex state")
                }
                new &^= mutexWoken // 新状态清除唤醒标记
            }
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                if old&mutexLocked == 0 { // 旧状态锁已释放，新状态成功持有了锁，直接返回
                    break
                }
                runtime_Semacquire(&m.sema) // 阻塞等待
                awoke = true // 被唤醒
                iter = 0
            }
        }
    }
```

如果可以 spin 的话，第 9 行的 for 循环会重新检查锁是否释放。对于临界区代码执行非常短的场景来说，这是一个非常好的优化。因为**临界区的代码耗时很短，锁很快就能释放**，而抢夺锁的 goroutine 不用通过休眠唤醒方式等待调度，直接 spin 几次，可能就获得了锁。

###### **解决饥饿问题**

在之前的几代Mutex优化中，考虑的主要是为“新来的goroutine”分配临界区的占有权，而在极端情况下，很可能出现**等待中的goroutine一直获取不到锁的情况，出现饥饿问题**。

最新的Mutex实现：

```go

   type Mutex struct {
        state int32
        sema  uint32
    }
    
    const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken
        mutexStarving // 从state字段中分出一个饥饿标记
        mutexWaiterShift = iota
    
        starvationThresholdNs = 1e6
    )
    
    func (m *Mutex) Lock() {
        // Fast path: 幸运之路，一下就获取到了锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }
        // Slow path：缓慢之路，尝试自旋竞争或饥饿状态下饥饿goroutine竞争
        m.lockSlow()
    }
    
    func (m *Mutex) lockSlow() {
        var waitStartTime int64
        starving := false // 此goroutine的饥饿标记
        awoke := false // 唤醒标记
        iter := 0 // 自旋次数
        old := m.state // 当前的锁的状态
        for {
            // 锁是非饥饿状态，锁还没被释放，尝试自旋
            if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                old = m.state // 再次获取锁的状态，之后会检查是否锁被释放了
                continue
            }
            new := old
            if old&mutexStarving == 0 {
                new |= mutexLocked // 非饥饿状态，加锁
            }
            if old&(mutexLocked|mutexStarving) != 0 {
                new += 1 << mutexWaiterShift // waiter数量加1
            }
            if starving && old&mutexLocked != 0 {
                new |= mutexStarving // 设置饥饿状态
            }
            if awoke {
                if new&mutexWoken == 0 {
                    throw("sync: inconsistent mutex state")
                }
                new &^= mutexWoken // 新状态清除唤醒标记
            }
            // 成功设置新状态
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 原来锁的状态已释放，并且不是饥饿状态，正常请求到了锁，返回
                if old&(mutexLocked|mutexStarving) == 0 {
                    break // locked the mutex with CAS
                }
                // 处理饥饿状态

                // 如果以前就在队列里面，加入到队列头
                queueLifo := waitStartTime != 0
                if waitStartTime == 0 {
                    waitStartTime = runtime_nanotime()
                }
                // 阻塞等待
                runtime_SemacquireMutex(&m.sema, queueLifo, 1)
                // 唤醒之后检查锁是否应该处于饥饿状态
                starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
                old = m.state
                // 如果锁已经处于饥饿状态，直接抢到锁，返回
                if old&mutexStarving != 0 {
                    if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                        throw("sync: inconsistent mutex state")
                    }
                    // 有点绕，加锁并且将waiter数减1
                    delta := int32(mutexLocked - 1<<mutexWaiterShift)
                    if !starving || old>>mutexWaiterShift == 1 {
                        delta -= mutexStarving // 最后一个waiter或者已经不饥饿了，清除饥饿标记
                    }
                    atomic.AddInt32(&m.state, delta)
                    break
                }
                awoke = true
                iter = 0
            } else {
                old = m.state
            }
        }
    }
    
    func (m *Mutex) Unlock() {
        // Fast path: drop lock bit.
        new := atomic.AddInt32(&m.state, -mutexLocked)
        if new != 0 {
            m.unlockSlow(new)
        }
    }
    
    func (m *Mutex) unlockSlow(new int32) {
        if (new+mutexLocked)&mutexLocked == 0 {
            throw("sync: unlock of unlocked mutex")
        }
        if new&mutexStarving == 0 {
            old := new
            for {
                if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                    return
                }
                new = (old - 1<<mutexWaiterShift) | mutexWoken
                if atomic.CompareAndSwapInt32(&m.state, old, new) {
                    runtime_Semrelease(&m.sema, false, 1)
                    return
                }
                old = m.state
            }
        } else {
            runtime_Semrelease(&m.sema, true, 1)
        }
    }
```

与之前的实现相比，当前的 Mutex 最重要的变化，就是增加饥饿模式。第 123行将饥饿模式的最大等待时间阈值设置成了 1 毫秒，这就意味着，一旦等待者等待的时间超过了这个阈值，Mutex 的处理就有可能进入饥饿模式。

###### **饥饿模式 vs 正常模式**

请求锁时调用的 Lock 方法中一开始是 fast path，这是一个幸运的场景，当前的 goroutine 幸运地获得了锁，没有竞争，直接返回，否则就进入了 lockSlow 方法。

在lockSlow方法下就会进行正常模式与饥饿模式的切换。

正常模式下，waiter 都是进入先入先出队列，被唤醒的 waiter 并不会直接持有锁，而是要和新来的 goroutine 进行竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。

在饥饿模式下，Mutex 的拥有者将**直接把锁交给队列最前面的 waiter**。新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin，它会乖乖地加入到等待队列的尾部。如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正常模式:

- 此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；
- 此 waiter 的等待时间小于 1 毫秒。

正常模式拥有更好的性能，因为即使有等待抢锁的 waiter，goroutine 也可以连续多次获取到锁。

饥饿模式是对公平性和性能的一种平衡，它避免了某些 goroutine 长时间的等待锁。在饥饿模式下，优先对待的是那些一直在等待的 waiter。

综上，则是Mutex整个发展过程，可以看出Mutex设计者的一个核心理念： 在**Mutex的设计中，绝不容忍一个goroutine被落下，尽可能地让等待时间较长的goroutine更有机会获取到锁。**

#### **RWMutex**

Mutex已经能够为我们保证临界区资源的并发安全，但相对得牺牲了并发性能，为此，我们需要引入“读写分离”的概念，把并发中的写和读单独抽出考虑。

##### **RWMutex接口包含方法**

**Lock/Unlock**：写操作时调用的方法。如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。

**RLock/RUnlock**：读操作时调用的方法。如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。

**RLocker**：这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

##### **Mutex使用方法**

RWMutex和Mutex一样，零值是未加锁的状态，当使用时，可以嵌入到其他strcut中，不必显示初始化。

```go
func main() {
    var counter Counter
    for i := 0; i < 10; i++ { // 10个reader
        go func() {
            for {
                counter.Count() // 计数器读操作
                time.Sleep(time.Millisecond)
            }
        }()
    }

    for { // 一个writer
        counter.Incr() // 计数器写操作
        time.Sleep(time.Second)
    }
}
// 一个线程安全的计数器
type Counter struct {
    mu    sync.RWMutex
    count uint64
}

// 使用写锁保护
func (c *Counter) Incr() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// 使用读锁保护
func (c *Counter) Count() uint64 {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.count
}
```

##### **RWMutex内部实现**

Golang中RWMutex是基于Mutex实现的。

readers-writers 问题一般有三类，基于对读和写操作的优先级，读写锁的设计和实现也分成三类。

**Read-preferring：**读优先的设计可以提供很高的并发性，但是，在竞争激烈的情况下可能会导致写饥饿。这是因为，如果有大量的读，这种设计会导致只有所有的读都释放了锁之后，写才可能获取到锁。

**Write-preferring：**写优先的设计意味着，如果已经有一个 writer 在等待请求锁的话，它会阻止新来的请求锁的 reader 获取到锁，所以优先保障 writer。当然，如果有一些 reader 已经请求了锁的话，新请求的 writer 也会等待已经存在的 reader 都释放锁之后才能获取。所以，写优先级设计中的优先权是针对新来的请求而言的。这种设计主要避免了 writer 的饥饿问题。

**不指定优先级：**这种设计比较简单，不区分 reader 和 writer 优先级，某些场景下这种不指定优先级的设计反而更有效，因为第一类优先级会导致写饥饿，第二类优先级可能会导致读饥饿，这种不指定优先级的访问不再区分读写，大家都是同一个优先级，解决了饥饿的问题。

**Go中的RWmutex采用Write-preferring。**

RWMutex结构包含一个Mutex，和四个辅助字段：

```go
type RWMutex struct {
  w           Mutex   // 互斥锁解决多个writer的竞争
  writerSem   uint32  // writer信号量
  readerSem   uint32  // reader信号量
  readerCount int32   // reader的数量
  readerWait  int32   // writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30
```

**字段 w：**Mutex锁，为 writer 的竞争锁而设计。

**字段 readerCount：**记录当前 reader 的数量（以及是否有 writer 竞争锁）。

**readerWait：**记录 writer 请求锁时需要等待 read 完成的 reader 的数量

***writerSem 和 readerSem：***都是为了阻塞设计的信号量。

###### **RLock/RUnlock 的实现**

```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
            // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先级高，所以把后来的reader阻塞休眠
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow(r) // 有等待的writer
    }
}
func (rw *RWMutex) rUnlockSlow(r int32) {
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 最后一个reader了，writer终于有机会获得锁了
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

**readerCount字段的双重含义：**

- 没有 writer 竞争或持有锁时，readerCount 和我们正常理解的 reader 的计数是一样的。
- 如果有 writer 竞争锁或者持有锁时，那么，readerCount 不仅仅承担着 reader 的计数功能，还能够标识当前是否有 writer 竞争或持有锁，在这种情况下，请求锁的 reader 的处理进入第 4 行，阻塞等待锁的释放。

在RUnlock时，需要先检查是否存在writer竞争锁（readerCount为负值），在这种情况下，还会调用方rUnlockSlow方法等待所有的reader锁释放。

**当 writer 请求锁的时候，是无法改变既有的 reader 持有锁的现实的，也不会强制这些 reader 释放锁，它的优先权只是限定后来的 reader 不要和它抢。**

###### **Lock 的实现**

```
func (rw *RWMutex) Lock() {
    // 首先解决其他writer竞争问题
    rw.w.Lock()
    // 反转readerCount，告诉reader有writer竞争锁
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 如果当前有reader持有锁，那么需要等待
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```

一旦一个 writer 获得了内部的互斥锁，就会反转 readerCount 字段，把它从原来的正整数 readerCount(>=0) 修改为负数（readerCount-rwmutexMaxReaders），让这个字段保持两个含义（既保存了 reader 的数量，又表示当前有 writer）。

如果 readerCount 不是 0，就说明当前有持有读锁的 reader，RWMutex 需要把这个当前 readerCount 赋值给 readerWait 字段保存下来（第 7 行）， 同时，这个 writer 进入阻塞等待状态（第 8 行）。

每当一个 reader 释放读锁的时候（调用 RUnlock 方法时），readerWait 字段就减 1，直到所有的活跃的 reader 都释放了读锁，才会唤醒这个 writer。

###### **Unlock 的实现**

```
func (rw *RWMutex) Unlock() {
    // 告诉reader没有活跃的writer了
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    
    // 唤醒阻塞的reader们
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // 释放内部的互斥锁
    rw.w.Unlock()
}
```

当一个 writer 释放锁的时候，它会再次反转 readerCount 字段。

当writer 要释放锁了，需要唤醒之后新来的 reader，不必再阻塞它们了（第7行）

Mutex在这里需要保障修改字段的互斥关系（实现代码中省略），在 Lock 方法中，是先获取内部互斥锁，才会修改的其他字段；而在 Unlock 方法中，是先修改的其他字段，才会释放内部互斥锁，这样才能保证字段的修改也受到互斥锁的保护。



