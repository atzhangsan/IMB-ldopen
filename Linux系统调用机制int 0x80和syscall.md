Linux系统调用机制int 0x80、sysenter/sysexit、syscall/sysret的原理与代码分析
=========================================================================
```
近来学习Linux内核过程中，发现书籍和大部分的博文对x86架构系统调用的说明还是基于老旧的软中断“int 0x80”机制。实际上只有早期2.6内核及其以前的版本，才用软中断机制进行系统调用。因为软中断机制性能较差，所以在2.6后期版本以及3.x内核之后，已经完全使用快速系统调用指令。32位系统使用sysenter指令，64位系统使用syscall指令。所以，我们有必要了解当前广泛使用的Linux内核版本的系统调用实现原理，才能在实际工作中学以致用。

一、“talk is cheap, show me the code!”
先直观感受一下三种系统调用的差别，下面是用GNU C内联汇编<1>直接调用exit系统调用的例子[1]：

① 过时的int 0x80方式：“ 下载源码 >> asmsyscall_int80.c

int main(void)
{
  unsigned int syscall_nr = 1;
  int exit_status = 99;

  __asm__ __volatile__ (
  "movl %0, %%eax\n\t"  /* 调用号syscall_nr存入eax寄存器 */
  "movl %1, %%ebx\n\t"  /* 第一个参数返回码exit_status存入ebx寄存器 */
  "int $0x80\n\t"
  :
  :"m"(syscall_nr), "m"(exit_status)
  :"eax", "ebx"
  );  // gcc -o asmsyscall_int80 asmsyscall_int80.c
}
② x86_32<2>使用的sysenter方式：“ 下载源码 >>asmsyscall_sysenter.c

#include <stdlib.h>
#include <elf.h>

int main(int argc, char *argv[], char *envp[])
{
  unsigned int syscall_nr = 1;
  int exit_status = 99;
  Elf32_auxv_t *auxv;

  while(*envp++ != NULL);

  for (auxv = (Elf32_auxv_t *)envp; auxv->a_type != AT_NULL; auxv++)
  {
    if (auxv->a_type == AT_SYSINFO)
    {
      break;
    }
  }

  __asm__ __volatile__ (
  "movl %0, %%eax\n\t"  /* 调用号syscall_nr存入eax寄存器 */
  "movl %1, %%ebx\n\t"  /* 第一个参数返回码exit_status存入ebx寄存器 */
  "call *%2\n\t"
  :
  :"m"(syscall_nr), "m"(exit_status), "m"(auxv->a_un.a_val)
  :"eax", "ebx"
  );
}  // gcc -m32 -o asmsyscall_sysenter asmsyscall_sysenter.c
③ x86_64使用的syscall方式：“ 下载源码 >>asmsyscall_syscall.c

int main(void)
{
  unsigned long syscall_nr = 60;
  long exit_status = 99;

  __asm__ __volatile__ (
  "movq %0, %%rax\n\t"  /* 调用号syscall_nr存入rax寄存器 */
  "movq %1, %%rdi\n\t"  /* 第一个参数返回码exit_status存入rdi寄存器 */
  "syscall"
  :
  :"m"(syscall_nr), "m"(exit_status)
  :"rax", "rdi"
  );
}  // gcc -o asmsyscall_syscall asmsyscall_syscall.c
编译、执行后，立即查看程序的返回码“echo $?”，可以看到返回码为“99”，说明三种方式都成功调用了exit这个系统调用。

asmsyscall_syscall执行结果
下面重点分析一下实际环境中使用较多的x86_64 syscall系统调用的原理与代码实现。关于“int 0x80”和“sysenter/sysexit”机制的信息，请参看“Linux系统调用指南”[1]相关内容。

二、“终极”快速系统调用syscall/sysret
用户空间程序需要把系统调用号存入rax寄存器。syscall的参数存入其它通用寄存器中。
x86-64 ABI文档[4]第A.2.1章<4>写到：

    1. 用户程序使用%rdi, %rsi, %rdx, %rcx, %r8 和 %r9寄存器传递参数序列，内核接口使用%rdi, %rsi, %rdx, %r10, %r8 和 %r9寄存器。
    2. syscall指令完成系统调用，内核会破坏%rcx 和 %r11寄存器值。 
    3. 系统调用号用%rax寄存器传递。
    4. 系统调用最多6个参数，不能用直接用栈传递。
    5. syscall的返回结果保存在%rax寄存器中。-4095到-1的值表示错误，他是错误号。
    6. 只能传递整数或者内存值到内核。
前面的GNU C内联汇编，我们正是按调用规约执行了syscall指令，成功调用了exit系统调用。
英特尔开发人员手册[3]的1788页，说明了syscall指令的工作方式。
“
SYSCALL触发一个权限等级0的操作系统系统调用执行程序。它通过从IA32_LSTAR MSR加载RIP来实现（把SYSCALL后面指令的地址保存到RCX之后）。[1]
”
所以，IA32_LSTAR MSR<3>寄存器就保存着使用syscall进行系统调用的例程的入口地址。下面让我们从内核初始化入口开始，分析一下syscall系统调用的代码流程。Linux内核版本：linux-3.10.107。
1、init/main.c:472 start_kernel

      …… other code ……
      trap_init();
2、arch/x86/kernel/traps.c:758 trap_init

      …… other code ……
      cpu_init();
3、arch/x86/kernel/cpu/common.c:1224 cpu_init

      …… other code ……
      syscall_init();
4、arch/x86/kernel/cpu/common.c:1120 syscall_init

      …… other code ……
      wrmsrl(MSR_LSTAR, system_call);
MSR_LSTAR值在arch/x86/include/uapi/asm/msr-index.h中定义为0xc0000082。system_call在arch/x86/kernel/entry_64.S:613中定义。wrmsrl的作用就是把system_call例程的地址写入MSR_LSTAR寄存器中。
5、arch/x86/kernel/entry_64.S:613

ENTRY(system_call)
      …… other code ……
      call *sys_call_table(,%rax,8)  # XXX:  rip relative
和其它系统调用的方式一样，最终都会用保存在rax寄存器的调用号去系统调用表调用实际的处理例程。sys_call_table是在arch/x86/kernel/syscall_64.c定义的。
6、arch/x86/kernel/syscall_64.c:26

const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
 /*
  * Smells like a compiler bug -- it doesn't work
  * when the & below is removed.
  */
 [0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_64.h>
};
这边实际上使用了GNU C的数组初始化的扩展语法[5]。被包含的“asm/syscall_64.h”在是在编译的过程中，shell脚本依据arch/x86/syscalls/syscall_64.tbl生成的<4>。“syscall_64.h”文件内容如下：

__SYSCALL_COMMON(0, sys_read, sys_read)
        …… ……
__SYSCALL_COMMON(60, sys_exit, sys_exit)

        …… ……
__SYSCALL_X32(544, compat_sys_io_submit, compat_sys_io_submit)
可以看到我们示例中使用的exit系统调用的调用号就是60。“call *sys_call_table(,%rax,8)”指令将保存在rax的系统调用号60，乘以系统调用表元素的尺寸（表元素都是指针，占8字节），就得到exit系统调用的入口地址。
7、arch/x86/kernel/entry_64.S:675

      …… other code ……
      USERGS_SYSRET64
系统调用例程执行结束后，进入返回用户态的操作。“USERGS_SYSRET64”在arch/x86/include/asm/irqflags.h:133定义为

swapgs; \
sysretq;
调用了sysret指令，进行用户态上下文的恢复工作，然后返回用户态继续执行。

注解

<1> 对GNU C内联汇编不熟悉的话，可以快速浏览一下IBM developerWorks的“Linux 中 x86 的内联汇编”[2]
这篇博文。
<2> 在64位linux下编译运行32位程序，需要安装“glibc.i686”的包；“sudo yum install glibc.i686”。
<3> Model Specific Registers (MSRs)，特殊模块寄存器，是提供CPU的某些控制功能为目的的寄存器。你可以分别使用rdmsr和wrmsr来读写MSRs。
<4> 可以从编译路径下的目录arch/x86/include/generated找到“asm/syscall_64.h”文件。

参考

[1] Linux系统调用指南，packagecloud:blog著，levon译
[2] Linux 中 x86 的内联汇编，Bharata B. Rao
[3] 英特尔? 64 位和 IA-32 架构开发人员手册，Intel
[4] System V Application Binary Interface AMD64 Architecture Processor Supplement，Intel
[5] Linux Kernel代码艺术——数组初始化，郭海林

修订记录

2017-10-31 PM：完成初稿

版权声明：自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）

作者：叶雨珍
链接：https://www.jianshu.com/p/f4c04cf8e406
來源：简书
```
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
Linux Kernel代码艺术——数组初始化
==============================
```
前几天看内核中系统调用代码，在系统调用向量表初始化中，有下面这段代码写的让我有点摸不着头脑：

const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    /*
     * Smells like a compiler bug -- it doesn't work
     * when the & below is removed.
     */
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_32.h>
};
咱先不管上面代码的意思，先来回顾一下 C 语言中数组初始化的相关知识，然后再回头来理解上面这段代码。

数组初始化
C 语言中数组的初始化，可以在定义时就给出其初始值，以逗号隔开，用花括号括起来，例如：

int my_array[5] = {0, 1, 2, 3, 4};
当然你可以不用显示地去初始化所有的元素，例如，下面的代码就是显示初始化了数组的前三项，后面两项默认为0：

int my_array[5] = {0, 1, 2};
在 C89 标准中，要求按照数组中元素固定的顺序对数组的元素进行初始化；然而在 ISO C99 中，你可以以任意的顺序对数组元素初始化，只是需要给出数组元素所在的索引号；当然 GNU 编译器 GCC 对 C89 进行了扩展，也允许这么做。为了指明初始特殊的数组元素，需要在元素值前加上 [index] =，如：

int my_array[6] = { [4] = 29, [2] = 15 };
或者写成：
int my_array[6] = { [4] 29, [2] 15 };     //省略到索引与值之间的=，GCC 2.5 之后该用法已经过时了，但 GCC 仍然支持
两者均等价于：
int my_array[6] = {0, 0, 15, 0, 29, 0};
GNU 还有一个扩展：在需要将一个范围内的元素初始化为同一值时，可以使用 [first ... last] = value 这样的语法：

int my_array[100] = { [0 ... 9] = 1, [10 ... 98] = 2, 3 };
这是将my_array数组的第0~9个元素初始化为1， 第10~98个元素初始化为2， 第99个元素初始化为3（你也可以显示地写成[99] = 3)。** 注意 **：在语法中... 两边必须要留有空格符。

回到上面
对数组特定元素进行初始化我之前还真没遇到过，但也是 C 标准所支持的。内核中系统调用表是指根据系统调用号来找到系统调用的函数入口地址，结合上面数组初始化这个语法点，再回头看看上面系统调用表的定义：

const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_32.h>
};
先对表中所有 __NR_syscall_max+1 项初始化为指向 sys_ni_syscall 的函数，该函数只返回 -ENOSYS，表示该系统调用未实现。接下来包含一个头文件#include <asm/syscalls_32.h>，该文件是在编译时生成的，内容为：

__SYSCALL_I386(0, sys_restart_syscall, sys_restart_syscall)
__SYSCALL_I386(1, sys_exit, sys_exit)
__SYSCALL_I386(2, sys_fork, stub32_fork)
__SYSCALL_I386(3, sys_read, sys_read)
__SYSCALL_I386(4, sys_write, sys_write)
__SYSCALL_I386(5, sys_open, compat_sys_open)
...
__SYSCALL_I386 是一个宏定义：

#define __SYSCALL_I386(nr, sym, compat) [nr] = sym,
这样上面的系统调用表定义就展开为：

const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
    [0] = sys_restart_syscall,
    [1] = sys_exit,
    [2] = sys_fork,
    [3] = sys_read,
    //...
};
当用户进程发生系统调用，通过软中断 int 0x80 或者 sysenter 指令陷入到内核态，首先保存寄存器，然后检查系统调用号是否合法，最后跳转到相应的内核系统调用函数中执行：

ENTRY(system_call)
    pushl_cfi %eax          # 保存原始 eax
    SAVE_ALL                # 保存寄存器帧
    GET_THREAD_INFO(%ebp)
    testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp)    # 检查是否跟踪系统调用标志
    jnz syscall_trace_entry
    cmpl $(NR_syscalls), %eax    # 检查系统调用号是否合法
    jae syscall_badsys
syscall_call:
    call *sys_call_table(,%eax,4)   # 调用相应函数，等价于 call sys_call_table[%eax*4]
上面就是系统调用的进入过程，比较简单，这里只是说明了我们之前定义的系统调用表 sys_call_table 的用处。

再举一例
内核中还有其他地方应用到此种初始化数组的方法：

/* There are machines which are known to not boot with the GDT
   being 8-byte unaligned.  Intel recommends 16 byte alignment. */
static const u64 boot_gdt[] __attribute__((aligned(16))) = {
    /* CS: code, read/execute, 4 GB, base 0 */
    [GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
    /* DS: data, read/write, 4 GB, base 0 */
    [GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
    /* TSS: 32-bit tss, 104 bytes, base 4096 */
    /* We only have a TSS here to keep Intel VT happy;
       we don't actually use it for anything. */
    [GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
};
这是对系统启动时对全局符号表GDT的初始化。
```


