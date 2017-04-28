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

### 多块分配(Multiblock allocation)

当需要将新数据写入磁盘上时，需要块分配器决定将数据写入哪一个空闲块中。

但是 Ext3 写入的时候，每次只分配一个 block(4kb), 也就是说如果要写入 100 Mb 的数据时会调用块分配器 25600 词，效率很低，分配器也无法作优化。

Ext4 使用多块分配器，一次调用分配多个块


## 参考

1. [File system](https://en.wikipedia.org/wiki/File_system#Linux)
1. [Ext4 Howto](https://ext4.wiki.kernel.org/index.php/Ext4_Howto)
