为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习： 

一、从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。  
答：  
CPU在加电之后，CPU里面的ROM存储器会将其里面保存的初始值传给各个寄存器，其中CS:IP = 0Xf000 : fff0(CS：代码段寄存器；IP：指令寄存器)，可以通过这个值继而得到从内存中读数据的位置(PC = 16*CS + IP=0Xffff0)。   
1.修改/moocos/uocre_lab/labcodes/lab1/tools目录下的gdbinit文件为(set architecture i8086 \n target remote :1234)。    
2.在lab1文件夹下make debug 并在gdb窗口用si命令单步追踪。  
3.在gdb窗口下，用(x /2i $pc)命令查看BIOS代码。  

二、在初始化位置0x7c00设置实地址断点,测试断点正常。  

三、从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。 

四、自己找一个bootloader或内核中的代码位置，设置断点并进行测试。  

