# v9-cpu

## 概述
v9-cpu是一个假想的简单CPU，用于操作系统教学实验和练习．

## 寄存器组:
总共有 9 个寄存器,其中 7 个为 32 位,2 个为 64 位(浮点寄存器)。本文档只针对 CPU
进行描述,对于相关的硬件配套外设(例如中断控制器等)在此不做介绍。其中：

 - a, b, c : 三个通用寄存器
 - f, g 两个浮点寄存器,是用来进行各种指令操作的。
 - sp 为当前栈指针
 - pc 为程序计数器（指向下一条指令），其指向的内存内容（即具体的指令值）会放到ir中，给CPU解码并执行
 - tsp 为栈顶指针(本 CPU 的栈是从顶部往底部增长)
 - flags 为状态寄存器(包括当前的运行模式,是否中断使能,是否有自陷,以及是否使用虚拟地址等)。
 
 

### flags寄存器标志位
 - user   : 1; 用户态或内核态(user mode or not.)
 - iena   : 1; 中断使能/失效 (interrupt flag.)
 - trap   : 1; 异常/错误编码 (fault code.)
 - paging : 1; 页模式使能/失效（ virtual memory enabled or not.）

 
## 指令集

总共有 209 条指令,具体的命令在指令的低 8 位,高 24 位为操作数 0。对于具体的命令,
其中 4 条既不需要操作数,也不需要当前的 CPU 信息;还有 92 条也不需要操作数,但需要当
前 CPU 的信息;剩下的 113 条命令,需要一个 24 位的操作数和当前的 CPU 信息。

整体来看,指令分为三大类:运算比较指令、流程控制指令、装载卸载指令。另外还有其
他的辅助指令:例如系统命令(例如 HALT,IDLE,RTI,BIN 等)、系统设置(例如 SSP,
USP,IVEC, PDIR 等)、扩展函数库(NET 类,MCP 类等)。


v9-cpu的指令集如下：

### system
- HALT, // halt system,
- ENT,// sp += operand0
- LEV, //  pc= *sp, sp + = operand0+8, 
- JMP, // jump to operand0
- JMPI, // jump to operand0 + (a * sizeof(union insnfmt_t))
- JSR,  // save current pc, *sp=pc, sp -= 8; jump to operand0 OR pc+=operand0. 
- JSRA, // save current pc, *sp=pc, sp -= 8; jump to a reg,  pc+=(a * sizeof(union insnfmt_t)).
- LEA, LEAG, // a = sp/pc + operand0
- CYC, // a = current cycle related with pc.
- MCPY, MCMP, MCHR, MSET, // memcpy/memcmp/memchr/memset(a, b, c)

### load to register a
a = *(uint/short/ushort/char/uchar/double/float *)addr

- LL, LLS, LLH, LLC, LLB, LLD, LLF, // a=content(local_addr) ; local addr= operand0 + sp
- LG, LGS, LGH, LGC, LGB, LGD, LGF, // a=content(global_addr) ; global_addr = operand0 + pc
- LX, LXS, LXH, LXC, LXB, LXD, LXF, // a=content(virt_addr); virt_addr = vir2phy(operand0)

### load register b
b = *(uint/short/ushort/char/uchar/double/float *)addr

- LBL, LBLS, LBLH, LBLC, LBLB, LBLD, LBLF, // b=content(local_addr) ; local addr= operand0 + sp
- LBG, LBGS, LBGH, LBGC, LBGB, LBGD, LBGF, // b=content(global_addr) ; global_addr = operand0 + pc
- LBX, LBXS, LBXH, LBXC, LBXB, LBXD, LBXF, // b=content(virt_addr); virt_addr = vir2phy(operand0)


### load a immediate
- LI, // a = operand0
- LHI, // a = (a << 24) | ((uint)operand0 >> 8)
- LIF, // f = operand0


### store register a to memory 
*(uint/short/ushort/char/uchar/double/float *)addr = a

- SL, SLH, SLB, SLD, SLF, // *(local_addr)=a; local_addr = operand0 + sp
- SG, SGH, SGB, SGD, SGF, // *(global_addr)=a; global_addr = operand + pc
- SX, SXH, SXB, SXD, SXF, // *(virt_addr)=a; virt_addr = vir2phy(operand0)

### arithmetic
- ADDF, SUBF, MULF, DIVF, // floating point arithmetic: f = fx(f, g)
- ADD, ADDI, ADDL, // a += b/operand0/(operand0 + sp)
- SUB, SUBI, SUBL, // a -= b/operand0/(operand0 + sp)
- MUL, MULI, MULL, // (int)a *= (int)b/(int)operand0/(int)(operand0 + sp)
- DIV, DIVI, DIVL, // (int)a /= (int)b/(int)operand0/(int)(operand0 + sp)
- DVU, DVUI, DVUL, // (uint)a /= (uint)b/(uint)operand0/(uint)(operand0 + sp)
- MOD, MODI, MODL, // (int)a %= (int)b/(int)operand0/(int)(operand0 + sp)
- MDU, MDUI, MDUL, // (uint)a %= (uint)b/(uint)operand0/(uint)(operand0 + sp)
- AND, ANDI, ANDL, // a &= b/operand0/(operand0 + sp)
- OR, ORI, ORL, // a |= b/operand0/(operand0 + sp)
- XOR, XORI, XORL, // a ^= b/operand0/(operand0 + sp)
- SHL, SHLI, SHLL, // a <<= b/operand0/(operand0 + sp)
- SHR, SHRI, SHRL, // (int)a >>= (int)b/(int)operand0/(int)(operand0 + sp)
- SRU, SRUI, SRUL, // (uint)a >>= (uint)b/(uint)operand0/(uint)(operand0 + sp)
- EQ, EQF, // (a = a == b)/(a = f == g)
- NE, NEF, // (a = a != b)/(a = f != g)
- LT, LTU, LTF,// (a = (int/uint)a < (int/uint)b)/(a = f < g)
- GE, GEU, GEF,// (a = (int/uint)a >= (int/uint)b)/(a = f >= g)


### conditional branch
- BZ, BZF, // branch to operand0 if a/f is zero
- BNZ, BNZF,  // branch to operand0 if a/f is non-zero
- BE, BEF, // branch to operand0 if (a == b)/(f == g)
- BNE, BNEF, // branch to operand0 if (a != b)/(f != g)
- BLT, BLTU, BLTF, // branch to operand0 if ((int/uint)a < (int/uint)b)/(f < g)
- BGE, BGEU, BGEF, // branch to operand0 if ((int/uint)a >= (int/uint)b)/f >= g)

### conversion
- CID, // f = (int)a
- CUD, // f = (uint)a
- CDI, // a = (int)f
- CDU, // a = (uint)f

### misc
- CLI, // a = iena, iena = 0 -- clear interrupt flag, return orig val.
- STI, // if generated by hardware: set trap, and process the interrupt;
          else: iena = 1 -- set interrupt flag
- RTI, // return from interrupt, set pc, sp, may switch user/kernel mode;
          if has pending interrupt, process the interrupt
- BIN, // a = kbchar -- kbchar is the value from outside io
- BOUT, // a = write(a, &b, 1);
- NOP, // no operation.
- SSP, // ksp = a -- ksp is kernel sp
- PSHA, // sp -= 8, *sp = a
- PSHI, // sp -= 8, *sp = operand0
- PSHF, // sp -= 8, *(double *)sp = f
- PSHB, // sp -= 8, *sp = b
- POPB, // b = *sp, sp += 8
- POPF,  // f = *(double *)sp, sp += 8
- POPA,  // a = *sp, sp += 8
- IVEC, // ivec = a -- set interrupt vector by a
- PDIR, // pdir = mem + a -- set page directory physical memory by a
- SPAG, // paging = a -- enable/disable virtual memory feature by a
- TIME, // if operand0 is 0: timeout = a -- set current timeout from a;
           else: printk("timer%d=%u timeout=%u", operand0, timer, timeout)
- LVAD, // a = vadr -- vadr is bad virtual address
- TRAP, // trap = FSYS
- LUSP, SUSP, // (a = usp)/(usp = a) -- usp is user stack pointer 
- LCL,  // c = *(uint *)(sp + operand0)
- LCA, // c = a
- PSHC, POPC, // (sp -= 8, *sp = c)/(c = *sp, sp += 8)
- MSIZ, // a = memsz -- move physical memory to a.
- PSHG, POPG, // (sp -= 8, *sp = g)/(g = *sp, sp += 8)

### math 
f = fx(f)/fx(f, g)

- POW, ATN2, FABS, ATAN, LOG, LOGT, EXP, FLOR, CEIL, HYPO, SIN, COS, TAN, ASIN,
ACOS, SINH, COSH, TANH, SQRT, FMOD,

### cpu idle
- IDLE // response hardware interrupt (include timer).

## 内存
缺省内存大小为128MB，可以通过启动参数"-m XXX"，设置为XXX MB大小．
在TLB中，设置了4个1MB大小页转换表（page translation buffer array）
 - kernel read page translation table
 - kernel write page translation table
 - user read page translation table
 - user write page translation table

有两个指针tr/tw, tw指向内核态或用户态的read/write　page translation table．
```
tr/tw[page number]=phy page number //页帧号
```
还有一个tpage buffer array, 保存了所有tr/tw中的虚页号，这些虚页号是tr/tw数组中的index 
```
tpage[tpages++] = v //v是page number
```

## IO操作
### 写外设（类似串口写）的步骤
 - 1 --> a
 - 一个字符'char' --> b
 - BOUT　　//如果在内核态，在终端上输出一个字符'char', 1-->a，如果在用户态，产生FPRIV异常

### 读外设（类似串口读）的步骤
　- BIN  //如果在内核态，kchar -->a  kchar是模拟器定期轮询获得的一个终端输入字符
 　　
如果iena(中断使能)，则在获得一个终端输入字符后，会产生FKEYBD中断 
 
### 设置timer的timeout
 - val --> a
 - TIME // 如果在内核态，设置timer的timeout为a; 如果在用户态，产生FPRIV异常
 


## 中断/异常
### 一些变量的含义：
 - ivec: 中断向量的地址
 
### 中断/异常类型
- FMEM,          // bad physical address 
- FTIMER,        // timer interrupt
- FKEYBD,        // keyboard interrupt
- FPRIV,         // privileged instruction
- FINST,         // illegal instruction
- FSYS,          // software trap
- FARITH,        // arithmetic trap
- FIPAGE,        // page fault on opcode fetch
- FWPAGE,        // page fault on write
- FRPAGE,        // page fault on read
- USER 　　　　      // user mode exception 

### 设置中断向量
 - val --> a
 - IVEC // 如果在内核态，设置中断向量的地址ivec为a; 如果在用户态，产生FPRIV异常

### 中断/异常产生的处理
 - 如果终端产生了键盘输入，且iean=1，则ipend |= FKEYBD，0-->iena
 - 如果timer产生了timeout，且iean=1，则ipend |= FTIMER，0-->iena
 - 如果产生了其他异常，则会有相应的处理，
 
 然后，保存中断的地址到kkernel mode的sp中，pc会跳到中断向量的地址ivec处执行
 
　
## CPU执行过程
### 一些变量的含义：
主要集中在em.c的cpu()函数中

 - a: a寄存器
 - b: b寄存器
 - c: c寄存器
 - f: f浮点寄存器
 - g: g浮点寄存器
 - ir:　指令寄存器
 - xpc: pc在host内存中的值
 - fpc: pc在host内存中所在页的下一页的起始地址值
 - tpc: pc在host内存中所在页的起始地址值
 - xsp: sp在host内存中的值
 - tsp: sp在host内存中所在页的起始地址值
 - fsp: 辅助判断是否要经过tr/tw的分析
 - ssp:
 - usp:
 - cycle: 
 - xcycle:
 - timer:
 - timeout:
 -　detla:
 
 ###执行过程概述
 
 1. 首先，读入os kernel文件到内存的底部，并把pc放置到os kernel文件指定的内存位置，
 设置可用sp为　MEM_SZ-FS_SZ=124MB
 1. 然后从os kernel文件的起始地址开始执行
 1. 如果碰到异常或中断，则保存中断的地址，并跳到中断向量的地址ivec处执行
