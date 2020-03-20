练习一、理解通过make生成执行文件的过程  

在此练习中，大家需要通过静态分析代码来了解： 

一、操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)  
答：  
   1.用GCC把kern文件夹下的各种相关.c和.S文件编译，再用ld命令把gcc生成的目标文件链接生成bin/kernelt。   
   2.用GCC把boot文件夹下的各种相关.c和.S文件编译，并将tools文件夹下面的sign.c也进行编译，然后通过ld命令生成obj/bootblock.o。  
   3.ucore.img的生成依赖于bootlock和kernel的ELF文件，并把两个文件复制进ucore.img被分配的10000*512字节大小的空间下，bootlock占前512字节(bootlocsk是引导区)，kernel接在其后（kernel是os内核）。  

二、一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？  
答：  
   1.通过sign.c对bootblock进行了限制，大小为512字节，多余的空间填0，最后两个字节为0x55,0xAA。  




