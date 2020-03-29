# Lab1 report

## 练习一

一、操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

1.
```
bin/ucore.img
| 生成ucore.img的相关代码为
|       $(UCOREIMG): $(kernel) $(bootblock)
|	$(V)dd if=/dev/zero of=$@ count=10000
|	$(V)dd if=$(bootblock) of=$@ conv=notrunc
|	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
|
| 为了生成ucore.img，首先需要生成bootblock、kernel的ELF文件
|
|>	bin/bootblock
|	| 生成bootblock的相关代码为
|	| $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
|	|	@echo + ld $@
|	|	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ \
|	|		-o $(call toobj,bootblock)
|	|	@$(OBJDUMP) -S $(call objfile,bootblock) > \
|	|		$(call asmfile,bootblock)
|	|	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) \
|	|		$(call outfile,bootblock)
|	|	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
|	|
|	| 为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign
|	|
|	|>	obj/boot/bootasm.o, obj/boot/bootmain.o
|	|	| 生成bootasm.o,bootmain.o的相关makefile代码为
|	|	| bootfiles = $(call listf_cc,boot) 
|	|	| $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
|	|	|	$(CFLAGS) -Os -nostdinc))
|	|	| 实际代码由宏批量生成
|	|	| 
|	|	| 生成bootasm.o需要bootasm.S
|	|	| 实际命令为
|	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
|	|	| 	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
|	|	| 	-c boot/bootasm.S -o obj/boot/bootasm.o
|	|	| 其中关键的参数为
|	|	| 	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
|	|	|	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
|	|	| 	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
|	|	| 	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
|	|	|	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
|	|	| 	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
|	|	| 	-I<dir>  添加搜索头文件的路径
|	|	| 
|	|	| 生成bootmain.o需要bootmain.c
|	|	| 实际命令为
|	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
|	|	| 	-fno-stack-protector -Ilibs/ -Os -nostdinc \
|	|	| 	-c boot/bootmain.c -o obj/boot/bootmain.o
|	|	| 新出现的关键参数有
|	|	| 	-fno-builtin  除非用__builtin_前缀，
|	|	|	              否则不进行builtin函数的优化
|	|
|	|>	bin/sign
|	|	| 生成sign工具的makefile代码为
|	|	| $(call add_files_host,tools/sign.c,sign,sign)
|	|	| $(call create_target_host,sign,sign)
|	|	| 
|	|	| 实际命令为
|	|	| gcc -Itools/ -g -Wall -O2 -c tools/sign.c \
|	|	| 	-o obj/sign/tools/sign.o
|	|	| gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
|	|
|	| 首先生成bootblock.o
|	| ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
|	|	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
|	| 其中关键的参数为
|	|	-m <emulation>  模拟为i386上的连接器
|	|	-nostdlib  不使用标准库
|	|	-N  设置代码段和数据段均可读写
|	|	-e <entry>  指定入口
|	|	-Ttext  制定代码段开始位置
|	|
|	| 拷贝二进制代码bootblock.o到bootblock.out
|	| objcopy -S -O binary obj/bootblock.o obj/bootblock.out
|	| 其中关键的参数为
|	|	-S  移除所有符号和重定位信息
|	|	-O <bfdname>  指定输出格式
|	|
|	| 使用sign工具处理bootblock.out，生成bootblock
|	| bin/sign obj/bootblock.out bin/bootblock
|
|>	bin/kernel
|	| 生成kernel的相关代码为
|	| $(kernel): tools/kernel.ld
|	| $(kernel): $(KOBJS)
|	| 	@echo + ld $@
|	| 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
|	| 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
|	| 	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
|	| 		/^$$/d' > $(call symfile,kernel)
|	| 
|	| 为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o
|	|	kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
|	|	trapentry.o vectors.o pmm.o  printfmt.o string.o
|	| kernel.ld已存在
|	|
|	|>	obj/kern/*/*.o 
|	|	| 生成这些.o文件的相关makefile代码为
|	|	| $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
|	|	|	$(KCFLAGS))
|	|	| 这些.o生成方式和参数均类似，仅举init.o为例，其余不赘述
|	|>	obj/kern/init/init.o
|	|	| 编译需要init.c
|	|	| 实际命令为
|	|	|	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
|	|	|		-gstabs -nostdinc  -fno-stack-protector \
|	|	|		-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
|	|	|		-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \
|	|	|		-o obj/kern/init/init.o
|	| 
|	| 生成kernel时，makefile的几条指令中有@前缀的都不必需
|	| 必需的命令只有
|	| ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
|	| 	obj/kern/init/init.o obj/kern/libs/readline.o \
|	| 	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
|	| 	obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
|	| 	obj/kern/driver/clock.o obj/kern/driver/console.o \
|	| 	obj/kern/driver/intr.o obj/kern/driver/picirq.o \
|	| 	obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
|	| 	obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
|	| 	obj/libs/printfmt.o obj/libs/string.o
|	| 其中新出现的关键参数为
|	|	-T <scriptfile>  让连接器使用指定的脚本
|
| 生成一个有10000个块的文件，每个块默认512字节，用0填充
| dd if=/dev/zero of=bin/ucore.img count=10000
|
| 把bootblock中的内容写到第一个块
| dd if=bin/bootblock of=bin/ucore.img conv=notrunc
|
| 从第二个块开始写kernel中的内容
| dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

二、 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

1.从tools/sign.c的代码来看。  
```
char buf[512];
memset(buf, 0, sizeof(buf));   // 把buf初始化为全零
FILE *ifp = fopen(argv[1], "rb");
int size = fread(buf, 1, st.st_size, ifp);
if (size != st.st_size) {      // 检查buf的大小是否为512字节
     fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
     return -1;
}
fclose(ifp);
buf[510] = 0x55;   // 把第510个字节赋值为0x55
buf[511] = 0xAA;   // 把第511个字节赋值为0xAA
```
一个磁盘主引导扇区的特征是大小只有512字节，多余空间填零，且第510个字节是0x55，第511个字节是0xAA。



## 练习二

一、从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

1.修改 lab1/tools/gdbinit,内容为:
```
set architecture i8086  // 设置当前调试的CPU是8086
target remote :1234     // gdb接连qemu虚拟机
```

2.在lab1目录下，执行
```
make debug
```

3.在看到gdb的调试界面(gdb)后，在gdb调试界面下执行如下命令
```
si 
```
即可单步跟踪BIOS了。

4.在gdb界面下，可通过如下命令来看BIOS的代码
```
 x /2i $pc  // 显示当前eip处的汇编指令
```

> 拓展

```
改写Makefile文件
	debug: $(UCOREIMG)
	       $(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
	       $(V)sleep 2
	       $(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中。
为防止qemu在gdb连接后立即开始执行，删除了`tools/gdbinit`中的`continue`行。

二、在初始化位置0x7c00 设置实地址断点,测试断点正常。

1.在tools/gdbinit结尾加上

```
    set architecture i8086  // 设置当前调试的CPU是8086
    b *0x7c00  // 在0x7c00处设置断点。此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处
    c          // continue简称，表示继续执行
    x /5i $pc  // 显示当前eip处的汇编指令
    set architecture i386  // 设置当前调试的CPU是80386
```
	
2.在lab1目录下，运行make debug命令便可得到，gdb打印出 ：
```
	Breakpoint 2, 0x00007c00 in ?? ()
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
```

三、在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。

1.在tools/gdbinit结尾加上:
```
	b *0x7c00
	c
	x /10i $pc
```

2.使用si和 x/i $pc 指令一行一行的跟踪，将得到的反汇编代码为:
```
0x00007c01 in ?? ()
(gdb) x/i $pc
=> 0x7c01:      cld    
(gdb) si
0x00007c02 in ?? ()
(gdb) x/i $pc
=> 0x7c02:      xor    %eax,%eax
(gdb) si
0x00007c04 in ?? ()
(gdb) x/i $pc
=> 0x7c04:      mov    %eax,%ds
(gdb) 
```

bootasm.S的部分代码：
```
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment

```

bootblock.asm中的部分代码：
```
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    7c00:   fa                      cli    
    cld                                             # String operations increment
    7c01:   fc                      cld    

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    7c02:   31 c0                   xor    %eax,%eax
    movw %ax, %ds                                   # -> Data Segment
    7c04:   8e d8                   mov    %eax,%ds
    movw %ax, %es                                   # -> Extra Segment
    7c06:   8e c0                   mov    %eax,%es
    movw %ax, %ss                                   # -> Stack Segment
    7c08:   8e d0                   mov    %eax,%ss

```
3.对比发现，反汇编指令与两个汇编文件中相对应部分代码相同。  

## 练习三  

一、分析bootloader 进入保护模式的过程。  
（提示：需要阅读小节“保护模式和分段机制”和lab1/boot/bootasm.S源码，了解如何从实模式切换到保护模式，需要了解：  
为何开启A20，以及如何开启A20  
如何初始化GDT表  
如何使能和进入保护模式）  

1.首先清理环境：关闭中断，将各个段寄存器重置   
修改控制方向标志寄存器DF=0，使得内存地址从低到高增加，并先将各个寄存器置0
```
.code16                            // CPU启动为16位模式                  
	cli                        // 关中断
	cld                        // 清方向标志位
	xorw %ax, %ax              // 置零
	movw %ax, %ds              // -> 数据段寄存器
	movw %ax, %es              // -> 附加段寄存器
	movw %ax, %ss              // -> 堆栈段寄存器
```

2.开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，可以访问4G的内存空间。
下面的代码打开A20地址线 
``` 
seta20.1:                   // 等待8042键盘控制器不忙
	inb $0x64, %al      // 从0x64端口读入一个字节的数据到al中  
	testb $0x2, %al     // test指令对al的第2位进行位测试(test指令可作and指令进行，只不过它不会影响操作数）
	jnz seta20.1        // 如果上面的测试中发现al的第2位为0，就不执行该指令；否则就循环检查。  
	
	movb $0xd1, %al     // 将0xd1写入到al中 
	outb %al, $0x64     // 将al中的数据写入到端口0x64中  
	
seta20.2:                   
        inb $0x64, %al      
        testb $0x2, %al
        jnz seta20.2

        movb $0xdf, %al     // 将0xdf写入到al中(即将A20置1)
        outb %al, $0x60     // 将al中的数据写入到端口0x60中 
```

3.初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可:
```
	lgdt gdtdesc     // 将全局描述符表描述符加载到全局描述符表寄存器  
```

4.进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式。
```
	movl %cr0, %eax           // 控制寄存器cr0中的数值写入到eax中。
	orl $CR0_PE_ON, %eax      // 用或运算使得PE位置1
	movl %eax, %cr0           // eax中修改过的数据写入cr0。
```

5.通过长跳转更新cs的基地址
```
	ljmp $PROT_MODE_CSEG, $protcseg
	.code32                 // 长跳转到32位代码段 
	protcseg:               // 初始化保护模式的数据段寄存器      
```
设置段寄存器，并建立堆栈
```
	movw $PROT_MODE_DSEG, %ax     // Our data segment selector
	movw %ax, %ds                 // -> DS: Data Segment
	movw %ax, %es                 // -> ES: Extra Segment
	movw %ax, %fs                 // -> FS
	movw %ax, %gs                 // -> GS
	movw %ax, %ss                 // -> SS: Stack Segment
	movl $0x0, %ebp               // 初始化栈底指针
	movl $start, %esp             // 初始化栈顶指针
```

6.转到保护模式完成后，进入boot主函数   
```
	call bootmain                // 调用bootmain函数。
``` 


## 练习四  

一、bootloader如何读取硬盘扇区的？

1.首先看bootmain.h文件中的readsect函数，`readsect`从设备的第secno扇区读取数据到dst位置
```
	static void
	readsect(void *dst, uint32_t secno) {
	    waitdisk();                             // 首先等待磁盘就绪
	
	    outb(0x1F2, 1);                         // 设置需要读取得参数，即扇区个数count=1
	    outb(0x1F3, secno & 0xFF);              // 即读取相应的内容到寄存器里面.
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	    // 上面四条指令联合制定了扇区号
	    // 在这4个字节线联合构成的32位参数中
	    // 29-31位强制设为1
	    // 28位(=0)表示访问"Disk 0"
	    // 0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 发出读取磁盘命令 cmd 0x20 - read sectors
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 把磁盘扇区数据读到指定内存dest中
	}
```

2.readseg函数简单包装了readsect函数，可以从设备读取任意长度的内容。
```
	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
	    uintptr_t end_va = va + count;    // 设置结束地址
	
	    va -= offset % SECTSIZE;          // 设置块首地址
	
	    uint32_t secno = (offset / SECTSIZE) + 1;   // 设置需要读取的磁盘的位置
	    // 加1因为0扇区被引导占用
	    // ELF文件从1扇区开始
	
	    for (; va < end_va; va += SECTSIZE, secno ++) {
	    // 继续对虚存va和secno进行自加操作，直到读完所需读的东西为止
	        readsect((void *)va, secno);   // 磁盘中读取一个整块 存到相应的虚存va中
	    }
	}
```

二、分析bootloader是如何加载ELF格式的OS？

1.在bootmain函数中，
```
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);  // 调用readseg函数从ELFHDR处读取8个扇区的大小。
	
	    // 通过储存在头部的e_e_magic判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;  // 加载到错误得操作系统, 则跳转到bad
	    }
	
	    struct proghdr *ph, *eph;	
	    // 在elf.h中有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph中
	    
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);   // ph表示ELF段表首地址 
	    eph = ph + ELFHDR->e_phnum;    // eph表示ELF段表末地址
	
	    // 按照描述表将ELF文件中数据载入相应的虚存p_va程序块中
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口并运行
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```

三、总结  

1.从硬盘中将bin/kernel文件的第一页内容加载到内存地址为0x10000的位置，并把这里强制转换成ELFHDR使用。  
2.校验e_magic字段，确保这是一个ELF文件。  
3.读取ELF Header的e_phoff字段，得到Program Header表的起始地址；读取ELF Header的e_phnum字段，得到Program Header表的元素数目。  
4.遍历Program Header表中的每个元素，并根据偏移量分别把程序段的数据读取到内存中。  
5.读取完毕，通过ELF Header的e_entry得到内核的入口地址，并跳转到该地址开始执行内核代码。  

## 练习五  

一、实现函数调用堆栈跟踪函数 

ss:ebp指向的堆栈位置储存着caller的ebp，以此为线索可以得到所有使用堆栈的函数ebp。
ss:ebp+4指向caller调用时的eip，ss:ebp+8等是（可能的）参数。

1.输出中，堆栈最深一层为
```
 ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d6d --
```
其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。
bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。call指令压栈，所以bootmain中ebp为0x7bf8。

PS；  
ebp:0x0007bf8 这意味着kern_init函数的调用者（也就是bootmain函数）没有传递任何输入参数给它。  
eip:0x00007d6e eip的值是kern_init函数的返回地址，也就是bootmain函数调用kern_init对应的指令的下一条指令的地址。
args存放的就是boot loader指令的前16个字节。      

## 练习六

完善中断初始化和处理

一、中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

1.打开lab1/kern/mm/mmu.h文件，找到中断描述符表数据结构gatedesc。
```
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // 低16位是段偏移
    unsigned gd_ss : 16;              // 段选择子
    unsigned gd_args : 5;             // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;             // 保留位
    unsigned gd_type : 4;             // 类型(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // 必须为零 (system)
    unsigned gd_dpl : 2;              // 描述符(meaning new) privilege level优先级？
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;       // 高16位是段偏移
};
```
中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移。  
通过段选择子去GDT中找到对应的基地址，然后基地址加上偏移量就是中断处理程序的地址。  

二、请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。

1.补充代码如下：
```
    extern uintptr_t __vectors[];      // 声明__vertors[]，保存在vectors.S中的256个中断处理例程的入口地址数组。
    int i;
    // 使用SETGATE宏，对IDT中的每一个gatedesc进行初始化
    for(i = 0 ; i < sizeof(idt) / sizeof(struct gatedesc) ; i++ )  // IDT表项的个数
    {
        SETGATE(idt[i],0,GD_KTEXT,__vectors[i],DPL_KERNEL);        // 初始化idt数组
    }
    SETGATE(idt[T_SWITCH_TOK],0,GD_KTEXT,__vectors[T_SWITCH_TOK],DPL_USER);        //在这里先把所有的中断都初始化为内核级的中断
    lidt(&idt_pd);      // 使用lidt指令加载中断描述符表 
```

2.执行make debug指令验证。  

三、请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数。  

1.补充代码如下：
```
    case IRQ_OFFSET + IRQ_TIMER:
        //“volatile size_t ticks" from <clok.h>
        if (++ticks == TICK_NUM) {     // define TICK_NUM = 100
            ticks= 0;
            print_ticks();            //用于info的函数
        }
        break;
```

2. 执行make debug指令验证。  

## 练习七(challenge)

一、增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务。  

1.在kern/init/init.c中用lab1_switch_test函数对用户态和内核态的状态进行测试和info，并用lab1_switch_to_user() 和lab1_switch_to_kernel()这两个函数中的内嵌汇编产生中断信号。
```
static void
lab1_switch_to_user(void) {
  //LAB1 CHALLENGE 1 : TODO
  cprintf("1");
  asm volatile (                              // 内嵌汇编
      "sub $0x8, %%esp \n"                    
      "int %0 \n"                             // 产生中断
      "movl %%ebp, %%esp"                     // 对错误码和中断号进行压栈         
      :
      : "i"(T_SWITCH_TOU)
  );
  cprintf("1");
}
```
```
static void
lab1_switch_to_kernel(void) {
  //LAB1 CHALLENGE 1 :  TODO
  asm volatile (
      "int %0 \n"
      "movl %%ebp, %%esp" 
      :
      : "i"(T_SWITCH_TOK)
  );
}
```

2.在kern/trap/trapentry.S中__alltraps会保存相应的寄存器上下文，然后切换到内核上下文，最后传递trapframe数据*tf给kern/trap/trap.c中的trap函数进行中断处理。

3.trap函数传递trapframe数据tf给trap_dispatch函数，运行其中的case T_SWITCH_TOu和 case T_SWITCH_TOk。
```
struct trapframe switchk2u, *switchu2k;    //  先定义两个trapframe声明用于临时trapframe和trapframe指针。


case T_SWITCH_TOU:                    // T_SWITCH_TOU：中断信号的意义由内核空间跳转到用户空间。
if(tf->tf_cs != USER_CS){             // 判定tf的cs类别是不是用户cs
switchk2u = *tf;                      // 获取此时的trapframe
switchk2u.tf_cs = USER_CS;               
switchk2u.tf_ds = switchk2u.tf_es = switchk2u.tf_ss = USER_DS;       // 更新cs、ds、es、ss
switchk2u.tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;      // esp的首地址在trapframe末尾偏移8字节
switchk2u.tf_eflags |= FL_IOPL_MASK;                                 // 更新eflags（特权级）
*((uint32_t *)tf - 1) = (uint32_t)&switchk2u;                        // 用((uint32_t *)tf - 1)指针存放堆栈中的返回地址
}
break;
case T_SWITCH_TOK:
if (tf->tf_cs != KERNEL_CS) {
tf->tf_cs = KERNEL_CS;
tf->tf_ds = tf->tf_es = KERNEL_DS;
tf->tf_eflags &= ~FL_IOPL_MASK;                                      // 一个是|=，一个是&=。
switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));  // switchu2k指向esp，esp
memmove(switchu2k, tf, sizeof(struct trapframe) - 8);                             
// 用memmove函数将tf中trapframe除最后8个字节的内容copy给switchu2k
*((uint32_t *)tf - 1) = (uint32_t)switchu2k;       // 用((uint32_t *)tf - 1)指针存放堆栈中的返回地址   
}
break;
```
二、用键盘实现用户模式内核模式切换。具体目标是:“键盘输入3时切换到用户模式,键盘输入0时切换到内核模式”。基本思路是借鉴软中断(syscall功能)的代码,并且把trap.c中软中断处理的设置语句拿过来。  

1.在idt_init中，将用户态调用SWITCH_TOK中断的权限打开。  
```
	SETGATE(idt[T_SWITCH_TOK], 1, KERNEL_CS, __vectors[T_SWITCH_TOK], 3);
```

2.在trap_dispatch中，将iret时会从堆栈弹出的段寄存器进行修改。  
TO User
```
	    tf->tf_cs = USER_CS;
	    tf->tf_ds = USER_DS;
	    tf->tf_es = USER_DS;
	    tf->tf_ss = USER_DS;
```
TO Kernel
```
	    tf->tf_cs = KERNEL_CS;
	    tf->tf_ds = KERNEL_DS;
	    tf->tf_es = KERNEL_DS;
```

在lab1_switch_to_user中，调用T_SWITCH_TOU中断。
注意从中断返回时，会多pop两位，并用这两位的值更新ss,sp，损坏堆栈。
所以要先把栈压两位，并在从中断返回后修复esp。
```
	asm volatile (
	    "sub $0x8, %%esp \n"
	    "int %0 \n"
	    "movl %%ebp, %%esp"
	    : 
	    : "i"(T_SWITCH_TOU)
	);
```

在lab1_switch_to_kernel中，调用T_SWITCH_TOK中断。
注意从中断返回时，esp仍在TSS指示的堆栈中。所以要在从中断返回后修复esp。
```
	asm volatile (
	    "int %0 \n"
	    "movl %%ebp, %%esp \n"
	    : 
	    : "i"(T_SWITCH_TOK)
	);
```

但这样不能正常输出文本。根据提示，在trap_dispatch中转User态时，将调用io所需权限降低。
```
	tf->tf_eflags |= 0x3000;
```

