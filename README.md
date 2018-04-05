# IMB-ldopen
跟踪一个ldopen的实现。
```
__dlopen (const char *file, int mode DL_CALLER_DECL)
{
# ifdef SHARED
  if (__builtin_expect (_dlfcn_hook != NULL, 0))
    return _dlfcn_hook->dlopen (file, mode, DL_CALLER);
# endif

  struct dlopen_args args;
  args.file = file;
  args.mode = mode;
  args.caller = DL_CALLER;

# ifdef SHARED
  return _dlerror_run (dlopen_doit, &args) ? NULL : args.new;
# else
  if (_dlerror_run (dlopen_doit, &args))
    return NULL;

  __libc_register_dl_open_hook ((struct link_map *) args.new);
  __libc_register_dlfcn_hook ((struct link_map *) args.new);

  return args.new;
# endif
}
```
#define定义宏
#undef取消已定义的宏
#if如果给定条件为真，则编译下面代码
#ifdef如果宏已经定义，则编译下面代码
#ifndef如果宏没有定义，则编译下面代码
```
GCC __builtin_expect的作用

将流水线引入cpu，可以提高cpu的效率。更简单的说，让cpu可以预先取出下一条指令，可以提供cpu的效率。如下图所示：
+--------------------------------
|取指令 | 执行指令 | 输出结果
+--------------------------------
|             | 取指令     | 执行
+--------------------------------
可见，cpu流水钱可以减少cpu等待取指令的耗时，从而提高cpu的效率。
        如果存在跳转指令，那么预先取出的指令就无用了。cpu在执行当前指令时，从内存中取出了当前指令的下一条指令。执行完当前指令后，cpu发现不是要执行下一条指令,而是执行offset偏移处的指令。cpu只能重新从内存中取出offset偏移处的指令。因此，跳转指令会降低流水线的效率，也就是降低cpu的效率。
        综上，在写程序时应该尽量避免跳转语句。那么如何避免跳转语句呢？答案就是使用__builtin_expect。
        这个指令是gcc引入的，作用是"允许程序员将最有可能执行的分支告诉编译器"。这个指令的写法为：__builtin_expect(EXP, N)。意思是：EXP==N的概率很大。一般的使用方法是将__builtin_expect指令封装为LIKELY和UNLIKELY宏。这两个宏的写法如下。
        #define LIKELY(x) __builtin_expect(!!(x), 1) //x很可能为真
        #define UNLIKELY(x) __builtin_expect(!!(x), 0) //x很可能为假
如下是一个实际的例子。
//test_builtin_expect.c  
#define LIKELY(x) __builtin_expect(!!(x), 1)  
#define UNLIKELY(x) __builtin_expect(!!(x), 0)  
  
int test_likely(int x)  
{  
    if(LIKELY(x))  
    {  
        x = 5;  
    }  
    else  
    {  
        x = 6;  
    }  
  
    return x;  
}  
  
int test_unlikely(int x)  
{  
    if(UNLIKELY(x))  
    {  
        x = 5;  
    }  
    else  
    {  
        x = 6;  
    }  
  
    return x;  
}  
运行如下命令:
        gcc -fprofile-arcs -O2 -c test_builtin_expect.c
        objdump -d test_builtin_expect.o
输出的汇编码为:
<test_likely>:  
00    push     %ebp  
01    mov      %esp,%ebp  
03    mov      0x8(%ebp),%eax  
06    addl     $0x1,0x38  
0d    adcl     $0x0,0x3c  
14    test     %eax,%eax  
16    jz       2d <test_likely+0x2d>//主要看这里。此处的效果是eax不为零时，不需要跳转。即x为真是不跳转。  
18    addl     $0x1,0x40  
1f    mov      $0x5,%eax  
24    adcl     $0x0,0x44  
2b    pop      %ebp  
2c    ret  
2d    addl     $0x1,0x48  
34    mov      $0x6,%eax  
39    adcl     $0x0,0x4c  
40    pop      %ebp  
41    ret  
42    lea      0x0(%esi,%eiz,1),%esi  
49    lea      0x0(%edi,%eiz,1),%edi  
  
<test_unlikely>:  
50    push     %ebp  
51    mov      %esp,%ebp  
53    mov      0x8(%ebp),%edx  
56    addl     $0x1,0x20  
5d    adcl     $0x0,0x24  
64    test     %edx,%edx  
66    jne      7d <test_unlikely+0x2d>//主要看这里。此处的效果是edx为零时,不需跳转。即x为假时不跳转。  
68    addl     $0x1,0x30  
6f    mov      $0x6,%eax  
74    adcl     $0x0,0x34  
7b    pop      %ebp  
7c    ret  
7d    addl     $0x1,0x28  
84    mov      $0x5,%eax  
89    adcl     $0x0,0x2c  
90    pop      %ebp  
91    ret  
92    lea      0x0(%esi,%eiz,1),%esi  
99    lea      0x0(%edi,%eiz,1),%edi 
可见，编译器利用程序员作出的判断，生成了高效的汇编码。即，跳转语句不生效的概率很大。
```

