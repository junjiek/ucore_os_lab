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
            1. 生成一个有10000个块的文件，每个块默认512字节，用0填充
            2. 把bootblock中的内容写到第一个块
            3. 从第二个块开始写kernel中的内容

    > `cc`命令的参数含义如下：

        + `-I dir` 把`dir`添加到搜索头文件的目录集合
        + `-fno-builtin` 表示不识别没有以`__builtin_`开头的built-in函数
        + `-Wall` 显示所有的warning
        + `-ggdb` 当结合gdb使用时输出调试信息
        + `-gstabs` 使用`stabs`格式输出调试信息
        + `-o file` 输出到文件`file`
        + `-nostdinc` 不在标准的系统目录里搜索头文件
        + `-c file` 输入源文件`file`
    
    > `ld`命令的参数含义如下：

        + `-m env` 设置程序的模拟运行环境
        + `-nostdlib` 不使用标准函数库
        + `-N` 设置代码段和数据段可读、可写。
        + `-e entry` 把`entry`作为程序开始执行的显式标志。
        + `-o file` 指定输出到文件`file`

    > `dd`命令的参数含义如下：

        + `if=file` 指定输入文件
        + `of=file` 指定输出文件
        + `count=n` 从输入文件拷贝`n`个字节块（默认512字节）
        + `conv=notrunc` 不截断输出文件
        + `seek=n` 跳过输出文件的前`n`个字节快（默认512字节）

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


