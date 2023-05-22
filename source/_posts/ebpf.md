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

# Learning eBPF总结

## 1.eBPF是什么，为何对于如今的操作系统如此重要

### 1.1.概述

eBPF是一种革命性的内核技术，允许开发者编写一些内核代码（例如重写内核函数），并动态加载到内核中，改变Kernel的行为。

通过eBPF可以缩短内核调用链路，改变System Call方式，并监控内核调用链路，比较常见的eBPF使用场景：

- 构建高性能网络
- 跟踪内核调用链
- 阻止不安全的内核调用（网络DDOS）

eBPF彻底改变了内核开发方式，内核至此诞生了无数可能性。

### 1.2.BPF、eBPF

一句话：**BPF（cBPF）只是一个包过滤器（tcpdump就是基于这玩意写的），只有两个32位的寄存器；而eBPF是基于事件的内核调用修改器，有10个64位寄存器**，支持JIT-Compilation、方便的用户态 <-> 内核态通信、maps支持、方便的helper func...

### 1.3.Kernel

用户应用一般存在于User space，通过高级语言调用System Call。

![image-20230512155031730](/images/image-20230512155031730.png)

exmaple（一个echo可以包含100多个系统调用）：

![image-20230512155241487](/images/image-20230512155241487.png)

因为Linux代码的复杂性，且Linus Torvalds的执着与Linux release数量日益增长，如果想增加额外的code来实现对System Call进行中断，观测“Open files、Network IO”等，几乎是不可能的。

![image-20230512155648348](/images/image-20230512155648348.png)

### 1.4.Kernel Modules

上文提到，向Kernel中提交代码是非常复杂且艰巨的，所以，Linus大佬也没把路给封死，另一种修改Kernel的方式是通过内核模块（Kernel Modules）。

Kernel Modules可以按需加载或卸载，并且独立于官方Linux kernel release发布，不需要merge在upstream branch上。

但是，Kernel Module还是有一定的危险性，原因很简单，不稳定的Kernel Module和内核程序一起运行，可能会导致Kernel crash，另外，由于Kernel Modules的作者来源不一定得到验证，无法确保是否安全（Kernel Modules运行在Kernel space，可以访问机器上所有东西）。所以大家对Kernel Modules的使用一直是非常谨慎的。

eBPF提供了一种非常不同的方式来确保安全性：[eBPF verifier](https://docs.kernel.org/bpf/verifier.html)，其只允许安全的eBPF程序（不会造成Kernel crash或者死锁）运行，也不对数据安全有所妥协。

### 1.5.Dynamic Loading of eBPF Programs

eBPF能够被Kernel动态加载，一旦eBPF attch到一个指定的event，会直接被event trigge到，不论这个发起这个event的进程是否在之前已经启动了，这样不需要重启机器就可以直接使用eBPF程序。

这促使eBPF可以形成具备超强的安全、观测工具集，它可以观测机器上发生的所有事，如果跑着容器里，和跑在host一样，能观测到容器里发生的所有事，史上最强CNI：Cilium就是这么诞生的。

另外，很重要的一点：引入内核功能再也不需要挨Linus大佬批了。

![image-20230512171104918](/images/image-20230512171104918.png)

### 1.6.High Perofrmance of eBPF Programs

eBPF程序添加检测的过程是相当高效的，当一个eBPF程序被加载并即时编译（JIT-compiled）后，程序会直接运行CPU机器指令。另外，在接受每个事件的时候，用户态到内核态的拷贝也是不存在的(hook在copy前)，所以几乎没有开销。

以基于eBPF的高性能内核包过滤器：XDP（eXpress Data Path）为例，传统的包过滤都是在服务进程侧进行过滤（例如iptables），其经过了内核态解包，copy到用户态，性能会很差：

<img src="/images/netstack.png" alt="img" style="zoom: 50%;" />

而XDP将eBPF的event hook放在网卡驱动侧，在分配skb之前就可以直接对包进行过滤，通过XDP进行包过滤、转发，其性能是ipvs的4.3倍（iptables更不用说了）。

<img src="/images/netstackebpf.png" style="zoom:50%;" />

### 1.7.eBPF in Cloud Native Environments

如今，多数用户选择以云原生的方式运行eBPF程序，比如容器、K8S或者公有云的Serverless服务（AWS Lambda）等等；

好处显而易见，节点上的容器share同一个kernel，例如，在K8S中，所有的Pods共享同一个kernel，所以我们只需要起一个容器，就可以观测节点上所有workloads：

![image-20230512174830637](/images/image-20230512174830637.png)

利用节点上所有进程的观测能力，以及eBPF程序的动态编译能力，一些eBPF-based的工具可以在集群中发挥superpower：

- 应用程序不需要做任何改动、配置，就可以接入eBPF工具集。
- 当eBPF程序被动态加载时，不需要重启节点或应用，就可以对已经运行的应用进行观测。

我们知道，一般在K8S里，我们可以通过给container挂sidecar来对网络流量、进程进行过滤、追踪、转发，实现service mesh、logging等功能。这种方式相较于直接在代码中调用对应的SDK来说，更为方便，对代码无侵入性，但是，sidecar其实存在很多缺陷：

- 当为Pod插入Sidecar时，Pod需要重启。

- 往往需要修改应用的yaml，才可以插入sidecar。例如，我们一般通过添加annontation的方式来触发injector来修改某个workload的yaml，插入sidecar，整个过程是自动的，但如果这个workload（比如deployment），漏加了annontation，或者没有加某个label，这个sidecar就不会注入，从而无法观测这个应用了。

- 当一个Pod中存在多个Container时，由于彼此达到Readiness的时间点是无法预测的，sidecar的注入会可能导致容器启动速度下降，甚至会导致竞争情况。

- 当网络功能（例如service mash）以sidecar的方式注入时，整条网络路径非常冗长，会导致出现抖动、丢包的情况：

  ![image-20230512193453205](/images/image-20230512193453205.png)

幸运的是，我们可以通过eBPF来解决sidecar模式的诸多问题，原因还是在于eBPF可以观测节点上发生的所有事件，不论是否有sidecar的注入，依赖eBPF，可以解决一系列因sidecar带来的不稳定与安全问题。



## 2.eBPF's "Hello World"

### 2.1. Run first eBPF Program

eBPF如此强大，我们从一个简单的例子来编写eBPF应用。

有很多lib库和框架可以帮助我们编写一个eBPF应用，例如[BCC Python framework](https://github.com/iovisor/bcc)，提供了相当简单的方式去编写eBPF程序。

首先，我们需要安装BCC框架，可以参考官方文档：https://github.com/iovisor/bcc/blob/master/INSTALL.md。

以下是一个“Hello World”的demo：

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

1.eBPF内核态程序

2.eBPF用户态程序

![image-20230518193033191](/images/image-20230518193033191.png)

![image-20230518190719115](/images/image-20230518190719115.png

eBPF有一组`helper function`来与内核进行交互，在demo中，包含一个`helper function`: `bpf_trace_printk()`，用来输出一行信息到trace_pipe（/sys/kernel/debug/tracing/trace_pipe）文件中。eBPF代码本身是通过C来编写的，理论上需要进行编译才可以运行（实际上单独编写一个eBPF程序也需要编译），但当通过BCC框架来开发时，不需要额外进行编译操作，只需要声明一个BPF object：

```b = BPF(text=program)```

eBPF程序还需要attach到一个event上，在demo中attach到了`execve`这个syscall上，当任何一个可执行程序被执行时，这个syscall都会被调用，接着就会trigger eBPF程序。一般来说，syscall会根据系统架构的不同，而带一些特定的前缀，例如：

```
"sys_",
"__x64_sys_",
"__x32_compat_sys_",
"__ia32_compat_sys_",
"__arm64_sys_",
"__s390x_sys_",
"__s390_sys_",
```

BCC框架提供了get_syscall_fnname的函数来获取对应的真实syscall名称，所以我们就不需要考虑系统架构了：

```syscall = b.get_syscall_fnname("execve")```

接下来，我们通过需要通过内核的[kprobes](https://docs.kernel.org/trace/kprobes.html)（在05年的时候被加入Kernel，通过这个功能可以允许开发者在几乎所有的指令上增加trap）功能来attach到一个syscall的events上:

```b.attach_kprobe(event=syscall, fn_name="hello")```

最后，通过一个死循环来hang住程序，并获取trace_pipe中的内容，就可以开始监听内核调用了。

执行下（很多都是vsocde server执行的程序）：

![image-20230518193303378](/images/image-20230518193303378.png)

```
Tips:
如果不是通过root运行，可能会没有权限，eBPF必须的三个内核capabilitied分别为：
CAP_BPF(https://mdaverde.com/posts/cap-bpf/), CAP_PERFMON, CAP_NET_ADMIN
```

在输出结果中，除了“Hello World”外，还有一些额外的event上下文信息，包括调用程序，process ID等。

但这个demo其实有很多问题，除了几乎不能做除了观测以外的其他事，最重要的是，这个demo无法获取eBPF程序本身的信息，当多个“Hello-World”同时运行时，并不知道是“谁”往trace_pipe中塞了内容，所以，BPF maps登场了。



### 2.2.BPF maps

**map**是用于eBPF程序和用户态程序之间交互的数据结构（在绝大多数场景，BPF maps和eBPF maps是一个东西），比较典型的用法是：

1.eBPF程序加载用户态程序配置

2.eBPF程序间传递状态

3.eBPF将结果传递到用户态程序

在Kernel中定义了很多[BPF maps类型](https://elixir.bootlin.com/linux/v5.15.86/source/include/uapi/linux/bpf.h#L878)，可以在[kernel文档](https://docs.kernel.org/bpf/map_bloom_filter.html)中找到他们的说明。一般来说，maps是以KV的形式存储的，但同样可以被扩展为hash表、perf、ring buffers或者数组。

被扩展成数组的map，由一个4字节的key构成；而hash表类型的map可以存在任何类型的key。一些map类型也被扩展成一些周知的类型，例如先入先出队列，先入后出栈，最长匹配队列，或者Bloom filters。

有一些常用的BPF maps存在一些特定的对象，例如：

[sockmaps](https://lwn.net/Articles/731133/)和[devmaps](https://docs.kernel.org/bpf/map_devmap.html)存放了sockets和网络设备相关的信息，可以给网络功能相关的eBPF程序所使用，用来重定向流量；

数组类型的map存放了eBPF程序的index，可以用于和其他eBPF程序交互；

一些map类型还会分散在不同的内存块上，由不同的CPU来读写，一般用于并行场景，使用这种map时，需要考虑并行锁的问题（Spin lock）。

接下来，我们通过一个例子来使用一个hash表类型的BPF map，通过BCC框架，我们可以很简单地去扩展一个hash表。

#### 2.2.3.Hash Table Map

在“Hello World”的基础上，我们将hello函数进行扩展，定义出一个hash表来存放user ID和对应的user调用```execve```的次数。

BPF程序：

```C
BPF_HASH(counter_table); // BPF_HASH是用来定义hash table的宏

int hello(void *ctx) {
    u64 uid;
    u64 counter = 0;
    u64 *p;
    // bpf_get_current_uid_gid是一个helper func，用来定位被kprobe event trigger到的user ID, user ID的前32位是group ID，我们通过32为掩码过滤
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

我们仔细观察这段“C”代码：

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

下面我们以`Perf Buffer`为例，来将“hello world”进行一版改进，不再将`execve`写入trace文件，而通过`Perf Buffer`来传递。

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
   bpf_probe_read_kernel(&data.message, sizeof(data.message), message); // 在这里例子里，在内核中的message“Hello Wrold”，被传递到data结构体中
 
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

从上面几个例子可以看到，在BPF程序中，我们调用了很多Kernel的helper function，从程序编写的风格上说，我们应该将自己的公共代码逻辑也包装为函数。当在BPF程序的早期，BPF程序是不允许调用除了helper function以外其他函数的（进栈限制），这就给编码带来了很多不便，所以，早期采用了一种workaround的方式来处理。（总是定义内联函数）“Always inline“：

```C
static __always_inlinevoid my_function(void *ctx, int val)
```

内联函数是C语言中可以定义的一种特殊函数，一般来说，函数被编译后的执行方式是“函数初始位置（堆） - > 执行函数（进栈） -> 返回初始位置（出栈，返回堆）”。当使用内联函数时，编译器会将函数代码指令直接复制到堆中。

![image-20230522124958075](/images/image-20230522124958075.png)

实际上，由于编译器的优化，一些内联函数会被强制编译为栈函数，而一些栈函数也会被强制编译为内联函数（所以有些Kernel function没法被kprobe attatch），这会导致我们在使用内联上存在一定的问题。

在Kernel 4.16（LLVM 6.0）后，内核取消了对函数栈调用的限制，不需要通过“Always inline”的方式来workaround了，这个特性被称为“BPF function calls”或者“BPF subprograms”（BPF子程序）。但目前BCC框架并不支持类型BPF function calls的功能，取而代之的将复杂能力分解的方式：tail calls。

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



## 3.深入解析eBPF程序

在实际大型项目中，除了通过BCC框架，我们仍然需要编写相对复杂的原生eBPF程序。本章中，我们会使用完整的C代码来编写“Hello World”。

一个eBPF程序实际上是一组eBPF字节码指令，一般会通过高级语言编写（当然你可以直接写字节码～），最终编译为eBPF字节码。eBPF程序用的最多的高级语言是C，也有一些eBPF程序用Rust编写，因为Rust编译器也支持eBPF字节码：

![image-20230522160351303](/images/image-20230522160351303.png)

字节码最终会跑在kernel的eBPF虚拟机中。



### 3.1.The eBPF Virtual Machine

eBPF虚拟机和其他虚拟机一样，是通过软件实现的一个计算机。在这个“计算机”里，能够运行eBPF的字节码指令，这些字节码指令最终会转化为在CPU上跑的原生机器指令。

在eBPF的早期实现中，eBPF字节码会直接被Kernel解析，一旦eBPF程序开始运行，Kernel会检查eBPF字节码并将其转换为机器码，然后再执行。目前，处于性能和安全性考虑，Kernel的字节码解释器已经被JIT（Just In Time）编译所取代，这意味着在程序被加载到内核时，只会被转换一次（即编译）。

eBPF字节码包含一组指令，这些指令会运行在eBPF（虚拟）寄存器（eBPF registers）上。eBPF指令集和寄存器模型与通用CPU架构相匹配，这样编译机器码的过程相对比较简单。

#### 3.1.1.eBPF Registers

eBPF虚拟机有10个一般用途的寄存器（0～9），还有一个10号寄存器，用来充当栈帧指针（只能读，不能写）。当BPF程序运行时，值会被存储到这些寄存器汇总以持续记录状态。

和CPU寄存器不同，这些寄存器是通过软件方式实现的，可以在[Kernel代码](https://elixir.bootlin.com/linux/v5.19.17/source/include/uapi/linux/bpf.h)中找到。

eBPF程序运行前，上下文参数会被加载到Register 1，返回值则会存储到Register 0。

在调用eBPF代码中函数前，函数的参数会存储到Register1到5中（如果低于5个参数，则其他寄存器不会使用）。

