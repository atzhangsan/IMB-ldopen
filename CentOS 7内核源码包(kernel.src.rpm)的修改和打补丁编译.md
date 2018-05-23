CentOS 7内核源码包(kernel.src.rpm)的修改和打补丁编译
==================================================
```
在某些特定应用环境中，我们会涉及到Linux内核的源码修改和编译安装，通常是选择从kernel.org获取最新的稳定发行版内核进行修改并编译安装，但实际情况中，CentOS的内核是基于官方内核做了一些定制裁剪的，选用CentOS的内核源码包做一些适应性修改，重编译打包成RPM做分发会更适合生产需要，因此记录下源码包kernel.src.rpm的打补丁和重编译过程。

1
[root@localhost ~]#uname -a
2
Linux localhost 3.10.0-327.36.3.el7.x86_64 #1 SMP Mon Oct 24 16:09:20 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
//获取当前系统的内核版本号

1
[root@localhost ~]# cat /etc/redhat-release
2
CentOS Linux release 7.2.1511 (Core)&nbsp;
//获取当前系统的发行版本号

1
[root@localhost ~]# wget http://vault.centos.org/7.2.1511/updates/Source/SPackages/kernel-3.10.0-327.22.2.el7.src.rpm
//从http://valut.centos.org 获取对应版本内核src源码包并下载，这里以kernel-3.10.0-327.22.2.el7.src.rpm为例.

1
[root@localhost ~]# rpm -ivh kernel-3.10.0-327.36.3.el7.src.rpm
//安装源码包,会在当前目录下创建一个 rpmbuild 文件夹

在安装过程中出现一下提示,builder用户名和组不存在提示可以忽略,官方的源码包是在builder用户下构建的.
warning: user builder does not exist – using root
warning: group builder does not exist – using root

1
[root@localhost ~]# yum install wget gcc gc bc gd make perl ncurses-devel xz rpm-build xmlto asciidoc hmaccalc python-devel \
2
newt-devel pesign binutils-devel audit-libs-devel numactl-devel pciutils-devel perl-ExtUtils-Embed -y
//为避免编译缺少对应的依赖包头文件和工具，使用yum安装构建编译所需的工具

在使用rpmbuild 自动构建内核rpm包时,首先读取SEPCS目录下的kernel.spec 配置文件，解压SOURCES 目录中的内核源码包,并打上对应的patch 文件, 配置后进行编译生成rpm包, 因此修改内核源码有2种方式。
1. 直接解压内核tar.xz包,修改编辑完成后直接打包覆盖原有的tar.zx包，执行rpmbuild命令构建.
2. 解压内核tar.zx包,备份需要修改的源文件,修改后对比源文件生成patch补丁,并在kernel.spec配置中指定补
   丁文件,执行rpmbuild构建

第一种方式解压打包比较繁琐，不如制作patch补丁来的方便,在大版本不变的情况下只需要引入patch文件即可,通用性更好.推荐使用

1
[root@localhost ~]# cd rpmbuild/SOURCES/
2
[root@localhost SOURCES]# tar -xf linux-3.10.0-327.36.3.el7.tar.xz -C /tmp
//解压内核源码包到 /tmp下

这里以修改TCP参数头文件tcp.h为例

1
[root@localhost ~]# cd /tmp/linux-3.10.0-327.36.3.el7/include/net
2
[root@localhost net]# cp tcp.h tcp_orig.h
//进入需要修改的文件目录,备份源文件为tcp_orig.h

此时可对tcp.h 进行参数修改,完成后，在内核源码包根路径下，使用diff生成patch补丁文件

1
[root@localhost ~]# cd /tmp/linux-3.10.0-327.36.3.el7/
2
[root@localhost linux-3.10.0-327.36.3.el7]# diff -up include/net/tcp_orig.h include/net/tcp.h >> /tmp/custom.patch
//生成patch文件, 第一个参数为原始文件,第二个为修改后的文件

如需要对文件夹下大量文件修改,可以先备份目录后修改，使用以下命令生成批量文件补丁
diff -uprN include_orig/net include/net >> /tmp/custom.patch

1
[root@localhost ~] cp /tmp/custom.patch /root/rpmbuild/SOURCES/
//将patch补丁复制到rpmbuild源码包目录

1
[root@localhost ~] vim /root/rpmbuild/SPECS/kernel.spec
//编辑spec配置文件

查找ApplyOptionPatch开头的行,新增一行自定义补丁
ApplyOptionalPatch linux-kernel-test.patch
ApplyOptionalPatch debrand-single-cpu.patch
ApplyOptionalPatch debrand-rh_taint.patch
ApplyOptionalPatch debrand-rh-i686-cpu.patch
ApplyOptionalPatch custom.patch

继续查找Patch1开头的行，增加一行自定义补丁,Patch后面的数字为优先级,可以自定义.
# empty final patch to facilitate testing of kernel patches
Patch999999: linux-kernel-test.patch
Patch1000: debrand-single-cpu.patch
Patch1001: debrand-rh_taint.patch
Patch1002: debrand-rh-i686-cpu.patch
Patch4000: custom.patch

完成后，可以将custom.patch 和 kernel.spec 文件保留作备用。

1
[root@localhost ~] cd ~/rpmbuild/SPECS
2
//进入spec配置文件夹.
1
[root@localhost rpmbuild] rpmbuild -bb --target=$(uname -m) SPECS/kernel.spec
//使用rpmbuild -bb 命令直接构建二进制rpm安装包，会自动解压内核至BUILDS文件夹，并根据kernel.spec配置打补丁等操作，等待结束，会在RPMS目录中生成修改完成后的RPM包，将包复制到目标服务器执行
rpm -Uvh kernel-3.10.0-327.36.3.el7.x86_64.rpm 重启并进行验证.
硬件配置对内核编译时间影响较大，从几十分钟甚至几个小时，建议使用i7级别以上的多核CPU提高效率。
```
