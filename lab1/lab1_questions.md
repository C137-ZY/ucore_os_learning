练习一：  

一、  
    在Terminal中要进入~/moocos/ucore_lab/labcodes_answer/lab1_result的时候，我先是用cd /home命令，再用ls命令行可以看到moocos,且ls -l 后moocos的权限也有，为什么cd ./moocos命令却无效呢？而未使用cd /home时直接用ls命令却也能看到moocos且cd ./ moocos有效？  

练习六：  

二、  
1.在trap.c补充中，首先是声明了一个vectors数组，而vector定义在vector.S中，用来借此指针来跳转到某一个中断处理地点。
```
.text
.globl __alltraps
.globl vector0
vector0:
pushl $0
pushl $0
jmp __alltraps

.data
.globl __vectors
__vectors:
  .long vector0
```
vectors.S在vectors数组中声明了256个.long数据vector,然后通过中断号进行跳转。跳转时会先把该中断信息（中断号）进行压栈，再调用统一的中断预处理函数__alltraps；该函数的作用很简单：保存好用户空间的上下文（一些寄存器变量），并切换到内核的上下文。  

2.setaget这个函数的作用是设置正确的interrupt/trap gate描述符。 
```
#define SETGATE(gate, istrap, sel, off, dpl) {            
    // gate(门描述符)，istrap(用是否为0区分interrupts和trap), sel(段选择，值为GD_KTEXT), off(偏移，中断处理函数的入口), dpl(特权级)
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;       //对gatedesc的8个字节中的部分进行赋值。
    (gate).gd_ss = (sel);                               
    (gate).gd_args = 0;                                    
    (gate).gd_rsv1 = 0;                                    
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    
    (gate).gd_s = 0;                                    
    (gate).gd_dpl = (dpl);                                
    (gate).gd_p = 1;                                    
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        
}
```
PS: 对T_SWITCH_TOK的发生时机是在用户空间的，所以对应的dpl需要修改为DPL_USER。  

3.lidt(&idt_pd)将idt的首地址和size装进idtr寄存器。  

三、  
