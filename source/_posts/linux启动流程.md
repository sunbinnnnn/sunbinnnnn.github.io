---
title: Linux启动过程刨析
date: 2021-04-08 21:38:04
tags: system
---

Linux启动过程主要包含boot和startup两个阶段。

<!--more-->



\*本文不包含硬件相关的加载细节



### 引导(Boot)阶段

Boot阶段始于按下开机键或通过内核指令执行reboot操作，之后会经历以下过程：

#### 1.BIOS POST

第一阶段启动和Linux本身关系不大，主要围绕硬件启动部分，在绝大大数操作系统上行为是一致的。当计算机被启动时，首先运行的是POST（Power On Self Test 开机自检）过程，这是BIOS（Basic I/O System）的一部分。

当IBM在1981年设计了第一台PC开始，BIOS就被设计为初始化硬件的组件。POST是BIOS检查计算机硬件，并确保计算机硬件功能正常的阶段。如果POST失败了，计算机将无法正常启动，Boot程序也不会继续。

BIOS POST过程中会检查硬件的基本引导能力，当定位到可引导设备上的引导扇区时，会发出BIOS中断指令INT 13H中断BIOS POST过程。第一个被找到的引导扇区上包含了一个可用的的引导设备。第一个被找到的引导扇区将会包含一个可用的引导记录并被加载到内存，随后将引导的控制权转移从引导扇区加载的代码中。

加载引导扇区实际上是引导加载程序的第一阶段。大多数Linux发行版使用了三种引导加载程序：GRUB、GRUB2和LILO，目前用的最多的是GRUB2。

#### 2.GRUB2

GRUB2的全称是“GRand Unified Bootloader, version 2”，目前是Linux发行版中最主流的引导加载程序（bootloader）。GRUB2的作用是让计算机能够灵活得找到操作系统Kernel并且加载到内存中。（下文中将用GRUB指代GRUB2）。

GRUB被设计为多引导规范（[Multiboot specification](https://en.wikipedia.org/wiki/Multiboot_specification)）支持，这使得GRUB可以引导Linux的多数版本以及其他开放操作系统，也可以链接到其他专有操作系统的引导记录（例如通过GRUB加载Windows的引导程序）。

GRUB允许用户选择从任意给定的Linux发行版的不通内核中引导，如果因为内核版本变化导致引导失败，GRUB还支持了故障回滚机制，能够引导到之前的内核版本。GRUB可以使用 `/boot/grub/grub.conf` 进行回滚策略的变更。

以下将介绍GRUB2的三个引导阶段：

##### Stage 1

在BIOS POST的POST阶段的末尾，BIOS会搜索连接的磁盘上的引导记录，通常位于主引导记录（[MBR](https://en.wikipedia.org/wiki/Master_boot_record)）中，POST会找到第一个引导记录并加载到内存中，然后开始执行引导记录。GRUB2 stage 1即执行引导程序代码，这块被设计得非常小，因为它必须与分区表一起放在硬盘驱动器的第一个512字节扇区中。在经典的通用MBR中，为实际引导代码分区的空间量是446字节。Stage 1中的446字节文件名为`boot.img`，头部的446中不包含单独的引导分区。

由于引导记录大小的限制，使得其无法直接获取文件系统结构，因此，Stage 1阶段的唯一目的是定义并加载Stage 1.5，GRUB Stage 1.5阶段必须位于引导记录本身和驱动器第一个分区之间，GRUB Stage 1.5被加载到Ram后，控制权随即转为Stage 1.5

##### Stage 1.5

Stage 1.5的工作主要是加载寻找 /boot目录中Stage 2阶段所必须的驱动。

GRUB Stage 1.5必须位于引导记录本身与磁盘驱动器第一个分区之间的空间中。硬盘驱动器的第一个分区从扇区63开始，而MBR在扇区0剩下的62个扇区（512字节-31744字节）中存放了执行Stage 1.5的`core.img`，`core.img`大小被设计为25389字节，可以被存放在MBR和磁盘第一个分区之间。

对于Stage 1.5来说，它能够存放比Stage 1多的多的代码，一般来说，一些常见文件系统的驱动程序会存放在此（例如EXT、FAT、NTFS等）。GRUB2的core.img比GRUB1更复杂，功能更强大，GRUB2的core.img可以直接位于标准EXT文件系统上（不支持放在逻辑卷上），所以Stage 2的标准路径是`/boot` 目录，一般是`/boot/grub2`。

需要注意的是，`/boot`目录必须创建在GRUB支持的文件系统上。

##### Stage 2



与GRUB1一样，GRUB2也支持从多种Linux内核之一进行引导。红帽软件包管理器DNF支持保留内核的多个版本，因此，如果最新版本的内核出现问题，则可以引导较旧版本的内核。默认情况下，GRUB提供已安装内核的预引导菜单，包括救援选项和恢复选项（如果已配置）。



GRUB Stage2阶段所需文件都在 `/boot/grub2`目录中，GRUB Stage2阶段中没有Stage1和1.5那样的image文件，主要会由运行时Kernel Modules构成，一般从`/boot/grub2/i386-pc`目录中加载。

GRUB Stage2的职责主要是定位Linux Kernel并加载到RAM中，并将整个控制权移交给kernel。

Kernel及其相关文件都位于/boot目录中，Kernel文件一般以vmlinuz开头命名，通过列出 /boot 目录的内容，可以查看当前系统安装的内核。

GRUB2 和 1都支持多内核引导。RedHat的包管理系统DNF支持在仓库中保留多个Kernel版本，如果最新版本内核出现问题，可以通过GRUB提供的内核预引导菜单（pre-boot menu），引导前一个版本的内核。



##### Kernel

为了节省磁盘空间，所有的内核均采用自解压的压缩格式。内核、初始RAM磁盘映像（InitRamFs）以及硬盘的社区映射都位于 /boot目录中。

选定的内核被加载到内存开始执行时，首先进行自解压，提取可执行作业，并加载。内核提取后首先systemd，并将启动控制权进行交接。

Boot阶段到此全部结束，此时，Linux Kernel和systemd程序正在运行。



### 启动(Startup)阶段

Startup阶段和Boot阶段完成后随即开始，主要工作是让计算机能够真正执行生产作业。

##### 0号进程和1号进程

systemd是系统的1号进程，在1号进程启动前，系统会自动创建0号进程，0号进程的pid=0，也是唯一一个没有通过`fork`或者`kernel_thread`生成的进程。

0号进程是系统所有进程的先祖，进程描述符`init_task`是内核静态创建的，而它在初始化的时候通过`kernel_thread`的方式创建了两个内核线程，分别是`kernel_init`和`kthreadd`，其中`kernel_init`为1号进程。

0号进程创建1号进程的方式如下：

```c
kernel_thread(kernel_init, NULL, CLONE_FS);
```

我们发现1号进程的执行函数就是kernel_init，kernel_init函数将完成设备驱动程序的初始化，并调用init_post函数启动用户空间的init进程即systemd。

##### systemd

systemd是所有用户进程的祖先，它负责使Linux主机达到可以完成生产性工作的状态。它的某些功能比旧的init程序要更为广泛。

首先，systemd会挂载`/etc/fstab`定义的文件系统，包括所有交换分区和普通分区。此时，它可以访问/etc目录来确定系统配置相关信息。当找到配置文件`/etc/systemd/system/default.target`时，会使用该配置文件来确定将主机引导到的状态或目标。 `default.target`文件是指向真实目标文件的链接。对于带GUI界面的系统，通常将其作为`graphic.target`，等效于旧SystemV init中的runlevel 5。对于不带GUI解密的系统，默认值则为`multi-user.target`，类似于SystemV中的runlevel 3，`emergency.target`则与单用户模式相似。

下表是对systemd targets与的systemV runlevel的比较。 systemd target aliases 由systemd提供，用于向后兼容。target aliases 允许脚本使用init 3这样的SystemV命令来更改运行级别。SystemV命令将转发给systemd进行解释和执行。

| **SystemV Runlevel** | **systemd target** | **systemd target aliases** | **Description**                                              |
| -------------------- | ------------------ | -------------------------- | ------------------------------------------------------------ |
|                      | halt.target        |                            | Halts the system without powering it down.                   |
| 0                    | poweroff.target    | runlevel0.target           | Halts the system and turns the power off.                    |
| S                    | emergency.target   |                            | Single user mode. No services are running; filesystems are not mounted. This is the most basic level of operation with only an emergency shell running on the main console for the user to interact with the system. |
| 1                    | rescue.target      | runlevel1.target           | A base system including mounting the filesystems with only the most basic services running and a rescue shell on the main console. |
| 2                    |                    | runlevel2.target           | Multiuser, without NFS but all other non-GUI services running. |
| 3                    | multi-user.target  | runlevel3.target           | All services running but command line interface (CLI) only.  |
| 4                    |                    | runlevel4.target           | Unused.                                                      |
| 5                    | graphical.target   | runlevel5.target           | multi-user with a GUI.                                       |
| 6                    | reboot.target      | runlevel6.target           | Reboot                                                       |
|                      | default.target     |                            | This target is always aliased with a symbolic link to either multi-user.target or graphical.target. systemd always uses the default.target to start the system. The default.target should never be aliased to halt.target, poweroff.target, or reboot.target. |

每个target都有对应的配置文件可供配置，并且具备相互依赖的特性，下图描述了target的依赖关系。

```
local-fs-pre.target
            |
            v
   (various mounts and   (various swap   (various cryptsetup
    fsck services...)     devices...)        devices...)       (various low-level   (various low-level
            |                  |                  |             services: udevd,     API VFS mounts:
            v                  v                  v             tmpfiles, random     mqueue, configfs,
     local-fs.target      swap.target     cryptsetup.target    seed, sysctl, ...)      debugfs, ...)
            |                  |                  |                    |                    |
            \__________________|_________________ | ___________________|____________________/
                                                 \|/
                                                  v
                                           sysinit.target
                                                  |
             ____________________________________/|\________________________________________
            /                  |                  |                    |                    \
            |                  |                  |                    |                    |
            v                  v                  |                    v                    v
        (various           (various               |                (various          rescue.service
       timers...)          paths...)              |               sockets...)               |
            |                  |                  |                    |                    v
            v                  v                  |                    v              rescue.target
      timers.target      paths.target             |             sockets.target
            |                  |                  |                    |
            v                  \_________________ | ___________________/
                                                 \|/
                                                  v
                                            basic.target
                                                  |
             ____________________________________/|                                 emergency.service
            /                  |                  |                                         |
            |                  |                  |                                         v
            v                  v                  v                                 emergency.target
        display-        (various system    (various system
    manager.service         services           services)
            |             required for            |
            |            graphical UIs)           v
            |                  |           multi-user.target
            |                  |                  |
            \_________________ | _________________/
                              \|/
                               v
                     graphical.target
```

根据依赖拓扑完成启动后，Linux系统结束启动过程。



### 附录：关于initramd的定制

解压：

```xz -dc < initrd.img| cpio -idmv```

打包：

```find . 2>/dev/null | cpio -c -o | xz -9 --format=xz --check=crc32 > /tmp/new.img```



### 引用

https://opensource.com/article/17/2/linux-boot-and-startup

https://blog.csdn.net/gatieme/article/details/51532804

https://www.golinuxcloud.com/update-rebuild-initrd-image-centos-rhel-7-8/#Method_2_Extract_initrd_image