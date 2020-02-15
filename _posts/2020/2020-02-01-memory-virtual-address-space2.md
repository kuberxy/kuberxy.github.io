---

layout: post
title: 虚拟地址空间的分布（二）
date:   2020-02-01 10:02:42 +0800
category: 计算机基础
tags: [内存]
typora-root-url: ../..
---



## ELF文件的链接视图和执行视图

### 从Section到Segment

对于一个正常的进程来说，在它对应的可执行文件中往往会包含很多不同类型的段（也称为Section），比如代码段、数据段、BSS段的等。

![image-20200203180427527](/assets/images/2020/image-20200203180427527.png)

我们知道，ELF文件被映射到进程的虚拟地址空间时，是以系统的页大小为单位进行映射的。因此，每个段在映射时的长度都会是系统页大小的整数倍，即不足一页的段仍会占用一个页的空间。显然，当ELF文件的段数量很多时，就会产生空间浪费的问题。

> 那么该如何减少这种浪费呢？

实际上，操作系统在装载在可执行文件时，它并不关心可执行文件的各个段所包含的实际内容，它只关心一些跟装载相关的属性，其中最主要的是段的权限（可读、可写、可执行）。

在ELF文件中，段的权限往往只有为数不多的几种组合，基本上是以下三种：

- 可读可执行段，以代码段为代表

- 可读可写段，以数据段和BSS段为代表

- 只读段，以只读数据段为代表

因此，我们可以将权限相同的段合并在一起当作一个整体进行映射。



我们来看一个简单的示例：有一个ELF文件，它由两个段组成，分别是“.text”和“.init”，它们分别是程序的可执行代码和初始化代码，并且它们的权限相同，都是可读可执行的。

![image-20200203181005872](/assets/images/2020/image-20200203181005872.png)

现假设，.text为4097字节，.init为512字节。如果这两个段分别映射的话，要找占用三个页，但如果将它们合并后一起映射的话，只需占用两个页。其中，这两个合并后的段，我们称之为Segment。

实际上，Segment是从程序装载的角度重新划分了ELF文件中的段。在将目标文件链接成可执行文件的时候，链接器会尽量把属性相同的段分配在同一个空间。比如，将可读可执行的段都放在一起，这种典型的段是代码段；将可读可写的段都放在一起，这种典型的段是数据段。

在ELF文件中，那些属性相似、又连在一起的段就是一个Segment。操作系统在映射可执行文件时，是按照Segment而不是Section来划分可执行文件的。



### 示例：查看可执行文件的section和segment

#### 生成可执行文件

下面我们看一个简单的小程序，程序本身是不停地循环执行“sleep”操作，它的源代码如下：

```c
#include <stdlib.h>

int main()
{
    while(1) {
        sleep(1000);
    }
    return 0;
}
```

首先，我们使用静态连接的方式将其编译成可执行文件，编译命令如下：

```bash
gcc -static SectionMapping.c -o SectionMapping.elf
```



#### 查看可执行文件的sction

然后，我们使用readelf命令查看这个可执行文件包含的段（section）：

```bash
readelf -S SectionMapping.elf
```

输出如下：

```
There are 33 section headers, starting at offset 0x8eac8:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .note.ABI-tag     NOTE            080480d4 0000d4 000020 00   A  0   0  4
  [ 2] .note.gnu.build-i NOTE            080480f4 0000f4 000024 00   A  0   0  4
  [ 3] .rel.plt          REL             08048118 000118 000028 08   A  0   5  4
  [ 4] .init             PROGBITS        08048140 000140 000030 00  AX  0   0  4
  [ 5] .plt              PROGBITS        08048170 000170 000050 00  AX  0   0  4
  [ 6] .text             PROGBITS        080481c0 0001c0 069fbc 00  AX  0   0 16
  [ 7] __libc_thread_fre PROGBITS        080b2180 06a180 00008c 00  AX  0   0 16
  [ 8] __libc_freeres_fn PROGBITS        080b2210 06a210 001838 00  AX  0   0 16
  [ 9] .fini             PROGBITS        080b3a48 06ba48 00001c 00  AX  0   0  4
  [10] .rodata           PROGBITS        080b3a80 06ba80 0154fb 00   A  0   0 32
  [11] __libc_thread_sub PROGBITS        080c8f7c 080f7c 000004 00   A  0   0  4
  [12] __libc_subfreeres PROGBITS        080c8f80 080f80 000030 00   A  0   0  4
  [13] __libc_atexit     PROGBITS        080c8fb0 080fb0 000004 00   A  0   0  4
  [14] .stapsdt.base     PROGBITS        080c8fb4 080fb4 000001 00   A  0   0  1
  [15] .eh_frame         PROGBITS        080c8fb8 080fb8 00c9f4 00   A  0   0  4
  [16] .gcc_except_table PROGBITS        080d59ac 08d9ac 0000ff 00   A  0   0  1
  [17] .tdata            PROGBITS        080d6aac 08daac 000010 00 WAT  0   0  4
  [18] .tbss             NOBITS          080d6abc 08dabc 000018 00 WAT  0   0  4
  [19] .ctors            PROGBITS        080d6abc 08dabc 00000c 00  WA  0   0  4
  [20] .dtors            PROGBITS        080d6ac8 08dac8 00000c 00  WA  0   0  4
  [21] .jcr              PROGBITS        080d6ad4 08dad4 000004 00  WA  0   0  4
  [22] .data.rel.ro      PROGBITS        080d6ad8 08dad8 000030 00  WA  0   0  4
  [23] .got              PROGBITS        080d6b08 08db08 00000c 04  WA  0   0  4
  [24] .got.plt          PROGBITS        080d6b14 08db14 000020 04  WA  0   0  4
  [25] .data             PROGBITS        080d6b40 08db40 000bc0 00  WA  0   0 32
  [26] .bss              NOBITS          080d7700 08e700 001740 00  WA  0   0 32
  [27] __libc_freeres_pt NOBITS          080d8e40 08e700 000018 00  WA  0   0  4
  [28] .note.stapsdt     NOTE            00000000 08e700 00023c 00      0   0  4
  [29] .comment          PROGBITS        00000000 08e93c 00002d 01  MS  0   0  1
  [30] .shstrtab         STRTAB          00000000 08e969 00015e 00      0   0  1
  [31] .symtab           SYMTAB          00000000 08eff0 007e10 10     32 888  4
  [32] .strtab           STRTAB          00000000 096e00 006d81 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

我们可以看到，这个可执行文件一共有33个段。在段表中，详细记录了每个段的属性信息。



#### 查看可执行文件的segment

接着，我们再使用readelf命令查看这个可执行文件的Segment：

```bash
readelf -l SectionMapping.elf 
```

输出如下：

```
Elf file type is EXEC (Executable file)
Entry point 0x80481c0
There are 5 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x08048000 0x08048000 0x8daab 0x8daab R E 0x1000
  LOAD           0x08daac 0x080d6aac 0x080d6aac 0x00c54 0x023ac RW  0x1000
  NOTE           0x0000d4 0x080480d4 0x080480d4 0x00044 0x00044 R   0x4
  TLS            0x08daac 0x080d6aac 0x080d6aac 0x00010 0x00028 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00     .note.ABI-tag .note.gnu.build-id .rel.plt .init .plt .text __libc_thread_freeres_fn __libc_freeres_fn .fini .rodata __libc_thread_subfreeres __libc_subfreeres __libc_atexit .stapsdt.base .eh_frame .gcc_except_table 
   01     .tdata .ctors .dtors .jcr .data.rel.ro .got .got.plt .data .bss __libc_freeres_ptrs 
   02     .note.ABI-tag .note.gnu.build-id 
   03     .tdata .tbss 
   04     
```

我们可以看到，这个可执行文件一共有5个Segment。在程序头表中，详解描述了这个可执行文件应该如何被操作系统映射到进程的虚拟地址空间中。



### 链接视图 & 执行视图

从程序装载的角度看，我们只需要关心“LOAD”类型的Segment，因为只有它是需要被映射的，其他的诸如“NOTE”、“TLS”、“GNU_STACK”都是在装载时起辅助作用的。如下，是SectionMapping.elf这个可执行文件与进程虚拟地址空间的映射关系。

![image-20200208173103121](/assets/images/2020/image-20200208173103121.png)

我们可以看到，可执行文件“SectionMapping.elf”中的段被重新划分成了三个部分，对于一些可读可执行的段，它们被统一映射到VMA0；对于一些可读可写的段，它们被统一映射到VMA1；而对于那些包含调试信息和字符串表的其他段，它们没有被映射，因为这些段在程序执行时没有用，所以它们不需要被映射。显然，所有相同属性的Section被归类到一个Segment，并且映射到同一个VMA。

总的来说，Section和Segment是从不同的角度对同一个ELF文件进行划分。从section的角度划分出来的ELF文件，我们称之为ELF文件的链接视图，从segment的角度划分出来的ELF文件，我们称之为ELF文件的执行视图。

> 注：在程序装载时，我们说所的“段”专指“Segment”，而在其他情况下，“段”指的是“Section”。



### 程序头表

在每一个ELF的可执行文件中，都存在一个专门的数据结构叫做程序头表（Program Header Table），它的作用是保存程序的segment信息。程序头表详细描述了可执行文件应该如何被操作系统映射到进程的虚拟地址空间中。

> 注：ELF的目标文件没有程序头表，因为它不需要被装载，而ELF的可执行文件和共享库文件都有。



程序头表是一个结构体数组，它的结构体如下：

```c
typedef struct {
Elf32_Word p_type;
Elf32_Off p_offset;
Elf32_Addr p_vaddr;
Elf32_Addr p_paddr;
Elf32_Word p_filesz;
Elf32_Word p_memsz;
Elf32_Word p_flags;
Elf32_Word p_align;
} Elf32_Phdr;
```

Elf32_Phdr结构体的几个成员与我们使用“readelf -l”命令输出的程序头表的结果相对应，它们的基本含义如下：

- p_type：segment的类型。通常，我们只需要关注LOAD类型的segment，LOAD类型的常量为1。在动态链接中，我们还会看到“DYNAMIC”、“INTERP”等类型。
- p_offset： segment在文件中的偏移。
- p_vaddr： segment的第一个字节在进程的虚拟地址空间中的起始位置。在程序头表中，所有LOAD类型的segment会按照p_vaddr从小到大排列。
- p_paddr： segment的物理装载地址。一般情况下，p_paddr与p_vaddr是一样的。
- p_filesz：segment在ELF文件中所占空间的长度。该值可能为0，表示segment在ELF文件中没有内容。
- p_memsz：在进程的虚拟地址空间中所占用的长度。该值也可能为0。
- p_flags：segment的权限属性。比如，R（可读）、W（可写）、X（可执行）。
- p_align：segment的对齐属性。实际对齐字节等于2的p_align次方。比如，p_align等于10，那么实际的对齐属性就是2的10次方，即1024字节。

对于LOAD类型的segment来说，p_memsz的值不可以小于p_filesz，否则就是不符合常理的。



> 那么，当p_memsz的值大于p_filesz时意味着什么呢？

p_memsz大于p_filesz，表示该segment在内存中所分配的空间超过了其在文件中的实际大小，而这超出的部分会全部用“0”填充。这样做的好处是，在装载ELF可执行文件时，不需要额外设置BSS segment。我们可以把数据Segment的p_memsz扩大，那些扩大的部分就是BSS segment 。因为数据段和BSS段的唯一区别就是，数据段从文件中初始化内容，而BSS段的内容全都初始化为0。所以，在前面的示例中，我们只看了两个LOAD类型的Segment，而不是三个，因为BSS段已经被合并到数据段中了。



## 堆、栈的真面目

操作系统实际上是通过使用VMA来管理进程的虚拟地址空间的。我们知道进程在执行的时候，除了代码和数据外，还会用到堆（Heap）、栈（Stack）等空间。因此， 不仅是可执行文件中的各种segment，还包括进程在运行时所用到的堆、栈等空间也是以VMA的形式存在于进程的虚拟地址空间中的。

一般情况下，一个进程中的堆和栈分别都有一个对应的VMA。在Linux系统下，我们可以通过“/proc”来查看进程虚拟地址空间的分布：

```bash
# ./SectionMapping.elf &
[1] 3292
# cat /proc/3292/maps 
08048000-080b9000 r-xp 00000000 08:01 25007      ./SectionMapping.elf
080d9000-080bb000 rw-p 00070000 08:01 25007      ./SectionMapping.elf
080bb000-080de000 rw-p 080bb000 00:00 0          [heap]
bf7ec000-bf802000 rw-p bf7ec000 00:00 0          [stack]
ffffe000-fffff000 r-xp 00000000 00:00 0          [vdso]
```

其中：

- 第一列是VMA的地址范围。

- 第二列是VMA的权限。r表可读，w表示可写，x表示可执行，p表示私有（COW，Copy on Write），s表示共享。

- 第三列是偏移。表示VMA对应的segment在可执行文件中的偏移。

- 第四列表示可执行文件所在设备的主设备号和次设备号。

- 第五列表示可执行文件的节点号（inode）。

- 第六列是可执行文件的路径。

我们可以看到，该进程有5个VMA。其中，只有前两个VMA是被映射到可执行文件中的两个segment；其它的VMA，它们的主设备号、次设备号以及文件节点号都是0，这表示它们没有被映射到文件中，而这种VMA叫做匿名虚拟内存区域（Anonymous Virtual Memory Area）。

同时，我们还可以看到，有两个区域分别是堆（heap）和栈（stack），它们的大小分别为140KB和88KB。在所有的进程中，几乎都会存在这两个VMA，在C语言中最常用的内存分配函数是malloc()，它所申请的空间就是从堆中分配的，而对于栈来说，我们知道每个线程都有属于自己的栈，对于单线程的程序来说，这个VMA栈全都归它使用。

此外，我们还可以看到，有一个很特殊的VMA叫做“vdso”，它的地址已经位于内核空间了（即大于0xC000 0000的地址），其实它是一个内核模块，进程可以通过访问这个VMA来跟内核进行通信。



## 虚拟地址空间的分布

操作系统通过在进程的虚拟地址空间中划分出一个个的VMA来管理进程的虚拟地址空间。其划分的基本原则是将具有相同权限属性的、有相同可执行文件的映射成一个VMA。通常，一个进程的虚拟地址空间可以分为如下几个VMA区域：

- 代码VMA，权限只读、可执行；有可执行文件
- 数据VMA，权限可读写、可执行；有可执行文件
- 堆VMA，权限可读写、可执行；无可执行文件，匿名，可向上扩展。
- 栈VMA，权限可读写、不可执行；无执行文件，匿名，可向下扩展。

在进程的虚拟地址空间中，常提到的段（Segment），通常就是指这几种VMA。它们的具体分布大致如下：

![image-20200208175413650](/assets/images/2020/image-20200208175413650.png)

> 前面我们在Linux的“/proc”目录下看到的SectionMapping.elf的VMA2的结束地址跟原先预测的不一样，按照计算应该是0x080B C000，但实际上显示出来的是0x080B B000。这是为什么呢？

因为Linux系统在装载ELF文件时使用了一种“Hack”的做法。

在Linux系统中，VMA的概念并非与segment完成对应，Linux规定一个VMA可以映射到某个文件的一个区域，或者不映射到任何文件；而我们这里的第二个segment要求是，前面部分映射到文件中，而后面一部分不映射到任何文件直接为0，也就是说前面的从“.tdata”段到“.data”段部分要建立从虚拟空间到文件的映射，而“.bss”和“\__libcfreeres_ptrs”部分直接为0。

这样两个概念就不同了，所以Linux实际上采用了一种取巧的办法，它在映射完第二个segment之后，把最后一个页剩余的部分清零，然后调用内核中的do_brk()，把“.bbs”和“__libcfreeres_ptrs”的剩余部分放到堆段中。



## Segment地址对齐

我们知道，一个可执行文件最终是要被操作系统装载到物理内存中然后开始运行的，而这个装载过程一般是通过虚拟内存的页映射机制完成的。

在映射过程中，页是映射的最小单位，其大小通常为4096字节，即当我们将一段物理内存和进程的虚拟地址空间建立映射关系时，这段内存空间的长度必须是4096的整数倍，并且这段空间在物理内存和进程的虚拟地址空间中的起始地址必须是4096的整数倍。由于有着长度和起始地址的限制，因此对于可执行文件来说，它应该尽量优化自己的空间和地址的安排，从而节省空间。



### 直接装载segment

下面我们看一个简单的例子：假设有一个ELF文件，它有三个segment需要装载，分别为SEG0、SEG1和SEG2，每个段的长度和在ELF文件中的偏移如下：

|  段  | 长度（字节） | 偏移（字节） |    权限    |
| :--: | :----------: | :----------: | :--------: |
| SEG0 |     127      |      34      | 可读可执行 |
| SEG1 |     9899     |     164      |  可读可写  |
| SEG2 |     1988     |              |    只读    |

我们可以看到，每个segment的长度都不是页大小的整数倍（这也符合实际情况）。而要想装载这个程序，最简单、直接的方法就是每个段分开映射，对于长度不足一页的段则按一个页算。

我们假设这个ELF文件的起始虚拟地址为0x0804 8000，按照这样的映射方式，该ELF文件的各个段的虚拟地址和长度如下：

|  段  | 起始虚拟地址 |  大小  | 有效字节 | 偏移 |    权限    |
| :--: | ------------ | :----: | :------: | :--: | :--------: |
| SEG0 | 0x0804  8000 | 0x1000 |   127    |  34  | 可读可执行 |
| SEG1 | 0x0804  9000 | 0x3000 |   9899   | 164  |  可读可写  |
| SEG2 | 0x0804  C000 | 0x1000 |   1988   |      |    只读    |

因此，我们可以得出如下映射关系：

![image-20200208181616299](/assets/images/2020/image-20200208181616299.png)

我们可以看到，这种对齐方式会产生很多的碎片，从而造成物理空间的浪费。整个可执行文件的三个segment总长度为12 014字节，却占用了5页，即20 480字节，因此空间使用率只有58.6%。

> 那么，该如何解决这种浪费问题呢？



### segment合并

在Unix系统中采取的方法是，让两个segment之间接壤的部分共享同一个物理页，然后将这个物理页在进程的虚拟地址空间中分别映射两次。



这种方式的映射逻辑是这样的，将整个ELF文件看作是一个整体，先把整个ELF文件从最开始到结束，以4096字节为单位在逻辑上分成若干个块，然后每个块都会被装载到物理内存中。而对于那些包含多个segment的块，在进程的虚拟地址空间中，它会被映射多次。

此外，Unix系统会将ELF的文件头也看作是一个segment，将其映射到进程的地址空间中。这样做的好处是，进程中的某一块区域就是整个ELF文件，对于一些要访问ELF文件头的操作（比如，动态链接需需要读取ELF文件头）可以直接通过读写内存地址空间进程。



在这种方式下，上例中的ELF文件的各个段的虚拟地址和长度如下：

|  段  | 起始虚拟地址 | 大小 | 偏移 |    权限    |
| :--: | :----------: | :--: | :--: | :--------: |
| SEG0 | 0x0804  8022 | 127  |  34  | 可读可执行 |
| SEG1 | 0x0804  90A4 | 9899 | 164  |  可读可写  |
| SEG2 | 0x0804  C74F | 1988 |      |  可读可写  |

因此，我们可以得出如下映射关系：

![image-20200208181828466](/assets/images/2020/image-20200208181828466.png)

如上，对于SEG0和SEG1接壤部分的那个物理页，操作系统将它映射两份到进程的虚拟地址空间中，一份为SEG0，另一份为SEG1；对于SEG1和SEG2接壤部分也采取同样的操作；而其他的页按照正常的页颗粒进行映射。

我们可以看到，内存空间得到了充分的利用，原来要用5个物理页，现在只需3个页。在这种映射方式下，一个物理页可能包含多个segment的数据，比如文件头、代码段、数据段和BSS段的长度加起来都没有超过4096字节，那么可以将它们装载到同一个物理页。