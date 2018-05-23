用qemu搭建linux环境的最简单步骤(硬盘启动)
=======================================
```
只有有了最基本的东西，才能在此基础上起飞！
环境: ubuntu14 x86_64 cpu
过程很简单，准备，制作和运行
----------------------------------------
甲: 准备
----------------------------------------
1. 获取源码: a. kernel  b. busybox
    方法任意，这里从略。目录如下:

    ~/misc/qemu$ ls
    busybox  linux-3.13.0  


2. 源码编译:
    cd ~/misc/qemu/linux-3.13.0G
    make defconfig
    make
    成功编译出内核.

   defconfig 会根据当前系统架构,自动选择系统默认的配置文件,例如我的会找到先x86_64_defconfig

  如果你知道自己的机器是x86_64, 等价于直接执行 make x86_64_defconfig, 结果生成.config 文件


    cd ~/misc/qemu/busybox
    make menuconfig
        选择静态编译: BusyboxSettings->Build options->Build Busybox as a static binary
    make
    成功编译出busybox


----------------------------------------
乙: 制作
----------------------------------------
    a. 内核文件准备
    cd ~/misc/qemu/linux-3.13.0
    cp arch/x86/boot/bzImage ..

    b 根文件系统制作
    cd ~/misc/qemu
    dd if=/dev/zero of=rootfs.img bs=1M count=10 #创建大小为10M到根文件系统
    mkfs.ext3 rootfs.img #以ext3类型来格式化
    mkdir  rootdir
    sudo mount -t ext3 -o loop rootfs.img rootdir #将img mount 到 loop设备上
    cd rootdir
    mkdir dev proc sys  创建三个目录

    #把busybox文件系统安装到根文件系统中
    cd ~/misc/qemu/busybox
    sudo make install CONFIG_PREFIX=~/misc/qemu/rootdir

    cd ~/misc/qemu
    sudo umount rootdir
    rmdir rootdir

    制作完成后的目录：
    ~/misc/qemu$ ls
    busybox  bzImage  linux-3.13.0  rootfs.img

----------------------------------------
丙： 运行
----------------------------------------
    cd ~/misc/qemu
    qemu-system-x86_64  -kernel bzImage -hda rootfs.img -append "root=/dev/sda init=/bin/ash"
    内核启动， 熟悉的linux 环境出来了，ls, cat, .....
    ctrl-alt 释放qemu 鼠标。

其他:

   使用 -hda, 指明硬盘镜像， qemu-system 能够仿真硬盘， -cdrom 还可以仿真CD-ROM -boot 选项指定启动设备

默认为硬盘： d 从 CD-ROM 引导， a 从软盘引导，c 从硬盘引导（默认），而n 从网络引导



 -append 是内核启动参数, root=/dev/XXX, root 是根文件系统之意，这个启动参数/dev 与 dev设备没有关系。只是约定名称

    init=XXX, init 指明根文件系统第一个运行的程序。



补充: 升级了qemu版本后,执行上述命令有一个警告信息,看着不爽,

WARNING: Image format was not specified for 'rootfs.img' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.


我猜测着试了试去不掉这个WARNING, google,baidu 大部分说不清楚, 后来发现下面的参数可以去掉警告,来指明format=raw,

注意, 多一个标点甚至空格都可能失败啊, qemu 在这方面怎么这么不友好呢?!

qemu-system-x86_64 -kernel bzImage -drive format=raw,file=rootfs.img,media=disk -append "root=/dev/sda init=/bin/ash"


media=disk 我是猜的, 我本来想指定它是第一个磁盘, 但不知道怎样表达, 就这样吧,有空再研究!  it works now!





qemu-system-x86_64 -kernel bzImage -drive format=raw,file=rootfs.img,media=disk -append "root=/dev/sda init=/bin/ash console=ttyS0" -nographic

添加上 -append "console=ttyS0" -nographic 后, 可以启动控制台输出, 观察所有log.  退出qemu Ctrl+ax, 

提示/ # QEMU: Terminated



----------------------------------------
进阶篇:
----------------------------------------
    1. 调试内核
    a. 编译时加上-g 选项
    b.  qemu-system-x86_64 启动时加-S 选项， 使内核启动冻结
    c. ctrl+alt+2 可切换到qemo控制台，输入“gdbserver"
    d. 另起一个终端，    
        cd ~/misc/qemu/linux-3.13.0
        gdb vmlinux
        target remote localhost:1234

    开始调试内核。
    还可以使用ddd 前端或 vimgdb 前端等， 已经着陆了。
qemu-system-x86_64 -kernel bzImage -drive format=raw,file=rootfs.img,media=disk -append "root=/dev/sda init=/bin/ash"
```
```
$ qemu -kernel arch/x86/boot/bzImage -append "noapic"

有时候内核会这样崩溃：

MP-BIOS BUG 8254 timer not connected

trying to set up timer as Virtual Wire IRQ

所以需要添加-append "noapic"参数
```
