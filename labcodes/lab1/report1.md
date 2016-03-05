# Lab1 实验报告

计24 柯均洁 2012011335

## 练习1：理解通过make生成执行文件的过程。

1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

    > 执行命令

    ```
    $ make "V="

    ```

    > 部分输出如下结果

    ```
    + cc kern/init/init.c
    gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
    ......
    + cc libs/printfmt.c
    gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/printfmt.c -o obj/libs/printfmt.o
    + ld bin/kernel
    ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
    + cc boot/bootasm.S
    gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
    + cc boot/bootmain.c
    gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
    + cc tools/sign.c
    gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
    gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
    + ld bin/bootblock
    ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
    'obj/bootblock.out' size: 472 bytes
    build 512 bytes boot sector: 'bin/bootblock' success!
    dd if=/dev/zero of=bin/ucore.img count=10000
    dd if=bin/bootblock of=bin/ucore.img conv=notrunc
    dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
    ```

    > 从以上的输出可以看出，为了生成ucore.img，首先需要生成kernel、bootblock。
    
    > 具体步骤如下：

        1. 执行`cc`命令将kernel源代码编译成目标文件(.o)
        2. 执行`ld`命令将目标文件链接起来得到kernel的可执行文件
        3. 执行`cc`命令将bootloader的汇编代码(bootasm.S)和c代码(bootmain.c, sign.c)编译生成目标文件
        4. 执行`ld`将bootasm.o bootmain.o链接起来，得到bootblock可执行文件
        5. 执行`dd`命令将bootloader和kernel装入虚拟硬盘`ucore.img`
            - 生成一个有10000个块的文件，每个块默认512字节，用0填充
            - 把bootblock中的内容写到第一个块
            - 从第二个块开始写kernel中的内容

    > `cc`命令的参数含义如下：

        - `-I dir` 把`dir`添加到搜索头文件的目录集合
        - `-fno-builtin` 表示不识别没有以`__builtin_`开头的built-in函数
        - `-Wall` 显示所有的warning
        - `-ggdb` 当结合gdb使用时输出调试信息
        - `-gstabs` 使用`stabs`格式输出调试信息
        - `-o file` 输出到文件`file`
        - `-nostdinc` 不在标准的系统目录里搜索头文件
        - `-c file` 输入源文件`file`
    
    > `ld`命令的参数含义如下：

        - `-m env` 设置程序的模拟运行环境
        - `-nostdlib` 不使用标准函数库
        - `-N` 设置代码段和数据段可读、可写。
        - `-e entry` 把`entry`作为程序开始执行的显式标志。
        - `-o file` 指定输出到文件`file`

    > `dd`命令的参数含义如下：

        - `if=file` 指定输入文件
        - `of=file` 指定输出文件
        - `count=n` 从输入文件拷贝`n`个字节块（默认512字节）
        - `conv=notrunc` 不截断输出文件
        - `seek=n` 跳过输出文件的前`n`个字节快（默认512字节）

2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
   
    > 主引导扇区的读取代码在`tools/sign.c`中，该文件完成了特征的标记。
    ```
    if (st.st_size > 510) {
        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
        return -1;
    }
    ......
    buf[510] = 0x55;
    buf[511] = 0xAA;
    ```
    > 以上两段代码表明，主引导扇区应该满足两个条件：

        1. 不能超过512字节
        2. 最后两个字节应该是0x55AA

## 练习2：使用qemu执行并调试lab1中的软件。

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

    > 修改lab1init文件
    ```
    file bin/kernel
    target remote :1234
    set architecture i8086
    b *0x7c00
    ```

    > 在命令行中运行`make lab1-mon`，则程序会在BIOS的第一条指令处，可以通过`si`命令进行单步调试。

2. 在初始化位置0x7c00设置实地址断点,测试断点正常。

    > 在gdb中输入如下命令设置断点
    ```
    (gdb) b *0x7c00
    ```

    > 运行c继续执行，则程序会在0x7c00停下来

3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

    > 不修改lab1init，运行`make lab1-mon`后，在gdb中通过`si`命令单步执行。

    > 或者通过使用 `x /5i $pc` 查看当前 PC 寄存器指向的地址及之后的 5 条指令。
    ```
    => 0x7c00:      cli
       0x7c01:      cld
       0x7c02:      xor    %ax,%ax
       0x7c04:      mov    %ax,%ds
       0x7c06:      mov    %ax,%es
    ```

    > 通过比对可以发现，程序所执行的汇编代码跟`bootasm.S`中的代码相同。

4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

    > 在gdb中输入
    ```
    (gdb) b kern_init
    (gdb) c
    ```

    > 则程序顺利停止在kern_init函数中

## 练习3：分析bootloader进入保护模式的过程
0. bootloader进入保护模式的过程

    > 首先清理环境：包括将flag置0和将段寄存器置0
    > 打开A20 Gate。从理论上讲，打开A20 Gate的方法是通过设置8042芯片输出端口（64h）的2nd-bit，但事实上，当你向8042芯片输出端口进行写操作的时候，在键盘缓冲区中或许还有别的数据尚未处理，因此须首先处理这些数据。
    ```
    seta20.1:
        inb $0x64, %al
        testb $0x2, %al
        jnz seta20.1

        movb $0xd1, %al
        outb %al, $0x64

    seta20.2:
        inb $0x64, %al
        testb $0x2, %al
        jnz seta20.2

        movb $0xdf, %al
        outb %al, $0x60
    ```
    > 装载全局描述符，并设置cr0（PE位设置为1）进入使系统进入保护模式
    ```
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    ```
    > 接下来系统进入32位的保护模式，进行了一个关键跳转
    ```
    ljmp $PROT_MODE_CSEG, $protcseg
    ```
    > 进行保护模式下段寄存器的初始化，并跳入c代码运行
    ```
    protcseg:
        movw $PROT_MODE_DSEG, %ax
        movw %ax, %ds
        movw %ax, %es
        movw %ax, %fs
        movw %ax, %gs
        movw %ax, %ss

        movl $0x0, %ebp
        movl $start, %esp
        call bootmain
    ```


1. 为何开启A20，以及如何开启A20

    > Intel 8086处理器提供20根地址线，但是在80286处理器中，系统的地址总线变为24根，为了解决向下兼容问题，IBM在PC AT计算机上加了一个A20硬件模块。
    > 当A20地址线控制禁止时，实模式程序在访问1MB之上的部分地址都会被 wrap 回来，强制使用 1MB 的地址空间，1MB以上的地址不可访问。因此，在实模式下访问高端内存区和在保护模式下，都必须开启A20，这样就可以访问所有内存地址了。  
    > 开启A20的时候需要经过以下几个步骤：
        1. 从0x64端口读入input buffer的状态，看第一位是否为1，如果为1说明输入缓冲区有内容，则不断循环等待，直到第一位为零，8042输入缓冲区为空。
        2. 将0xb1写到0x64端口，表明要写数据到8042的P2端口
        3. 重复1，等待8042输入缓冲区为空。
        4. 将0xdf写入0x60，表明P2的A20被设置为1，内存回卷模式被禁止。

2. 如何初始化GDT表

    > 使用`lgdt gdtdesc`指令对GDT表进行初始化。将含有表的基址和大小的信息装入GDTR寄存器gdtdesc中，其中gdt的大小被设置为0x18，gdt基址被设置为0x0.
    ```
    gdtdesc:
        .word 0x17
        .long gdt 
    ```

3. 如何使能和进入保护模式

    > 将cr0寄存器中的PE位置为1，即可使能保护模式。
    ```
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
    ```
    
    > 使用一个长跳转指令`ljmp $PROT_MODE_CSEG, $protcseg`进入保护模式。通过长跳转更新cs的基地址后设立段寄存器，建立堆栈
    ```
    ljmp $PROT_MODE_CSEG, $protcseg
    .code32
    protcseg:
        movw $PROT_MODE_DSEG, %ax
        movw %ax, %ds
        movw %ax, %es
        movw %ax, %fs
        movw %ax, %gs
        movw %ax, %ss
        movl $0x0, %ebp
        movl $start, %esp
    ```
    > 则转到保护模式完成，进入bootmain方法开始执行。
    ```
    call bootmain
    ```

## 练习4：分析bootloader加载ELF格式的OS的过程
1. bootloader如何读取硬盘扇区的？

    > 调用`readseg`函数，再调用`readect`函数依次读取磁盘扇区。
    > `readsect`函数从设备的第secno扇区读取数据到dst位置。读取磁盘扇区的时候，先等待硬盘就绪，然后向相应端口0x1F2到0x1F7写入需要的数据，再次等待硬盘就绪，然后读取扇区数据。
```
    static void
    readsect(void *dst, uint32_t secno) {
        waitdisk();
    
        outb(0x1F2, 1);                         // 设置读取扇区的数目为1
        outb(0x1F3, secno & 0xFF);
        outb(0x1F4, (secno >> 8) & 0xFF);
        outb(0x1F5, (secno >> 16) & 0xFF);
        outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
        // 将32位的磁盘号secno分成四段，每段8位，依次写入0x1F6~0x1F3

        outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
    
        waitdisk();

        insl(0x1F0, dst, SECTSIZE / 4);         // 读取数据到dst位置
    }
```

    > `readseg`函数简单包装了`readsect`，可以从设备读取任意长度的内容。
```
    static void
    readseg(uintptr_t va, uint32_t count, uint32_t offset) {
        uintptr_t end_va = va + count;
    
        va -= offset % SECTSIZE;
    
        uint32_t secno = (offset / SECTSIZE) + 1; 
        // 加1因为0扇区被引导占用
        // ELF文件从1扇区开始
    
        for (; va < end_va; va += SECTSIZE, secno ++) {
            readsect((void *)va, secno);
        }
    }
```

2. bootloader是如何加载ELF格式的OS？

    > 在bootmain函数中，
```
    void
    bootmain(void) {
        // 首先读取ELF的头部，大小为8个扇区
        readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    
        // 通过储存在头部的幻数判断是否是合法的ELF文件
        if (ELFHDR->e_magic != ELF_MAGIC) {
            goto bad;
        }
    
        struct proghdr *ph, *eph;
    
        // 用ELF文件头中的e_phoff和e_phnum信息来创建程序头，程序头是描述了ELF文件应加载到内存什么位置的描述表头地址
        ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
        eph = ph + ELFHDR->e_phnum;
    
        // 按照描述表将ELF文件中数据载入内存
        for (; ph < eph; ph ++) {
            readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
        }

        // 根据ELF头部储存的入口信息，找到内核的入口
        ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    
    bad:
        outw(0x8A00, 0x8A00);
        outw(0x8A00, 0x8E00);
        while (1);
    }
```

## 练习5：实现函数调用堆栈跟踪函数

    > 在C函数调用栈中，`ss:ebp`存着caller's ebp，`ss:[ebp+4]`存着返回地址，`ss:[ebp+8]`之后存着函数调用的参数
    > 实现过程：在循环中依次输出eip、ebp、参数，并通过print_debuginfo打印调用函数的信息。然后将`ss:[ebp+4]`存着的返回地址赋给eip，将ss:ebp存着caller's ebp赋给ebp，便可以跳转到上一层调用的函数中去。
    > 输出如下，显示了函数调用栈的回溯信息，依次为栈中每层的ebp、eip、函数的（四个）参数、调用函数信息。

```
    ebp:0x00007b08 eip:0x001009a6 args0x00010094 0x00000000 0x00007b38 0x00100092 
        kern/debug/kdebug.c:306: print_stackframe+21
    ebp:0x00007b18 eip:0x00100c95 args0x00000000 0x00000000 0x00000000 0x00007b88 
        kern/debug/kmonitor.c:125: mon_backtrace+10
    ebp:0x00007b38 eip:0x00100092 args0x00000000 0x00007b60 0xffff0000 0x00007b64 
        kern/init/init.c:48: grade_backtrace2+33
    ebp:0x00007b58 eip:0x001000bb args0x00000000 0xffff0000 0x00007b84 0x00000029 
        kern/init/init.c:53: grade_backtrace1+38
    ebp:0x00007b78 eip:0x001000d9 args0x00000000 0x00100000 0xffff0000 0x0000001d 
        kern/init/init.c:58: grade_backtrace0+23
    ebp:0x00007b98 eip:0x001000fe args0x001032fc 0x001032e0 0x0000130a 0x00000000 
        kern/init/init.c:63: grade_backtrace+34
    ebp:0x00007bc8 eip:0x00100055 args0x00000000 0x00000000 0x00000000 0x00010094 
        kern/init/init.c:28: kern_init+84
    ebp:0x00007bf8 eip:0x00007d68 args0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
        <unknow>: -- 0x00007d67 --
```

    > 最后一行的含义：
    > 其对应的是栈中的第1个函数，即bootmain.c中的bootmain。由于最深一层是载入bootloader的位置，因此深入到该层时调用位置显示为unknown，`0x7d67`为bootloader调用kernel时eip的地址。在bootblock.asm存在下面代码：

```
    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    7d5c:   a1 18 00 01 00          mov    0x10018,%eax
    7d61:   25 ff ff ff 00          and    $0xffffff,%eax
    7d66:   ff d0                   call   *%eax
    asm volatile ("outb %0, %1" :: "a" (data), "d" (port));
```
    > 对比地址可知，在该处程序跳转进入了kernel。`0x7bf8`是程序在进入kernel之后显示函数栈的基准地址。`0x7d68`是程序跳转进入kernel时eip寄存器的值，表示原本应该执行的下一条指令的地址。

## 练习6：完善中断初始化和处理

1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

struct gatedesc定义如下，一共64bit，即中断向量表一个表项占用8字节。其中2-3字节是段描述符，通过段描述符在GDT中进行索引，找到基地址，再加上IDT中0-1字节和6-7字节拼成位移就可以得到中断处理代码的入口地址。

```
    /* Gate descriptors for interrupts and traps */
    struct gatedesc {
        unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
        unsigned gd_ss : 16;            // segment selector
        unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
        unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
        unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
        unsigned gd_s : 1;                // must be 0 (system)
        unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
        unsigned gd_p : 1;                // Present
        unsigned gd_off_31_16 : 16;        // high bits of offset in segment
    };
```

2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

    > 遍历所有可能的中断号，通过`SETGATE(gate, istrap, sel, off, dpl)`宏来完成idt表的建立。其中段选择址`sel`采用kernel代码段，即`GD_KTEXT`，段内偏移地址`off`从`__vectors`向量直接读取， `dpl`则根据特权级的不同来选择。

    > 最后通过`lidt`指令设置IDT的起始地址。


3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

    > 遇到时钟中断ticks加1，若ticks能被TICK_NUM整除则调用`print_ticks()`进行输出。

