---
title: Learn DGB
date: 2023-12-04 17:18:39
tags: [GDB]
---

# 前言

`GDB 的全称是 GNU Debugger`

"Caveman debugging" 这个词语的使用是一种夸张和幽默的说法，强调了使用最基本工具进行调试的原始感觉，好像程序员是在用石器时代的工具一样。不要只会最简单的调试，掌握更切近实际的高效技巧会为你调试工作带来不小的便利。

<!-- more -->

## 练习

```shell
cd yourworkspace
git clone --depth=1 git@github.com:zhendewokusi/Basic_GDB.git
```

如果你有一定C语言基础，且对gdb的基本使用还不怎么熟练，可以克隆下面仓库，里面有很多代码用以练手，本篇文章的`markdown`文件也在里面。

```shell
# 编译举例
gcc basic1.c -g3 -o basic1
```

# GDB基本命令

b（break的缩写）命令在main()函数入口设置断点

r（run的缩写）命令开始执行程序，程序执行到main函数时，触发断点。

n（next的缩写）命令进行单步执行。

s（step的缩写）：单步调试如果有函数调用，则进入函数；与命令n不同，n是不进入调用的函数的

函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息。

watch 表达式：设置一个监视点，一旦被监视的“表达式”的值改变，gdb将强行终止正在被调试的程序。如： `watch a`
    rwatch awatch

x/p:x 命令用于查看内存中的数据，而 /p 是其中的一种输出格式选项之一。

where/bt ：当前运行的堆栈列表

call:用于在调试会话中调用一个函数，从而可以直接查看函数的返回值或执行一些函数内的操作。

up/down/frame:

1. **up命令：**
   - 用法：`up [count]`
   - 作用：将当前调用堆栈帧上移 `count` 个帧。如果没有提供 `count`，则默认上移一个帧。每个帧对应于一个函数调用。

2. **down命令：**
   - 用法：`down [count]`
   - 作用：将当前调用堆栈帧下移 `count` 个帧。如果没有提供 `count`，则默认下移一个帧。

3. **frame命令：**
   - 用法：`frame [frame-number]`
   - 作用：选择调用堆栈中的特定帧。


- 分割窗口:
layout：用于分割窗口，可以一边查看代码，一边测试：
layout src：显示源代码窗口
layout asm：显示反汇编窗口
layout regs：显示源代码/反汇编和CPU寄存器窗口
layout split：显示源代码和反汇编窗口
Ctrl + L：刷新窗口
winheight:改变窗口高度

info :看断点信息，局部变量等等

command / condition:

1. **`command` 命令：**
   - `command` 命令允许你定义一个自定义 GDB 命令，该命令可以包含一系列 GDB 命令，这些命令将在程序执行时被执行。
   - 语法：
     ```bash
     command [COUNT] [SHELL-COMMANDS]
     ```

     - `COUNT` 表示该命令在程序执行多少次后将被删除。如果省略 `COUNT`，则该命令会一直存在。
     - `SHELL-COMMANDS` 是在达到断点时要执行的 GDB 命令列表。

   - 示例：
     ```bash
     (gdb) break 10
     (gdb) command 3
     > silent
     > printf "At line %d\n", __LINE__
     > continue
     > end
     ```
save breakpoints file_name

2. **`condition` 命令：**
   - `condition` 命令用于设置断点的条件。这意味着，只有在满足指定条件时，断点才会中断程序执行。
   - 语法：
     ```bash
     condition BREAKPOINT-NUMBER EXPRESSION
     ```

     - `BREAKPOINT-NUMBER` 是断点的编号。
     - `EXPRESSION` 是一个表达式，如果该表达式的值为真（非零），则断点会中断程序执行。

   - 示例：
     ```bash
     (gdb) break 20
     (gdb) condition 1 i == 5
     ```

     这将在程序执行到第 20 行时，只有当变量 `i` 的值等于 5 时，才会中断程序执行。

until:直到多少行，哪个函数....

tbreak:临时断点



# GDB打印复杂数据结构

正常数据结构太难看，让它输出好看点
`.gdbinit`文件
```
set print pretty on
set print array-indexes on
```

# 宏定义

GCC的调试选项 -g
我们知道，要想用GDB进行调试，必须在用GCC编译时加上“-g”选项。但很多童鞋可能不知道的是，和优化选项“-Ox”一样，调试选项“-g”也有几个等级可选：

-g 默认选项，同-g2

-g0 不生成任何调试信息，和编译时不加“-g”是一样的。

-g1 生成最少量的调试信息，这些信息足够用来通过backtrace查看调用栈符号信息。主要包括了函数声明、外部变量和行号等信息，但是不包含局部变量信息。这个选项比较少用。

-g2 生成足够多的调试信息，可以用GDB进行正常的程序调试，这个是默认选项。

-g3 在-g2的基础上产生更多的信息，如宏定义。

可见，我们编译时加的“-g”选项，其实等同于“-g2”，它产生了足够多的调试信息，我们可以用gdb查看调用栈、查看局部变量等。但是，要想查看宏定义，则必须要使用“-g3”选项。

开`g3`才能输出宏

# GDB Dynamic Printf

问题：
- 代码添加打印信息进行调试，突然发现添加打印的位置不对，或者别的地方也需要添加打印信息。

- 于是，重新修改源码，重新添加打印，重新编译，重新部署，重新运行，重新调试，重新分析。

- 当我们费了九牛二虎之力把这些都弄好之后，很不幸地又发现了新的问题，然后不得不反复进行这些过程。
误而崩溃，GDB 会在进程终止后无法再执行 call 命令。你需要在程序崩溃之前设置调试信息，还必须从程序中删除掉。

上面问题你或许可以克服，新的问题：大型项目printf太麻烦，编译时间太久。还要修改源码。

那么，有没有一种方法，既不需要修改源码，又能随时在程序中任何地方任意添加打印信息呢？

设置动态打印的命令是dprintf，格式如下：

```shell
dprintf location,format string,arg1,arg2,...
```

dprintf命令和C语言中的printf的用法很相似，支持格式化打印。

相比printf函数，dprintf命令多了一个location参数，用于指定动态打印被触发的位置。

和break命令设置断点时一样，location可以是文件名:行号、函数名、或者具体的地址等。

除了location外，剩余的几个参数，就和printf()函数一致了。format指定字符串打印的格式，后面几个参数指定打印的数据来源。

## 保存断点信息

为了解决上面提到的问题，GDB很贴心地提供了对断点信息保存和加载的功能。

GDB中，可以把当前所设置的各种类型的断点信息全部保存在一个脚本文件中。这其中当然也包括dprintf设置的动态打印信息。

只需要执行下面的命令即可：
```shell
save breakpoints file_name
```
这条命令会把当前所有的断点信息都保存在file_name指定的文件中。

等下次进行调试时，可以把file_name文件中的断点信息重新加载起来。有两种方法：

1. 启动GDB时使用“-x file_name”参数。

2. 在GDB中执行source file_name命令。

gdb后加`-x`加`file_name`加载断点信息。

# GDB Dynamic Printf 多余吗?
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

int global = 6;

void alarm_handler() {
    global++;
}

int main(int argc,char* argv[]) {
    int a,b,c;

    signal(SIGALRM, alarm_handler);
    alarm(1);

    a = 9;
    b = 9;
    c = 6;

    c = (a + b) / (global - c);
    return 0;
}
```
## 普通断点的问题
我们前面尝试了单步执行，以及直接在第23行设置断点并打印变量的方式，可程序都能够正常执行结束，浮点异常错误无法重现。

这两种方式存在的共同问题是，**在程序触发断点后，需要和用户进行交互，用户必须手动输入命令并恢复程序的执行。而和用户交互，势必引入延迟。**

实际上，无论是单步执行，还是在断点触发后打印a、b、c、global_variable的值，都无法b保证程序在在1秒钟内执行完毕。因此，第17行设置的闹钟就会到期。

**尽管我们用GDB调试程序时，被调试程序本身的执行被暂时停止，但是alarm函数设置的闹钟是由底层OS内核提供的服务。无论我们的程序执行是否被暂停，OS内核仍然会在设置的闹钟到期后，向应用程序发送SIGALRM信号。**

如此以来，无论我们是用step命令还是continue命令恢复程序执行，程序都会首先处理SIGALRM信号，然后才去执行接下来的代码。

在SIGALRM的处理函数alarm_handler()中，会把global_variable加1，它的值变成了5,。接下来执行第23行代码时，global_variable - c的值就变成了1，当然不会再触发除零错误了。

接下来，我们用GDB的动态打印功能来调试一下。

# 反向调试

反向调试技术的核心原理，简单来说，是在程序运行中，记录每一条指令对程序执行的所有状态变化，包括变量、寄存器、内存的数据变化等，并将这些信息存储在一个history文件中。当需要回溯到过去的状态时，调试器会按照相反的顺序逐条指令恢复这些状态，使得程序的执行状态回到已经被记录的任意时间点。

- reverse-next(rc): 参考next(n), 逆向执行一行代码，遇函数调用不进入
- reverse-nexti(rni): 参考nexti(ni), 逆向执行一条指令，与函数调用不进入
- reverse-step(rs): 参考step(s), 逆向执行一行代码，遇函数调用则进入
- reverse-stepi(rsi): 参考setpi(si)， 逆向执行一条指令，与函数调用则进入
- reverse-continue(rc): 参考continue(c), 逆向继续执行
- reverse-finish: 参考finish，逆向执行，一直到函数入口处
- reverse-search(): 参考search，逆向搜索
- set exec-direction reverse: 设置程序逆向执行，执行完此命令后，所有常用命令如next, nexti, step, stepi, continue、finish等全部都变成逆向执行
- set exec-direction forward: 设置程序正向执行，这也是默认的设置

**在绝大多数环境下，在使用这些反向调试命令之前，必须要先通过record命令让GDB把程序执行过程中的所有状态信息全部记录下来。通常是在程序运行起来之后，先设置断点让让程序停下来，然后输入record命令，开启状态信息记录，然后再继续执行。**

- record: 记录程序执行过程中所有状态信息
- record stop: 停止记录状态信息
- record goto: 让程序跳转到指定的位置, 如record goto start、record goto end、record goto n
- record save filename: 把程序执行历史状态信息保存到文件，默认名字是gdb_record.process_id
- record restore filename: 从历史记录文件中恢复状态信息
- show record full insn-number-max：查看可以记录执行状态信息的最大指令个数，默认是200000
- set record full insn-number-max limit：设置可以记录执行状态信息的最大指令个数
- set record full insn-number-max unlimited：记录所有指令的执行状态信息