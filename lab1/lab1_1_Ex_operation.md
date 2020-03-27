# 实验操作具体步骤

## 练习一

1.进入/moocos/ucore_lab/labcodes_answer/lab1_result目录。
2.执行make qemu命令。 
3.执行make clean命令清除上次make命令后产生的object文件(后缀为.o的文件)及可执行文件。  
4.执行cat Makefile命令查看Makefile。
```
  1 PROJ    := challenge
  2 EMPTY    :=
  3 SPACE    := $(EMPTY) $(EMPTY)
  4 SLASH    := /
  5 
  6 V       := @
  7 #need llvm/cang-3.5+
  8 #USELLVM := 1
  9 # try to infer the correct GCCPREFX
 10 ifndef GCCPREFIX
 11 GCCPREFIX := $(shell if i386-elf-objdump -i 2>&1 | grep '^elf32-i386$$' >/dev/null 2>&1; \
 12     then echo 'i386-elf-'; \
 13     elif objdump -i 2>&1 | grep 'elf32-i386' >/dev/null 2>&1; \
 14     then echo ''; \
 15     else echo "***" 1>&2; \
 16     echo "*** Error: Couldn't find an i386-elf version of GCC/binutils." 1>&2; \
 17     echo "*** Is the directory with i386-elf-gcc in your PATH?" 1>&2; \
 18     echo "*** If your i386-elf toolchain is installed with a command" 1>&2; \
 19     echo "*** prefix other than 'i386-elf-', set your GCCPREFIX" 1>&2; \
 20     echo "*** environment variable to that prefix and run 'make' again." 1>&2; \
 21     echo "*** To turn off this error, run 'gmake GCCPREFIX= ...'." 1>&2; \
 22     echo "***" 1>&2; exit 1; fi)
 23 endif
 24 
 25 # try to infer the correct QEMU
 26 ifndef QEMU
 27 QEMU := $(shell if which qemu-system-i386 > /dev/null; \
 28     then echo 'qemu-system-i386'; exit; \
 29     elif which i386-elf-qemu > /dev/null; \
 30     then echo 'i386-elf-qemu'; exit; \
 31     elif which qemu > /dev/null; \
 32     then echo 'qemu'; exit; \
 33     else \
 34     echo "***" 1>&2; \
 35     echo "*** Error: Couldn't find a working QEMU executable." 1>&2; \
 36     echo "*** Is the directory containing the qemu binary in your PATH" 1>&2; \
 37     echo "***" 1>&2; exit 1; fi)
 38 endif
 39 
 40 # eliminate default suffix rules
 41 .SUFFIXES: .c .S .h
 42 
 43 # delete target files if there is an error (or make is interrupted)
 44 .DELETE_ON_ERROR:
 45 
 46 # define compiler and flags
 47 ifndef  USELLVM
 48 HOSTCC        := gcc
 49 HOSTCFLAGS    := -g -Wall -O2
 50 CC        := $(GCCPREFIX)gcc
 51 CFLAGS    := -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
 52 CFLAGS    += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
 53 else
 54 HOSTCC        := clang
 55 HOSTCFLAGS    := -g -Wall -O2
 56 CC        := clang
 57 CFLAGS    := -march=i686 -fno-builtin -fno-PIC -Wall -g -m32 -nostdinc $(DEFS)
 58 CFLAGS    += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
 59 endif
 60 
 61 CTYPE    := c S
 62 
 63 LD      := $(GCCPREFIX)ld
 64 LDFLAGS    := -m $(shell $(LD) -V | grep elf_i386 2>/dev/null | head -n 1)
 65 LDFLAGS    += -nostdlib
 66 
 67 OBJCOPY := $(GCCPREFIX)objcopy
 68 OBJDUMP := $(GCCPREFIX)objdump
 69 
 70 COPY    := cp
 71 MKDIR   := mkdir -p
 72 MV        := mv
 73 RM        := rm -f
 74 AWK        := awk
 75 SED        := sed
 76 SH        := sh
 77 TR        := tr
 78 TOUCH    := touch -c
 79 
 80 OBJDIR    := obj
 81 BINDIR    := bin
 82 
 83 ALLOBJS    :=
 84 ALLDEPS    :=
 85 TARGETS    :=
 86 
 87 include tools/function.mk
 88 
 89 listf_cc = $(call listf,$(1),$(CTYPE))
 90 
 91 # for cc
 92 add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))
 93 create_target_cc = $(call create_target,$(1),$(2),$(3),$(CC),$(CFLAGS))
 94 
 95 # for hostcc
 96 add_files_host = $(call add_files,$(1),$(HOSTCC),$(HOSTCFLAGS),$(2),$(3))
 97 create_target_host = $(call create_target,$(1),$(2),$(3),$(HOSTCC),$(HOSTCFLAGS))
 98 
 99 cgtype = $(patsubst %.$(2),%.$(3),$(1))
100 objfile = $(call toobj,$(1))
101 asmfile = $(call cgtype,$(call toobj,$(1)),o,asm)
102 outfile = $(call cgtype,$(call toobj,$(1)),o,out)
103 symfile = $(call cgtype,$(call toobj,$(1)),o,sym)
104 
105 # for match pattern
106 match = $(shell echo $(2) | $(AWK) '{for(i=1;i<=NF;i++){if(match("$(1)","^"$$(i)"$$")){exit 1;}}}'; echo $$?)
107 
108 # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
109 # include kernel/user
110 
111 INCLUDE    += libs/
112 
113 CFLAGS    += $(addprefix -I,$(INCLUDE))
114 
115 LIBDIR    += libs
116 
117 $(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
118 
119 # -------------------------------------------------------------------
120 # kernel
121 
122 KINCLUDE    += kern/debug/ \
123                kern/driver/ \
124                kern/trap/ \
125                kern/mm/
126 
127 KSRCDIR        += kern/init \
128                kern/libs \
129                kern/debug \
130                kern/driver \
131                kern/trap \
132                kern/mm
133 
134 KCFLAGS        += $(addprefix -I,$(KINCLUDE))
135 
136 $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
137 
138 KOBJS    = $(call read_packet,kernel libs)
139 
140 # create kernel target
141 kernel = $(call totarget,kernel)
142 
143 $(kernel): tools/kernel.ld
144 
145 $(kernel): $(KOBJS)
146     @echo + ld $@
147     $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
148     @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
149     @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
150 
151 $(call create_target,kernel)
152 
153 # -------------------------------------------------------------------
154 
155 # create bootblock
156 bootfiles = $(call listf_cc,boot)
157 $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
158 
159 bootblock = $(call totarget,bootblock)
160 
161 $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
162     @echo + ld $@
163     $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
164     @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
165     @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
166     @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
167 
168 $(call create_target,bootblock)
169 
170 # -------------------------------------------------------------------
171 
172 # create 'sign' tools
173 $(call add_files_host,tools/sign.c,sign,sign)
174 $(call create_target_host,sign,sign)
175 
176 # -------------------------------------------------------------------
177 
178 # create ucore.img
179 UCOREIMG    := $(call totarget,ucore.img)
180 
181 $(UCOREIMG): $(kernel) $(bootblock)
182     $(V)dd if=/dev/zero of=$@ count=10000
183     $(V)dd if=$(bootblock) of=$@ conv=notrunc
184     $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
185 
186 $(call create_target,ucore.img)
187 
188 # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
189 
190 $(call finish_all)
191 
192 IGNORE_ALLDEPS    = clean \
193                   dist-clean \
194                   grade \
195                   touch \
196                   print-.+ \
197                   handin
198 
199 ifeq ($(call match,$(MAKECMDGOALS),$(IGNORE_ALLDEPS)),0)
200 -include $(ALLDEPS)
201 endif
202 
203 # files for grade script
204 
205 TARGETS: $(TARGETS)
206 
207 .DEFAULT_GOAL := TARGETS
208 
209 .PHONY: qemu qemu-nox debug debug-nox
210 qemu-mon: $(UCOREIMG)
211     $(V)$(QEMU)  -no-reboot -monitor stdio -hda $< -serial null
212 qemu: $(UCOREIMG)
213     $(V)$(QEMU) -no-reboot -parallel stdio -hda $< -serial null
214 log: $(UCOREIMG)
215     $(V)$(QEMU) -no-reboot -d int,cpu_reset  -D q.log -parallel stdio -hda $< -serial null
216 qemu-nox: $(UCOREIMG)
217     $(V)$(QEMU)   -no-reboot -serial mon:stdio -hda $< -nographic
218 TERMINAL        :=gnome-terminal
219 debug: $(UCOREIMG)
220     $(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
221     $(V)sleep 2
222     $(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
223     
224 debug-nox: $(UCOREIMG)
225     $(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
226     $(V)sleep 2
227     $(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
228 
229 .PHONY: grade touch
230 
231 GRADE_GDB_IN    := .gdb.in
232 GRADE_QEMU_OUT    := .qemu.out
233 HANDIN            := proj$(PROJ)-handin.tar.gz
234 
235 TOUCH_FILES        := kern/trap/trap.c
236 
237 MAKEOPTS        := --quiet --no-print-directory
238 
239 grade:
240     $(V)$(MAKE) $(MAKEOPTS) clean
241     $(V)$(SH) tools/grade.sh
242 
243 touch:
244     $(V)$(foreach f,$(TOUCH_FILES),$(TOUCH) $(f))
245 
246 print-%:
247     @echo $($(shell echo $(patsubst print-%,%,$@) | $(TR) [a-z] [A-Z]))
248 
249 .PHONY: clean dist-clean handin packall tags
250 clean:
251     $(V)$(RM) $(GRADE_GDB_IN) $(GRADE_QEMU_OUT) cscope* tags
252     -$(RM) -r $(OBJDIR) $(BINDIR)
253 
254 dist-clean: clean
255     -$(RM) $(HANDIN)
256 
257 handin: packall
258     @echo Please visit http://learn.tsinghua.edu.cn and upload $(HANDIN). Thanks!
259 
260 packall: clean
261     @$(RM) -f $(HANDIN)
262     @tar -czf $(HANDIN) `find . -type f -o -type d | grep -v '^\.*$$' | grep -vF '$(HANDIN)'`
263 
264 tags:
265     @echo TAGS ALL
266     $(V)rm -f cscope.files cscope.in.out cscope.out cscope.po.out tags
267     $(V)find . -type f -name "*.[chS]" >cscope.files
268     $(V)cscope -bq 
269     $(V)ctags -L cscope.files
```
5.执行make V=命令查看make执行的命令。  
```
moocos-> make V=
 2 + cc kern/init/init.c
 3 gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
 4 + cc kern/libs/readline.c
 5 gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/readline.c -o obj/kern/libs/readline.o
 6 + cc kern/libs/stdio.c
 7 gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/stdio.c -o obj/kern/libs/stdio.o
 8 + cc kern/debug/kdebug.c
 9 gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/kdebug.c -o obj/kern/debug/kdebug.o
10 + cc kern/debug/kmonitor.c
11 gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/kmonitor.c -o obj/kern/debug/kmonitor.o
12 + cc kern/debug/panic.c
13 gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/panic.c -o obj/kern/debug/panic.o
14 + cc kern/driver/clock.c
15 gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/clock.c -o obj/kern/driver/clock.o
16 + cc kern/driver/console.c
17 gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/console.c -o obj/kern/driver/console.o
18 + cc kern/driver/intr.c
19 gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/intr.c -o obj/kern/driver/intr.o
20 + cc kern/driver/picirq.c
21 gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/picirq.c -o obj/kern/driver/picirq.o
22 + cc kern/trap/trap.c
23 gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trap.c -o obj/kern/trap/trap.o
24 + cc kern/trap/trapentry.S
25 gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trapentry.S -o obj/kern/trap/trapentry.o
26 + cc kern/trap/vectors.S
27 gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/vectors.S -o obj/kern/trap/vectors.o
28 + cc kern/mm/pmm.c
29 gcc -Ikern/mm/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/mm/pmm.c -o obj/kern/mm/pmm.o
30 + cc libs/printfmt.c
31 gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/printfmt.c -o obj/libs/printfmt.o
32 + cc libs/string.c
33 gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/string.c -o obj/libs/string.o
34 + ld bin/kernel
35 ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
36 + cc boot/bootasm.S
37 gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
38 + cc boot/bootmain.c
39 gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
40 + cc tools/sign.c
41 gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
42 gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
43 + ld bin/bootblock
44 ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
45 'obj/bootblock.out' size: 488 bytes
46 build 512 bytes boot sector: 'bin/bootblock' success!
47 dd if=/dev/zero of=bin/ucore.img count=10000
48 10000+0 records in
49 10000+0 records out
50 5120000 bytes (5.1 MB) copied, 0.0601803 s, 85.1 MB/s
51 dd if=bin/bootblock of=bin/ucore.img conv=notrunc
52 1+0 records in
53 1+0 records out
54 512 bytes (512 B) copied, 0.000141238 s, 3.6 MB/s
55 dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
56 146+1 records in
57 146+1 records out
58 74923 bytes (75 kB) copied, 0.00356787 s, 21.0 MB/s
```

## 练习五

1.查看lab1/kern/debug/kdebug.c文件中print_stackframe函数注释。
2.按照注释提醒，填补代码:
```
uint32_t ebp = read_ebp(), eip = read_eip();  //用read_ebp和read_eip函数分别读取ebp和eip的值
int i, j;
for (i = 0; ebp != 0 && i < STACKFRAME_DEPTH && ebp !=0 ; i ++)  //#define STACKFRAME_DEPTH 20 并且检测ebp的值
{
    cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip); 
    uint32_t *args = (uint32_t *)ebp + 2; //参数的首地址
    for (j = 0; j < 4; j ++)
       cprintf("0x%08x ", args[j]); //打印4个参数
       cprintf("\n");
       
    print_debuginfo(eip - 1);  //打印函数信息
    eip = ((uint32_t *)ebp)[1]; //更新eip
    ebp = ((uint32_t *)ebp)[0]; //更新ebp
```
3.执行make qemu命令，并查看打印结果。  

