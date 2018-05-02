gdb断点高级技巧
==============
条件断点
-------
```
设置一个条件断点，条件由cond指定；在gdb每次执行到此
断点时,cond都被计算。当cond的值为非零时，程序在断点处停止。



用法：

break [break-args] if (condition)



例如：

break main if argc > 1
break 180 if (string == NULL && i < 0)
break test.c:34 if (x & y) == 1
break myfunc if i % (j+3) != 0
break 44 if strlen(mystring) == 0
b 10 if ((int)$gdb_strcmp(a,"chinaunix") == 0)
b 10 if ((int)aa.find("dd",0) == 0)



condition

可以在我们设置的条件成立时，自动停止当前的程序，先使用break(或者watch也可以）设置断点，
然后用condition来修改这个断点的停止（就是断）的条件。



用法：
condition <break_list> (conditon)



例如：
cond 3 i == 3

condition 2 ((int)strstr($r0,".plist") != 0)



ignore

如果我们不是想根据某一条件表达式来停止，而是想断点自动忽略前面多少次的停止，从某一次开始
才停止，这时ignore就很有用了。



用法：

ignore <break_list> count

上面的命令行表示break_list所指定的断点号将被忽略count次。



例如：

ignore 1 100，表示忽略断点1的前100次停止


为断点设置命令列表

设置一个断点并且在上面中断后，我们必须会查询一些变量或者做一些其他动作。
如果这些动作可以一起呵成，岂不妙哉！使用命令列表(commands)就能实现这个
功能。



步骤：
1.建立断点。
2.使用commands命令



用法:
commands <break_list>



例如：
(gdb) commands 1
Type commands for when breakpoint 1 is hit,one per line.
End with a line saying just "end".
>silent
>print "n= %d \n",n
>continue
>end

 

文件记录 ：

断点2在open函数开头

(gdb) commands 2
Type commands for when breakpoint 2 is hit,one per line.
End with a line saying just "end".
>x/s $r0
>continue
>end
```
