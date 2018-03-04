# lab0 SPOC思考题

##**提前准备**
(不在此赘述)
---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？

硬件设计上需要实现CPU、MMU、内存和硬盘寻址等支持，至少应该提供寻址、地址转换、寄存器读写等指令

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？

区别：寻址方式和范围不同、段大小实模式固定为64k而保护模式不固定、段地址的存放位置不同、实模式对段没有保护而保护模式有
物理地址：内存等存储设备上的实际的存储地址
线性地址：程序代码产生的逻辑地址加上相应的段的基地址得到的就是线性地址
逻辑地址：程序产生的与段相关的偏移地址部分，是相对于当前进程数据段的地址

- 理解list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
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
gd_off_15_0占最低的16位，后面gd_ss占16位，gd_args占5位，gd_rsv1占3位，gd_type占4位，gd_s占1位,gd_dp1占2位，gd_p占1位，gd_off_31_16占16位

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？
515

### 课堂实践练习

#### 练习一

请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。

  - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)

  - ##### [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)

  - ##### [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)

  - ##### [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)

  - ##### [[IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)]
  选择的汇编代码段
```
   switch_to:                      # switch_to(from, to)

    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
    movl %esp, 4(%eax)
    movl %ebx, 8(%eax)
    movl %ecx, 12(%eax)
    movl %edx, 16(%eax)
    movl %esi, 20(%eax)
    movl %edi, 24(%eax)
    movl %ebp, 28(%eax)

    # restore to's registers
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp
    movl 24(%eax), %edi
    movl 20(%eax), %esi
    movl 16(%eax), %edx
    movl 12(%eax), %ecx
    movl 8(%eax), %ebx
    movl 4(%eax), %esp

    pushl 0(%eax)               # push eip

    ret
```

含义：前半段将各个寄存器的值压入栈，再将目标进程的需要的各个寄存器的值存入相应的各个寄存器，从而实现进程切换

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore中宏定义的用途，并举例描述其含义。

> 利用宏进行复杂数据结构中的数据访问；
 
```
 #define \_\_vop_op(node, sym)                                                                         \\
    ({                                                                                              \\
        struct inode \*\_\_node = (node);                                                              \\
        assert(\_\_node != NULL && \_\_node->in\_ops != NULL && \_\_node->in\_ops->vop\_##sym != NULL);      \\
        inode_check(\_\_node, #sym);                                                                  \\
        \_\_node->in\_ops->vop\_##sym;                                                                  \\
     })
```
 使用这个宏可以实现对node的非空判断和调用其一个成员实现访问，而且访问的是一个函数指针，实现了函数调用
 
> 利用宏进行数据类型转换；如 to_struct, 
 
```
 #define to_struct(ptr, type, member)                               \\
   ((type \*)((char \*)(ptr) - offsetof(type, member)))
```
 从结构体的一个数据成员的地址和类型推知这个数据成员所在的结构体的地址
 
> 常用功能的代码片段优化；如  ROUNDDOWN, SetPageDirty
 
```
 #define ROUNDDOWN(a, n) ({                                          \\
            size_t \_\_a = (size_t)(a);                               \\
            (typeof(a))(\_\_a - \_\_a % (n));                           \\
        })
```
 将a减小到最靠近a的一个n的倍数
