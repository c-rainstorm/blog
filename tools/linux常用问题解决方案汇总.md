## Linux 常用问题解决方案汇总

之前的解决方法都是从网上找资料，然后记录到为知笔记里，但这一点，那一点，查找的时候很不方便。写这篇汇总目的是方便以后使用时查阅。同时也希望能够帮到大家。

<!-- TOC -->

- [Linux 常用问题解决方案汇总](#linux-常用问题解决方案汇总)
    - [Linux 下制作启动盘](#linux-下制作启动盘)
    - [安装 CentOS](#安装-centos)
    - [删除内核升级后多余的启动项](#删除内核升级后多余的启动项)
    - [恢复 Windows 启动项](#恢复-windows-启动项)
    - [配置字体](#配置字体)

<!-- /TOC -->

---


### Linux 下制作启动盘

1. 查看 U 盘所对应的分区
1. 使用 `dd` 命令写入镜像
1. 写入以后即可使用。**特别注意的是 `dd` 命令写入以后 U 盘会变成只读的，如果安装完系统之后不打算将其专门作为启动盘的话还需要将 U 盘恢复一下，稍后会作简要介绍**

```
[chen@localhost Downloads]$ df -hl
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda5       197G   21G  167G  11% /
devtmpfs        3.8G     0  3.8G   0% /dev
tmpfs           3.8G  8.0M  3.8G   1% /dev/shm
tmpfs           3.8G   34M  3.8G   1% /run
tmpfs           3.8G     0  3.8G   0% /sys/fs/cgroup
tmpfs           775M   28K  775M   1% /run/user/1001
tmpfs           775M     0  775M   0% /run/user/0
/dev/sdb1        15G  8.0K   15G   1% /run/media/chen/CentOS 7 x86_64

// sdb 是设备号，sdb1 是分区号，sdb1 是设备 sdb 的第一个分区
[chen@localhost Downloads]$ sudo dd bs=4M if=CentOS-7-x86_64-Everything-1611.iso of=/dev/sdb
[sudo] password for chen:
1974+1 records in
1974+1 records out
8280604672 bytes (8.3 GB) copied, 878.385 s, 9.4 MB/s

```

- 恢复 U 盘正常使用

```
[chen@localhost Downloads]$ df -hl
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda5       197G   21G  167G  11% /
devtmpfs        3.8G     0  3.8G   0% /dev
tmpfs           3.8G  8.0M  3.8G   1% /dev/shm
tmpfs           3.8G   34M  3.8G   1% /run
tmpfs           3.8G     0  3.8G   0% /sys/fs/cgroup
tmpfs           775M   28K  775M   1% /run/user/1001
tmpfs           775M     0  775M   0% /run/user/0
/dev/sdb1        15G  8.0K   15G   1% /run/media/chen/CentOS 7 x86_64

[chen@localhost ~]$ sudo mkfs.vfat /dev/sdb1
mkfs.fat 3.0.20 (12 Jun 2013)

```

### 安装 CentOS

1. 查看 U 盘对应分区
1. 修改引用位置
1. 安装

```
//直接选择安装的话可能会出现找不到位置的错误并且进入以下模式：

dracut:/#

// 查看 U 盘对应分区
dracut:/# ls /dev/

接下来找到以sd开头的，有的是sdd，有的是sdb，还有的是sdc，不过貌似一般都是sdb，这里你可以看到以sdb开头的文件有两个分别是sdb和sdb1，sdb1代表的就是你的U盘了，这里是假设你看到的是sdb1,我的显示是sdb4,记住你看到的是哪个。
然后重新开机

// 修改引用位置
进入安装页时按 TAB 键，下面会出现
append initrd=initrd.img root=live:CDLABEL=Fedora\x2017\x20i386 quiet
这时你把后面改为  append initrd=initrd.img repo=hd:/dev/sdb1:/ quiet

//安装
```

### 删除内核升级后多余的启动项

1. 查看当前使用的内核版本
1. 查看已安装的内核版本
1. 使用包管理器卸载版本较旧的内核版本

```
[chen@localhost ~]$ uname -a
Linux localhost.localdomain 3.10.0-514.10.2.el7.x86_64 #1 SMP Fri Mar 3 00:04:05 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux

[chen@localhost ~]$ rpm -qa | sort | grep kernel
abrt-addon-kerneloops-2.1.11-45.el7.centos.x86_64
kernel-3.10.0-514.10.2.el7.x86_64
kernel-3.10.0-514.el7.x86_64
kernel-devel-3.10.0-514.10.2.el7.x86_64
kernel-devel-3.10.0-514.el7.x86_64
kernel-headers-3.10.0-514.10.2.el7.x86_64
kernel-tools-3.10.0-514.10.2.el7.x86_64
kernel-tools-libs-3.10.0-514.10.2.el7.x86_64

[root@localhost chen]# yum remove kernel-*3.10.0-514.el7.x86_64
Loaded plugins: fastestmirror, langpacks
Resolving Dependencies
--> Running transaction check
---> Package kernel.x86_64 0:3.10.0-514.el7 will be erased
---> Package kernel-devel.x86_64 0:3.10.0-514.el7 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package             Arch          Version               Repository        Size
================================================================================
Removing:
 kernel              x86_64        3.10.0-514.el7        @anaconda        148 M
 kernel-devel        x86_64        3.10.0-514.el7        @anaconda         34 M

Transaction Summary
================================================================================
Remove  2 Packages

Installed size: 182 M
Is this ok [y/N]: y
...
```

### 恢复 Windows 启动项

1. 添加 EPEL 源
1. 安装 ntfs-3g
1. 重新生成引导项

```
# yum install epel-release
# yum install ntfs-3g
# grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 配置字体

1. 下载喜欢的字体
1. 将 xxx.ttf 文件移动到`～/.fonts/` 下
1. 添加字体缓存 `fc-cache ~/.fonts/`
