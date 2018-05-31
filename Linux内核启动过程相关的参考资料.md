```
Linux内核启动过程相关的参考资料

计算机的启动过程概述

x86 CPU启动的第一个动作CS:EIP=FFFF:0000H（换算为物理地址为000FFFF0H，因为16位CPU有20根地址线)，即BIOS程序的位置。
http://wenku.baidu.com/view/4e5c49eb172ded630b1cb699.html

BIOS例行程序检测完硬件并完成相应的初始化之后就会寻找可引导介质，找到后把引导程序加载到指定内存区域后，
就把控制权交给了引导程序。这里一般是把硬盘的第一个扇区MBR和活动分区的引导程序加载到内存（即加载BootLoader），
加载完整后把控制权交给BootLoader。

引导程序BootLoader开始负责操作系统初始化，然后起动操作系统。
启动操作系统时一般会指定kernel、initrd和root所在的分区和目录，
比如root (hd0,0)，kernel (hd0,0)/bzImage root=/dev/ram init=/bin/ash，initrd (hd0,0)/myinitrd4M.img

内核启动过程包括start_kernel之前和之后，之前全部是做初始化的汇编指令，之后开始C代码的操作系统初始化，
最后执行第一个用户态进程init。

一般分两阶段启动，先是利用initrd的内存文件系统，然后切换到硬盘文件系统继续启动。initrd文件的功能主要有两个：
1、提供开机必需的但kernel文件(即vmlinuz)没有提供的驱动模块(modules)
2、负责加载硬盘上的根文件系统并执行其中的/sbin/init程序进而将开机过程持续下去
