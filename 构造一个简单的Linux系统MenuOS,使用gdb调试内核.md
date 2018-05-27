使用自己的Linux系统环境搭建MenuOS的过程
=====================================
```
# 下载内核源代码编译内核
cd ~/LinuxKernel/
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.18.6.tar.xz
xz -d linux-3.18.6.tar.xz
tar -xvf linux-3.18.6.tar
cd linux-3.18.6
make i386_defconfig
make # 一般要编译很长时间，少则20分钟多则数小时
 
# 制作根文件系统
cd ~/LinuxKernel/
mkdir rootfs
git clone  https://github.com/mengning/menu.git
cd menu
gcc -o init linktable.c menu.c test.c -m32 -static –lpthread
cd ../rootfs
cp ../menu/init ./
find . | cpio -o -Hnewc |gzip -9 > ../rootfs.img
 
# 启动MenuOS系统
cd ~/LinuxKernel/
qemu -kernel linux-3.18.6/arch/x86/boot/bzImage -initrd rootfs.img


重新配置编译Linux使之携带调试信息







在原来配置的基础上，make menuconfig选中如下选项重新配置Linux，使之携带调试信息








kernel hacking—>

[*] compile the kernel with debug info




make重新编译（时间较长）







使用gdb跟踪调试内核




qemu -kernel linux-3.18.6/arch/x86/boot/bzImage -initrd rootfs.img -s -S # 关于-s和-S选项的说明：
# -S freeze CPU at startup (use ’c’ to start execution)
# -s shorthand for -gdb tcp::1234 若不想使用1234端口，则可以使用-gdb tcp:xxxx来取代-s选项
 
另开一个shell窗口



gdb
（gdb）file linux-3.18.6/vmlinux # 在gdb界面中targe remote之前加载符号表
（gdb）target remote:1234 # 建立gdb和gdbserver之间的连接,按c 让qemu上的Linux继续运行
（gdb）break start_kernel # 断点的设置可以在target remote之前，也可以在之后
```
```
qemu-system-x86_64 -kernel /usr/src/kernels/linux-3.10/arch/x86_64/boot/bzImage -initrd rootfs.img -append "noapic" -s -S
```
