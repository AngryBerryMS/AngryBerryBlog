---
title: GDB For XV6
categories:
  - 计算机/操作系统
tags:
  - 操作系统
  - OS
  - GDB
keywords:
  - 操作系统
  - GDB
date: 2021-02-22 22:30:39
summary: xv6 的 GDB 调试
---



> 课程已经提供了 `.gdbinit` 来自动为 QEMU 配置 GDB 环境。
>
> - 需要从实验代码的根目录运行 GDB
> - Edit ~/.gdbinit to allow other gdbinits

##  1. GDB 常用指令

### Help

执行 `help <command-name>` 来查看指令的用法。

所有的指令都可以在没有歧义的前提下缩写，例如

- c = co = cont = continue
- s = step
- si = stepi

### 单步执行

- `step` 指令每次执行一行指令。当遇到函数调用的时候，将会进入被调用的函数（step into）。
- `next` 指令也是每次执行一行指令，但是遇到函数调用时，不会进入被调用的函数（step over）。
- `stepi` 和 `nexti` 指令为汇编指令执行单步。

> 所有上述指令都可以在后面追加一个参数 `[N]` 来指定重复的次数，比如 `step 10` 指明执行10 次。
>
> 不输入任何指令，直接按下 `Enter` 键是重复执行上一条指令的意思。 

### 执行

- `continue` 执行代码直到遇到断点或者用户按下 `Ctrl+C` 中断执行‘
- `finish` 执行代码直到当前函数返回
- `advance <location>` 执行代码到指定的位置

### 断点

- `break <location>` 在指定位置设定断点，每个断点会有一个断点序号
  - `location` 可以是具体的内存地址或者函数名字或者行号
- 使用 `delete`，`disable` 和 `enable` 修改断点
  - delete 断点编号（通过info break查到）：delete 2
  - delete 起始断点-终止断点：delete 1-5
- 条件断点
  - `break <location> if <condition>`
  - `condition <break number> <condition>`

### 观测断点

- `watch <expression>` 当 expression 的值变化是停止执行
- `watch -l <address>` 当指定地址的内容发生变化时停止执行
- `rwatch [-l] <expression>` 当每次 expression 的值被读取时停止执行

### 检验数据

- `x` 指令可以按照指定格式打印指定内存处的原始内容
  - `x/x` 以十六进制格式打印
  - `x/i` for assembly
- `print` 可以给 C 表达式求值并打印结果。通常这比 `x` 指令更加有用
- `info registers` 打印所有寄存器的值
- `info frame` 打印当前所有的栈帧
- `list <location>` 打印指定位置的函数代码
- `backtrace` 回溯函数调用栈。例如：当出现段错误的时候，执行backtrace，可以看到是哪里的调用，产生的这个段错误。
- `layout` 可以输出一个文本用户界面来展示一些有用的信息，比如 code listing， disassembly and register content
  - layout split
  - layout src
  - layout asm
  - layout reg
  - layout tui

### 其它

你可以使用 `set` 指令来改变执行中的变量的值。

## 2. 调试 xv6

按照官方文档配置好环境后，就应该可以正藏调试了。

下面以 `sleep.c` 代码的调试为例学习一下怎么调试 xv6 的代码。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    int time;

    if (argc < 2)
    {
        fprintf(2, "Usage: Sleep seconds\n");
        exit(0);
    }

    time = atoi(argv[1]);

    sleep(time);
    
    exit(0);
}
```

首先，在终端输入 `make qemu-gdb`，会得到类似下面的输出：

```bash
(base) zsc@BerryLap:~/xv6-labs-2020$ make qemu-gdb
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::26000
```

在另一个终端窗口中输入 `gdb-multiarch`，报错：

```bash
(base) zsc@BerryLap:~/xv6-labs-2020$ gdb-multiarch
GNU gdb (Ubuntu 9.1-0ubuntu1) 9.1
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
warning: File "/home/zsc/xv6-labs-2020/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /home/zsc/xv6-labs-2020/.gdbinit
line to your configuration file "/home/zsc/.gdbinit".
To completely disable this security protection add
        set auto-load safe-path /
line to your configuration file "/home/zsc/.gdbinit".
For more information about this security protection see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
        info "(gdb)Auto-loading safe path"
(gdb) run
Starting program:
No executable file specified.
Use the "file" or "exec-file" command.
```

根据报错信息，在用户home目录新建 `.gdbinit` 文件并添加指定内容 ：`add-auto-load-safe-path /home/zsc/xv6-labs-2020/.gdbinit`

再次运行 `gdb-multiarch` 得到以下输出，可以看出目标架构已经是 `riscv:rv64` 了：

```bash
(base) zsc@BerryLap:~/xv6-labs-2020$ gdb-multiarch
GNU gdb (Ubuntu 9.1-0ubuntu1) 9.1
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
The target architecture is assumed to be riscv:rv64
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000001000 in ?? ()
(gdb)
```

输入要 debug 的程序：

```bash
(gdb) file user/_sleep
Reading symbols from user/_sleep...
```

可以在第 16 行打个断点：

```bash
(gdb) break 16
Breakpoint 1 at 0xe: file user/sleep.c, line 16.
```

在第一个窗口中输入 `sleep 10` 调用 sleep 程序，

使用 `continue` 指令执行至断点：

```bash
(gdb) continue
Continuing.
[Switching to Thread 1.3]

Thread 3 hit Breakpoint 1, main (argc=1, argv=0x2 <main+2>)
    at user/sleep.c:16
16          time = atoi(argv[1]);
```

`next` 指令执行至下一步，此时使用 `print time` 可以得到我们指定的数值10:

```bash
(gdb) c
Continuing.

Thread 2 hit Breakpoint 1, main (argc=2, argv=0x2fc0)
    at user/sleep.c:16
16          time = atoi(argv[1]);
(gdb) next
18          sleep(time);
(gdb) print time
$1 = 10
(gdb)
```

尝试调试更多的内容吧！

## 3. 更多

本文记录并学习了 xv6 代码的 GDB 调试过程，但是比较浅显。

我找到了一篇很好的博客，后面有时间可以仔细看看：

https://www.cnblogs.com/KatyuMarisaBlog/p/13727565.html