# Report Lab1

计76 周煜威 2016010165



## ex1

#### 列出本实验各练习中对应的OS原理的知识点，并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。

1) make和makefile，通过阅读静态代码进行学习和了解；

2) ucore.img的组成及其组成部分的生成方式；

3) bootloader主引导扇区的相关特征。



#### 操作系统镜像文件ucore.img是如何一步一步生成的？

**1 生成ucore.img之前，需要首先生成kernel和bootblock。**

```makefile
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

# dd: 用指定大小和转换方式复制文件
# if: 输入文件
# of: 输出文件
# /dev/zero: 写入字符串0
# count: 复制的块数
# seek: 进行复制前跳过的块数
# conv: 转换方式
# notrunc: 不截短输出文件
```

**2 而生成kernel和bootblock，又需要通过之前所生成的各种.o目标文件，如下：**

**kernel**: init.o stdio.o readline.o panic.o kdebug.o kmonitor.o clock.o console.o picirq.o intr.o trap.o vectors.o trapentry.o pmm.o  string.o printfmt.o

**bootblock**: bootasm.o bootmain.o sign

```makefile
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)

# ld: 链接目标文件生成可执行文件
# -m elf_i386: 编译所生成的代码用于i386
# -nostdlib:不使用标准库
# -T: 可执行文件入口
```

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)

# -N: 代码段和数据段均为可读可写
# -e start: 函数入口为start
# -Ttext 0x7c00: 代码段起始地址为0x7c00
```

**3 生成.o文件则需要.c或.s文件进行编译**

```makefile
# .c文件生成.o文件举例
gcc -Ikern/init/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o

# -fno-builtin: 不使用C语言内建函数
# -fno-PIC: 不生产位置无关代码
# -Wall: 开启编译警告
# -ggdb: 为gdb生成调试信息
# -m32: 生成32位环境代码
# -gstabs: 生成stabs格式调试信息
# -nostdinc: 不使用标准库
# -fno-stack-protector: 不检测栈溢出
```

```makefile
# .s文件生成.o文件举例
gcc -Iboot/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
```



#### 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

**1 bin/sign用作判断硬盘主引导扇区是否符合规范，根据makefile中的命令，可以从sign.c观察其特征。**

```makefile
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

**2 文件sign.c中，return -1的情况不满足规范，因此可以得出符合规范的主引导扇区特征为：**

**1) 扇区大小为512字节**

```c
if (st.st_size > 510) {
	fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
	return -1;
}
if (size != st.st_size) {
	fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
	return -1;
}
if (size != 512) {
	fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
	return -1;
}
```

**2) 扇区最后两个字节为0x55和0xAA**

```c
buf[510] = 0x55;
buf[511] = 0xAA;
```



## ex2

#### 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

```makefile
# gdbinit
file bin/kernel
target remote :1234
set architecture i8086
```

```makefile
# terminal
make debug
0x0000fff0 in ?? ()
The target architecture is assumed to be i8086
(gdb) x $pc
0xfff0: 0x00000000
(gdb) si
0x0000e05b in ?? ()
```

#### 在初始化位置0x7c00设置实地址断点,测试断点正常。

```makefile
# terminal
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.

Breakpoint 1, 0x00007c00 in ?? ()
```

#### 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

反汇编得到的代码与两个文件中的汇编代码基本一致。

```makefile
# terminal
# si进行单步跟踪，为记录方便，不将每一次单步列出
(gdb) x /50i $pc
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
   0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
   0x7c12:      out    %al,$0x64
   0x7c14:      in     $0x64,%al
   0x7c16:      test   $0x2,%al
   0x7c18:      jne    0x7c14
   0x7c1a:      mov    $0xdf,%al
   0x7c1c:      out    %al,$0x60
   0x7c1e:      lgdtl  (%esi)
   0x7c21:      insb   (%dx),%es:(%edi)
   0x7c22:      jl     0x7c33
   0x7c24:      and    %al,%al
   0x7c26:      or     $0x1,%ax
   0x7c2a:      mov    %eax,%cr0
   0x7c2d:      ljmp   $0xb866,$0x87c32
   0x7c34:      adc    %al,(%eax)
   0x7c36:      mov    %eax,%ds
   0x7c38:      mov    %eax,%es
   0x7c3a:      mov    %eax,%fs
   0x7c3c:      mov    %eax,%gs
   0x7c3e:      mov    %eax,%ss
   0x7c40:      mov    $0x0,%ebp
   0x7c45:      mov    $0x7c00,%esp
   0x7c4a:      call   0x7d11
   0x7c4f:      jmp    0x7c4f
   ......
```

#### 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

在readreg函数设置断点，得到的反汇编代码与bootblock.asm中的部分一致。

```makefile
# terminal
(gdb) b *0x7c72
Breakpoint 2 at 0x7c72
(gdb) c
Continuing.

Breakpoint 2, 0x00007c72 in ?? ()
(gdb) x /10i $pc
=> 0x7c72:      push   %ebp
   0x7c73:      mov    %esp,%ebp
   0x7c75:      push   %edi
   0x7c76:      lea    (%eax,%edx,1),%edi
   0x7c79:      mov    %ecx,%edx
   0x7c7b:      and    $0x1ff,%edx
   0x7c81:      push   %esi
   0x7c82:      sub    %edx,%eax
   0x7c84:      shr    $0x9,%ecx
   0x7c87:      mov    %eax,%esi
(gdb) 
```



## ex3

#### 请分析bootloader是如何完成从实模式进入保护模式的。

**1 实模式下初始化**

将中断关闭，并将段寄存器置零

```assembly
.globl start
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

**2 开启A20，也就是使能32位地址线**

根据下一段开启A20的方式，通过8042输入输出缓冲打开A20 gate

```assembly
seta20.1:
    inb $0x64, %al                        # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                       # 0xd1 -> port 0x64
    outb %al, $0x64                       # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                        # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                       # 0xdf -> port 0x60
    outb %al, $0x60                       # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

**3 初始化gdt表，并进入保护模式**

加载gdt表，将cr0寄存器置为0x1，长跳转至保护模式代码段

```assembly
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    ljmp $PROT_MODE_CSEG, $protcseg
```

**4 段寄存器设置初值，进入bootmain**

将寄存器ds, es, fs, gs, ss赋值为0x10，并初始化ebp和esp寄存器，调用bootmain

```assembly
.code32                                             # Assemble for 32-bit mode
protcseg:
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```



#### 为何开启A20，以及如何开启A20

1. 实模式下为了保持原有的向下兼容性，模仿早期20根地址线的情形。但当进入保护模式时，需要32位地址线寻址，因此需要打开A20。

2. 开启A20的方法是：
   1. 等待8042 Input buffer为空；
   2. 发送Write 8042 Output Port （P2）命令到8042 Input buffer；
   3. 等待8042 Input buffer为空；
   4. 将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer。



#### 如何初始化GDT表

载入引导区中的gdt



#### 如何使能和进入保护模式

使能：将cr0寄存器设为0x1

进入：地址长跳转



## ex4

#### bootloader如何读取硬盘扇区的？

由“硬盘访问概述”内容可以了解到，每次单独访问一个硬盘扇区时，首先根据0x1f7地址中的状态判断磁盘当前是否空闲。若空闲，那么依次提供读写扇区数、扇区地址、主盘从盘等信息。再次等待完成后读取扇区。

```c
static void
readsect(void *dst, uint32_t secno) {
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    waitdisk();

    insl(0x1F0, dst, SECTSIZE / 4);
}
```

#### bootloader是如何加载ELF格式的OS？

1 首先读取ELF文件头，根据第一个magic属性判断是否合法。

```c
	// read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
```

2 获取program header 表的位置偏移和表中的入口数目，即phoff和phnum。根据以上信息对磁盘扇区进行依次读取。从读取的内容中获取入口地址，并进行访问。

```c
    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```



## ex5

#### 请完成实验，看看输出是否与上述显示大致一致，并解释最后一行各个数值的含义。

输出与练习五中所给的输出类似。

最后一行的含义分别为：

1. ebp：存有caller的ebp的栈中地址
2. eip：caller中call指令后面的一条指令地址
3. args：调用callee时的参数

具体而言，最后一行的含义为：

1. 在kernel_init函数之前的函数，即bootmain函数
2. 其ebp为0x00007bf8， eip为0x00007d72，且栈中地址为0x00007bf8处的内容为0x00000000
3. 调用bootmain函数处在bootasm.S中的call bootmain



## ex6

#### 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

中断描述符表一个表项8字节，其中0-1字节为前16位，6-7字节为后16位，拼起来即为中断处理代码的入口。



