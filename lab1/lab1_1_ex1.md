在此练习中，大家需要通过静态分析代码来了解： 

一、操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)  
答：0.ucore.img的生成依赖于bootlock和kernel文件，并把两个文件复制进ucore.img被分配的10000*512字节大小的空间下，bootlock占前512字节，kernel接在其后。bootlock目录下的bootsm.S和bootmain.c会进行编译和链接，kernel目录下的.c文件也会进行相应的编译，并链接成ucore.img.  
    1.先是用GCC把bootload的各种相关.c源代码编译成多个相关的.o文件(即目标文件，是机器可执行的指令)。   
    2.再用ld命令把gcc生成的目标文件链接生成可执行文件--bootlock.out。  
    3.再用dd命令把bootlock.out文件放到虚拟硬盘ucore.img中，等待硬盘模拟器调用。
    

二、一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？  
答：1.通过sign.c对bootblock进行了限制，大小为512字节，多余的空间填0，最后两个字节为0x55,0xAA。  




