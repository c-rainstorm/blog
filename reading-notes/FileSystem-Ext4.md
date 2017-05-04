# Ext4 文件系统简要总结

最近在看操作系统的文件系统这块，感觉只有理论还是不行，所以就选了一种文件系统作一下拓展。

选择 Ext4 的原因是： 他是 Linux 的主流文件系统 `ext*` 家族中最新的版本

主要内容：

1. Ext4 的特性
1. Ext4 磁盘布局
- Tips： 本文内容主要是 [Ext4 文档](https://ext4.wiki.kernel.org/index.php/Main_Page) 的概括翻译加上自己的解释说明，所以英文水平还行的话建议直接看英文文档。

本文假设读者对文件系统有基本的了解，否则请先查阅相关资料（《现代操作系统》《操作系统概念(影印版)》）

---

<!-- TOC -->

- [Ext4 文件系统简要总结](#ext4-文件系统简要总结)
    - [主要特性](#主要特性)
        - [兼容性(Compatibility)](#兼容性compatibility)
        - [更大的文件系统和文件大小(Bigger File System and File Sizes )](#更大的文件系统和文件大小bigger-file-system-and-file-sizes-)
        - [拓展子目录数量(Sub directory scalability )](#拓展子目录数量sub-directory-scalability-)
        - [拓展块大小(Extents)](#拓展块大小extents)
        - [多块分配(Multiblock allocation)](#多块分配multiblock-allocation)
        - [延迟分配(Delayed allocation)](#延迟分配delayed-allocation)
        - [快速文件系统检测(Fast fsck)](#快速文件系统检测fast-fsck)
        - [日志校验(Journal checksumming)](#日志校验journal-checksumming)
        - [禁用日志模式("No Journaling" mode )](#禁用日志模式no-journaling-mode-)
        - [在线磁盘整理(Online defragmentation )](#在线磁盘整理online-defragmentation-)
        - [Inode 相关特性(Inode-related features)](#inode-相关特性inode-related-features)
        - [磁盘预分配(Persistent preallocation )](#磁盘预分配persistent-preallocation-)
        - [屏障默认开启(Barriers on by default)](#屏障默认开启barriers-on-by-default)
    - [Ext4 磁盘布局](#ext4-磁盘布局)
        - [概述(Overview)](#概述overview)
            - [块(Blocks)](#块blocks)
            - [布局(layout)](#布局layout)
            - [弹性块组(Flexible Block Groups)](#弹性块组flexible-block-groups)
            - [元块组(Meta Block Groups)](#元块组meta-block-groups)
        - [超级块(The Super Block)](#超级块the-super-block)
        - [块组描述符(Block Group Descriptors)](#块组描述符block-group-descriptors)
    - [参考](#参考)

<!-- /TOC -->

---

## 主要特性

### 兼容性(Compatibility)

Ext4 兼容 Ext3，升级只需运行一些命令即可，不需要变动磁盘格式，升级中不会影响已有的数据。

### 更大的文件系统和文件大小(Bigger File System and File Sizes )

|File System| Max FS Size | Max File Size|block addressing bits|
|:---:|:---:|:---:|:---:|
|Ext3|16TB|2TB|32|
|Ext4|1EB|16TB|48|

Tips:
- 1 EB = 1024 *  1024 TB
- block size： 4 bytes

### 拓展子目录数量(Sub directory scalability )

在一个目录中，

- Ext3 支持 32000 个子目录
- Ext4 支持 64000 个子目录

### 拓展块大小(Extents)

Ext3 为每个文件维护一个 block 表，用于保存这个文件在磁盘上的块号，因为一个 block 只有 4kb 的大小，所以对于一个大文件来说的话，需要维护的 block 表占用的空间就比较可观了，删除和截断等操作的效率也就比较低。

Ext4 使用 extents 代替 block。
extents 由多个连续的 block 组成。能够有效的减少需要维护的 block 表的长度，进而提高在文件上操作的效率

### 多块分配(Multiblock allocation)

当需要将新数据写入磁盘上时，需要块分配器决定将数据写入哪一个空闲块中。

但是 Ext3 写入的时候，每次只分配一个 block(4kb), 也就是说如果要写入 100 Mb 的数据时会调用块分配器 25600 词，效率很低，分配器也无法作优化。

Ext4 使用多块分配器，根据需要，一次调用分配多个块(一个 extents)

### 延迟分配(Delayed allocation)

传统的文件系统尽可能早的分配磁盘 blocks，当进程调用 `write()` 时，文件系统立即为其分配 block，即使数据并没有立即写入磁盘(在缓存中临时存放)。这种方式的缺点是当进程持续向文件写入数据，文件增长时需要分配另外的 block 来存放新增的数据，块分配器无法对分配方式作优化。

而延迟分配策略解决了这个问题，当进程调用 `write()` 时它并不立即分配 blocks，直到数据从缓存写入磁盘时进行分配。写入磁盘时，数据基本就不再增长了，此时使用多块分配器为该文件分配多个 extents

### 快速文件系统检测(Fast fsck)

文件系统检测是一项非常慢的操作，特别是检查文件系统中所有的 inode 节点。

Ext4 跳过未使用的 inode 节点来加快检测速度，根据已使用的 inode 节点的数量不同，性能会提升 2 到 20 倍。

### 日志校验(Journal checksumming)

使用校验和来判断一个日志块是否已失效。

Ext3 使用两阶段(执行 + commit/rollback)提交来保证正确性。

Ext4 使用一阶段提交 + 日志校验来保证正确性，性能提升大约 20%。

### 禁用日志模式("No Journaling" mode )

日志确保了磁盘上内容变动时文件系统的完整性，但是却带来了少量的额外开销(日志记录)。

通过禁用日志特性可以获得少量的性能提升

### 在线磁盘整理(Online defragmentation )

这个特性正在开发中，会包含到之后的版本中。

通过使用延迟分配、extents 和 多块分配能够有效减少磁盘碎片，但是文件内容变动(可以需要另外的 block 来存放数据，这个 block 可能会离原来的地方比较远，从而引发一次额外的寻道)也会带来很多碎片，磁盘碎片整理可以将文件尽可能的重分配到连续的 block 中，从而减少磁盘碎片，提高访问效率。

### Inode 相关特性(Inode-related features)

1. 更大的 inodes：Ext3 支持配置 inode 大小，默认为 128 bytes，Ext4 默认为 256 bytes。增加了一些额外的域(比如纳秒级的 timestamps 或 inode 版本)，剩余的空间用来保存拓展属性。这种方式可以使访问这些属性的速度更快，从而提高应用程序的性能。
1. 当创建目录时，直接为其创建几个保留的 inode 节点，当在这个目录中创建新文件时，就可以直接使用这些保留的 inode 节点，从而提高文件创建和删除的效率。
1. Ext3 的时间属性是秒级的，Ext4 的时间属性是纳秒级的。

### 磁盘预分配(Persistent preallocation )

这个特性允许应用程序预先分配磁盘空间，应用通知文件系统预先分配空间，文件系统预先分配需要的块和数据结构，直到应用程序向该空间写数据前，该空间中是没有数据的。

### 屏障默认开启(Barriers on by default)

这个选项改善了文件系统的完整性，但损失了一些性能。

文件系统在写入数据之前必须先将事务信息记录到日志，然后根据顺序写入，但是这种方式效率比较低。现代的驱动有很大的内部缓存并且为了得到更好的性能会进行操作重排序，所以在写入数据之前，文件系统必须先显式的指示磁盘加载所有的日志。

内核的 阻塞 I/O 子系统使用屏障来实现，即在加载日志时进行阻塞，其他数据 I/O 操作就无法再进行了。


## Ext4 磁盘布局

### 概述(Overview)

![](../res/fs-layout.jpg)

- MBR 为主引导记录用来引导计算机。在计算机启动时，BIOS 读入并执行 MBR，MBR 作的第一件事就是确定活动分区(这对应于双系统的计算机开机时选择启动项，单系统的直接就能确定了所以就不需要选择)，读入活动分区的引导块(Boot block)，引导块再加载该分区中的操作系统。
- 分区表(Partition table)用来记录每个分区的起始和结束地址，表中的一个分区为活动分区。
- 引导块(boot block)用于转载该分区的操作系统。
- 超级块(super block)包含文件系统的所有关键参数。

Ext4 文件系统被分为一系列的块组(block groups)。为了减少磁盘碎片带来的性能问题，减少寻道时间，块分配器总是尝试将每个文件的所有块分配到同一个块组中。块组的大小由超级块中的属性来定义，通常是 `8 * block_size_in_bytes` 对于块大小为 4Kb 的磁盘来说，每一个组可以包含 32768 个 block，总共 128Mb。块组的数目为磁盘空间除以块组大小。

在 Ext4 中处日志以外的数据都是以小端法存储的。

#### 块(Blocks)

一个块由偶数个扇区(sector)组成，一个块组又由多个块组成。块大小在 build file system 时被指定，通常是 4Kb。如果块大小大于内存页的大小，装载文件系统时可能会遇到问题。

#### 布局(layout)

![](../res/fs-ext4-layout.png)

为了允许安装启动扇区和其他用途，块组 0 的开头 1024 bytes 未被使用。超级块从 第 1024 bytes 开始，通常在第 0 块，但如果块大小是 1Kb，则从第 1 块开始。

Ext4 驱动主要使用在第 0 块组的超级块和组描述符(Group Descriptors)来工作。为了防止磁盘开头部分崩溃而无法访问文件系统，会将超级块和组描述符的副本保存在多个块组中，没有保存副本的块组以数据块位图 (Data Block Bitmap) 开头。

inode table 的位置保存在 `grp.bg_inode_table_*` 中，这是足够容纳 `sb.s_inodes_per_group * sb.s_inode_size` bytes 的连续块。

#### 弹性块组(Flexible Block Groups)

从 Ext4 开始，有了一个叫弹性块组(flex_bg)的新特性。在一个弹性块组中几个普通块组连在一起组成一个逻辑块组，几个普通块组的元数据(inode 节点表和位图等等)都存放在一起，然后弹性块组中剩余的空间都用来存放数据。这种方法可以加快加载速度并且可以使大文件连续存放。组成弹性块组的普通块组的数量为 `2 ^ sb.s_log_groups_per_flex`

#### 元块组(Meta Block Groups)

//TODO


### 超级块(The Super Block)

超级块记录了文件系统的关键参数。

`sparse_super` 特性标志被设置后，冗余的超级块和组描述符的副本只保留在组号为 0 或组号为 3，5，7 的倍数的块组中，否则冗余副本会保存在所有的块组中。

超级块占用 1Kb 空间。

详细的参数描述请查阅官方文档，不再赘述。

### 块组描述符(Block Group Descriptors)

//TODO

## 参考

1. [File system](https://en.wikipedia.org/wiki/File_system#Linux)
1. [Ext4 Howto](https://ext4.wiki.kernel.org/index.php/Ext4_Howto)
1. [Ext4 Disk Layout](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Overview)
