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
