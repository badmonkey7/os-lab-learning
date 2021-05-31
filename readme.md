# ucore learning

> 实验环境: Manjaro Linux 21.0.5
>
> 编译器:clang 11.1.0
>
> 硬件模拟器:qemu
>
> 文档:https://objectkuan.gitbooks.io/
>
> 答案:https://oscourse-tsinghua.github.io/

## day1

编译器改用clang,gcc编译出来的bootloader会大于510字节。

https://github.com/chyyuu/os_kernel_lab/issues/50

**make流程**

1. 编译kernel源码并链接
2. 编译bootloader并链接(注意要小于510字节)，因为作为引导扇区(第一个block，一个block 512字节)
3. 将bootloader和kernel写入到一起,bootloader作为第一个block,kernel紧随其后

## day2

gdb 调试qemu启动的bios时，无法正确识别指令地址的问题。

![image-20210529125830184](img/day2-1.png)

https://stackoverflow.com/questions/62513643/qemu-gdb-does-not-show-instructions-of-firmware

https://stackoverflow.com/questions/32955887/how-to-disassemble-16-bit-x86-boot-sector-code-in-gdb-with-x-i-pc-it-gets-tr/32960272#32960272

这是由于bios启动的时候处于实模式，20位寻址空间，cs:eip的寻址模式，而gdb在计算地址时只考虑了eip,没有计算考虑cs寄存器，所以调试的时候会有问题。解决方法参考第一个stackoverflow

![image-20210529131039195](img/day2-2.png)

顺便回顾一下实模式和保护模式的概念

https://zhuanlan.zhihu.com/p/42309472

**实模式**，段寄存器:通用寄存器 => 段基址:段内偏移 16位寄存器 20位寻址空间 

**保护模式**,32位寄存器(段寄存器仍然是16位)，通用寄存器保存偏移，段寄存器不再保存段基址，而是保存全局描述符表(GDT)的索引，通过索引找到段描述符,段描述符记录了段的基址和相关信息(读写权限之类的)，用于保护段！！

需要注意的是，进入kernel后就是保护模式了，所以需要再改一下gdb的配置。

**A20 Gate**

a20是80286后产生的一个概念，由于80286有24根寻址线，寻址能力比8086高了16倍，为了向下兼容使用a20位表示是否启用80286的24位寻址。可以用于实模式到保护模式的转换，具体的如何开启a20是通过写io端口实现的。

**GDT表**

全局描述符表，保存多个段描述符，其地址在GDTR(全局描述表寄存器)，GDTR为48位，高32位为基址，低16位为段界限。lgdt汇编指令将gdt表地址加载到gdtr中

GDT表第一项为空段描述符，GDTR中的段界限为8*N-1其中N为段描述符的个数(包含空段描述符)

**段描述符**

64bit 8字节 结构如下

```c
/* segment descriptors */
struct segdesc {
    unsigned sd_lim_15_0 : 16;        // low bits of segment limit
    unsigned sd_base_15_0 : 16;        // low bits of segment base address
    unsigned sd_base_23_16 : 8;        // middle bits of segment base address
    unsigned sd_type : 4;            // segment type (see STS_ constants)
    unsigned sd_s : 1;                // 0 = system, 1 = application
    unsigned sd_dpl : 2;            // descriptor Privilege Level 特权级实现保护机制
    unsigned sd_p : 1;                // present
    unsigned sd_lim_19_16 : 4;        // high bits of segment limit 段界限粒度位为0时默认单位是字节
    unsigned sd_avl : 1;            // unused (available for software use)
    unsigned sd_rsv1 : 1;            // reserved
    unsigned sd_db : 1;                // 0 = 16-bit segment, 1 = 32-bit segment
    unsigned sd_g : 1;                // granularity: limit scaled by 4K when set 粒度 
    unsigned sd_base_31_24 : 8;        // high bits of segment base address
};
```

主要包括了段的基址，段的界限，段的属性(粒度，类型，特权级，存在位，访问位)

![segdesc-img](img/day2-3.png)

**选择子**

保护模式下16位的段寄存器，现在用来作为选择子(类似于索引的概念)而不是段基址。其中高13位为索引，倒数第三位为表指示位(LDT和GDT),最低的两位为请求特权级。

![selector](img/day2-4.png)

特权级有4中，最高特权级为0,最低特权级为3.ucore中0为内核态，3为用户态。

**特权级保护模型**

主要涉及到几个PL(privilege level)的概念

CPL(current privilege level)当前特权级，保存在cs寄存器的最低2位

DPL(Descriptor privilege level)描述符特权级，描述符表中的特权级也是两位

RPL(Request privilege level)请求特权级，选择子中的最低两位。

![protected-model](img/day2-5.png)

允许内核代码加载特权级比较低的段。注意对于堆栈寄存器要求CPL,RPL,DPL完全一致才可以被加载。

## day3

elf文件格式，bootloader在第一扇区，而os的elf文件在第二扇区，bios需要加载elf到内存中，这个过程是根据elf文件头的格式来进行的。bios会将elf中的代码段依次加载到内存中，然后执行elf的entry。

**elf文件格式**

关于elf链接的一些知识，可以参考这个文档  https://docs.oracle.com/cd/E19683-01/816-1386/index.html

三种类型:可执行文件(类似exe文件)，可重定位文件(还不太懂)，共享目标文件(一些动态链接库，linux下.so文件，windows下.dll文件)

下面是可执行elf文件的文件头

```c
/* file header */
struct elfhdr {
    uint32_t e_magic;     // must equal ELF_MAGIC
    uint8_t e_elf[12];
    uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
    uint16_t e_machine;   // 3=x86, 4=68K, etc.  
    uint32_t e_version;   // file version, always 1
    uint32_t e_entry;     // entry point if executable
    uint32_t e_phoff;     // file position of program header or 0 程序头表的偏移
    uint32_t e_shoff;     // file position of section header or 0 表偏移
    uint32_t e_flags;     // architecture-specific flags, usually 0
    uint16_t e_ehsize;    // size of this elf header
    uint16_t e_phentsize; // size of an entry in program header
    uint16_t e_phnum;     // number of entries in program header or 0 程序头表中的项数
    uint16_t e_shentsize; // size of an entry in section header
    uint16_t e_shnum;     // number of entries in section header or 0 表中项数
    uint16_t e_shstrndx;  // section number that contains section name strings 字符section的表号
};
```

可以看到elf中有两个表一个是program header table一个是section header table，首先看一下program header是干嘛的。

> program header描述与程序执行直接相关的目标文件结构信息，用来在文件中定位各个段的映像，同时包含其他一些用来为程序创建进程映像所必需的信息。

结构如下

```
/* program section header */
struct proghdr {
    uint32_t p_type;   // loadable code or data, dynamic linking info,etc.
    uint32_t p_offset; // file offset of segment
    uint32_t p_va;     // virtual address to map segment
    uint32_t p_pa;     // physical address, not used
    uint32_t p_filesz; // size of segment in file
    uint32_t p_memsz;  // size of segment in memory (bigger if contains bss）
    uint32_t p_flags;  // read/write/execute bits
    uint32_t p_align;  // required alignment, invariably hardware page size
};
```

实际的elf结构图

![elf](img/day3-1.gif)

可以看到program header 是紧跟在elf header后面的一串结构体数组。

对于可执行文件好像有没有section header 无所谓，但是对于可重定位文件必须有section header。section header包含了用于定位和isolate文件中每个section的信息。

```c
 typedef struct {
               uint32_t   sh_name;// name of section 
               uint32_t   sh_type;// 类型
               uint32_t   sh_flags;//section 的一些属性 
               Elf32_Addr sh_addr; // 内存中的地址
               Elf32_Off  sh_offset; //距离文件头(file offset)起始的偏移量
               uint32_t   sh_size;// 大小
               uint32_t   sh_link;//section header table index link 根据不同的section type 有不同的含义
               uint32_t   sh_info;// 额外的一些信息
               uint32_t   sh_addralign;// 对齐限制
               uint32_t   sh_entsize;// entry size
           } Elf32_Shdr;
```

更详细的信息查阅 https://docs.oracle.com/cd/E19455-01/806-3773/elf-2/index.html

**中断**

中断向量表(Interrupt Vector Table简称IVT),在实模式下使用，用于存放256个中断对应的中断处理程序的地址，IVT通常位于`0000:0000H`的位置大小为`0x400`每个中断向量为`0x4`字节。

中断描述符(Interrupt Description Table简称IDT),IA-32体系架构使用。实模式中断向量表的保护模式副本，用于保存中断服务例程(Interrupt Service Routines 即ISR)所在的位置。类似于全局描述符表。

中断的分类:

- 硬件中断(hardware interrupt)
	- 可屏蔽中断(maskable interrupt)，可以通过中断屏蔽寄存器配置。
	- 非可屏蔽中断(non-maskeable interrupt简称nmi)，常见的有时钟中断
	- 处理器间中断(interprocessor interrupt)，一个处理器发出，另一个处理器接收，用于多处理器系统。
	- 伪中断(spurious interrupt),电气信号异常等不希望被产生的硬件中断，通常是设备的问题.
- 软件中断(software interrupt) 通过CPU指令将用于态切换到内核态，通常用于实现系统调用。

## day4

