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
```
(gdb) p/x args
$5 = {file = 0x400c20, mode = 0x400b81, new = 0x0, caller = 0x7ffff7ffe150}
return _dlerror_run (dlopen_doit, &args) 
```
```
/* Call handler iff the first call.  */
#define __libc_once(ONCE_CONTROL, INIT_FUNCTION) \
  do {									      \
    if ((ONCE_CONTROL) == 0) {						      \
      INIT_FUNCTION ();							      \
      (ONCE_CONTROL) = 1;						      \
    }									      \
  } while (0)
  ```
  ```
    /* If we have not yet initialized the buffer do it now.  */
  __libc_once (once, init);
  /* Initialize buffers for results.  */
static void
init (void)
{
  if (__libc_key_create (&key, free_key_mem))
    /* Creating the key failed.  This means something really went
       wrong.  In any case use a static buffer which is better than
       nothing.  */
    static_buf = &last_result;
}
  ```
  跟踪代码
  ```
  p *args
$13 = {file = 0x400c20 "./libadd_c.so", mode = 1, new = 0x0, 
  caller = 0x400a6b <main()+30>}
```
 解释 
 #define RTLD_LAZY	0x0001	/* Lazy function call binding.  */
#define RTLD_NOW	0x0002	/* Immediate function call binding.  */
#define RTLD_BINDING_MASK  0x3	/* Mask of binding time value.  */
```
591	  if ((mode & RTLD_BINDING_MASK) == 0)
592	    /* One of the flags must be set.  */
593	    _dl_signal_error (EINVAL, file, NULL, N_("invalid mode for dlopen()"));
```
这里讲mode=1与b11相与，意思是既不是0x01 也不是0x2 则出现错误
```
_dl_open (file=0x400c20 "./libadd_c.so", mode=-2147483647, 
    caller_dlopen=0x400a6b <main()+30>, nsid=-2, argc=1, argv=0x7fffffffe008, 
    env=0x7fffffffe018) 
 ```
 在程序中，可能需要为某些整数定义一个别名，我们可以利用预处理指令#define来完成这项工作，您的代码可能是：
 
```
#define MON  1
#define TUE   2
#define WED  3
#define THU   4
#define FRI    5
#define SAT   6
#define SUN   7
在此，我们定义一种新的数据类型，希望它能完成同样的工作。这种新的数据类型叫枚举型。

 

1. 定义一种新的数据类型 - 枚举型

 以下代码定义了这种新的数据类型 - 枚举型

enum DAY
{
      MON=1, TUE, WED, THU, FRI, SAT, SUN
};
 

(1) 枚举型是一个集合，集合中的元素(枚举成员)是一些命名的整型常量，元素之间用逗号,隔开。

(2) DAY是一个标识符，可以看成这个集合的名字，是一个可选项，即是可有可无的项。

(3) 第一个枚举成员的默认值为整型的0，后续枚举成员的值在前一个成员上加1。

(4) 可以人为设定枚举成员的值，从而自定义某个范围内的整数。

(5) 枚举型是预处理指令#define的替代。

(6) 类型定义以分号;结束。
```
```
__dlopen (const char *file, int mode DL_CALLER_DECL)
mode后面的 宏定义如下，本本例子中的 define DL_CALLER_DECL /* Nothing */
#ifdef SHARED
# define DL_CALLER_DECL /* Nothing */
# define DL_CALLER RETURN_ADDRESS (0)
#else
# define DL_CALLER_DECL , void *dl_caller
# define DL_CALLER dl_caller
#endif
```
```
dlsym(RTLD_DEFAULT, name);

也就是说，handle=RTLD_DEFAULT，在网上查了下，这种情况下会发生的事情是，会在当前进程中按照 default library search order搜索name这个symbol，网上的介绍摘录如下：

 

http://www.qnx.de/developers/docs/6.4.0/neutrino/lib_ref/d/dlsym.html?lang=cn

If handle is a handle returned by dlopen(), you must not have closed that shared object by calling dlclose(). The dlsym() functions also searches for the named symbol in the objects loaded as part of the dependencies for that object.

 

If handle is RTLD_DEFAULT, dlsym() searches all objects in the current process, in load-order.

 

In the case of RTLD_DEFAULT, if the objects being searched were loaded with dlopen(), dlsym() searches the object only if the caller is part of the same dependency hierarchy, or if the object was loaded with global search access (using the RTLD_GLOBAL mode).
```
函数指针
```
初学C语言的童鞋，通常在学完函数和指针的知识后，已经是萌萌哒，学习到了函数指针（请注意不是函数和指针），更是整个人都不好了，这篇文章的目的，就是帮助我的童鞋们理解函数指针。�

函数指针概述
首先我们需要回顾一下函数的作用：完成某一特定功能的代码块。
再来回忆一下指针的作用：一种特殊的变量，用来保存地址值，某类型的指针指向某类型的地址。
下面定义了一个求两个数最大值的函数：

int maxValue (int a, int b) {
    return a > b ? a : b;
}     
而这段代码编译后生成的CPU指令存储在代码区，而这段代码其实是可以获取其地址的，而其地址就是函数名，我们可以使用指针存储这个函数的地址——函数指针。
函数指针其实就是一种特殊的指针——指向一个函数的指针。在很多高级语言中，它的思想是很重要的，尤其是它的“回调函数”，所以理解它是很有必要的。

函数指针定义与使用
任何变量定义都包含三部分: 变量类型 + 变量名 = 初值，那么定义一个函数指针，首先我们需要知道要定义一个什么样的函数指针（指针类型），那么问题来了，函数的类型又是什么呢？我们继续分析这段代码：

int maxValue (int a, int b) {
    return a > b ? a : b;
}    

这个函数的类型是有两个整型参数，返回值是个整型。对应的函数指针类型：

int (*) (int a, int b);  
对应的函数指针定义：

int (*p)(int x, int  y);    
参数名可以去掉，并且通常都是去掉的。这样指针p就可以保存函数类型为两个整型参数，返回值是整型的函数地址了。

int (*p)(int, int);
通过函数指针调用函数：

int (*p)(int, int) = NULL;
p = maxValue;
p(20, 45);
回调函数
上述内容是函数指针的基础用法，然而我们可以看得出来，直接使用函数maxValue岂不是更方便？没错，其实函数指针更重要的意义在于函数回调，而上述内容只是一个铺垫。
举个例子:
现在我们有这样一个需求：实现一个函数，将一个整形数组中比50大的打印在控制台，我们可能这样实现：

void compareNumberFunction(int *numberArray, int count, int compareNumber) {
    for (int i = 0; i < count; i++) {
        if (*(numberArray + i) > compareNumber) {
            printf("%d\n", *(numberArray + i));
        }
    }
}
int main() {

    int numberArray[5] = {15, 34, 44, 56, 64};
    int compareNumber = 50;
    compareNumberFunction(numberArray, 5, compareNumber);

    return 0;
}   
这样实现是没有问题的，然而现在我们又有这样一个需求：实现一个函数，将一个整形数组中比50小的打印在控制台。”What the fuck!”对于提需求者，你可能此时的心情是这样：



</img>
然而回到现实，这种需求是不可避免的，你可能想过复制粘贴，更改一下判断条件，然而作为开发者，我们要未雨绸缪，要考虑到将来可能添加更多类似的需求，那么你将会有大量的重复代码，使你的项目变得臃肿，所以这个时候我们需要冷静下来思考，其实这两个需求很多代码都是相同的，只要更改一下判断条件即可，而判断条件我们如何变得更加灵活呢？这时候我们就用到回调函数的知识了，我们可以定义一个函数，这个函数需要两个int型参数，函数内部实现代码是将两个整形数字做比较，将比较结果的bool值作为函数的返回值返回出来，以大于被比较数字的情况为例：

BOOL compareGreater(int number, int compareNumber) {
    return number > compareNumber;
}   
同理，小于被比较的数字函数定义如下：

BOOL compareLess(int number, int compareNumber) {
    return number < compareNumber;
}
接下来，我们可以将这个函数作为compareNumberFunction的一个参数进行传递（没错，函数可以作为参数），那么我们就需要一个函数指针获取函数的地址，从而在compareNumberFunction内部进行对函数的调用，于是，compareNumberFunction函数的定义变成了这样：

void compareNumberFunction(int *numberArray, int count, int compareNumber, BOOL (*p)(int, int)) {
    for (int i = 0; i < count; i++) {
        if (p(*(numberArray + i), compareNumber)) {
            printf("%d\n", *(numberArray + i));
        }
    }
}
具体使用时代吗如下：

int main() {

    int numberArray[5] = {15, 34, 44, 56, 64};
    int compareNumber = 50;
    // 大于被比较数字情况：
    compareNumberFunction(numberArray, 5, compareNumber, compareGreater);
    // 小于被比较数字情况：
    compareNumberFunction(numberArray, 5, compareNumber, compareLess);

    return 0;
}

根据上述案例，我们可以得出结论：函数回调本质为函数指针作为函数参数，函数调用时传入函数地址，这使我们的代码变得更加灵活，可复用性更强。

动态排序
上面的案例如果你已经理解的话那么动态排序其实你已经懂了。首先我们应该理解动态这个词，我的理解就是不同时刻，不同场景，发生不同的事，这就是动态。话不多说，直接上案例。
需求： 有30个学生需要排序
按成绩排
按年龄排
…
这种无法预测的需求变更，就是我们上文说的动态场景，那么解决方案就是函数回调：

typedef struct student{
    char name[20];
    int age;
    float score;
}Student;

//比较两个学生的年龄
BOOL compareByAge(Student stu1, Student stu2) {
    return stu1.age > stu2.age ? YES : NO;
}
//比较两个学生的成绩
BOOL compareByScore(Student stu1, Student stu2) {
    return stu1.score > stu2.score ? YES : NO;
}
void sortStudents(Student *array, int n, BOOL(*p)(Student, Student)) {
    Student temp;
    int flag = 0;
    for (int i = 0; i < n - 1 && flag == 0; i++) {
        flag = 1;
        for (int j = 0; j < n - i - 1; j++) {
            if (p(array[j], array[j + 1])) {
                temp = array[j];
                array[j] = array[j + 1];
                array[j + 1] = temp;
                flag = 0;
            }
        }
    }
}
int main() {

    Student stu1 = {"小明", 19, 98};
    Student stu2 = {"小红", 20, 78};
    Student stu3 = {"小白", 21, 88};
    Student stuArray[3] = {stu1, stu2, stu3};
    sortStudents(stuArray, 3, compareByScore);

    return 0;
}
没错，动态排序就是这么简单！

函数指针作为函数返回值
没错，既然函数指针可以作为参数，自然也可以作为返回值。再接着上案例。
需求：定义一个函数，通过传入功能的名称获取到对应的函数。



整理一下发型，然后我们分析下需求，当前我们需要定义一个叫做findFunction的函数，这个函数传入一个字符串之后会返回一个 int (*)(int, int)类型的函数指针，那么我们这个函数的声明是不是可以写成这样呢？

int (*)(int, int) findFunction(char *);   
这看起来很符合我们的理解，然而，这并不正确，编译器无法识别两个完全并行的包含形参的括号(int, int)和(char *),真正的形式其实是这样：

int (*findFunction(char *))(int, int);  
这种声明从外观上看更像是脸滚键盘出来的结果，现在让我们来逐步的分析一下这个声明的组成步骤：

findFunction是一个标识符
findFunction()是一个函数
findFunction(char *)函数接受一个类型为char *的参数
*findFunction(char *)函数返回一个指针
(*findFunction(char*))()这个指针指向一个函数
(*findFunction(char*))(int, int)指针指向的函数接受两个整形参数
int (*findFunction(char *))(int, int)指针指向的函数返回一个整形

现在我们的分析已经完成了，编译器可以通过了，现在程序员疯了，这对我们来说就像鲱鱼罐头一样难以下咽，那么我们是不是有更好的书写方式呢？（老司机友情提示：typedef）



最终代码演变成了这样：

// 重定义函数指针类型
typedef int (*FUNC)(int, int);

// 求最大值函数
int maxValue(int a, int b) {
    return a > b ? a : b;
}

// 求最小值函数
int minValue(int a, int b) {
    return a < b ? a : b;
}
// findFunction函数定义
FUNC findFunction(char *name) {
    if (0 == strcmp(name, "max")) {
        return maxValue;
    } else if (0 == strcmp(name, "min")) {
        return minValue;
    }

    printf("Function name error");
    return NULL;
}   

int main() {

    int (*p)(int, int) = findFunction("max");
    printf("%d\n", p(3, 5));

    int (*p1)(int, int) = findFunction("min");
    printf("min = %d\n", p1(3, 5));

    return 0;
}
到了这里，函数指针的基础内容已经结束了，有的同学还有可能困惑，为什么我要以函数去获取函数呢，直接使用maxValue和minValue不就好了么，其实在以后的编程过程中，很有可能maxValue和minValue被封装了起来，类的外部是不能直接使用的，那么我们就需要这种方式，如果你学习了Objective-C你会发现，所有的方法调用的实现原理都是如此。

函数指针数组
现在我们应该清楚表达式“char * (*pf)(char * p)”定义的是一个函数指针pf。既然pf 是一个指针，那就可以储存在一个数组里。把上式修改一下：
char * (*pf[3])(char * p);
这是定义一个函数指针数组。它是一个数组，数组名为pf，数组内存储了3 个指向函数的指针。这些指针指向一些返回值类型为指向字符的指针、参数为一个指向字符的指针的函数。这念起来似乎有点拗口。不过不要紧，关键是你明白这是一个指针数组，是数组。

函数指针数组怎么使用呢？给一个非常简单的例子，只要真正掌握了使用方法，再复杂的问题都可以应对。如下：

char * fun1(char * p)
{
   printf("%s\n",p);
   return p;
}
char * fun2(char * p)
{
   printf("%s\n",p);
   return p;
}
char * fun3(char * p)
{
   printf("%s\n",p);
   return p;
}
int main(){
   char * (*pf[3])(char * p);
   pf[0] = fun1; // 可以直接用函数名
   pf[1] = &fun2; // 可以用函数名加上取地址符
   pf[2] = &fun3;
   pf[0]("fun1");
   pf[0]("fun2");
   pf[0]("fun3");
   return 0;
}
是不是感觉上面的例子太简单，不够刺激？好，那就来点刺激的。

函数指针数组的指针
看着这个标题没发狂吧？函数指针就够一般初学者折腾了，函数指针数组就更加麻烦，现在的函数指针数组指针就更难理解了。

其实，没这么复杂。前面详细讨论过数组指针的问题，这里的函数指针数组指针不就是一个指针嘛。只不过这个指针指向一个数组，这个数组里面存的都是指向函数的指针。仅此而已。

下面就定义一个简单的函数指针数组指针：
char * (*(*pf)[3])(char * p);
注意，这里的pf 和上面的pf 就完全是两码事了。上一节的pf 并非指针，而是一个数组名；这里的pf 确实是实实在在的指针。这个指针指向一个包含了3 个元素的数组；这个数字里面存的是指向函数的指针；这些指针指向一些返回值类型为指向字符的指针、参数为一个指向字符的指针的函数。这比面的函数指针数组更拗口。其实你不用管这么多，明白这是一个指针就ok 了。其用法与前面讲的数组指针没有差别。下面列一个简单的例子：

char * fun1(char * p)
{
   printf("%s\n",p);
   return p;
}
char * fun2(char * p)
{
   printf("%s\n",p);
   return p;
}
char * fun3(char * p)
{
   printf("%s\n",p);
   return p;
}
int main(){
   char * (*a[3])(char * p);
   char * (*(*pf)[3])(char * p);
   pf = &a;
   a[0] = fun1;
   a[1] = &fun2;
   a[2] = &fun3;
   pf[0][0]("fun1");
   pf[0][1]("fun2");
   pf[0][2]("fun3");
   return 0;
}
好了，到了这里C语言已经没有什么能阻挡你了。

作者：Ostkaka丶
链接：https://www.jianshu.com/p/f1cf2aa531d9
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```
```

 通过__builtin_return_address获取调用者函数地址
 2.1 背景介绍：__builtin_return_address是GCC提供的一个内置函数，用于判断给定函数的调用者。

6.49 Getting the Return or Frame Address of a Function里面有更多获取函数调用者的介绍。

void * __builtin_return_address (unsigned int level)
level为参数，如果level为0，那么就是请求当前函数的返回地址；如果level为1，那么就是请求进行调用的函数的返回地址。
```
```
(gdb) 
0x00000000004005ea	14	{
1: x/10i $pc
=> 0x4005ea <func_d+1>:	mov    %rsp,%rbp
   0x4005ed <func_d+4>:	callq  0x40052d <func_e>
   0x4005f2 <func_d+9>:	pop    %rbp
   0x4005f3 <func_d+10>:	retq   
   0x4005f4 <func_c>:	push   %rbp
   0x4005f5 <func_c+1>:	mov    %rsp,%rbp
   0x4005f8 <func_c+4>:	callq  0x4005e9 <func_d>
   0x4005fd <func_c+9>:	pop    %rbp
   0x4005fe <func_c+10>:	retq   
   0x4005ff <func_b>:	push   %rbp

```
```
func_e(0)=0x4005f2
func_e(1)=0x4005fd
func_e(2)=0x400608
func_e(3)=0x400613
func_e(4)=0x400629
func_e(5)=0x7ffff7a39c05
func_a=0x40060a, func_b=0x4005ff, func_c=0x4005f4, func_d=0x4005e9, func_e=0x40052d

查看function-e函数的返回地址（level为参数，如果level为0，那么就是请求当前函数的返回地址；）
则当前函数的代码地址func_e=0x40052d，那调用他的函数为func_d，调用函数返回地址为 call func_e下一个指令地址，查看反汇编代码
0x4005ed <func_d+4>:	callq  0x40052d <func_e>
   0x4005f2 <func_d+9>:	pop    %rbp
则0x0x4005f2与  __builtin_return_address(0) 对应
```
线程函数key
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

pthread_key_t key; // 好吧，这个玩意究竟是干什么的呢？

struct test_struct { // 用于测试的结构
    int i;
    float k;
};

void *child1(void *arg)
{
    struct test_struct struct_data; // 首先构建一个新的结构
    struct_data.i = 10;
    struct_data.k = 3.1415;
    pthread_setspecific(key, &struct_data); // 设置对应的东西吗？
    printf("child1--address of struct_data is --> 0x%p\n", &(struct_data));
    printf("child1--from pthread_getspecific(key) get the pointer and it points to --> 0x%p\n", (struct test_struct *)pthread_getspecific(key));
    printf("child1--from pthread_getspecific(key) get the pointer and print it's content:\nstruct_data.i:%d\nstruct_data.k: %f\n", 
        ((struct test_struct *)pthread_getspecific(key))->i, ((struct test_struct *)pthread_getspecific(key))->k);
    printf("------------------------------------------------------\n");
}
void *child2(void *arg)
{
    int temp = 20;
    sleep(2);
    printf("child2--temp's address is 0x%p\n", &temp);
    pthread_setspecific(key, &temp); // 好吧，原来这个函数这么简单
    printf("child2--from pthread_getspecific(key) get the pointer and it points to --> 0x%p\n", (int *)pthread_getspecific(key));
    printf("child2--from pthread_getspecific(key) get the pointer and print it's content --> temp:%d\n", *((int *)pthread_getspecific(key)));
}
int main(void)
{
    pthread_t tid1, tid2;
    pthread_key_create(&key, NULL); // 这里是构建一个pthread_key_t类型，确实是相当于一个key
    pthread_create(&tid1, NULL, child1, NULL);
    pthread_create(&tid2, NULL, child2, NULL);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    pthread_key_delete(key);
    return (0);
}
```
一个函数strchr（）
```
头文件：#include <string.h>

strchr() 用来查找某字符在字符串中首次出现的位置，其原型为：
    char * strchr (const char *str, int c);

【参数】str 为要查找的字符串，c 为要查找的字符。

strchr() 将会找出 str 字符串中第一次出现的字符 c 的地址，然后将该地址返回。

注意：字符串 str 的结束标志 NUL 也会被纳入检索范围，所以 str 的组后一个字符也可以被定位。

【返回值】如果找到指定的字符则返回该字符所在地址，否则返回 NULL。

返回的地址是字符串在内存中随机分配的地址再加上你所搜索的字符在字符串位置。设字符在字符串中首次出现的位置为 i，那么返回的地址可以理解为 str + i。

提示：如果希望查找某字符在字符串中最后一次出现的位置，可以使用 strrchr() 函数。
```
