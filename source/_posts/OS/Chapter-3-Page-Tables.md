---
title: Chapter3 Page Tables
categories:
  - 计算机/操作系统
tags:
  - 操作系统
  - OS
  - 页表
keywords:
  - 操作系统
date: 2021-02-28 10:44:34
summary: 页表
---


> 操作系统通过页表（Page Tables）来向每个进程提供私有的地址空间和内存。页表决定了内存地址的含义，哪些物理内存可以访问。xv6使用页表分隔不同进程的地址空间，并把它们多路复用到单一的物理内存。xv6还使用页表实现了一些技巧：把同一块内存映射到多个地址空间(a trampoline page)，使用未映射的页来保护内核和用户的栈。本文主要介绍 RISC-V 硬件提供的页表功能以及 xv6 如何使用这些功能。

## 3.1 分页硬件（Paging hardware）

记住一点，RISC-V 指令（包括用户指令和内核指令）操作**虚拟地址**。机器的 RAM（或者叫做物理内存），是被物理地址索引的。RISC-V页表硬件通过把虚拟地址映射到物理地址的方式来把它们连接在一起。

xv6运行在 RISC-V 的 Sv39的分页方式，这种分页方式只使用64位虚拟地址里的低39位，高25位并未使用。在这39位有效位里，低12位是offset，剩下27位是index，这意味着每个页表在逻辑上对应着 $2^{27}(134,217,728)$ 个**PTE(Page Table Entry，页表入口)**。每个 PTE 包含一个 44 位的**物理页号（PPN, Physical Page Number）**和一些标志位。**每个PTE占用64位**，低54位有效，高10位是保留位。在这54位有效位里，低10位是flags，高44位是PPN(物理页号)。分页硬件使用39位有效位的高 27 位在页表中索引来找到对应的PTE，然后组成  56 位的物理地址，这 56 位里的高44 位来自 PPE 里保存的 PPN，低 12 位从虚拟地址的 12 位 offset 拷贝而来。下图 3.1 把一个页表看作一个简单的 PTE 数组来简单地展示这段处理逻辑。页表使得操作系统可以以 4096 字节块的粒度控制虚拟地址到物理地址的转换。这些 4096 字节块被称为页（page）。

![图 3.1 RISC-V 简单的物理地址和虚拟地址映射过程](https://gallery.angryberry.tech/blog/OS/3-1.png)

在 Sv39 RISC-V 中，虚拟地址的高 25 位不参与翻译，这些保留位是为了在未来做一些可能的扩展功能：可以用来支持更多层级的地址翻译等。物理地址也有增长余地：PTE的低十位给物理页号留有 10 位的增长空间。

在下图 3.2 中，实际的地址翻译分为三步。一个页表在内存中保存为拥有三级的树。

1. 首先，树的根节点是一个 4096 字节的页表页，包含 512 ($\frac{4096\times 8\ bits}{64\ bits/per\ PTE}=512$)个 PTE，每个 PTE 包含下一级页表的一页；
2. 第二级页表的每一页同样包含第三级页表 512 个 PTE

分页硬件根据虚拟地址的 27 位 index 的高 9 位用来选择树根的 PTE，中间 9 位用来选择二级页表页的 PTE，低 9 位用来选择最终的 PTE。

![图 3.2 RISC-V 地址翻译细节](https://gallery.angryberry.tech/blog/OS/3-2.png)

如果需要转换的虚拟地址的三级 PTE 中任一个不存在，分页硬件就会抛出一个页错误异常（Page Fault Exception），让内核来处理这个异常（chapter 4）。三级结构避免了页表记录所有页表项，因为通常情况下，大部分的虚拟地址都是没有映射的。

每条 PTE 有 10 位的 flag来告诉分页硬件该如何处理对应的虚拟地址，可以在 kernel/riscv.h 看到xv6对于它们的定义。上图 3.2 可以看到这些标志位及其含义。

- `PTE_V`的意思是此条 PTE 是否存在(1:存在，0:不存在)，如果不存在，对这一页会引发异常；
- `PTE_R`、`PTE_W`分别代表所对应的页是否可读、写。
- `PTE_X` 指明 CPU 是否可以将页面内容解析为指令并执行。
- `PTE_U`代表用户模式下是否可以访问此页。

为了使分页硬件能够找到页表，内核需要把**根页表页的物理地址放到 `satp` 寄存器里**。**每个CPU都有一个`satp`寄存器**。CPU会根据 `satp` 的内容找到页表并把接下来要执行的指令的地址翻译成物理地址。由于每个CPU都可以有自己私有的地址空间和私有的 `satp`，进而运行各自的进程。

> 一些值得注意的术语：
>
> 物理内存指的是 DRAM 中的存储单元，在物理内存中的地址成为物理地址。程序指令只能使用虚拟地址，因此需要借助分页硬件将其翻译成物理地址，然后发送给 DRAM 来进行读写。

## 3.2 内核地址空间（Kernel address space）

xv6 为每个进程维护一个页表，这个页表描述对应进程的用户地址空间（User address space），另外还有一个单独的页表描述内核地址空间。内核会配置自己的地址空间布局，是自己可以在可预测的虚拟地址访问物理内存以及多种硬件资源。下图 3.3 展示了此布局如何将内核虚拟地址映射到物理地址。`kernel/memlayout.h` 声明了 xv6 内核布局所需的常量值。

![图 3.3 左侧是 xv6 内核地址空间](https://gallery.angryberry.tech/blog/OS/3-3.png)

QEMU 模拟了一台包含 RAM (physical memory) 开始于物理地址 `0x80000000` ，并至少延续到 `0x86400000`（xv6 的 `PHYSTOP`） 的计算机。同时 QEMU 也模拟了一些 I/O 设备比如磁盘接口等。QEMU 以 内存映射 I/O （MMIO, Memory-mapped I/O）的形式向软件提供设备接口，设备的控制寄存器被映射在物理内存0~0x80000000之间，内核通过读写这部分内存与设备交互。第四章将对 MMIO 进行详细介绍。

内核的大部分虚拟地址都使用的是恒等映射（direct mapping，也可以叫直接映射），即将资源的虚拟地址直接映射到等值的物理地址。例如，内核的起始地址无论在虚拟地址还是物理地址都是`KERNBASE=0x80000000`。对于即需要读写虚拟页又需要通过PTE管理物理页的内核来说，这样的直接映射降低了复杂性。例如当 `fork` 为子进程分配用户内存时，分配器会返回那段内存的物理地址；`fork` 在为子进程拷贝父进程的用户内存时会直接使用此地址作为虚拟地址。

只有两处虚拟地址不是直接映射的：

- trampoline页（trampoline page）。它被映射在虚拟空间的顶端，它也被用户页表映射在用户空间的同样位置。chapter 4 会讨论这个页的作用。**它被内核虚拟地址空间映射了两次：一次是直接映射，一次是虚拟空间的顶端。**
- 内核栈所在的页（kernel stack page）。每个进程都有自己的内核栈，它被映射在高位，这样在它下面xv6可以保留一个未映射的**守护页**(guard page)。守护页的PTE是无效的(`PTE_V`为置位)，这就确保了即使内核让内核栈溢出，也仅仅是触发一个错误让内核panic。如果没有守护页，栈溢出将导致执行不正确的指令。与其这样，让内核panic是个更好的选择。

刚提到，内核使用 high-memory mapping 来使用栈，但是栈也可以被恒等映射的地址访问。可能你会想：是否可以直接使用恒等映射，然后在恒等映射的地址访问栈。答案是：否！ 把内核栈和守护页直接映射不可取，这样的话，守护页涉及到取消虚拟地址的映射，否则这些虚拟地址会访问物理内存，这导致守护页对应的物理内存将难以被使用。

对于内核的trampoline页和kernel text页，使用的权限是`PTE_R`和`PTE_X`，因为内核要从那些页里读取和执行指令。对于其它页则使用`PTE_R`和`PTE_W`，因为内核要从那些页里读写内存。对于保护页，则要将其设置为无效的。

## 3.3 代码：创建一个地址空间

xv6中负责管理地址空间和页表的代码主要在kernel/vm.c。核心数据结构是`pagetable_t`，它是RISC-V **根页表页的指针**；`pagetable_t`它要么是内核的页表，要么是某个进程的页表。核心函数是`walk`，它用来查找一个虚拟地址的PTE，或者用来查找`mappages`，`mappages`用来为新的映射安装PTE。以`kvm`开头的函数用来管理内核页表，以`uvm`开头的函数则用来管理用户页表，其它函数即可以管理内核页表也可以管理用户页表。`copyout`用来把数据复制到用户虚拟地址(由系统调用的参数提供)，`copyin`则是进行相反方向的复制；它们都定义在vm.c里，因为它们需要严格地转换这些地址以找到对应的物理地址。

在启动的早期，`main`函数调用了`kvminit`（kernel/vm.c:22）函数来创建内核的页表。此时 xv6 还没有开启 RISC-V 的分页功能，所以是直接使用的物理地址。`kvminit`首先分配了一个物理页用以保存根页表页。然后调用`kvmmap`来保存内核所需的转换。**这些转换包括了内核的指令和数据，至到`PHYSTOP`的物理内存，和实际设备的内存范围。**

`kvmmap`（kernel/vm.c:118）调用`mappages`（kernel/vm.c:149）来把一定范围虚拟地址对应的物理地址的映射关系保存到页表里。对于每个要映射的虚拟地址，`mappages`通过`walk`来找到对应`PTE`的地址。然后初始化`PTE`来保存相关的物理页号，所需的权限(`PTE_W, PTE_X`, and/or `PTE-R`)，以及 `PTE_V` 来标记该页有效（kernel/vm.c:161）。

`walk`（kernel/vm.c:72）在查找虚拟地址的 PTE 的时候模仿了分页硬件(paging hardware)，（参照图 3.2）。它是三级页表逐层查找（每次 9 位，指向下一级页表或者最终的页 kernel/vm.c:78）。如果查到的PTE无效，说明所需的页不存在；如果参数`alloc`设置为1，`walk`会分配新的页表页，并把它的物理地址放在PTE里。最终返回的是第三级的PTE的地址（kernel/vm.c:88）。

如上代码依赖于物理地址和虚拟地址的直接映射。例如，当 `walk` 按照页表层次查找时，会从一个 PTE 拉取下一级页表的物理地址（kernel/vm.c:80），然后使用这个地址作为得到下一级 PTE 的虚拟地址（kernel/vm.c:78）。

`main` 调用 `kvminithart`（kernel/vm.c:53）来使内核页表生效。它把根**页表页的物理地址保存在`satp`寄存器里**。然后CPU就会使用内核页表开始地址转换了。由于内核使用了一致性映射，当前虚拟地址的下一条指令将会映射在正确的物理地址上（the now virtual address of the next instruction will map to the right physical memory address）。

`main`调用 `procinit`（kernel/proc.c:26）来为每个进程分配内核栈。宏`KSTACK`用以生成每个内核栈的虚拟地址，它还为栈保护页(stack-guard pages)预留了空间。`kvmmap`把 PTE 加到内核页表里，`kvminithart`重新将内核页表载入 `satp`寄存器这样硬件才能识别新的PTE。

每个RISC-V CPU 都把页表缓存在TLB（转译后备缓冲区， Translation Look-aside Buffer）里，当 xv6 改变一个页表时，必须告知 CPU 使相应的 TLB 条目无效。如果不这样做的话，TLB 就可能会使用过期的缓存映射，从而指向一个错误的物理地址，这可能导致破坏其它进程的内存（因为此时此部分物理内存可能已经分配给其它进程了）。RISC-V 的指令`sfence.vma`用以刷新当前 CPU 的TLB以使新的页表生效。xv6在重新加载 `satp` 寄存器后，会在 `kvminithart` 执行`sfence.vma` 指令，并在返回用户空间之前 `trampoline` 代码中切换到用户页表（kernel/trampoline.S: 79）。

## 3.4 物理内存分配

内核必须在运行的时候(run-time)，为页表、用户内存、内核栈、管道缓存分配或释放物理内存。

xv6使用内核结尾到`PHYSTOP`之间的物理内存进行运行时(run-time)的分配。**它一次性地分配和释放整个4K的页。**它通过把空闲页插入一个链表里来保持对这些页的持续追踪。分配就是从链表里移除一个页，释放就是把空闲页再加到链表里。

## 3.5 代码：物理内存分配器

分配器(allocator)定义在 kernel/kalloc.c 里。**分配器的数据结构是由空闲物理页组成的链表**。每个空闲页在列表里都是`struct run`（kernel/kalloc.c:17）。分配器从哪里得到内存来保存这个数据结构呢？因为空闲页里什么都没有，所以空闲页的`run`数据结构就保存在空闲页自身里。这个空闲列表使用自旋锁（spin lock, kernel.kalloc.c:21-24）进行保护。链表和锁被包装在一个结构体中，以表明锁保护结构体中的字段。当下，可以忽略锁以及 `acquire` 和 `release` 调用，我们会在第六章详细介绍这部分内容。

`main`函数调用`kinit`来初始化分配器（kernel/kalloc.c:27）。`kinit`通过保存内核结束点之后到 `PHYSTOP` 之间的所有空闲页来初始化链表。xv6其实是可以通过分析配置信息来获取真实内存的大小的，但它没有这么做，而是假定系统只有128M的内存。`kinit`调用`freerange`来把空闲内存加到链表里，`freerange`是通过调用`kfree`把每个空闲页逐一加到链表里来实现此功能的。由于 PTE 只能指向 4K 对齐的物理地址，所以`freerange`使用宏`PGROUNDUP`来确保空闲内存是4K对齐的。分配器刚开始是无内存可用的，对`kfree`的调用使得它拥有了可以管理的内存。

分配器有时把地址作为整数以执行一些算法（例如，通过 `freerange` 遍历所有页），有时把它们作为指针以读写内存（例如，操作保存在每页的 `run` 结构体），这就使得分配器的代码里充满了强制类型转换。另外，分配和释放实际上也是改变了内存的类型。

`kfree`（kernel/kalloc.c:47）首先把内存的每个字节都填充为1，这是为了让之前使用它的代码不能再读取到有效的内容，期望这些代码能尽早崩溃以发现问题所在。然后`kfree`再把页加到空闲列表里，它把`pa`转换为指向 `struct run`的指针，把原空闲列表指向`r->next`，并使空闲列表等价于`r`。`kalloc`移除并返回空闲列表的第一个元素。

## 3.6 进程地址空间

每个进程有一个独立的页表。当 xv6 在进程间切换的时候，进程的页表也会跟着切换。正如图 2.3 所示，进程的**用户空间**从虚拟地址 0 到`MAXVA`（kernel/riscv.h:348），共计256G的内存($2^{38}$)。

当进程申请更多内存的时候，xv6首先用`kalloc`分配一个物理页,然后把这个物理页的 PTE 加到进程的页表里。xv6会设置该PTE对应的标志位: `PTE_W, PTE_R PTE_X,PTE_V`。大多数进程不会用到全部的虚拟地址空间，在 xv6 中未使用的 PTE 的 `PTE_V` 标志位是 0。

我们可以看到页表有如下好处：

1. 不同的页表把用户空间映射到不同的物理内存上，这样每个进程都有各自的用户空间。
2. 从进程的角度看虚拟地址是连续的，但实际上它所对应的物理地址却不必是连续的。
3. 内核把 `trampoline` 代码映射到地址空间的顶部，这样单一的物理内存就可以出现在所有的地址空间了，从而实现共享。

下图 3.4 展示了 xv6 中一个进程的用户内存布局。栈是一个单独的页，并展示了由 `exec` 创建的初始内容。在栈的最顶端，是命令行参数，和它们的指针数组。在这些参数的下面，是`main`的入口。

![带有初始栈的用户进程地址空间](https://gallery.angryberry.tech/blog/OS/3-4.png)



在栈的下面有一个保护页，它被设置为无效的，这样栈溢出的时候就会产生一个 `page-fault` 的异常。而真实世界的操作系统则可能会在栈溢出的时候给它分配更多的内存。

## 3.7 代码： sbrk

`sbrk` 是进程用于增加或减少进程内存的系统调用。这个系统调用是通过函数 `growproc` （kernel/proc.c:239） 实现的。`growproc` 根据 `n` 值的正或负来调用 `uvmalloc` 或者 `umvdealloc`。`uvmalloc` （kernel/vm.c:229）利用 `kalloc` 分配物理内存，并调用 `mappages` 把 PTE 加入用户页表。`uvmdealloc` 调用 `uvmunmap` （kernel/vm.c:174）使用 `walk` 找到 PTE 并调用 `kfree` 释放指向的物理内存。

xv6 的进程页表不仅告诉硬件如何映射虚拟地址，还记录了哪些物理内存页被分配给了当前进程。这也是为什么 `uvmunmap` 释放内存时需要检查用户页表。

## 3.8 代码： exec

`exec` 是创建用户地址空间的系统调用。它从文件系统的一个文件来初始化用户部分的地址空间。`exec` （kernel/exec.c:13）使用 `namei` （kernel/exec.c:26）打开指定路径path 的二进制文件，这会在第八章详细介绍。然后，读取文件的 `ELF` 头。xv6 程序使用 `ELF` 格式描述，见 kernel/efl.h。一个 `ELF` 二进制文件包含 `ELF` 头——`struct elfhdr` （kernel/elf.h:6），之后是一系列程序节头（program section header）——`struct proghdr`（kernel/elf.h:25）。每个 `proghdr` 都描述了必须加载到内存中的应用程序的一部分。xv6 程序只有一个 program section header，但是其他的系统可能会有多个独立的 section （比如，指令节，数据节等）。

第一步是检查文件包含 ELF binary。一个 ELF binary 以四字节“magic number”开始：`0x7F`, `E`, `L`, `F`（或者称为 `ELF_MAGIC` kernel/elf.h:3）。

`exec` 使用 `proc_pagetable` （kernel/exec.c:38）分配一个新的、没有用户映射的页表,使用 `uvmalloc` （kernel/exec.c:52）为每个 ELF 段（segment）分配内存，使用 `lodseg` （kernel/exec.c:10）将每个段加载到内存中。`loadseg` 利用`walkaddr` 找到已分配的内存地址，在这个物理地址写入每个 ELF 段的每个页，并使用 `readi` 从文件读取。

xv6 用 `exec` 创建的第一个用户程序 `/init` 的 program section 看起来如下：

```bash
# objdump -p _init
user/_init: file format elf64-littleriscv
Program Header:
	LOAD off 0x00000000000000b0 vaddr 0x0000000000000000
    			paddr 0x0000000000000000 align 2**3
		filesz 0x0000000000000840 memsz 0x0000000000000858 flags rwx
	STACK off 0x0000000000000000 vaddr 0x0000000000000000
				paddr 0x0000000000000000 align 2**4
		filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
```

program section header 的 `filesz` 可能比 `memsz`小，这意味着这二者之间的 gap 可能是 0 填充的（对于 C global variables 而言）而不是从文件读取的。对于 `/init` 而言，`filesz` 是 2112 字节，`memsz` 是2136 字节，因此，`uvmalloc` 分配了足够多的字节（2136 字节），而只从 `/init` 文件读取了 2112 字节。

现在，`exec` 分配和初始化用户栈。它只分配一个 stack page。`exec` 依次将参数字符串拷贝到栈的顶端，将指向它们的指针记录在 `ustack` 中。`exec` 在传递给 `main` 的参数列表 `argv` 的结尾放置一个空指针。`ustack` 的前三个 entry 是 fake return program counter, argc and argv pointer.

`exec` 在 stack page 的下方放置一个不可访问页，因此程序试图使用超过一页的栈就会 fault。这个不可访问页还可以允许 `exec` 处理参数过大的问题，在这种情况下，`exec` 用来拷贝参数的 `copyout ` （kernel/vm.c:355）函数会意识到目标页是不可访问的，于是返回 -1。

在准备新的内存映像（memory image）时，如果 `exec` 检测到一个错误，比如一个 invalid program segment，就会跳转到 bad 标签，释放新的映像并返回 -1。`exec` 必须等待 free the old image 直到确保系统调用成功：如果 old image 消失了，系统调用就不能为其返回 -1。`exec` 唯一的错误发生在创建 image 期间。一旦 image 创建完成，`exec` 可以提交新的页表（kernel/exec.c；113）并释放老的页表（kernel/exec.c:117）。

`exec` 从ELF 文件加载字节到 ELF 文件指定地址的内存中。用户或进程可以将任何它们想要的地址放入ELF 文件。因此，`exec` is risky，因为 ELF 文件中的地址可能有意或无意地指向内核。一个粗糙的内核可能导致崩溃或者被恶意软件劈坏隔离性（安全漏洞）。xv6 做了很多检查来规避这些风险。例如，`if(ph.vaddr + ph.memsz < ph.vaddr)` 检查是否会溢出 64 位整数。风险在于，用户可能构造一个ELF二进制文件，其中的`ph.vaddr`指向用户选择的地址，而`ph.memsz`足够大，以至于总和溢出到`0x1000`，这看起来像一个有效值。在xv6的旧版本中，用户地址空间也包含内核(但在用户模式下不可读/可写)，用户可以选择与内核内存对应的地址，从而将 ELF 二进制文件中的数据复制到内核中。在xv6的RISC-V版本中，这是不可能发生的，因为内核有自己单独的页表；`loadseg`加载到进程的页表中，而不是内核的页表中。

内核开发人员很容易忽略关键的检查，而且现实世界的内核很长一段时间以来都缺少检查，用户程序可以利用这些检查的缺失来获得内核特权。xv6很可能没有完成验证提供给内核的用户级数据的完整工作，恶意用户程序可能会利用这一点来绕过xv6的隔离。

## 3.9 Real World

Like most operating systems, xv6 uses the paging hardware for memory protection and mapping.
Most operating systems make far more sophisticated use of paging than xv6 by combining paging
and page-fault exceptions, which we will discuss in Chapter 4.
Xv6 is simplified by the kernel’s use of a direct map between virtual and physical addresses, and
by its assumption that there is physical RAM at address 0x8000000, where the kernel expects to be
loaded. This works with QEMU, but on real hardware it turns out to be a bad idea; **real hardware places RAM and devices at unpredictable physical addresses,** so that (for example) there might
be no RAM at 0x8000000, where xv6 expect to be able to store the kernel. More serious kernel
designs exploit the page table to turn arbitrary hardware physical memory layouts into predictable
kernel virtual address layouts.
RISC-V supports protection at the level of physical addresses, but xv6 doesn’t use that feature.
On machines with lots of memory it might make sense to use RISC-V’s support for “super
pages.” **Small pages make sense when physical memory is small,** to allow allocation and page-out
to disk with fine granularity. For example, if a program uses only 8 kilobytes of memory, giving
it a whole 4-megabyte super-page of physical memory is wasteful. **Larger pages make sense on**
**machines with lots of RAM, and may reduce overhead for page-table manipulation.**
The xv6 kernel’s lack of a malloc-like allocator that can provide memory for small objects
prevents the kernel from using sophisticated data structures that would require dynamic allocation.

Memory allocation is a perennial hot topic, the basic problems being efficient use of limited
memory and preparing for unknown future requests [7]. Today people care more about speed than
space efficiency. In addition, a more elaborate kernel would likely allocate many different sizes of
small blocks, rather than (as in xv6) just 4096-byte blocks; a real kernel allocator would need to
handle small allocations as well as large ones.