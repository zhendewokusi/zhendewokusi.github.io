---
title: 地址、section、vstart
date: 2023-10-14 14:55:49
tags: [x86汇编]
---

### 前言

我在学习汇编的时候，对于`vstart`的概念其实很模糊，感觉会又感觉不是很会，要说哪儿不会有说不出来，就很难受。`vstart`是`section`中的概念，但是要讲清楚什么是`section`，就必须说说什么是地址了。

<!-- more -->

### 地址

#### 地址的概念

计算机里面只有0和1,地址当然就是一串数字，这串数字用来描述各种符号在源程序中的位置。地址细分又能分为：物理地址、虚拟地址布拉布拉的，这里不做深究。

在汇编中，由于指令和变量占据的内存大小不一样，总不能想怎么来就怎么来，那么就得有一套统一的规则来分配。程序中各种数据结构的访问，本质上是通过“该数据结构的起始地址 + 该数据结构在该硬件平台规定的变量所占据的内存大小”来实现的。这里也就解释了，为什么`C语言`等高级语言中为什么要定义变量的类型，因为这样才能正确的分配和访问变量占据的内存。

那么这里的起始地址是如何得到的呢？编译器在做这件事情的时候，占据第一位的数据的地址便是整个程序的起始地址，后面的数据都是在它基础上的偏移。第n个数据所在的位置就是：第n-1个数据的偏移 + 数据 n-1 的内存空间，这就是所谓的**偏移量**，每个变量的地址都是前一个变量的地址 + 前一个变量的内存空间大小。

#### 简单例子

来看个简单例子：
```asm
     mov ax,$$
     mov ds,ax
     mov ax,[var]
     label : mov ax,$
     jmp label
     var dw 0x99
     infi: jmp near infi                 ;无限循环
   
   times (510-($-$$)) db 0               ;主引导程序512字节，空余的使用0填充
             db 0x55,0xaa                ;结束标志               
```
下面将其放在bochs中进行调试的输出信息：
```shell
<bochs:1> b 0x07c00
<bochs:2> c
(0) Breakpoint 1, 0x00007c00 in ?? ()
Next at t=17178863
(0) [0x000000007c00] 0000:7c00 (unk. ctxt): mov ax, 0x0000            ; b80000
<bochs:3> n
Next at t=17178864
(0) [0x000000007c03] 0000:7c03 (unk. ctxt): mov ds, ax                ; 8ed8
<bochs:4> 
Next at t=17178865
(0) [0x000000007c05] 0000:7c05 (unk. ctxt): mov ax, word ptr ds:0x000d ; a10d00
<bochs:5> 
Next at t=17178866
(0) [0x000000007c08] 0000:7c08 (unk. ctxt): mov ax, 0x0008            ; b80800
<bochs:6> 
Next at t=17178867
(0) [0x000000007c0b] 0000:7c0b (unk. ctxt): jmp .-5  (0x00007c08)     ; ebfb
<bochs:7> 
Next at t=17178868
(0) [0x000000007c08] 0000:7c08 (unk. ctxt): mov ax, 0x0008            ; b80800
<bochs:8> 
Next at t=17178869
(0) [0x000000007c0b] 0000:7c0b (unk. ctxt): jmp .-5  (0x00007c08)     ; ebfb
```
可以看到第一行的mov指令，`$$`被替换成了`0`。默认情况下如果程序没有定义`section`就将所有的文件当作一个大的`section`，因此此时`$$`为`0`。

第三行引用了`var`变量的值，而`[]`是去所在地址的内容。`mov ax, word ptr ds:0x000d ;`就是表明要将一个16位的数据从内存地址 `0x000d` 处加载到寄存器 `AX` 中。至于为什么是`0x000d`后面会解释。

第四行用了一个`$`标号，代表当前指令所在的地址。

最后一行就是一个数据的定义，没什么好讲的。

还记得上面讲的地址的计算吗？看调试第一行的左侧的地址`0x000000007c00`和第二行地址`0x000000007c03`，这两个之间相差3,而第一个指令转换成机器码是`b80000`刚好三个字节。有没有理解计算机的地址了？再依次往下算，刚好`var dw 0x99`的地址就是`0x000000007c0b` + 对应的机器码`ebfb`的长度，也就是`0x000000007c0d`，就将这个结果`0x000000007c0d - 0x000000007c00` 后得到的偏移量是不是 `0x000d`，也就是变量`var`的地址。

### section

我刚开始对`section`感觉就是在似懂非懂，这个东西称为节。编译器提供的这个关键字只是给程序员用的，处理器根本不知道有这么个东西，你会想：既然CPU又不知道这么个东西，我用不用`section`都行吧。是这样的，但是最好使用，因为`section`的功能类似与函数，人为的将代码划分为不同的部分，都是为了代码结构清晰，易于维护。

```asm
SECTION code
    mov ax,$$
    mov ax,section.data.start
    mov ax,section.code.start
    mov ax,[var1]
    label: jmp label

    infi: jmp near infi                 ;无限循环
 
  times (510-($-$$)) db 0
            db 0x55,0xaa

SECTION data
    var1 dd 0x4
```
下面将其放在bochs中进行调试的输出信息：

```shell
(0) [0x0000fffffff0] f000:fff0 (unk. ctxt): jmpf 0xf000:e05b          ; ea5be000f0
<bochs:1> b 0x07c00
<bochs:2> c
(0) Breakpoint 1, 0x00007c00 in ?? ()
Next at t=17178863
(0) [0x000000007c00] 0000:7c00 (unk. ctxt): mov ax, 0x0000            ; b80000
<bochs:3> n
Next at t=17178864
(0) [0x000000007c03] 0000:7c03 (unk. ctxt): mov ax, 0x0200            ; b80002
<bochs:4> 
Next at t=17178865
(0) [0x000000007c06] 0000:7c06 (unk. ctxt): mov ax, 0x0000            ; b80000
<bochs:5> 
Next at t=17178866
(0) [0x000000007c09] 0000:7c09 (unk. ctxt): mov ax, word ptr ds:0x0200 ; a10002
<bochs:6> 
Next at t=17178867
(0) [0x000000007c0c] 0000:7c0c (unk. ctxt): jmp .-2  (0x00007c0c)     ; ebfe
<bochs:7> 
Next at t=17178868
(0) [0x000000007c0c] 0000:7c0c (unk. ctxt): jmp .-2  (0x00007c0c)     ; ebfe
```

在调试信息中我们注意到：
- `SECTION code` 和 `SECTION data` 一个字节的机器码都没有产生，也印证了前面说的处理器并不知道有`SECTION`的存在。

- `section.data.start`实际上就是本文件中名为`data`的`section`的真实偏移。这个或许看不出来，可以看`mov ax,section.code.start`的偏移，也就是调试信息中的`0000:7c06 (unk. ctxt): mov ax, 0x0000`，可以看到`code`的偏移量为0，符合我们的预期。


### vstart

根据nasm官方手册的解释：`section`使用`vstart`修饰后，就可以被赋予一个虚拟起始地址`virtual start address`。而`org`和`vstart`实际上是同一功能，我就只展示其中一个，另外一个原理相同。

特别需要注意的是：`vstart`和`org`都不会让程序加载到地址xxx。它们做的只是告诉编译器将这个节之后的数据、指令的地址按照xxx为起始，就这么个功能。而加载是加载器的功能，编译器没这本事。

光听概念太枯燥，还是来看个代码：
```asm
SECTION code vstart=0x7c00
    mov ax,$$
    mov ax,section.data.start
    mov ax,section.code.start
    mov ax,[var1]
    label: jmp label

    infi: jmp near infi                 ;无限循环
 
  times (510-($-$$)) db 0
            db 0x55,0xaa

SECTION data 
    var1 dd 0x4
```
下面将其放在bochs中进行调试的输出信息：

```shell
(0) [0x0000fffffff0] f000:fff0 (unk. ctxt): jmpf 0xf000:e05b          ; ea5be000f0
<bochs:1> b 0x07c00
<bochs:2> c
(0) Breakpoint 1, 0x00007c00 in ?? ()
Next at t=17178863
(0) [0x000000007c00] 0000:7c00 (unk. ctxt): mov ax, 0x7c00            ; b8007c
<bochs:3> n
Next at t=17178864
(0) [0x000000007c03] 0000:7c03 (unk. ctxt): mov ax, 0x0200            ; b80002
<bochs:4> 
Next at t=17178865
(0) [0x000000007c06] 0000:7c06 (unk. ctxt): mov ax, 0x0000            ; b80000
<bochs:5> 
Next at t=17178866
(0) [0x000000007c09] 0000:7c09 (unk. ctxt): mov ax, word ptr ds:0x0900 ; a10009
<bochs:6> 
Next at t=17178867
(0) [0x000000007c0c] 0000:7c0c (unk. ctxt): jmp .-2  (0x00007c0c)     ; ebfe
<bochs:7> 
Next at t=17178868
(0) [0x000000007c0c] 0000:7c0c (unk. ctxt): jmp .-2  (0x00007c0c)     ; ebfe
```

可以看到`code`加了`vstart`后，`$$`从`0`变成`0x7c00`。所以该节中的数据地址从`0x7c00`为起始编址。这个是虚拟的地址，因为这个程序整个才512字节，根本到不了偏移量为`0x7c00`。`$$`以该节的虚拟起始地址为主，如果这个节没有`vstart`来指定起始地址，就输出在文件中偏移地址。而此时的`$`就是 当前的新的地址 + 偏移。 

`section.data.start`实际上就是本文件中名为`data`的`section`的真实偏移，没有因为`vstart`改变。



