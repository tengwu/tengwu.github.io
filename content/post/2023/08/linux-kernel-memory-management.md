+++
title = "Linux内存管理"
date = 2023-08-17T13:19:00+08:00
lastmod = 2023-08-21T13:20:38+08:00
tags = ["Kernel", [",", "OS"]]
categories = ["Kernel"]
draft = false
toc = true
+++

## 引子(废话 {#引子废话}

`Linux kernel` 的作用之一是管理计算机的硬件资源,其中一个硬件资源就是内存, `Linux kernel` 中管理内存的一部分被称为内存管理子系统.

从大的方面来说,内存管理子系统需要管理物理内存和虚拟内存两部分.物理内存是真实存在的,虚拟内存只存在于逻辑上,我们平时所说的虚拟内存空间就是指虚拟内存在逻辑上的布局,例如,虚拟内存的第一个内存单元到第100个内存单元的状态(也就是说这些内存单元存储了什么内容).对于应用程序员来说,只需要了解虚拟内存空间的知识就够了.Linux内存管理的作用就是对应用程序员隐藏比较复杂的底层内存管理逻辑,提供给他们一个易于操作的虚拟内存空间.

本文的目的是探索Linux内存管理子系统的工作方式,中间会涉及到具体的代码.理想情况下,本文会涉及内存管理子系统从Linux加电启动到普通程序运行的方方面面,所以这个战线会很长,本文也会持续很长时间才会完结(希望能顺利完结.


## 概述 {#概述}

如上所述,内存管理子系统需要管理物理内存资源和虚拟内存资源.

在 `Linux kernel v5.10` 中,物理内存管理包含了 `memblock` 和 `buddy system` 两大机制(另外还有基于 `buddy system` 的小块内存管理技术),虚拟内存管理分为用户空间内存管理和内核空间内存管理,用户空间内存管理主要包含 `malloc`, `mmap`,内核空间内存管理包含了 `kmalloc` 和 `vmalloc`.除此之外, `Linux kernel` 还实现了一些内存管理的优化机制,比如按需分页, `fork` 的写时复制, `swap` 机制,这些也会被本文所涵盖.


## 物理内存管理 {#物理内存管理}

`memblock` 和 `buddy system` 是物理内存管理的两大手段,其中, `memblock` **主要** 用在启动阶段(在启动后的阶段, `memblock` 的一部分也会被用到,请看后文), `buddy system` 主要用在运行阶段.

本文主要从初始化,内存分配,内存回收三方面来讨论这两大机制.初始化是指获取物理内存的大小,物理地址等信息,并且使用一个数据结构来维护这些信息;内存分配和回收也都是在维护这一数据结构.一旦物理内存管理初始化完毕,内核会 **主要** 通过相关接口来获取物理内存(有可能在一些需要优化的场景不会使用这些接口).


### memblock结构 {#memblock结构}

`memblock` 的结构比较简单,可以先通过一张图来感受一下.

{{< figure src="/images/memblock_struct.png" >}}

即 `memblock` 通过两个 `memblock_region` 数组来维护早期物理内存的状态.其中, `memocy->regions` 维护的是未使用内存的信息, `reserved->regions` 维护的是已使用内存的信息.

更详细地说 `memblock_region` 通过 `regions(struct memblock_region类型)` 这个有序数组来维护内存区域,针对每个region维护其起始地址 `base` 和其大小 `size` .内核保证 `regions` 里region之间没有相交的部分.并且在进行memblock相关操作时,会将相邻的region进行合并.

以下是具体的数据结构:

```c
struct memblock {
  bool bottom_up;  /* is bottom up direction? */
  phys_addr_t current_limit;
  struct memblock_type memory;
  struct memblock_type reserved;
};

struct memblock_type {
  unsigned long cnt;
  unsigned long max;
  phys_addr_t total_size;
  struct memblock_region *regions;
  char *name;
};

struct memblock_region {
  phys_addr_t base;
  phys_addr_t size;
  enum memblock_flags flags;
#ifdef CONFIG_NEED_MULTIPLE_NODES
  int nid;
#endif
};
```

`memblock` 的定义在 `mm/memblock.c` 中.

```c
struct memblock memblock __initdata_memblock = {
  .memory.regions		= memblock_memory_init_regions,
  .memory.cnt		= 1,	/* empty dummy entry */
  .memory.max		= INIT_MEMBLOCK_REGIONS,
  .memory.name		= "memory",

  .reserved.regions	= memblock_reserved_init_regions,
  .reserved.cnt		= 1,	/* empty dummy entry */
  .reserved.max		= INIT_MEMBLOCK_RESERVED_REGIONS,
  .reserved.name		= "reserved",

  .bottom_up		= false,
  .current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```


### 初始化阶段(启动阶段) {#初始化阶段--启动阶段}

在ARM64中,内核通过解析[设备树](https://docs.kernel.org/devicetree/usage-model.html)来获取设备的位置和大小,其中就包含了内存.

根据设备树文档的描述:

> Typically the early_init_dt_scan_chosen() helper
> is used to parse the chosen node including kernel parameters,
> early_init_dt_scan_root() to initialize the DT address space model,
> and early_init_dt_scan_memory() to determine the size and
> location of usable RAM.

即,内核通过 `early_init_dt_scan_memory()` 函数来获取内存信息,来看一下这个函数.

```c
int __init early_init_dt_scan_memory(unsigned long node, const char *uname,
                                     int depth, void *data)
{
  // ...
  reg = of_get_flat_dt_prop(node, "linux,usable-memory", &l);
  // ...
  while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
    // 遍历device tree的每个和内存相关的节点
    u64 base, size;

    base = dt_mem_next_cell(dt_root_addr_cells, &reg);
    size = dt_mem_next_cell(dt_root_size_cells, &reg);

    // ...

    early_init_dt_add_memory_arch(base, size);

    // ...
  }
}
```

在 `early_init_dt_add_memory_arch()` 函数里,通过 `memblock_add()` 将从设备树中解析到的内存添加到 `memblock.memory` 结构里.


### 运行阶段(启动后的阶段) {#运行阶段--启动后的阶段}


## 虚拟内存管理 {#虚拟内存管理}


### 内核空间 {#内核空间}


### 用户空间 {#用户空间}
