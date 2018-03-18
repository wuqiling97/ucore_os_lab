# Lab1

## 知识点

本实验涉及到了以下课程上提到的知识点：

1.  系统启动，包括BIOS，实模式到保护模式等
2.  x86中断和异常机制
3.  中断描述表
4.  C语言函数调用约定

本实验涉及的其他知识点：

1.  A20 gate如何开启
2.  如何访问硬盘
3.  ELF文件格式

OS原理中很重要，但在实验中没有对应上的知识点：

​    无

## 练习1

### 1.1 ucore.img的生成

#### 1. ucore内核代码的编译

命令 `gcc -I... -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -c kern/*.c -o obj/kern/*.o` 的含义为：

1.  `-I` 设置了附加的头文件查找目录
2.  `-fno-builtin -nostdinc` 禁用了内置的库
3.  `-Wall` 开启了所有警告
4.  `-ggdb -gstabs` 添加了调试信息
5.  `-m32` 输出的obj文件为32位
6.  `-fno-stack-protector` 关闭栈保护代码的生成
7.  `-c` 只编译和汇编，不链接
8.  `-o obj/kern/*.o` 指定obj文件的输出目录


#### 2. 链接生成内核elf文件

命令 `ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel obj/kern/*.o` 的含义为：

1.  `-m elf_i386` 生成32位elf文件
2.  `-nostdlib` 不使用C语言自带的库
3.  `-T tools/kernel.ld` 使用自定义的链接脚本 `kernel.ld` 
4.  `-o bin/kernel obj/kern/*.o` 将 `obj/kern/*.o` 链接生成 `bin/kernel` 文件

#### 3. bootloader编译

命令 `gcc -I... -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Os -c boot/bootasm.S -o obj/boot/bootasm.o` 的含义为：

1.  `-I... -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector` 已在 [1. ucore内核代码的编译](1. ucore内核代码的编译) 当中说明
2.  `-Os` 对代码的大小优化，防止超出512byte的限制
3.  将 `boot/bootasm.S boot/bootmain.c` 编译为obj文件

#### 4. 链接生成bootblock elf文件

命令 `ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o` 的含义为：

1.  `-m elf_i386 -nostdlib` 已经提到
2.  `-N` 不将数据对齐至页边界，不将 text 节只读
3.  `-e start` 将程序入口设置为 `start`
4.  `-Ttext 0x7C00` 将 `.text` 段的起始地址设为0x7C00
5.  `obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o` 将 `bootasm.o bootmain.o` 链接为 `bootblock.o`

#### 5. 生成bootblock二进制文件

命令 `objcopy -S -O binary obj/bootblock.o obj/bootblock.out` 的含义为：

1.  `-S` 不拷贝所有符号和重定位信息
2.  `-O binary` 将输入文件转换为二进制代码
3.  `obj/bootblock.o obj/bootblock.out` 将elf文件` bootblock.o` 拷贝为二进制代码 `bootblock.out`

#### 6. 生成boot扇区文件

命令 `bin/sign obj/bootblock.out bin/bootblock` 的含义为：

1.  用 `sign.c` 编译生成的可执行文件检查 `bootblock.out` 是否小于510 bytes，如果是，则用0将其补充到510 bytes，并加上2 bytes的结束字节0x55AA

#### 7. 生成ucore.img文件

包含三个命令：

1.  `dd if=/dev/zero of=bin/ucore.img count=10000` 从不断提供0的设备 `/dev/zero` 读取，输出到 `bin/ucore.img` ，共10000个扇区
2.  `dd if=bin/bootblock of=bin/ucore.img conv=notrunc`  读取 `bootblock` 并写入 `ucore.img` 当中，并且不截断 `ucore.img` ，即不清空改文件
3.  `dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc` 读取 `kernel` 并写入 `ucore.img` 当中，起始位置为第1个扇区

### 1.2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

有如下特征：

1.  代码段长度小于510字节
2.  最后2字节是0x55AA

## 练习2

### 2.1 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行

将 `tools/gdbinit` 改为：

```
target remote :1234
set arch i8086
```

并执行 `make debug` 进行调试，发现gdb指示当前地址为0xfff0，`x/2i $eip` 得到的指令都是0。联想到8086架构是通过 \$cs\*16+\$ip 来确定访问的地址，说明gdb对8086架构的支持不是很好，因此手动输入地址 `x/2i $cs\*16+$ip` 看到了 `ljmp $0xf000,$0xe05b` ，指令正确。使用 `si` 即可单步跟踪BIOS。

### 2.2 在初始化位置0x7c00设置实地址断点

进入gdb之后，输入

```
b *0x7c00
c
```

即可在0x7c00处停下，输入 `x/10i $eip` ，可以看到汇编代码和 `bootasm.S` 相同，说明断电正常。

### 2.3 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较

将 `tools/gdbinit` 改为

```
file obj/bootblock.o
target remote :1234
b *0x7c00
b bootmain
c
```

并且在Makefile中将debug改为

```
debug: $(UCOREIMG)
	$(V)$(QEMU) -d in_asm -D $(BINDIR)/q.log -S -s -parallel stdio -hda $< -serial null &  # 加入-d in_asm -D $(BINDIR)/q.log来记录汇编指令
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

执行 `make debug` 之后，即可在0x7c00停下，再次执行 `c` 之后可在bootmain停下。此时查看 `q.log` 会发现代码与 `bootasm.S` 相同；执行 `x/10i $pc` 可以看到汇编代码与 `obj/bootblock.asm` 当中 `bootmain` 函数的汇编代码相同

### 2.4 自己找一个bootloader或内核中的代码位置，设置断点并进行测试

在0x7c32设置断点，使用 `disas` 即可查看到 `call  0x7d0d<bootmain>` 以及之前的代码

## 练习3

分析bootloader 进入保护模式的过程：

在进入0x7c00，即bootasm.S之后，首先将寄存器 `ax, ds, es, ss` 清零。

随后开启A20 gate，开启之后，才能禁止地址回绕，否则CPU只能访问2k~2k+1 MB的内存，开启的过程如下

```assembly
seta20.1:
    inb $0x64, %al     # 等待8042键盘控制器不忙
    testb $0x2, %al
    jnz seta20.1   
                   
    movb $0xd1, %al    # 0xd1 -> port 0x64
    outb %al, $0x64    # 0xd1表示要写入Output Port
                   
seta20.2:          
    inb $0x64, %al     # 等待8042键盘控制器不忙
    testb $0x2, %al
    jnz seta20.2   
                   
    movb $0xdf, %al    # 0xdf -> port 0x60
    outb %al, $0x60    # 打开A20
```

然后初始化GDT表，一个简单的GDT已经硬编码在bootloader代码当中，可以直接载入

```assembly
	lgdt gdtdesc
```

随后开启保护模式：将CR0寄存器PE位置1即可

```assembly
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

然后通过ljmp更新CS

```assembly
	ljmp $PROT_MODE_CSEG, $protcseg
```

设置段寄存器，建立堆栈

```assembly
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
```

此时转到保护模式完成，可以进入bootmain了

## 练习4

分析bootloader加载ELF格式的OS的过程：

`bootmain.c` 当中有2个读取硬盘的函数。其中 `readsect(void *dst, uint32_t secno)` 将secno的扇区读取到dst；`readseg(uintptr_t va, uint32_t count, uint32_t offset)` 对 `readsect` 进行了包装，可以读取任意地址和长度的内容。

在bootmain当中，程序的作用如下

```c
void
bootmain(void) {
    // 读取elf头
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // 通过头4字节的magic number判断是否有效的elf文件
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // elf头描述了elf文件应该加载到内存的什么位置
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // 根据elf头存储的入口信息，跳转到内核入口
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

## 练习5

按照C语言的调用约定，在刚进入一个函数时，栈帧的结构如下

```
(high addr)
arg4          ebp + 5
arg3          ebp + 4
arg2          ebp + 3
arg1          ebp + 2
return addr   ebp + 1
old ebp       <= ebp, esp
(low addr)
```

再结合注释，这个函数就很容易写出了。

其中值得注意的是，传入 `print_debuginfo` 的是eip-1，这是因为eip指向的是下一条指令的地址，而 `print_debuginfo` 希望处理的是当前指令，所以要-1。

在更新eip, ebp的值得时候要注意顺序，要先更新eip，后ebp，否则eip使用的ebp+4的地址将会是上一层的ebp，导致错误。

调用栈当中最深的一层为

```
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d6d --
```

ebp是bootmain栈帧的起始位置，eip是bootmain调用内核函数的下一条指令，而4个参数是从0x7c00开始的，刚好为0x7c00~0x7c0f的数据，其实是代码。



与答案对比后，我发现我的循环体当中eip和ebp更新的顺序颠倒，导致除了最顶层以外的eip全部错误，对调后问题解决。

## 练习6

### 6.1 

中断向量表一个表项占用8字节，其中16-31位为段选择子，由于历史原因，段偏移量被拆散到0-15位和48-63位。二者联合就定义了中断处理程序的入口地址。

### 6.2 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init

首先用extern声明定义在vector.S里面的中断向量表，然后用SETGATE宏加循环填充向量表，其中T_SWITCH_TOK由于是从内核返回用户态，故dpl要使用用户态，也就是3；最后告诉CPU idt的地址。



与答案对比后，发现我直接硬编码了中断向量的条目数，答案的实现适用范围会更广。

### 6.3 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

注释已经说得很清楚了，只需要在时钟中断的时候将ticks+1，并且每100次调用一次 `print_ticks()` 即可。



与答案对比后，发现想写得不一样都很难。

