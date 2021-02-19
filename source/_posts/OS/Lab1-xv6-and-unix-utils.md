---
title: 'Lab1:xv6-and-unix-utils'
categories:
  - 计算机/操作系统
tags:
  - 操作系统
  - OS
  - xv6
keywords:
  - 操作系统
date: 2021-02-18 20:22:55
summary: 实验一：xv6 unix utilities
---

> 执行 `make qemu` 运行 xv6 程序

## sleep

在 `user/sleep.c` 中为 xv6 实现 unix 的 `sleep` 功能来实现暂停指定 tick 数。一个 tick 是 xv6 内核定义的一段时间，指明了时钟芯片两次终端之间的间隔。

一些提示：

- 阅读 https://blogs.angryberry.tech/2021/02/12/os/cao-zuo-xi-tong-jie-kou/#toc-heading-6 （第一章）

- 可以阅读一下 user 目录下的其他代码实现

- 如果用户没有传入参数，`sleep` 应该打印一条错误消息

- 命令行参数是以字符串形式传入的，需要转化为整数（参考 `user/ulib.c`）

- 使用 `sleep` 系统调用

- 查看代码 `kernel/sysproc.c` 以了解系统调用 `sleep` 是如何实现的 （查看 `sys_sleep`），`user/user.h` 中定义了用户程序调用 `sleep` 的句柄，`user/usys.S` 定义了从用户程序调用`sleep`跳转到内核的汇编代码

- 确保 `main` 函数调用 `exit()` 退出程序

- 将你的 `sleep` 程序添加到 `Makefile` 的 `UPROGS` 中。完成后，可以使用 `make qemu` 编译代码在shell中执行

   ```shell
    $ make qemu
    ...
    init: starting sh
    $ sleep 10
    (nothing happens for a little while)
    $
    ```

使用以下指令运行评分程序：

```shell
./grade-lab-util sleep
```

### 解答

`kernel/sysproc.c`中，程序 `sys_sleep` 代码如下：

```c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(myproc()->killed){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

`user/user.h` 中定义了 `sleep` 的原型：

```c
int sleep(int);
```

最终，我们可以如下实现 `sleep.c`：

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

## pingpong

写代码实现如下功能：使用 unix 系统调用实现在两个进程之间 “ping-pong” 一个字节，可以借助管道。父进程发送一个字节给子进程，子进程打印“\<pid\>: received ping”，其中， pid 是进程 ID；同时子进程向父进程写一个字节并退出。父进程需要读取子进程发送的字节，并打印 "\<pid\>: received pong"，之后退出。代码实现在 “user/pingpong.c”中

提示：

- 使用 `pipe` 创建管道
- 使用 `fork` 创建子进程
- 调用 `read` 从管道读取数据；调用 `write` 向管道写入数据
- 使用 `getpid` 得到调用进程的 PID
- 将程序添加至 Makefile 的 `UGPROGS` 中
- User programs on xv6 have a limited set of library functions available to them. You can see the list in `user/user.h`; the source (other than for system calls) is in `user/ulib.c`, `user/printf.c`, and `user/umalloc.c`.

程序执行结果应该像下面一样：

```shell
$ make qemu
...
init: starting sh
$ pingpong
4: received ping
3: received pong
$
```

### 解答

基本思路：

分别为父进程和子进程创建管道

- 在子进程中，关闭父进程管道的写和子进程管道的读，然后读取父进程管道的读，并按要求打印；之后向子进程管道写，然后关闭子进程管道的写和父进程管道的读；
- 在父进程中，关闭父进程管道的写和子进程管道的读，然后读取子进程管道的读，并按要求打印；之后向子进程管道的读端写入字节，等待子进程执行完毕

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define R 0
#define W 1

int 
main()
{
    int p_parent[2];  
    int p_child[2];
    pipe(p_parent); 
    pipe(p_child);

    if (fork() == 0) {
        close(p_parent[W]);
        close(p_child[R]);
        char buffer = 'c';
        read(p_parent[R], &buffer, 1);
        printf("%d: received ping\n", getpid());
        write(p_child[W], &buffer, 1);
        close(p_child[W]);
        close(p_parent[R]);
    }
    else {
        char buffer = 'p';
        close(p_child[W]);
        close(p_parent[R]);
        write(p_child[R], &buffer, 1);
        read(p_parent[W], &buffer, 1);
        printf("%d: received pong\n", getpid());
        close(p_child[R]);
        close(p_parent[W]);
        int status;
        wait(&status);
    }

    exit(0);
}
```

## xargs

编写简化的 unix xargs 程序，为每行标准输入调用指令。

`xargs`命令的作用，是将标准输入转为命令行参数。

```bash
$ echo "hello world" | xargs echo
hello world
```

上面的代码将管道左侧的标准输入，转为命令行参数`hello world`，传给第二个`echo`命令。

`xargs`命令的格式如下。

```bash
$ xargs [-options] [command]
```

真正执行的命令，紧跟在`xargs`后面，接受`xargs`传来的参数。

`xargs`的作用在于，大多数命令（比如`rm`、`mkdir`、`ls`）与管道一起使用时，都需要`xargs`将标准输入转为命令行参数。

```bash
$ echo "one two three" | xargs mkdir
```

上面的代码等同于`mkdir one two three`。如果不加`xargs`就会报错，提示`mkdir`缺少操作参数。

**提示：**

- 使用 `fork` 和 `exec` 为每一行输入调用命令。使用 `wait` 在父进程中等待子进程执行指令。
- 为了读取一行输入，每次读取一个字符指导遇到 `\n`
- `kernel/param.h` 中定义了 `MAXARG`，在声明参数数组时可能有用
- 在 `Makefile` 中添加文件至 `UPROGS` 中

以下指令会为当前目录下的每个 *b* 文件执行 `grep hello`

```bash
$ find . b | xargs grep hello
```

可以使用下面的指令测试我们的实现：

You may have to go back and fix bugs in your find program. The output has many `$` because the xv6 shell doesn't realize it is processing commands from a file instead of from the console, and prints a `$` for each command in the file.

```bash
$ make qemu
...
init: starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
$ $   
```

### 解答

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char* argv[])
{
    if (argc < 2) {
        fprintf(2, "usage: xargs cmd [...]\n");
        exit(1);
    }

    int flag = 1;
    char **newargv = malloc(sizeof(char*) * (argc + 2));
    memcpy(newargv, argv, sizeof(char*) * (argc + 1));

    newargv[argc + 1] = 0; // null pointer for the last one

    while (flag) {
        char buf[512], *p = buf;

        while ((flag = read(0, p, 1)) && *p != '\n') {
            ++p;
        }

        if (!flag)
            exit(0);
        
        *p = 0;
        newargv[argc] = buf;
        if (fork() == 0) {
            exec(argv[1], newargv + 1);
            exit(0);
        }

        wait(0);
    }

    exit(0);
}
```



## Optional challenge exercises

- Write an uptime program that prints the uptime in terms of ticks using the `uptime` system call. ([easy](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))
- Support regular expressions in name matching for `find`. `grep.c` has some primitive support for regular expressions. ([easy](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html))
- The xv6 shell (`user/sh.c`) is just another user program and you can improve it. It is a minimal shell and lacks many features found in real shell. For example, modify the shell to not print a `$` when processing shell commands from a file ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html)), modify the shell to support wait ([easy](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html)), modify the shell to support lists of commands, separated by ";" ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html)), modify the shell to support sub-shells by implementing "(" and ")" ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html)), modify the shell to support tab completion ([easy](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html)), modify the shell to keep a history of passed shell commands ([moderate](https://pdos.csail.mit.edu/6.S081/2020/labs/guidance.html)), or anything else you would like your shell to do. (If you are very ambitious, you may have to modify the kernel to support the kernel features you need; xv6 doesn't support much.)