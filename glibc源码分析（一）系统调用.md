glibc源码分析（一）系统调用
=========================
```
1.1   什么是glibc
glibc是GNU发布的libc库，即c运行库。glibc是linux系统中最底层的api，几乎其它任何运行库都会依赖于glibc。glibc除了封装linux操作系统所提供的系统服务外，它本身也提供了许多其它一些必要功能服务的实现。由于 glibc 囊括了几乎所有的 UNIX 通行的标准，可以想见其内容包罗万象。而就像其他的 UNIX 系统一样，其内含的档案群分散于系统的树状目录结构中，像一个支架一般撑起整个操作系统。在 GNU/Linux 系统中，其C函式库发展史点出了GNU/Linux 演进的几个重要里程碑，用 glibc 作为系统的C函式库，是GNU/Linux演进的一个重要里程碑。

glibc支持不同的体系结构，不同的体系结构之上又支持不同的操作系统。

支持的体系结构：alpha,arm,i386,ia64,powerpc等
支持的操作系统：bsd,linux等
本文及以后的一系列文章将对glibc源码进行一系列的分析，这些分析都是基于i386体系结构linux操作系统。glibc版本号为glibc-2.26。

1.2   什么是系统调用
1.2.1  概要

顾名思义，系统调用（system call）是指操作系统提供给程序调用的接口。

操作系统的主要功能是为管理硬件资源和为应用程序开发人员提供良好的环境来使应用程序具有更好的兼容性，为了达到这个目的，内核提供一系列具备预定功能的多内核函数，通过一组称为系统调用（system call)的接口呈现给用户。系统调用把应用程序的请求传给内核，调用相应的的内核函数完成所需的处理，将处理结果返回给应用程序。






作为开发人员，我们调用系统调用来实现系统功能。

有过linux下开发经验的人一定对glibc中的open,read,write,close,stat,mkdir等函数有所了解。这些函数其实都是是系统调用，准确的讲是系统调用的封装函数。glibc将诸多系统调用都封装成函数，使我们可以以函数的方式，方便的调用系统调用。本文及后续章节将详细讲解glibc对系统调用封装的过程。

1.2.2  实例

#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc,char **argv)
{

        struct stat buf;

        stat("/initrd.img",&buf);

        printf("size = %ld\n",buf.st_size);

        return 0;
}
1.3  系统调用的封装
系统调用的封装按照固定的规则进行。寄存器EAX传递系统调用号。系统调用号用来确定系统调用。寄存器EBX,ECX,EDX,ESI,EDI,EBP依次传递系统调用参数。参数个数决定设置寄存器的个数。int0x80指令切入内核执行系统调用。系统调用执行完成后返回。寄存器EAX保存系统调用的返回值。

glibc使用了多种不同的方式封装系统调用。但是，万变不离其宗，它们的封装过程一定是按照上面的规则进行的。

1.4  glibc封装系统调用
glibc实现了许多系统调用的封装。它们的封装方式大致可以分为两种：一  脚本生成汇编文件，汇编文件中汇编代码封装了系统调用。这种方式，简称脚本封装。二 .c文件中调用嵌入式汇编代码封装系统调用。一般使用.c文件封装系统调用，代码中除了嵌入式汇编封装代码外，还有一些C代码做其他处理。这种方式，简称.c封装。

1.5 脚本封装
1.5.1  概要

glibc中大多数系统调用都是使用脚本封装的方式封装的。

脚本封装的规则很简单。三种文件生成封装代码。一 make-syscall.sh文件  二 syscall-template.S文件 三 syscalls.list文件。

make-syscall.sh是shell脚本文件。它读取syscalls.list文件的内容，对文件的每一行进行解析。根据每一行的内容生成一个.S汇编文件，汇编文件封装了一个系统调用。

syscall-template.S是系统调用封装代码的模板文件。生成的.S汇编文件都调用它。

syscalls.list是数据文件，它的内容如下：

# File name Caller  Syscall name    Args    Strong name Weak names

accept      -   accept      Ci:iBN  __libc_accept   accept
access      -   access      i:si    __access    access
acct        -   acct        i:S acct
adjtime     -   adjtime     i:pp    __adjtime   adjtime
bind        -   bind        i:ipi   __bind      bind
chdir       -   chdir       i:s __chdir     chdir
......
它由许多行组成，每一行可分为6列。File name列指定生成的汇编文件的文件名。Caller指定调用者。Syscall name列指定系统调用的名称，系统调用名称可以转换为系统调用号以标示系统调用。Args列指定系统调用参数类型，个数及返回值类型。Strong name指定系统调用封装函数的函数名。Weak names列指定封装函数的别称，用户可以调用别称来调用封装函数。

make-syscall.sh分析syscalls.list每一行每一列的内容，生成汇编文件。以分析chdir行为例，生成的汇编文件内容为：

#define SYSCALL_NAME chdir
#define SYSCALL_NARGS 1
#define SYSCALL_SYMBOL __chdir
#define SYSCALL_CANCELLABLE 0
#define SYSCALL_NOERRNO 0
#define SYSCALL_ERRVAL 0
#include <syscall-template.S>
weak_alias (__chdir, chdir)
hidden_weak (chdir)
SYSCALL_NAME宏定义了系统调用的名字。是从Syscall name列获取。 

SYSCALL_NARGS宏定义了系统调用参数的个数。是通过解析Args列获取。 

SYSCALL_SYMBOL宏定义了系统调用的函数名称。是从Strong name列获取。 

SYSCALL_CANCELLABLE宏在生成的所有汇编文件中都定义为0。 

SYSCALL_NOERRNO宏定义为1，则封装代码没有出错返回。用于getpid这些没有出错返回的系统调用。是通过解析Args列设置。 

SYSCALL_ERRVAL宏定义为1，则封装代码直接返回错误号，不是返回-1并将错误号放入errno中。生成的所有.S文件中它都定义为0。

weak_alias (__chdir, chdir)定义了__chdir函数的别称，我们可以调用chdir来调用__chdir。 chdir从Weak names列获取。

汇编文件中引用了模板文件syscall-template.S，所有的封装代码都集中在syscall-template.S文件中。

3种文件，make-syscall.sh文件在sysdeps/unix/make-syscall.sh。syscall-template.S文件在sysdeps/unix/syscall-template.S。syscalls.list文件则有多个，分别在sysdeps/unix/syscalls.list，sysdeps/unix/sysv/linux/syscalls.list，sysdeps/unix/sysv/linux/generic/syscalls.list，sysdeps/unix/sysv/linux/i386/syscalls.list。

1.5.2  syscall-template.S

syscall-template.S作为模板文件，包含了所有封装代码。

#if SYSCALL_CANCELLABLE
# include <sysdep-cancel.h>
#else
# include <sysdep.h>
#endif

#define syscall_hidden_def(SYMBOL)      hidden_def (SYMBOL)

#define T_PSEUDO(SYMBOL, NAME, N)       PSEUDO (SYMBOL, NAME, N)
#define T_PSEUDO_NOERRNO(SYMBOL, NAME, N)   PSEUDO_NOERRNO (SYMBOL, NAME, N)
#define T_PSEUDO_ERRVAL(SYMBOL, NAME, N)    PSEUDO_ERRVAL (SYMBOL, NAME, N)
#define T_PSEUDO_END(SYMBOL)            PSEUDO_END (SYMBOL)
#define T_PSEUDO_END_NOERRNO(SYMBOL)        PSEUDO_END_NOERRNO (SYMBOL)
#define T_PSEUDO_END_ERRVAL(SYMBOL)     PSEUDO_END_ERRVAL (SYMBOL)

#if SYSCALL_NOERRNO

T_PSEUDO_NOERRNO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret_NOERRNO
T_PSEUDO_END_NOERRNO (SYSCALL_SYMBOL)

#elif SYSCALL_ERRVAL

T_PSEUDO_ERRVAL (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret_ERRVAL
T_PSEUDO_END_ERRVAL (SYSCALL_SYMBOL)

#else

T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret
T_PSEUDO_END (SYSCALL_SYMBOL)

#endif

syscall_hidden_def (SYSCALL_SYMBOL)
文件开头引入.h文件。如果SYSCALL_CANCELLABLE宏定义为1，则引入<sysdep-cancel.h>文件，否则引入<sysdep.h>文件。SYSCALL_CANCELLABLE宏在所有生成的汇编文件中都定义为0，所以汇编文件都是引用<sysdep.h>文件。<sysdep.h>文件位于sysdeps/unix/sysv/linux/i386/sysdep.h

#if SYSCALL_CANCELLABLE
# include <sysdep-cancel.h>
#else
# include <sysdep.h>
#endif
系统调用的封装代码由3种形式。

如果系统调用没有错误返回，则执行

T_PSEUDO_NOERRNO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret_NOERRNO
T_PSEUDO_END_NOERRNO (SYSCALL_SYMBOL)
如果系统调用有错误返回且直接返回错误，则执行

T_PSEUDO_ERRVAL (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret_ERRVAL
T_PSEUDO_END_ERRVAL (SYSCALL_SYMBOL)
如果系统调用有错误返回且返回-1，errno设置错误号，则执行

T_PSEUDO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret
T_PSEUDO_END (SYSCALL_SYMBOL)
1.5.3  T_PSEUDO_NOERRNO

在系统调用没有出错返回时，执行

T_PSEUDO_NOERRNO (SYSCALL_SYMBOL, SYSCALL_NAME, SYSCALL_NARGS)
    ret_NOERRNO
T_PSEUDO_END_NOERRNO (SYSCALL_SYMBOL)
T_PSEUDO_NOERRNO宏引用PSEUDO_NOERRNO宏

T_PSEUDO_END_NOERRNO宏引用PSEUDO_END_NOERRNO宏

#define T_PSEUDO_NOERRNO(SYMBOL, NAME, N)   PSEUDO_NOERRNO (SYMBOL, NAME, N)
#define T_PSEUDO_END_NOERRNO(SYMBOL)        PSEUDO_END_NOERRNO (SYMBOL)


#undef	 PSEUDO_NOERRNO
#define 	PSEUDO_NOERRNO(name, syscall_name, args)			      \
  .text;								      \
  ENTRY (name)								      \
    DO_CALL (syscall_name, args)
PSEUDO_NOERRNO宏在文件开头声明文件内容为代码段

.text;
定义了名字为name的函数

#define 	ENTRY(name)							      \
  .globl C_SYMBOL_NAME(name);						      \
  .type C_SYMBOL_NAME(name),@function;					      \
  .align ALIGNARG(4);							      \
  C_LABEL(name)								      \
  cfi_startproc;							      \
  CALL_MCOUNT

#ifndef C_SYMBOL_NAME
# define C_SYMBOL_NAME(name) name
#endif

#define ALIGNARG(log2) 1<<log2     //代码对齐

# define C_LABEL(name)	name##:      //函数名

# define cfi_startproc			.cfi_startproc

#define CALL_MCOUNT		/* Do nothing.  */
执行了系统调用

#undef	 DO_CALL
#define DO_CALL(syscall_name, args)			      		      \
    PUSHARGS_##args							      \
    DOARGS_##args							      \
    movl $SYS_ify (syscall_name), %eax;					      \
    ENTER_KERNEL							      \
    POPARGS_##args
DO_CALL宏根据命令行参数个数args的不同执行不同的宏。

当args为0时：

#define PUSHARGS_0	/* No arguments to push.  */
#define	 DOARGS_0	/* No arguments to frob.  */
#define	 POPARGS_0	/* No arguments to pop.  */
#define	 _PUSHARGS_0	/* No arguments to push.  */
#define _DOARGS_0(n)	/* No arguments to frob.  */
#define	 _POPARGS_0	/* No arguments to pop.  */
程序执行

movl $SYS_ify (syscall_name), %eax;					      
ENTER_KERNEL							      

//根据系统调用名，返回系统调用号
#undef SYS_ify
#define SYS_ify(syscall_name)	__NR_##syscall_name

//切入内核执行系统调用
#ifdef I386_USE_SYSENTER
# ifdef SHARED
#  define ENTER_KERNEL call *%gs:SYSINFO_OFFSET
# else
#  define ENTER_KERNEL call *_dl_sysinfo
# endif
#else
# define ENTER_KERNEL int $0x80
#endif


当args为1时：

#define PUSHARGS_1 	movl %ebx, %edx; L(SAVEBX1): PUSHARGS_0
#define	 DOARGS_1 	_DOARGS_1 (4)
#define	 POPARGS_1 	POPARGS_0; movl %edx, %ebx; L(RESTBX1):
#define	 _PUSHARGS_1 	pushl %ebx; cfi_adjust_cfa_offset (4); \
			cfi_rel_offset (ebx, 0); L(PUSHBX1): _PUSHARGS_0
#define _DOARGS_1(n)	 movl n(%esp), %ebx; _DOARGS_0(n-4)
#define	 _POPARGS_1	 _POPARGS_0; popl %ebx; cfi_adjust_cfa_offset (-4); \
			cfi_restore (ebx); L(POPBX1):
程序执行：

	movl %ebx, %edx;
movl 4(%esp), %ebx;
movl $SYS_ify (syscall_name), %eax;					      
ENTER_KERNEL
movl %edx, %ebx;


当args为2时：

#define PUSHARGS_2 	PUSHARGS_1
#define	 DOARGS_2 	_DOARGS_2 (8)
#define	 POPARGS_2	 POPARGS_1
#define _PUSHARGS_2 	_PUSHARGS_1
#define 	_DOARGS_2(n) 	movl n(%esp), %ecx; _DOARGS_1 (n-4)
#define	 _POPARGS_2	 _POPARGS_1
程序执行：

	movl %ebx, %edx;
	movl 8(%esp), %ecx;
movl 4(%esp), %ebx;
movl $SYS_ify (syscall_name), %eax;					      
ENTER_KERNEL
movl %edx, %ebx;


当args为3时：

#define PUSHARGS_3	 _PUSHARGS_2
#define DOARGS_3	 _DOARGS_3 (16)
#define POPARGS_3 	_POPARGS_3
#define _PUSHARGS_3 	_PUSHARGS_2
#define _DOARGS_3(n) 	movl n(%esp), %edx; _DOARGS_2 (n-4)
#define _POPARGS_3	 _POPARGS_2
程序执行

pushl %ebx;
movl 16(%esp), %edx;
movl 12(%esp), %ecx;
movl 8(%esp), %ebx;
movl $SYS_ify (syscall_name), %eax;
ENTER_KERNEL
popl %ebx


当args参数为4时：

#define PUSHARGS_4	 _PUSHARGS_4
#define DOARGS_4	 _DOARGS_4 (24)
#define POPARGS_4 	_POPARGS_4
#define _PUSHARGS_4 	pushl %esi; cfi_adjust_cfa_offset (4); \
			cfi_rel_offset (esi, 0); L(PUSHSI1): _PUSHARGS_3
#define _DOARGS_4(n) 	movl n(%esp), %esi; _DOARGS_3 (n-4)
#define _POPARGS_4 	_POPARGS_3; popl %esi; cfi_adjust_cfa_offset (-4); \
			cfi_restore (esi); L(POPSI1):
程序执行：

pushl %esi;
pushl %ebx;
movl 24(%esp), %esi;
movl 20(%esp), %edx;
movl 16(%esp), %ecx;
movl 12(%esp), %ebx;
movl $SYS_ify (syscall_name), %eax;
ENTER_KERNEL
popl %ebx;
popl %esi;


当参数为5时：

#define PUSHARGS_5	 _PUSHARGS_5
#define DOARGS_5	 _DOARGS_5 (32)
#define POPARGS_5 	_POPARGS_5
#define _PUSHARGS_5 	pushl %edi; cfi_adjust_cfa_offset (4); \
			cfi_rel_offset (edi, 0); L(PUSHDI1): _PUSHARGS_4
#define _DOARGS_5(n)	 movl n(%esp), %edi; _DOARGS_4 (n-4)
#define _POPARGS_5	 _POPARGS_4; popl %edi; cfi_adjust_cfa_offset (-4); \
			cfi_restore (edi); L(POPDI1):
程序执行

pushl %edi;
pushl %esi;
pushl %ebx;
movl 32(%esp), %edi;
movl 28(%esp), %esi;
movl 24(%esp), %edx;
movl 20(%esp), %ecx;
movl 16(%esp), %ebx;
movl $SYS_ify (syscall_name), %eax;
ENTER_KERNEL
popl %ebx;
popl %esi;
popl %edi;


当参数为6时：

#define PUSHARGS_6	 _PUSHARGS_6
#define DOARGS_6	 _DOARGS_6 (40)
#define POPARGS_6 	_POPARGS_6
#define _PUSHARGS_6 	pushl %ebp; cfi_adjust_cfa_offset (4); \
			cfi_rel_offset (ebp, 0); L(PUSHBP1): _PUSHARGS_5
#define _DOARGS_6(n) 	movl n(%esp), %ebp; _DOARGS_5 (n-4)
#define _POPARGS_6	 _POPARGS_5; popl %ebp; cfi_adjust_cfa_offset (-4); \
			cfi_restore (ebp); L(POPBP1):
程序执行

pushl %ebp; 
pushl %edi;
pushl %esi;
pushl %ebx;
movl 40(%esp), %ebp;
movl 36(%esp), %edi;
movl 32(%esp), %esi;
movl 28(%esp), %edx;
movl 24(%esp), %ecx;
movl 20(%esp), %ebx;
movl $SYS_ify (syscall_name), %eax;
ENTER_KERNEL
popl %ebx;
popl %esi;
popl %edi;
popl %ebp;
DO_CALL宏设置了系统调用参数，系统调用号，切入内核，并将系统调用返回值放入eax寄存器中。

接着，执行ret指令返回函数。

ret_NOERRNO
#define ret_NOERRNO ret
汇编文件结尾

#undef	PSEUDO_END_NOERRNO
#define	PSEUDO_END_NOERRNO(name)					      \
  END (name)

//汇编文件结束
#undef	END
#define END(name)							      \
  cfi_endproc;								      \
  ASM_SIZE_DIRECTIVE(name)

#define cfi_endproc			 .cfi_endproc
#define ASM_SIZE_DIRECTIVE(name) .size name,.-name;
到这里，整个封装代码已经全部完成。

1.5.4  T_PSEUDO_ERRVAL

#undef	PSEUDO_ERRVAL
#define	PSEUDO_ERRVAL(name, syscall_name, args) \
  .text;								      \
  ENTRY (name)								      \
    DO_CALL (syscall_name, args);					      \
    negl %eax
T_PSEUDO_ERRVAL宏定义了函数name，函数调用了系统调用syscall_name。执行完DO_CALL 宏后，系统调用执行完毕，系统调用返回值放入eax寄存器中。negl %eax取反eax寄存器的值。此时，eax寄存器中保存着错误号。

#define ret_ERRVAL ret
函数返回

#undef	PSEUDO_END_ERRVAL
#define	PSEUDO_END_ERRVAL(name) \
  END (name)
汇编文件结尾

1.5.5  T_PSEUDO

#undef	PSEUDO
#define	PSEUDO(name, syscall_name, args)				      \
  .text;								      \
  ENTRY (name)								      \
    DO_CALL (syscall_name, args);					      \
    cmpl $-4095, %eax;							      \
    jae SYSCALL_ERROR_LABEL
执行系统调用，如果其返回值大于等于-4095，则跳到SYSCALL_ERROR_LABEL处执行。

#define SYSCALL_ERROR_LABEL __syscall_error
SYSCALL_ERROR_LABEL指向__syscall_error函数。

int
__attribute__ ((__regparm__ (1)))
__syscall_error (int error)
{
  __set_errno (-error);
  return -1;
}
如果小于-4095，则直接返回

 ret
汇编文件结尾

#undef	PSEUDO_END
#define	PSEUDO_END(name)						      \
  SYSCALL_ERROR_HANDLER							      \
  END (name)

#define SYSCALL_ERROR_HANDLER	/* Nothing here; code in sysdep.c is used.  */
1.5.6  实例

chdir函数




umask函数




发布于 2017-11-28
```
