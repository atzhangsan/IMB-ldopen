给VMware下的Linux扩展磁盘空间（以CentOS7）
=======================================
```
当磁盘分区发现不够用时，能想道的第一个做法就是增加分区大小。但是一般Linux如果没有采用逻辑卷管理，则动态增加分区大小很困难，一个能想道的办法就是，备份分区文件系统数据，删除分区，然后再重新创建分区，恢复备份的文件系统，这个做法比较玄，可能删除分区后导致系统无法启动。
 
第二个做法就是，创建一个新的逻辑分区（当然必须有未使用的磁盘空间能分配），将文件系统从老分区拷贝到新分区，然后修改fstab，使用新分区/文件系统替换老的分区/文件系统
 
第三种做法是，创建一个新的逻辑分区，将新的逻辑分区格式化ext3（或其他类型）的文件系统，mount到磁盘空间不够的文件系统，就跟原来的分区/文件系统一样的使用。
 
这里采用的是第三种方式：
 



#查看挂载点：

df -h
#显示：
文件系统 容量 已用 可用 已用%% 挂载点
[root@localhost zoubf]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   18G   15G  2.9G   84% /
devtmpfs                 485M     0  485M    0% /dev
tmpfs                    494M   84K  494M    1% /dev/shm
tmpfs                    494M  7.1M  487M    2% /run
tmpfs                    494M     0  494M    0% /sys/fs/cgroup
/dev/sda1                497M  119M  379M   24% /boot
/dev/sr0                 3.9G  3.9G     0  100% /run/media/zoubf/CentOS 



 
一、扩展VMWare硬盘空间
　　关闭Vmware 的 Linux系统，这样，才能在VMWare菜单中设置：
　　VM -> Settings... -> Hardware -> Hard Disk -> Utilities -> Expand
　　输入你想要扩展到多少G。本文假设你新增加了 30G



二、对新增加的硬盘进行分区、格式化
　　增加了空间的硬盘是 /dev/sda。　　分区：
fdisk /dev/sda　　　　操作 /dev/sda 的分区表
p　　　　　　　查看已分区数量（我看到有两个 /dev/sda1 /dev/sda2）
n　　　　　　　新增加一个分区
p　　　　　　　分区类型我们选择为主分区
　　　　　　分区号选3（因为1,2已经用过了，见上）
回车　　　　　　默认（起始扇区）
回车　　　　　　默认（结束扇区）
t　　　　　　　修改分区类型
　　　　　　选分区3
8e　　　　　　修改为LVM（8e就是LVM）
w　　　　　　写分区表
q　　　　　　完成，退出fdisk命令
　　系统提示重启。



开机后，格式化：
mkfs.ext3 /dev/sda3





三、添加新LVM到已有的LVM组，实现扩容

lvm　　　　　　　　　　　　　　　　　　    进入lvm管理
lvm> pvcreate /dev/sda3　　　　　　　　　   这是初始化刚才的分区，必须的
lvm>vgextend centos /dev/sda3              　　　将初始化过的分区加入到虚拟卷组vg_dc01
lvm>lvextend -L +29.9G /dev/mapper/centos-root　　扩展已有卷的容量（29.9G这个数字在后面解释）
lvm>pvdisplay　　　　　　　　　　　　　　  查看卷容量，这时你会看到一个很大的卷了
lvm>quit　　　　　　　　　　　　　　　　　退出
上面那个 29.9G 怎么来的呢？因为你在VMWare新增加了30G，但这些空间不能全被LVM用了，你可以在上面的lvextend操作中一个一个的试探，比如 29.9G, 29.8G ... 直到不报错为止，这样你就可以充分使用新增加的硬盘空间了，当然这是因为我不懂才用的笨办法，高手笑笑就过了吧。（我更不懂啊，原作者，我直接上了29.9G，结果就OK了）
以上只是卷扩容了，下面是文件系统的真正扩容，输入以下命令：
resize2fs /dev/mapper/centos-root ..............centos6 xia
xfs_growfs /dev/mapper/centos-root
现在，屌丝们，再运行下：df -h
查看下我们机器性感的硬盘吧。
```
