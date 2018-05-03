```
下面正式介绍gdbtui。

#1. 打开TUI模式

方法一: 使用gdbtui or gdb-tui开始一个调试。

  gdbtui -q sample
友情提示：通过下面的方式调试一个正在运行的进程

  gdb  -p pid
如果出现如下错误，请参考这里。

  Could not attach to process.  If your uid matches the uid of the target
  process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
  again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
方法二: 直接使用gdb调试代码，在需要的时候使用切换键 ctrl+x a调出gdbtui。

#2. TUI模式下有4个窗口,

(cmd)command 命令窗口. 可以键入调试命令
(src)source 源代码窗口. 显示当前行,断点等信息
(asm)assembly 汇编代码窗口
(reg)register 寄存器窗口
最常用的也就是默认使用的方式，也可以通过layout命令来进行选择自己需要的窗口，可参见help layout.

#3. gdbtui相关的其他命令

layout

用以修改窗口布局

 help layout
 layout src
 layout asm
 layout split
winheight

调整各个窗口的高度。

 help winheight
 winheight src +5
 winheight src -4
space

当前窗口放大或者缩小以后，gdbtui窗口不会发生变化，我们可以通过space 键强行刷新gdbtui窗口。

focus next / prev

在默认设置下，方向键和PageUp PageDn 都是用来控制gdbtui的src窗口的，所以，我们常用的上下键用来显示前一条命令和后一条命令的功能就没有了， 不过这个时候我们可以通过ctrl + n / ctrl +p 来获取这个功能。

ps:当我们通过方向键调整了gdbtui 的src 窗口以后，可以通过update命令重新把焦点定位到当前执行的代码上。

我们可以通过focus命令来调整焦点位置，默认情况下是在src窗口，通过focus next命令， 焦点就移到cmd窗口了，这时候就可以像以前一样，通过方向键来切换到上一条命令和下一条命令。

 help focus
 focus cmd
 focus src
焦点不在src窗口以后，我们就不同通过方向键来浏览源码了。
```
