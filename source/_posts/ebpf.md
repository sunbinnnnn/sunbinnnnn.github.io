---
title: eBPF
date: 2023-05-22 19:01:22
tags:
- system
- kernel
- network
typora-copy-images-to: ../images
typora-root-url: ..
---

本文主要对eBPF相关进行总结。

<!--more-->

# eBPF

## 1.Why eBPF

*推荐学习：*

*https://github.com/eunomia-bpf*

https://cilium.isovalent.com/hubfs/Learning-eBPF%20-%20Full%20book.pdf

### 1.1.Overview

eBPF 是一种革命性的内核技术，允许开发者编写一些内核代码（例如重写内核函数），并动态加载到内核中，改变 Kernel 的行为。

通过 eBPF 可以缩短内核调用链路，改变 System Call 方式，并监控内核调用链路，比较常见的 eBPF 使用场景：

- 构建高性能网络
- 跟踪内核调用链
- 阻止不安全的内核调用（网络 DDOS）
- 内存分析
- WASM
- ...

eBPF彻底改变了内核开发方式，内核至此诞生了无数可能性。

### 1.2.BPF、eBPF

一句话：**BPF（cBPF）只是一个包过滤器（tcpdump就是基于这玩意写的），只有两个32位的寄存器；而eBPF是基于事件的内核调用修改器，有10个64位寄存器**，支持JIT-Compilation、方便的用户态 <-> 内核态通信、maps支持、方便的helper func...

### 1.3.Kernel

用户应用一般存在于 User space，通过高级语言调用 System Call。

![image-20230512155031730](/images/image-20230512155031730.png)

exmaple（一个 echo 可以包含100多个系统调用）：

![image-20230512155241487](/images/image-20230512155241487.png)

因为 Linux 代码的复杂性，且 Linus Torvalds 的执着与 Linux release 数量日益增长，如果想增加额外的 code 来实现对 System Call 进行中断，观测"Open files、Network IO”等，几乎是不可能的。

![image-20230512155648348](/images/image-20230512155648348.png)

### 1.4.Kernel Modules

上文提到，向 Kernel 中提交代码是非常复杂且艰巨的，所以，Linus 大佬也没把路给封死，另一种修改 Kernel 的方式是通过内核模块（Kernel Modules）。

Kernel Modules 可以按需加载或卸载，并且独立于官方 Linux kernel release 发布，不需要 merge 在 upstream branch 上。

但是，Kernel Module 还是有一定的危险性，原因很简单，不稳定的 Kernel Module 和内核程序一起运行，可能会导致 Kernel crash，另外，由于 Kernel Modules 的作者来源不一定得到验证，无法确保是否安全（Kernel Modules 运行在 Kernel space，可以访问机器上所有东西）。所以大家对 Kernel Modules 的使用一直是非常谨慎的。

eBPF提供了一种非常不同的方式来确保安全性：[eBPF verifier](https://docs.kernel.org/bpf/verifier.html)，其只允许安全的eBPF程序（不会造成 Kernel crash 或者死锁）运行，也不对数据安全有所妥协。

### 1.5.Dynamic Loading of eBPF Programs

eBPF 能够被 Kernel 动态加载，一旦 eBPF attch 到一个指定的event，会直接被 event trigger 到，不论这个发起这个 event 的进程是否在之前已经启动了，这样不需要重启机器就可以直接使用eBPF程序。

这促使 eBPF 可以形成具备超强的安全、观测工具集，它可以观测机器上发生的所有事，如果跑着容器里，和跑在 host 一样，能观测到容器里发生的所有事，史上最强 CNI：Cilium 就是这么诞生的。

另外，很重要的一点：引入内核功能再也不需要挨 Linus 大佬批了。

![image-20230512171104918](/images/image-20230512171104918.png)

### 1.6.High Perofrmance of eBPF Programs

eBPF 程序添加检测的过程是相当高效的，当一个 eBPF 程序被加载并即时编译（JIT-compiled）后，程序会直接运行 CPU 机器指令。另外，在接受每个事件的时候，用户态到内核态的拷贝也是不存在的(hook 在 copy 前)，所以几乎没有开销。

内核网络示意图：

![pic](/images/pic.jpg)

以基于 eBPF 的高性能内核包过滤器：XDP（eXpress Data Path）为例，传统的包过滤都是在服务进程侧进行过滤（例如 iptables），其经过了内核态解包，copy 到用户态，性能会很差：

<img src="/images/netstack.png" alt="img" style="zoom: 50%;" />

而 XDP 将 eBPF 的 event hook 放在网卡驱动侧，在分配 skb 之前就可以直接对包进行过滤，通过 XDP 进行包过滤、转发，其性能是ipvs 的4.3倍（iptables 更不用说了）。

<img src="/images/netstackebpf.png" style="zoom:50%;" />

BPF hook网络栈打点：

![image-20230615161801921](/images/image-20230615161801921.png)



### 1.7.eBPF in Cloud Native Environments

如今，多数用户选择以云原生的方式运行eBPF程序，比如容器、K8S 或者公有云的 Serverless 服务（AWS Lambda）等等；

好处显而易见，节点上的容器 share 同一个 kernel，例如，在 K8S 中，所有的 Pods 共享同一个 kernel，所以我们只需要起一个容器，就可以观测节点上所有 workloads：

![image-20230512174830637](/images/image-20230512174830637.png)

利用节点上所有进程的观测能力，以及 eBPF 程序的动态编译能力，一些 eBPF-based 的工具可以在集群中发挥 superpower：

- 应用程序不需要做任何改动、配置，就可以接入eBPF工具集。
- 当 eBPF 程序被动态加载时，不需要重启节点或应用，就可以对已经运行的应用进行观测。

我们知道，一般在 K8S 里，我们可以通过给 container 挂 sidecar 来对网络流量、进程进行过滤、追踪、转发，实现 service mesh、logging 等功能。这种方式相较于直接在代码中调用对应的SDK来说，更为方便，对代码无侵入性，但是，sidecar 其实存在很多缺陷：

- 当为 Pod 插入 Sidecar 时，Pod 需要重启。

- 往往需要修改应用的 yaml，才可以插入 sidecar。例如，我们一般通过添加 annontation 的方式来触发 injector 来修改某个workload的 yaml，插入 sidecar，整个过程是自动的，但如果这个 workload（比如 deployment），漏加了 annontation，或者没有加某个 label，这个 sidecar 就不会注入，从而无法观测这个应用了。

- 当一个 Pod 中存在多个 Container 时，由于彼此达到 Readiness 的时间点是无法预测的，sidecar 的注入会可能导致容器启动速度下降，甚至会导致竞争情况。

- 当网络功能（例如 service mash）以 sidecar 的方式注入时，整条网络路径非常冗长，会导致出现抖动、丢包的情况：

  ![image-20230512193453205](/images/image-20230512193453205.png)

幸运的是，我们可以通过 eBPF 来解决 sidecar 模式的诸多问题，原因还是在于eBPF可以观测节点上发生的所有事件，不论是否有sidecar的注入，依赖 eBPF，可以解决一系列因 sidecar 带来的不稳定与安全问题。



**Wasm-bpf**



## 2.eBPF's "Hello World"

### 2.1. Run first eBPF Program

eBPF 如此强大，我们从一个简单的例子来编写eBPF应用。

有很多 lib 库和框架可以帮助我们编写一个 eBPF 应用，例如[BCC Python framework](https://github.com/iovisor/bcc)，提供了相当简单的方式去编写eBPF程序。

首先，我们需要安装 BCC 框架，可以参考官方文档：https://github.com/iovisor/bcc/blob/master/INSTALL.md。

以下是一个 "Hello World” 的 demo：

```python
#!/usr/bin/python3

from bcc import BPF

program = r"""
int hello(void *ctx) {
    bpf_trace_printk("Hello, World!\\n");
    return 0;
}
"""

b = BPF(text=program)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")

b.trace_print()

```

这段代码包含两部分：

1.eBPF 内核态程序

2.eBPF 用户态程序

![image-20230518190719115](/images/image-20230518190719115.png)

eBPF 有一组`helper function`来与内核进行交互，在demo中，包含一个`helper function`: `bpf_trace_printk()`，用来输出一行信息到 trace_pipe（/sys/kernel/debug/tracing/trace_pipe）文件中。eBPF 代码本身是通过C来编写的，理论上需要进行编译才可以运行（实际上单独编写一个eBPF程序也需要编译），但当通过 BCC 框架来开发时，不需要额外进行编译操作，只需要声明一个BPF object：

```b = BPF(text=program)```

eBPF程序还需要 attach 到一个 event 上，在 demo 中 attach 到了`execve`这个syscall上，当任何一个可执行程序被执行时，这个 syscall都会被调用，接着就会 trigger eBPF程序。一般来说，syscall 会根据系统架构的不同，而带一些特定的前缀，例如：

```
"sys_",
"__x64_sys_",
"__x32_compat_sys_",
"__ia32_compat_sys_",
"__arm64_sys_",
"__s390x_sys_",
"__s390_sys_",
```

BCC框架提供了 `get_syscall_fnname ` 的函数来获取对应的真实 syscall 名称，所以我们就不需要考虑系统架构了：

```syscall = b.get_syscall_fnname("execve")```

接下来，我们通过需要通过内核的 [kprobes](https://docs.kernel.org/trace/kprobes.html)（在05年的时候被加入 Kernel，通过这个功能可以允许开发者在几乎所有的指令上增加trap）功能来attach到一个syscall的events上:

```b.attach_kprobe(event=syscall, fn_name="hello")```

最后，通过一个死循环来 hang 住程序，并获取 trace_pipe 中的内容，就可以开始监听内核调用了。

执行下（很多都是 vsocde server 执行的程序）：

![image-20230518193303378](/images/image-20230518193303378.png)

```
Tips:
如果不是通过 root 运行，可能会没有权限，eBPF 必须的三个内核 capabilitied 分别为：
CAP_BPF(https://mdaverde.com/posts/cap-bpf/), CAP_PERFMON, CAP_NET_ADMIN
```

在输出结果中，除了"Hello World”外，还有一些额外的event上下文信息，包括调用程序，process ID 等。

但这个demo其实有很多问题，除了几乎不能做除了观测以外的其他事，最重要的是，这个 demo无 法获取 eBPF 程序本身的信息，当多个"Hello-World”同时运行时，并不知道是"谁”往 trace_pipe 中塞了内容，所以，BPF maps 登场了。



### 2.2.BPF maps

**map **是用于 eBPF 程序和用户态程序之间交互的数据结构（在绝大多数场景，BPF maps 和eBPF maps 是一个东西），比较典型的用法是：

1.eBPF 程序加载用户态程序配置

2.eBPF 程序间传递状态

3.eBPF 将结果传递到用户态程序

在 Kernel 中定义了很多[BPF maps类型](https://elixir.bootlin.com/linux/v5.15.86/source/include/uapi/linux/bpf.h#L878)，可以在[kernel文档](https://docs.kernel.org/bpf/map_bloom_filter.html)中找到他们的说明。一般来说，maps 是以 KV 的形式存储的，但同样可以被扩展为 hash 表、perf、ring buffers 或者数组。

被扩展成数组的 map，由一个4字节的 key 构成；而 hash 表类型的 map 可以存在任何类型的 key。一些 map 类型也被扩展成一些周知的类型，例如先入先出队列，先入后出栈，最长匹配队列，或者 Bloom filters。

有一些常用的 BPF maps 存在一些特定的对象，例如：

[sockmaps ](https://lwn.net/Articles/731133/)和 [devmaps](https://docs.kernel.org/bpf/map_devmap.html) 存放了sockets和网络设备相关的信息，可以给网络功能相关的 eBPF 程序所使用，用来重定向流量；

数组类型的map存放了 eBPF 程序的 index，可以用于和其他 eBPF 程序交互；

一些 map 类型还会分散在不同的内存块上，由不同的 CPU 来读写，一般用于并行场景，使用这种map时，需要考虑并行锁的问题（Spin lock）。

接下来，我们通过一个例子来使用一个 hash 表类型的 BPF map，通过 BCC 框架，我们可以很简单地去扩展一个 hash 表。

#### 2.2.3.Hash Table Map

在"Hello World”的基础上，我们将 hello 函数进行扩展，定义出一个 hash 表来存放 user ID 和对应的user调用```execve```的次数。

BPF 程序：

```C
BPF_HASH(counter_table); // BPF_HASH 是用来定义 hash table 的宏

int hello(void *ctx) {
    u64 uid;
    u64 counter = 0;
    u64 *p;
    // bpf_get_current_uid_gid 是一个 helper func，用来定位被 kprobe event trigger 到的 user ID, user ID的前32位是group ID，我们通过32为掩码过滤
    uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    p = counter_table.lookup(&uid); // 查询hash表，找到对应key为uid的value，返回的p是指向value的指针
    if (p != 0) { // 之前已经记过数
        counter = *p;
    }
    counter ++;
    counter_table.update(&uid, &counter); // 更新hash表
    return 0;
}
```

我们仔细观察这段"C”代码：

```C
p = counter_table.lookup(&uid);
counter_table.update(&uid, &counter);
```

熟悉C语言的同学可能会发现，C语言是不能这样定义结构体方法的，C语言不能在struct中直接定义出方法体，需要通过函数指针，来指向某个方法（C++支持class，倒是可以）。这里其实可以理解成是BCC的语法糖，在BCC编译这段C代码时，会通过一些macros将代码转化为标准C语言，来简化编码过程（点个赞）。

接下来，我们将用户态程序补完：

```py
# Load BPF program
b = BPF(text=program)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")
while 1: # 死循传，每2秒输出一次
    sleep(2)
    s = ""
    for k, v in b["counter_table"].items(): # 从BPF object中获取couter_table的map，输出uid和counter
        s += f"ID {k.value}: {v.value}\t"
    print(s)
```

跑一下：

![image-20230519145005788](/images/image-20230519145005788.png)

我用ID为1000的用户执行了2次"ls"命令，可以看到，ID 1000用户被记录下了2次`execve`的系统调用。

在这个例子里，我们使用了int类型作为hash表的key，所以当key小于4字节时，会转换为arrary，而hash表可以支持任何形式的key。hash表在k-v场景里很好用，但用户态程序必须要轮训hash表才可以获取数据。

Kernel支持perfbuf（per-CPU环形缓存区），来将数据从内核态传到用户态，在eBPF中可以采用它，同样包括它的升级版：BPF ring buffer（非per-CPU，解决并行下不一致问题，需要内核版本5.8以上），其解决了perfbuf内存效率低下以及事件顺序不一致的问题。

#### 2.3.4.Perf and Ring Buffer Maps

关于Ring Buffer（如果你已经知道什么是Ring Buffer，可以跳过）：

*Ring Buffer和BPF的概念无关，你可以将Ring Buffer想象成由将一块内存按一定的逻辑构成的一个环（由于计算机内存地址是线性的，需要特别的算法设计才可以从逻辑上实现），这个有一个独立的写指针和一个读指针。数据的长度会包含在数据的header中，程序会从写指针的位置（最初的位置无关紧要）将数据写入Ring Buffer（包含数据长度），接着，写指针移动x位（x为数据长度）。*

*而读取时，程序在读指针的位置读取数据，根据写入时包含的数据长度，移动x位*。

*当读指针追上写指针时，代表数据读完了；如果写操作使写指针追上了读指针，则数据会被丢弃，且将丢弃计数器（drop counter）自增，通过读操作和丢弃计数器就能知道自上次成功读取数据后，是否有数据被丢弃*。

*如果写入速率和读取速率一致，那不需要调整缓冲区大小，否则，为了不让数据被丢弃（一般来说，业务会处理位阻塞），我们需要动态调整缓冲区大小。*

<img src="/images/v2-846878d6005f23f9f15afde8fe0ae028_b.gif" alt="动图" style="zoom:50%;" />

下面我们以`Perf Buffer`为例，来将"hello world”进行一版改进，不再将`execve`写入trace文件，而通过`Perf Buffer`来传递。

BPF程序代码：

```C
BPF_PERF_OUTPUT(output); // BPF Perf Buffer的macro，用于定义从Kernel Space往User Space传递信息的map, map的key为output

struct data_t { // 定义存储信息的结构体
    int pid;
    int uid;
    char command[16];
    char message[12];
};
 
int hello(void *ctx) {
   struct data_t data = {}; 
   char message[12] = "Hello World";
 
   data.pid = bpf_get_current_pid_tgid() >> 32; // bpf_get_current_pid_tgid()是一个helper func，前32位用存储进程的ID，后32位存储进行group ID，将前32位右移
   data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
   
   bpf_get_current_comm(&data.command, sizeof(data.command)); // 获取command或执行程序的helper func，通过地址传递给data.command
   bpf_probe_read_kernel(&data.message, sizeof(data.message), message); // 在这里例子里，在内核中的message"Hello Wrold”，被传递到data结构体中
 
   output.perf_submit(ctx, &data, sizeof(data)); //将data塞到map中
 
   return 0;
}
```

用户态Python代码：

```py
# Load BPF program
b = BPF(text=program)
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")
 
def print_event(cpu, data, size):  # print_event是一个回调函数，用于在屏幕上打印一行输出
   data = b["output"].event(data) # BCC做了高度抽象，将在BPF map中可以直接提取output,以及out中的data结构体
   print(f"{data.pid} {data.uid} {data.command.decode()} {data.message.decode()}")
 
b["output"].open_perf_buffer(print_event) # 打开perf buffer，通过print_event回调函数结构read的信息
while True:   # 死循环，开始执行perf poll
   b.perf_buffer_poll()
```

跑一下：

![image-20230522111340994](/images/image-20230522111340994.png)

在这个例子中，我们通过了名为"output"的ring map来将数据从Kernel传到User Space，这个ring map只能给这个程序本身用，这和案例一中采用trace文件时完全不同的，在`/sys/kernel/debug/tracing/trace_pipe`文件中，无法找到对应的上下文信息：

![image-20230522111612212](/images/image-20230522111612212.png)

Tips：Ring Buffer相较于Perf Buffer具备更小的内存占用，以及由于采用多核共享内存，解决了per-CPU的数据一致性问题。通过BCC实现Ring Buffer版本的Hello World和Perf Buffer其实区别不大，你可以根据[BCC接口文档](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#5-bpf_ringbuf_output)自己尝试将Perf Buffer换成Ring Buffer 

#### 2.3.5.Function Callls

从上面几个例子可以看到，在BPF程序中，我们调用了很多Kernel的helper function，从程序编写的风格上说，我们应该将自己的公共代码逻辑也包装为函数。当在BPF程序的早期，BPF程序是不允许调用除了helper function以外其他函数的（进栈限制），这就给编码带来了很多不便，所以，早期采用了一种workaround的方式来处理。（总是定义内联函数）"Always inline"：

```C
static __always_inline void my_function(void *ctx, int val)
```

内联函数是C语言中可以定义的一种特殊函数，一般来说，函数被编译后的执行方式是"函数初始位置（堆） - > 执行函数（进栈） -> 返回初始位置（出栈，返回堆）”。当使用内联函数时，编译器会将函数代码指令直接复制到堆中。

![image-20230522124958075](/images/image-20230522124958075.png)

实际上，由于编译器的优化，一些内联函数会被强制编译为栈函数，而一些栈函数也会被强制编译为内联函数（所以有些Kernel function没法被kprobe attatch），这会导致我们在使用内联上存在一定的问题。

在Kernel 4.16（LLVM 6.0）后，内核取消了对函数栈调用的限制，不需要通过"Always inline”的方式来workaround了，这个特性被称为"BPF function calls”或者"BPF subprograms”（BPF子程序）。但目前BCC框架并不支持类型BPF function calls的功能，取而代之的将复杂能力分解的方式：tail calls。

#### 2.3.6.Tail Calls

Tail Calls（尾调用）能够调用或者执行其他eBPF程序并且替换执行上下文信息，与`execve()`这个系统调用给常规进程使用类似。换句话说，执行程序在tail call完成前不会返回。

*Tips: [Tail Calls](https://www.baeldung.com/cs/tail-vs-non-tail-recursion)不是eBPF独有的概念，因为我们在其他地方一般会说另一个词：Tail Recursion（尾递归）。其提出的主要解决的问题是防止函数被递归调用时，反复向栈中添加栈帧（frames），最终导致堆栈溢出。而使用尾递归时，由于递归函数是当前当前堆栈中最后一条待执行语句，当这个调用返回后，当前栈帧中将无事可做，也就没有保留栈帧的必要了，于是就可以直接覆盖当前的栈帧，而不是添加栈帧，这样大大缩减了栈空间的大小，使得实际运行效率变高。在eBPF中，堆栈大小被限制在了512 bytes，所以Tail Calls可以解决很多问题*。

Tail Calls使用`bpf_tail_call()`这个helper function来定义，函数签名如下：

```C
long bpf_tail_call(void *ctx, structbpf_map *prog_array_map, u32 index)
```

函数中的三个参数的意义如下：

1.`ctx`：eBPF函数调用者和eBPF程序直接的上下文信息。

2.`prog_arryry_map`：`BPF_MAP_TYPE_PROG_ARRAY`类型的eBPF map，维护eBPF程序的一组文件描述符。

3.`index`：`index`表示哪一组eBPF程序需要被调用。

Tail call用法比较有意思，eBPF通过tail call调用成功后，不会返回原来的eBPF程序，因为tail call会重用调用方函数的栈帧。但是在失败的时候还是会返回，比如指定的程序不在map里，此时原函数还是能继续执行。

对于用户态程序来说，还是和之前一样，将eBPF程序加载到Kernel，并且创建程序的array map。

下面我们来尝试写一个tail call的例子。在例子中，eBPF程序attach到一个tracepoint，来监听所有的系统调用，并通过tail calls将特定opcode的syscall以特定的信息输出，其他的输出通用信息。

当使用BCC框架时，可以采用一行简单的方式来定义：

```C
prog_array_map.call(ctx, index)
```

在编译时，BCC会将这行重写为：

```
bpf_tail_call(ctx, prog_array_map, index)
```

BPF程序代码如下： 

```C
BPF_PROG_ARRAY(syscall, 300); // BCC提供BPF_PROG_ARRAY的macro，可以相对方便地定义一个BPF_MAP_TYPE_PROG_ARRAY的map（linux中总共有300种syscall，这个基本够了）

int hello(struct bpf_raw_tracepoint_args *ctx) { // hello函数的参数不是某个具体的系统调用，而是raw_tracepoint，下面的用户态程序选择sys_enter的raw tracepoint时，所有的系统调用都会被ctx接收
    int opcode = ctx->args[1];  // 提取bpf_raw_tracepoint_args struct中的opcod
    syscall.call(ctx, opcode); // 这里通过tail call在program array中，找到将符合opcode的程序，并调用. 这里在编译的时候会被BCC重写为bpf_tail_call()
    bpf_trace_printk("Another syscall: %d", opcode); // tail call失败的case，如果tail call成功了，这里不会执行，表示opcode没有hit
    return 0;
}

int hello_exec(void *ctx) { // hello_exec()是加载到syscall array map中的一个程序，当opcode为 execve()syscall时，它将被tail call调用
    bpf_trace_printk("Executing a program");
    return 0;
}

int hello_timer(struct bpf_raw_tracepoint_args *ctx) { // hello_timeer也是一个加载到syscall array map中的一个程序，用于处理其他opcode的情况
    int opcode = ctx->args[1];
    switch (opcode) {
        case 222:
            bpf_trace_printk("Creating a timer");
            break;
        case 226:
            bpf_trace_printk("Deleting a timer");
            break;
        default:
            bpf_trace_printk("Some other timer operation");
            break;
    }
    return 0;
}

int ignore_opcode(void *ctx) { // 为了过滤不想输出的opcode
    return 0;
}
```

用户态Python代码如下：

```python
b = BPF(text=program)
b.attach_raw_tracepoint(tp="sys_enter", fn_name="hello") # 使用sys_enter的tracepoint取代kprobe，以观察所有的sys_enter

ignore_fn = b.load_func("ignore_opcode", BPF.RAW_TRACEPOINT) # b.load_func()会返回tail call程序的文件描述符，需要注意的是，taill calls函数需要和父程序使用同样的程序类型，在这里是BPF.RAW_TRACEPOINT，不能使用kprobe
exec_fn = b.load_func("hello_exec", BPF.RAW_TRACEPOINT)
timer_fn = b.load_func("hello_timer", BPF.RAW_TRACEPOINT)

prog_array = b.get_table("syscall") # 用户态程序在syscall map中加入元素，将对应tail call程序的文件描述符和opcode一一映射
prog_array[ct.c_int(59)] = ct.c_int(exec_fn.fd)
prog_array[ct.c_int(222)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(223)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(224)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(225)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(226)] = ct.c_int(timer_fn.fd)

# Ignore some syscalls that come up a lot
prog_array[ct.c_int(21)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(22)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(25)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(29)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(56)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(57)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(63)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(64)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(66)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(72)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(73)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(79)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(98)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(101)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(115)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(131)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(134)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(135)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(139)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(172)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(233)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(291)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(212)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(260)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(94)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(221)] = ct.c_int(ignore_fn.fd)

b.trace_print() # 打印trace信息
```

跑一下：

![image-20230522155022177](/images/image-20230522155022177.png)

需要注意的是，Tail Calls在Kernel版本4.2后被支持，在5.10之前和BPF function call是不兼容的，在eBPF subprogram中调用tail calls依赖JIT编译（x86 5.10, arm 6.0）。



## 3.eBPF with native C

在实际大型项目中，除了通过BCC框架，我们仍然需要编写相对复杂的原生eBPF程序。本章中，我们会使用完整的C代码来编写"Hello World”。

一个eBPF程序实际上是一组eBPF字节码指令，一般会通过高级语言编写（当然你可以直接写字节码～），最终编译为eBPF字节码。eBPF程序用的最多的高级语言是C，也有一些eBPF程序用Rust编写，因为Rust编译器也支持eBPF字节码：

![image-20230522160351303](/images/image-20230522160351303.png)

字节码最终会跑在kernel的eBPF虚拟机中。



### 3.1.The eBPF Virtual Machine

eBPF虚拟机和其他虚拟机一样，是通过软件实现的一个计算机。在这个"计算机”里，能够运行eBPF的字节码指令，这些字节码指令最终会转化为在CPU上跑的原生机器指令。

在eBPF的早期实现中，eBPF字节码会直接被Kernel解析，一旦eBPF程序开始运行，Kernel会检查eBPF字节码并将其转换为机器码，然后再执行。目前，处于性能和安全性考虑，Kernel的字节码解释器已经被JIT（Just In Time）编译所取代，这意味着在程序被加载到内核时，只会被转换一次（即编译）。

eBPF字节码包含一组指令，这些指令会运行在eBPF（虚拟）寄存器（eBPF registers）上。eBPF指令集和寄存器模型与通用CPU架构相匹配，这样编译机器码的过程相对比较简单。

#### 3.1.1.eBPF Registers

eBPF虚拟机有10个一般用途的寄存器（0～9），还有一个10号寄存器，用来充当栈帧指针（只能读，不能写）。当BPF程序运行时，值会被存储到这些寄存器汇总以持续记录状态。

和CPU寄存器不同，这些寄存器是通过软件方式实现的，可以在[Kernel代码](https://elixir.bootlin.com/linux/v5.19.17/source/include/uapi/linux/bpf.h)中找到。

eBPF程序运行前，上下文参数会被加载到Register 1，返回值则会存储到Register 0。

在调用eBPF代码中函数前，函数的参数会存储到Register1到5中（如果低于5个参数，则其他寄存器不会使用）。

#### 3.1.2.eBPF Instructions

在 Kernel 的ebpf的[header代码](https://elixir.bootlin.com/linux/v5.19.17/source/include/uapi/linux/bpf.h#L71)中定义了一个名为`bpf_insn`的结构体，来描述eBPF指令（Instrcutions）

```
struct bpf_insn {
	__u8	code;		/* opcode */
	__u8	dst_reg:4;	/* dest register */
	__u8	src_reg:4;	/* source register */
	__s16	off;		/* signed offset */
	__s32	imm;		/* signed immediate constant */
};
```

`bpf_insn`结构体的操作码（opcode）分为这几类：

1.将值加载到寄存器

2.将值从寄存器存储到内存

3.进行算术操作

4.当条件满足时，跳跃到不同的寄存器

接下来，我们来尝试编写一个具体的例子。

### 3.2."Hello World” for a Network Interface

之前的例子，我们通过eBPF来观测系统调用相关的情况，这次，我们来观测下网络数据包相关的情况。

数据包处理是eBPF应用最多的场景，eBPF可以对每个经过网卡的数据包进行过滤、解析，甚至修改数据包的内容。

下面我们来编写eBPF程序代码，在这个例子中，我们只对数据包计数，二不会对数据包进行任何修改：

```C
#include <linux/bpf.h> // 引用bpf头文件
#include <bpf/bpf_helpers.h>

int counter = 0; // 定义自增计数器全局变量

SEC("xdp") // SEC 是一个 macro，用于定义eBPF程序类型
int hello(struct xdp_md *ctx) { // 这是eBPF程序的主体，eBPF的程序名和函数名一致，即hello
    bpf_printk("Hello World %d", counter); // 这里使用了bpf_printk()的helper function，来打印信息，和之前的bpf_trace_printk()一样，而后者是BCC框架提供的
    counter++; 
    return XDP_PASS; // XDP_PASS是向Kernel指明，该数据包字节交给Kernel，以一般数据的方式处理
}

char LICENSE[] SEC("license") = "Dual BSD/GPL"; // eBPF程序license声明，当调用的helper function是GPL license时，必须在你的代码中声明GPL license，否则会无法使用
```

这里例子中，我们将eBPF程序attach到 了XDP(eXpress Data Path) hook point上，你可以理解为，一旦数据包通过网卡，XDP event就会被触发。

*Tips：有些网卡可以直接运行XDP程序，这使得数据包在到达CPU之前就可以被处理，对于性能的提升会非常大。*

OK，那接下来，我们把这段代码编译一下。

#### 3.2.1.Compiling an eBPF Object File

eBPF源码需要编译为机器码（eBPF字节码）才能在eBPF虚拟机上运行。我们可以通过clang来进行编译，Makefile如下：

```makefile
TARGETS = hello hello-func

all: $(TARGETS)
.PHONY: all

$(TARGETS): %: %.bpf.o 

%.bpf.o: %.bpf.c
	clang \
	    -target bpf \
		-I/usr/include/$(shell uname -m)-linux-gnu \
		-g \
	    -O2 -o $@ -c $<

clean: 
	- rm *.bpf.o
	- rm -f /sys/fs/bpf/hello 
	- rm -f /sys/fs/bpf/hello-func
```

可能需要装一些依赖包：

```bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl clang llvm jq
apt-get install -y libelf-dev libpcap-dev libbfd-dev binutils-dev build-essential make 
apt-get install -y linux-tools-common linux-tools-5.15.0-41-generic bpfcc-tools
apt install -y libbpf-dev
```

执行编译：

```
make hello-net.bpf.o
```

#### 3.2.2.Inspecting an eBPF Object File

我们可以通过`file`命令来看下对象文件的内容：

```bash
# file hello-net.bpf.o 
hello-net.bpf.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info, not stripped
```

可以看到，eBPF的对象文件是一个ELF（Excutable and Linkable Format）文件，包含eBPF标识码，对应64位LSB（Least significant bit）架构。

我们可以还可以通过`llvm-objdmp`命令来看更详细反编译信息：

```bash
# llvm-objdump -S hello-net.bpf.o 

hello-net.bpf.o:        file format elf64-bpf

Disassembly of section xdp:

0000000000000000 <hello>:
; int hello(struct xdp_md *ctx) {
       0:       b7 01 00 00 00 00 00 00 r1 = 0
;     bpf_printk("Hello World %d", counter);
       1:       73 1a fe ff 00 00 00 00 *(u8 *)(r10 - 2) = r1
       2:       b7 01 00 00 25 64 00 00 r1 = 25637
       3:       6b 1a fc ff 00 00 00 00 *(u16 *)(r10 - 4) = r1
       4:       b7 01 00 00 72 6c 64 20 r1 = 543452274
       5:       63 1a f8 ff 00 00 00 00 *(u32 *)(r10 - 8) = r1
       6:       18 01 00 00 48 65 6c 6c 00 00 00 00 6f 20 57 6f r1 = 8022916924116329800 ll
       8:       7b 1a f0 ff 00 00 00 00 *(u64 *)(r10 - 16) = r1
       9:       18 06 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r6 = 0 ll
      11:       61 63 00 00 00 00 00 00 r3 = *(u32 *)(r6 + 0)
      12:       bf a1 00 00 00 00 00 00 r1 = r10
      13:       07 01 00 00 f0 ff ff ff r1 += -16
;     bpf_printk("Hello World %d", counter);
      14:       b7 02 00 00 0f 00 00 00 r2 = 15
      15:       85 00 00 00 06 00 00 00 call 6
;     counter++; 
      16:       61 61 00 00 00 00 00 00 r1 = *(u32 *)(r6 + 0)
      17:       07 01 00 00 01 00 00 00 r1 += 1
      18:       63 16 00 00 00 00 00 00 *(u32 *)(r6 + 0) = r1
;     return XDP_PASS;
      19:       b7 00 00 00 02 00 00 00 r0 = 2
      20:       95 00 00 00 00 00 00 00 exit
```

基本上是一些寄存器的指令操作，通过opcode，设置值到对应的寄存器，例如这里的`0xb7`，这里不展示讨论了，有兴趣的同学可以参考下eBPF指令集：https://www.kernel.org/doc/Documentation/networking/filter.txt。

#### 3.2.3.Loading the Program into the Kernel

我们可以通过`bpftool`工具来将程序加载到Kernel中：

```bash
# bpftool prog load hello-net.bpf.o /sys/fs/bpf/hello
```

通过`bpftool`工具可以列出所有加载到Kernel中的eBPF程序：

```bash
# bpftool prog list
...
14: cgroup_skb  tag 6deef7357e7b4530  gpl
        loaded_at 2023-05-24T02:46:54+0000  uid 0
        xlated 64B  jited 96B  memlock 4096B
27: xdp  name hello  tag 4ae0216d65106432  gpl
        loaded_at 2023-05-24T10:39:28+0000  uid 0
        xlated 168B  jited 200B  memlock 4096B  map_ids 3
        btf_id 117

# bpftool prog show id 103 --pretty
{
    "id": 103, # BPF程序ID
    "type": "xdp", # type为xdp，xdp类型的BPF程序可以attach到网卡上，监听网卡的XDP事件
    "name": "hello",
    "tag": "4ae0216d65106432", # tag标识，即该BPF程序的SHA HASH，
    "gpl_compatible": true,
    "loaded_at": 1685357932, # load时间
    "uid": 0, # load用户（root）
    "bytes_xlated": 168, # eBPF字节码大小
    "jited": true,
    "bytes_jited": 200, # 程序被JIT-compiled，且机器码为200 bytes
    "bytes_memlock": 4096, # 该程序预留4096 bytes的内存，不会分页
    "map_ids": [3 # eBPF maps ID
    ],
    "btf_id": 118 # BPF Type Format ID
}
```

同样，可以通过`name`,`tag`, `pinned`来查看。

此时，eBPF 程序已经被加载到 Kernel，接下来，我们来为它 attach 一个 Event。

#### 3.2.4.Attaching to an Event

eBPF 程序的类型和 Event 类型需要匹配，上述 eBPF 程序的类型是 XDP，所以我们需要将它 attach 到网卡的 XDP 事件上：

```bash
# bpftool net attach xdp id 103 dev enp0s1
```

此时，"Hello”已经被 attach 到了网卡 enp0s1，并监听 XDP 事件，可以通过 `bpftool` 来观察下：

```bash
# bpftool net list
xdp:
enp0s1(2) driver id 103

tc:

flow_dissector:
```

可以看到除了 XDP 以外，还可以绑定到其他的网络栈 event。

通过  `ip a `命 令也可以观测到网卡的attach情况，也能通过 `ip link` 命令来从网卡上绑定和解绑XDP程序：

```bash
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp/id:103 qdisc fq_codel state UP group default qlen 1000
    link/ether 1e:9e:30:12:f3:49 brd ff:ff:ff:ff:ff:ff
    inet 192.168.64.5/24 metric 100 brd 192.168.64.255 scope global dynamic enp0s1
       valid_lft 49211sec preferred_lft 49211sec
    inet6 fd3b:5ef9:8dd6:52de:1c9e:30ff:fe12:f349/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591996sec preferred_lft 604796sec
    inet6 fe80::1c9e:30ff:fe12:f349/64 scope link 
       valid_lft forever preferred_lft forever
```

此时，"Hello”程序已经开始监听网卡上的每个数据包了：

```bash
# cat /sys/kernel/debug/tracing/trace_pipe 
          <idle>-0       [003] d.s.. 80521.616068: bpf_trace_printk: Hello World 1929
          <idle>-0       [003] d.s.. 80521.616338: bpf_trace_printk: Hello World 1930
            node-1417    [003] d.s1. 80521.617439: bpf_trace_printk: Hello World 1931
          <idle>-0       [003] d.s.. 80521.617684: bpf_trace_printk: Hello World 1932
          <idle>-0       [003] d.s.. 80521.620113: bpf_trace_printk: Hello World 1933
          <idle>-0       [003] d.s.. 80521.622659: bpf_trace_printk: Hello World 1934
          <idle>-0       [003] d.s.. 80521.622664: bpf_trace_printk: Hello World 1935
          <idle>-0       [003] d.s.. 80521.626576: bpf_trace_printk: Hello World 1936
          <idle>-0       [003] d.s.. 80521.626584: bpf_trace_printk: Hello World 1937
          <idle>-0       [003] d.s.. 80521.626585: bpf_trace_printk: Hello World 1938
          <idle>-0       [003] dNs.. 80521.629195: bpf_trace_printk: Hello World 1939
          <idle>-0       [003] d.s.. 80521.629823: bpf_trace_printk: Hello World 1940
# bpftool prog tracelog
```

与之前 BCC 框架运行的结果所有差异，大部分的网络包都是直接发送到网卡，没有用户态调用 syscall的过程，所以前面的UID是 `<idle>-0`，在这个例子中，我们采用了一个全局变量 `counter` 来计数，下面我们来看下全局变量的 maps 实现。

#### 3.2.5.Global Variables

在之前的章节中提到过，eBPF map 是用户用户态程序到 eBPF 内核态程序传递信息或不通eBPF程序间传递信息的数据结构。在2019年后，eBPF 程序可以直接通过定义全局变量的来实现 map。

通过 `bpftool` 可以查看加载到Kernel中的所有 BPF maps：

```bash
# bpftool map list
3: array  name hello_ne.bss  flags 0x400
        key 4B  value 4B  max_entries 1  memlock 4096B
        btf_id 118
# bpftool map dump name hello_ne.bss
[{
        "value": {
            ".bss": [{
                    "counter": 60315
                }
            ]
        }
    }
]
```

每跑一次这个命令，都可以发现  `counter ` 的值在增加，即eBPF程序中"每一个XDP event增加一次 couter 的值"。

*Tips: 只有在编译的时候带上 `-g`(Generate source-level debug information)时，BTF信息才是可用了，才能显示这些信息。*

#### 3.2.6.Detaching& Unloading the Program

可以使用以下命令来从网卡上 Detach eBPF程序：

```sh
# bpftool net  detach xdp dev  enp0s1
# bpftool net list
xdp:

tc:

flow_dissector:
```

此时，Kernal中仍然加载了这个eBPF程序：

```bash
# bpftool prog show name hello
103: xdp  name hello  tag 4ae0216d65106432  gpl
        loaded_at 2023-05-29T10:58:52+0000  uid 0
        xlated 168B  jited 200B  memlock 4096B  map_ids 3
        btf_id 118
```

通过以下方式从Kernel中卸载：

```bash
# rm /sys/fs/bpf/hello
# bpftool prog show name hello
```

### 3.3.BPF to BPF Calls

在之前的章节中，我们介绍了有关  `tail calls`  的使用，除了 tail calls ，在原生的BPF程序中，我们可以直接使用 BPF calls，下面的例子我们通过attach到 `sys_enter`的 tracepoint，通过在eBPF程序中调用其他eBPF程序函数来跟踪syscall的opcode：

```C
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

// 获取opcode的函数
static __attribute((noinline)) int get_opcode(struct bpf_raw_tracepoint_args *ctx) // 强制编译器不进行内联编译
{
    return ctx->args[1];
}

SEC("raw_tp/")

int hello(struct bpf_raw_tracepoint_args *ctx)
{
    int opcode = get_opcode(ctx); // call func
    bpf_printk("Syscall: %d", opcode);
    return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";

```

编译并获取下 object 文件详情：

```Bash
# make hello-func
# bpftool prog load hello-func.bpf.o /sys/fs/bpf/hello
# bpftool prog list name hello
```

继续获取下该eBPF程序的字节码：

```bash
# bpftool prog dump xlated name hello
int hello(struct bpf_raw_tracepoint_args * ctx):
; int opcode = get_opcode(ctx);
   0: (85) call pc+7#bpf_prog_cbacc90865b1b9a5_get_opcode
; bpf_printk("Syscall: %d", opcode);
   1: (18) r1 = map[id:11][0]+0
   3: (b7) r2 = 12
   4: (bf) r3 = r0
   5: (85) call bpf_trace_printk#-73360
; return 0;
   6: (b7) r0 = 0
   7: (95) exit
int get_opcode(struct bpf_raw_tracepoint_args * ctx):
; return ctx->args[1];
   8: (79) r0 = *(u64 *)(r1 +8)
; return ctx->args[1];
   9: (95) exit
```

在第三行中，我们可以看到eBPF程序 `hello()`调用了函数 `get_opcode()`，偏移量（offset）0处的指令是 `0x85`，`0x85` 在eBPF指令集中的意义是 `Function call（call imm）`，所以，该处不会直接往下执行 offset 1 的指令，而是向前跳转7条指令，执行 offset=8 的指令。

在offset=8的处，可以看到 `get_opcode` 函数。

*Tips：函数调用指令需要将当前状态放到 BPF 虚拟机的堆栈中，以便当被调用函数退出时，可以在调用函数中继续执行。由于堆栈大小限制为 512 字节，因此 BPF 到 BPF 的调用不能嵌套得很深。*



## 4.The bpf() System Call

在之前的章节中，我们在用户态使用 `bpf()` 系统调用与内核运行的 BPF 程序进行交互，本章，我们重点介绍 `bpf()` 这个系统调用。

需要注意的是，在内核态中运行的 eBPF 程序之间不通过 `bpf()` 进行交互，而是通过 helper 函数（之前已经介绍过）来互访 BPF maps。

先来看下 `bpf()` 的函数签名(https://man7.org/linux/man-pages/man2/bpf.2.html)：

```C
int bpf(int cmd, unionbpf_attr *attr, unsignedint size);
```

第一个参数是 `cmd`，指定操作eBPF的指令。有很多命令可以操作 eBPF 程序，比如创建 maps，attach 程序的事件，访问 map 的 k-v：

![image-20230612120053059](/images/image-20230612120053059.png)

参数 `attr` 指定了 command 的参数，size指定了 `attr` 的大小（bytes）。

下面来看一个使用BCC框架调用 `bpf()` 的例子（和之前使用 perf buffer 的"hello world"例子有点类似，这次加了一个hash map来存UID, eBPF代码部分：

```C
struct user_msg_t {
   char message[12];
};

BPF_HASH(config, u32, struct user_msg_t); // BCC macro "BPF_HASH"用于声明一个hash table，存user_msg_t，key类型是u32（UID正好是32位，如果不指定，默认u64）

BPF_PERF_OUTPUT(output); 

struct data_t {     
   int pid;
   int uid;
   char command[16];
   char message[12];
};

int hello(void *ctx) {
   struct data_t data = {}; 
   struct user_msg_t *p;
   char message[12] = "Hello World";

   data.pid = bpf_get_current_pid_tgid() >> 32;
   data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

   bpf_get_current_comm(&data.command, sizeof(data.command));

   p = config.lookup(&data.uid); // 当config的hash map中获取到UID时，打印对应的mesasge，未获取到打印"Hello World"
   if (p != 0) {
      bpf_probe_read_kernel(&data.message, sizeof(data.message), p->message);       
   } else {
      bpf_probe_read_kernel(&data.message, sizeof(data.message), message); 
   }

   output.perf_submit(ctx, &data, sizeof(data)); 
 
   return 0;
}
```

用户代码部分：

```py
b = BPF(text=program) 
syscall = b.get_syscall_fnname("execve")
b.attach_kprobe(event=syscall, fn_name="hello")
b["config"][ct.c_int(0)] = ct.create_string_buffer(b"Hey root!") # 在config hash table中加入一个key为0，即UID为0时，打印Hey root!
 
def print_event(cpu, data, size):  
   data = b["output"].event(data)
   print(f"{data.pid} {data.uid} {data.command.decode()} {data.message.decode()}")
 
b["output"].open_perf_buffer(print_event) 
while True:   
   b.perf_buffer_poll()
```

跑一下：

```bash
# /usr/bin/python3 /root/workspace/ebpf-demo/s4/hello-buffer-config.py
68312 0 node Hey root!
68313 0 sh Hey root!
68314 0 node Hey root!
68315 0 sh Hey root!
68316 0 node Hey root!
68317 0 sh Hey root!
68318 0 node Hey root!
68319 0 sh Hey root!
68320 0 cpuUsage.sh Hey root!
```



### 4.1.Loading BTF Data

通过 `strace` 命令来看下 `bpf()` syscall：

```	bash
# strace -e bpf ./s4/hello-buffer-config.py 
bpf(BPF_BTF_LOAD, {btf="\237\353\1\0\30\0\0\0\0\0\0\0\364\5\0\0\364\5\0\0#\v\0\0\1\0\0\0\0\0\0\10"..., btf_log_buf=NULL, btf_size=4399, btf_log_size=0, btf_log_level=0}, 128) = 3
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_PERF_EVENT_ARRAY, key_size=4, value_size=4, max_entries=4, map_flags=0, inner_map_fd=0, map_name="output", map_ifindex=0, btf_fd=0, btf_key_type_id=0, btf_value_type_id=0, btf_vmlinux_value_type_id=0, map_extra=0}, 128) = 4
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_HASH, key_size=4, value_size=12, max_entries=10240, map_flags=0, inner_map_fd=0, map_name="config", map_ifindex=0, btf_fd=3, btf_key_type_id=1, btf_value_type_id=4, btf_vmlinux_value_type_id=0, map_extra=0}, 128) = 5
bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_KPROBE, insn_cnt=44, insns=0xffff858c3be8, license="GPL", log_level=0, log_size=0, log_buf=NULL, kern_version=KERNEL_VERSION(5, 15, 98), prog_flags=0, prog_name="hello", prog_ifindex=0, expected_attach_type=BPF_CGROUP_INET_INGRESS, prog_btf_fd=3, func_info_rec_size=8, func_info=0xaaaae38880f0, func_info_cnt=1, line_info_rec_size=16, line_info=0xaaaae3e9bd20, line_info_cnt=21, attach_btf_id=0, attach_prog_fd=0, fd_array=NULL}, 128) = 6
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=5, key=0xffff84de2410, value=0xffff84fd3390, flags=BPF_ANY}, 128) = 0
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, key=0xffff84de2410, value=0xffff84fd3390, flags=BPF_ANY}, 128) = 0
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, key=0xffff84de2410, value=0xffff84fd3390, flags=BPF_ANY}, 128) = 0
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, key=0xffff84de2410, value=0xffff84fd3390, flags=BPF_ANY}, 128) = 0
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=4, key=0xffff84de2410, value=0xffff84fd3390, flags=BPF_ANY}, 128) = 0
69248 0 node Hey root!
69249 0 sh Hey root!
```

先看第一个 `bpf()` 调用：

```bash
bpf(BPF_BTF_LOAD, {btf="\237\353\1\0\30\0\0\0\0\0\0\0\364\5\0\0\364\5\0\0#\v\0\0\1\0\0\0\0\0\0\10"..., btf_log_buf=NULL, btf_size=4399, btf_log_size=0, btf_log_level=0}, 128) = 3
```

在这个调用中，调用的系统命令是 `BPF_BTF_LOAD`，加载一个 BTF 数据的 blob 到Kernel，返回一个 BPF 数据的文件描述符（文件描述符用于给 read(), write() 这些syscall用，在本例中为 3，需注意，这里BPF数据不是一个真实的文件，仅用于作为其他系统调用的参数）。

### 4.2.Creating Maps

接下来一个 `bpf()` 调用创建了一个名为 `output` 的perf buffer map：

```bash
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_PERF_EVENT_ARRAY, key_size=4, value_size=4, max_entries=4, map_flags=0, inner_map_fd=0, map_name="output", map_ifindex=0, btf_fd=0, btf_key_type_id=0, btf_value_type_id=0, btf_vmlinux_value_type_id=0, map_extra=0}, 128) = 4
```

可以看出，创建 map 的指令是`BPF_MAP_CREATE`，且 map 的类型为`PERF_EVENT_ARRAY`，名为 "output"，key、value的长度是4字节，且最大能存储4个元素。返回值为文件描述符，值为4，这个可以供用户态的代码使用，来访问map。

再往下：

```bash
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_HASH, key_size=4, value_size=12, max_entries=10240, map_flags=0, inner_map_fd=0, map_name="config", map_ifindex=0, btf_fd=3, btf_key_type_id=1, btf_value_type_id=4, btf_vmlinux_value_type_id=0, map_extra=0}, 128) = 5
```

这里的`bpf()`系统调用又创建了一个`HASH TABLE`类型的map，Key为4字节（对应32位的UID），Value为12字节（message长度），名称为"config"，当未指定table大小时，BCC框架给定的默认大小为10240个元素。

在这个系统调用的参数中，还可以看到一个字段`btf_fd=3`，这个主要用作给 bpftool 展示 map，如果在 map 中存在 btf 的文件描述符，说明在K—V中存在BTF信息。

### 4.3.Loading a Program

在将 BTF 数据加载到Kernel，并创建好 BPF maps后，就开始加载 eBPF 程序了：

```bash
bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_KPROBE, insn_cnt=44, insns=0xffff858c3be8, license="GPL", log_level=0, log_size=0, log_buf=NULL, kern_version=KERNEL_VERSION(5, 15, 98), prog_flags=0, prog_name="hello", prog_ifindex=0, expected_attach_type=BPF_CGROUP_INET_INGRESS, prog_btf_fd=3, func_info_rec_size=8, func_info=0xaaaae38880f0, func_info_cnt=1, line_info_rec_size=16, line_info=0xaaaae3e9bd20, line_info_cnt=21, attach_btf_id=0, attach_prog_fd=0, fd_array=NULL}, 128) = 6
```

prog_type 声明了程序类型，即`KPROBE`；

insn_cnt 的意思是"instruction count"，即字节码指令的数量；

insns 是 eBPF 字节码指令的内存地址；

expected_attach_type 在 kprobe 类型中可以忽略；

prog_btf_fd 字段告诉内核之前加载 BTF 数据的哪个 blob 是该程序需要的。

返回 6 即该 eBPF程序的文件描述符。

| 文件描述符 | 意义                    |
| ---------- | ----------------------- |
| 3          | BTF 数据                |
| 4          | perf buffer map: output |
| 5          | hash map: config        |
| 6          | eBPF程序                |

### 4.4.Modifying Map From User Space

在这段用户态代码中，我们通过 python 程序，在config hash表中插入了一个uid：

```py
b["config"][ct.c_int(0)] = ct.create_string_buffer(b"Hey root!")
```

对应syscall：

```bash
bpf(BPF_MAP_UPDATE_ELEM, {map_fd=5, key=0xffff84de2410, value=0xffff84fd3390, flags=BPF_ANY}, 128) = 0
```

`BPF_MAP_UPDATE_ELEM`指令用于更新map中的K-V，`BPF_ANY` flag 用于指明如果key不存在，则创建。

map的文件描述符是5，即 config 这个hash map，Kernel中分配的文件描述符只能给特定进程用，所以 5 号描述符只能给指定的用户态的Python程序使用。然而，多个用户态程序（或者多个内核态eBPF程序）可以访问同一个map，文件描述符很可能不一样；访问不同map结构时，可能分到一样的文件描述符。

Key和Value都是指针类型，通过 strace 命令可以看到值，此外，通过 bpftool也可以看到值：

```
# bpftool map dump name config
[{
        "key": 0,
        "value": {
            "message": "Hey root!"
        }
    }
]
```

bpftool通过在 `BPF_MAP_CREATE` syscall中指定的 BTF 参数来获取BTF的信息，而BTF中就包含了k-v的结构、类型等。

当中断、退出eBPF程序时，Kernel通过 引用计数（reference counts） 来实现自动卸载eBPF程序和maps，下面来看下eBPF程序的回收。

### 4.5.BPF Program and Map Reference

上文中提到，当 eBPF 程序被加载到 Kernel 时，`bpf()` syscall 会返回一个文件描述符。在 Kernel 中，文件描述符是程序的一个引用（reference）。发起 syscall 的用户态程序拥有这个eBPF内核态程序的文件描述，当用户态进程退出时，文件描述符会释放，从而eBPF程序的引用计数（reference counts）会递减，当递减到0时，Kernel会移除这个eBPF程序。

当将 eBPF 程序 pin 到文件系统时，会增加一个额外的引用。

#### 4.5.1.Pinning

我们之前已经介绍过如何将一个 eBPF 程序 pin 到文件系统：

```bash
# bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

*Tips: pin到文件系统的对象并不是磁盘中的真实文件，它们在 pseudo 文件系统（/sys）中创建，虽然和常规文件几乎一样，但它们是在内存中的，重启后丢失。*

*<Reference部分，待补充XDP、BPF Links>*

### 4.6.Additional Syscalls Involved in eBPF

(略)

map

perf buffer  - ppoll

```bash
ppoll([{fd=8, events=POLLIN}, {fd=9, events=POLLIN}, {fd=10, events=POLLIN},{fd=11, events=POLLIN}], 4, NULL, NULL, 0) = 1 ([{fd=8, revents=POLLIN}])
```

attach kprobe events

ring buffer  - epoll

```bash
epoll_create1(EPOLL_CLOEXEC) = 
epoll_ctl(8, EPOLL_CTL_ADD, 4, {events=EPOLLIN, data={u32=0, u64=0}}) = 0
epoll_pwait(8,  [{events=EPOLLIN, data={u32=0, u64=0}}], 1, -1, NULL, 8) = 1
```



## 5.CO-RE, BTF, and Libbpf

上文，我们提到了有关 BTF（BPF Type Format），本章我们将了解为什么会有 BTF ， 它又是如何使得 eBPF 程序兼容不同版本的 Kernel 的，这是 BPF 的一个关键特性：一次编译，多处运行（compile once, run everywhere(CO-RE)）。

在深入 CO-RE 的工作原理之前，我们先看下 BCC 项目最初实现的 Kernel 可移植性方案。

### 5.1.BCC's Approach to Portability

BCC通过在目标机器的运行时编译来解决跨Kernel的移植性问题，但这种方式有很多问题：

1.编译工具包需要在每个目标机器上都安装，同样 Kernel header 文件也需要在目标机器上（很多时候是缺失的）。

2.编译会导致每次运行有延时。

3.每次编译会浪费大量的计算资源。

4.有些基于BCC的项目，会将 eBPF 源码和工具链打包成容器镜像进行分发，但仍然解决不了Kernel header文件的问题。

5.嵌入式设备上可能没有足够的内存来跑编译。

所以虽然上文例子中多次使用了 BCC 框架进行开发，但实际上，由于BCC框架移植性问题，不是非常建议在软件分发的场景下使用（[BCC 框架项目](https://github.com/iovisor/bcc)中的 libbpf-tools 目录中提供的工具采用了CO-RE，而 tools 目录中的则没有）。

### 5.2.CO-RE Overview

CO-RE的技术包含这部分：

**BTF：**一种描述数据结构和函数签名的格式。在 CO-RE 中，BTF的作用主要是判定数据结构在编译时和运行时的不同点。

**Kernel headers：**Linux 头文件在 Linux 源码中，维护了一些被eBPF程序使用的 Kernel 内部数据结构。Linux 头文件在不同的 Linux 版本存在一定的差异，eBPF 开发者可以选择单独的头文件，或者通过 bpftool 从运行的系统中生成头文件：`vmlinux.h`，其包含了一个 eBPF 程序可能会使用的所有 Kernel 内部数据结构。

**Compiler support：**目前 clang 编译器已经支持了 eBPF CO-RE，通过 `-g`参数，可以看到 BTF 描述的内核数据结构

**Library support for data structure relocations：**用户态程序将 eBPF 程序加载到 Kernel 时，需要基于编译在对象中的 CO-RE 迁移信息，来判定编译时和运行时的字节码差异，并进行字节码补偿，这个过程需要一些库的支持：比如 C库 `libbpf`, Golang 库 `Cilium eBPF`, Rust 库 `Aya`。

**a BPF skeleton：**在 BPF 程序被编译成 BPF object file时，BPF 代码框架（BPF skeleton）会自动生成，包括加载到内核、attach events等函数。如果用户态代码使用 C 编写，可以使用 bpftool命令（`bpftool gen skeleton`）来生成 BPF 代码框架，这比直接使用底层库（libbfp, cilium/ebpf）要方便很多。

#### 5.2.1.BTF Use Cases

<待补充，CO-RE原理>

<待补充，BP Verifier原理>

## 6.eBPF Program and Attachmenet Types

在之前的章节中，我们介绍了一些attach类型，例如 krpbes...实际上有超过30种 program types (https://elixir.bootlin.com/linux/v5.19.14/source/include/uapi/linux/bpf.h)，以及超过40种 attachment types，大部分 attachment types 和 program types 是一样的，但有些 programs types 可以 attch 多种 Kernel points，这种情况下， attachment types 也需要指定。

所有的 eBPF 都有一个指针类型的 context 参数，它指向的结构取决于触发的事件类型。eBPF 开发者需要编写代码来接收准确的 context 类型；假设 attach 到的是一个 tracepoint，但将上下文参数指向网络数据包，就没有意义。所以，通过定义不同的 program types，可以让 verifier来确保 context 信息是被正确处理，并且对应的 `helper functions` 是可以使用的。

### 6.1.Helper Functions and Return Codes

eBPF verifier 会校验和 program types符合的辅助函数，比如在程序类型为 XDP 的 BPF 程序中使用 `bpf_get_current_pid_tgid()`会被禁止，因为当数据库通过网卡时，不会有用户态的进程或者线程被调起，没有调用这个辅助函数是没有意义的。

program types 同样决定了程序的返回值的含义，还是之前 XDP 的例子，返回值表示了数据包通过网卡时，内核的处理方式，同样在被 tracepoint trigger 的 BPF 程序类型中就没有意义了。

通过命令 `bptftool feature` 可以查看每个辅助函数对应的程序类型。（helper functions手册：https://man7.org/linux/man-pages/man7/bpf-helpers.7.html#top_of_page）

Helper Function 是一种 *UAPI*（The userpace API），Linux Kernel外部稳定的接口，一旦一个 helper function 在 Kernel 中被定义好，即时 Kernel 内部的函数、数据结构发生变化，它都一般不会变。

除了外部接口外，在某些情况下 eBPF 程序也需要能够访问 Kernel 内部函数，这种访问方式称为 `kfuncs`（BPF kernel function）。

### 6.2.Kfuncs

Kfuncs 允许将 Kernel functions 注册到 BPF subsystem，每种 BPF 程序类型都会通过注册机与给定的 kfuncs 绑定。

不同于 Helper func， kfunc 不提供兼容性保障，也就是说开发者需要确保对应的 Kernel 支持对应内部函数。（reference：https://docs.kernel.org/bpf/kfuncs.html#core-kfuncs）

### 6.3.Tracing

从大类来看，Program types 分为两大类：tracing program types 和 networking-releated program types。

attach 到 kprobes、tracepoint、raw tracepoint、fentry/fexit probes 和 perf 的 events都被设计用于给内核中的 eBPF 程序 一种方便的方式来跟踪事件信息并传递给用户态程序。这些与跟踪相关的类型不会影响内核响应它们 attach 到的事件的行为方式。









