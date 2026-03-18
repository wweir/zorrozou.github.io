# Linux软RAID与LVM性能优化指南——从内核源码看实现与调优

版权声明：
本文章内容在非商业使用前提下可无需授权任意转载、发布。
转载、发布请务必注明作者和其微博、微信公众号地址，以便读者询问问题和甄误反馈，共同进步。

微博：https://weibo.com/orroz/
博客：https://zorrozou.github.io/
微信公众号：Linux系统技术

---

## 前言

在生产环境中，软RAID和LVM是Linux存储方案的重要组成部分。很多人知道"RAID5性能不好"、"LVM条带化可以提升性能"，却说不清楚为什么，也不知道该从哪些维度进行调优。本文从内核源码出发，结合`drivers/md/`目录下的实现，系统地梳理Device Mapper（DM）框架、LVM条带化、软RAID5/6的IO路径和性能关键点，帮助读者真正理解这些机制并做出有依据的优化决策。

本文分析的内核版本为tkernel5（基于Linux 6.6）。

---

## 一、架构概览：Device Mapper是一切的基础

### 1.1 DM框架：虚拟块设备的"路由器"

Linux的LVM和软RAID（md）虽然面向用户的工具链不同（lvm2 vs mdadm），但在内核IO路径上都依赖同一个框架：**Device Mapper（DM）**。

DM的核心思想可以用一句话概括：**将一个虚拟块设备的IO请求，通过"映射表（mapping table）"转换后分发给一个或多个底层真实块设备。** 如果把IO路径比作网络数据包的路由，那DM就是一个"路由器"——它根据映射表（相当于路由表）决定每个IO请求应该被转发到哪个底层设备。

这个转换过程由各种**target**（目标模块）实现，每种target定义了不同的映射策略。DM框架本身不关心具体的映射逻辑，它只负责：接收bio → 查映射表 → 调用target的map回调 → 将转换后的bio提交给底层设备。

### 1.2 DM的核心数据结构

理解DM的工作方式，需要先看几个关键的数据结构：

```c
/* dm.c / dm-core.h */
struct mapped_device {
    struct dm_table __rcu   *map;           /* 当前活跃的映射表（RCU保护） */
    struct gendisk          *disk;          /* 对应的虚拟块设备 */
    struct request_queue    *queue;         /* 请求队列 */
    ...
};

struct dm_table {
    struct dm_target        *targets;       /* target数组 */
    unsigned int            num_targets;    /* target数量 */
    ...
};

struct dm_target {
    struct dm_table         *table;
    struct target_type      *type;          /* target类型（linear/striped/raid...） */
    sector_t                begin;          /* 该target在虚拟设备上的起始扇区 */
    sector_t                len;            /* 该target管理的扇区数 */
    void                    *private;       /* target的私有数据 */
    ...
};
```

`mapped_device`是DM的顶层结构，代表一个虚拟块设备（如`/dev/dm-0`）。它持有一个`dm_table`，表中包含一个或多个`dm_target`——每个target负责虚拟设备上的一段地址空间。这种设计允许一个虚拟设备由多种不同类型的target拼接而成。

用一张图来表示它们的关系：

```
                        mapped_device (/dev/dm-0)
                              │
                              ▼
                          dm_table
                       ┌──────┴──────┐
                       ▼             ▼
                  dm_target[0]   dm_target[1]
                  type: linear   type: striped
                  begin: 0       begin: 1000000
                  len: 1000000   len: 2000000
                       │              │
                       ▼              ▼
                   /dev/sda      /dev/sdb + /dev/sdc
                  (直接1:1映射)   (数据在两盘间交替分布)
```

每个`dm_target`还有一个关键字段`type`，指向`target_type`结构体——这就是各种target模块注册的"方法表"：

```c
/* 每种target模块都要注册一个target_type */
struct target_type {
    const char *name;                       /* target名称，如"linear"/"striped" */
    ...
    dm_map_fn map;                          /* 核心：IO映射回调 */
    dm_io_hints_fn io_hints;                /* 向上层报告最优IO参数 */
    dm_ctr_fn ctr;                          /* 构造函数 */
    dm_dtr_fn dtr;                          /* 析构函数 */
    ...
};
```

其中`map`是最核心的回调——DM框架对每个bio都会调用它来完成地址转换。不同target的性能差异，本质上就是这个`map`函数的实现复杂度差异。

### 1.3 target类型详解：从简单到复杂

DM框架下有多种target类型，它们的IO路径复杂度和性能特征差异很大。下面从最简单的开始逐一分析：

#### linear——最简单的1:1映射

`linear`是最基本的target，对应LVM的普通逻辑卷。它的map函数只做一件事：**把虚拟设备的扇区地址加上一个偏移量，直接转换成底层设备的扇区地址。**

```c
/* dm-linear.c */
static int linear_map(struct dm_target *ti, struct bio *bio)
{
    struct linear_c *lc = ti->private;
    bio_set_dev(bio, lc->dev->bdev);               /* 重定向到底层设备 */
    bio->bi_iter.bi_sector = linear_map_sector(ti,  /* 扇区地址偏移 */
                                bio->bi_iter.bi_sector);
    return DM_MAPIO_REMAPPED;                       /* 告诉DM：bio已映射完成 */
}

static sector_t linear_map_sector(struct dm_target *ti, sector_t bi_sector)
{
    struct linear_c *lc = ti->private;
    return lc->start + dm_target_offset(ti, bi_sector);  /* 就是一次加法 */
}
```

整个map函数只有一次加法运算和一次设备指针赋值，没有任何锁、没有内存分配、没有额外IO。因此linear target的性能开销可以忽略不计，IO延迟几乎等于底层设备的原始延迟。

#### striped——将IO分散到多个设备

`striped`对应LVM条带卷，它把虚拟设备的地址空间按chunk大小切分，轮流映射到多个底层设备上：

```c
/* dm-stripe.c */
static int stripe_map(struct dm_target *ti, struct bio *bio)
{
    struct stripe_c *sc = ti->private;
    uint32_t stripe;
    sector_t result;
    /* 根据bio的扇区地址，计算应该落在哪个底层设备的哪个位置 */
    stripe_map_sector(sc, bio->bi_iter.bi_sector, &stripe, &result);
    bio_set_dev(bio, sc->stripe[stripe].dev->bdev);
    bio->bi_iter.bi_sector = result + sc->stripe[stripe].physical_start;
    return DM_MAPIO_REMAPPED;
}
```

比linear多了一步：需要通过**取模运算**计算目标设备编号和设备内偏移。但这仍然是纯数学计算，没有额外IO，性能开销极小。详细的计算过程在第二章分析。

#### raid——封装md RAID，复杂度陡增

`raid` target是DM对Linux md RAID子系统的封装。与前两种target有本质区别：**它的map函数不再是简单的地址转换，而是要处理奇偶校验计算、stripe cache管理、RMW/RCW策略选择等复杂逻辑。**

```c
/* dm-raid.c: raid target的注册 */
static struct target_type raid_target = {
    .name    = "raid",
    .map     = raid_map,
    .ctr     = raid_ctr,
    ...
};

static int raid_map(struct dm_target *ti, struct bio *bio)
{
    struct raid_set *rs = ti->private;
    struct mddev *mddev = &rs->md;
    /* 不是简单映射，而是将bio交给md子系统处理 */
    md_handle_request(mddev, bio);
    return DM_MAPIO_SUBMITTED;  /* 注意：返回SUBMITTED而不是REMAPPED */
}
```

注意返回值的差异：linear和striped返回`DM_MAPIO_REMAPPED`，表示"我已经改好了bio的目标地址，DM框架你帮我提交就行"；而raid返回`DM_MAPIO_SUBMITTED`，表示"bio我自己处理了，DM框架不用管了"。这个差异意味着raid的IO路径完全脱离了DM的简单映射流程，进入了md子系统的复杂处理逻辑。

#### thin——精简置备，按需分配

`thin` target实现了精简置备（thin provisioning）——虚拟设备可以声明一个很大的容量，但实际的物理存储空间在写入时才按需分配。

```c
/* dm-thin.c */
static int thin_bio_map(struct dm_target *ti, struct bio *bio)
{
    struct thin_c *tc = ti->private;
    struct dm_thin_lookup_result result;
    /* 查询映射树：这个虚拟块是否已经分配了物理空间？ */
    r = dm_thin_find_block(tc->td, block, 0, &result);
    if (r == -ENODATA) {
        /* 未分配 → 需要从池中分配新的物理块 */
        /* 这里涉及元数据更新、可能的写时复制等复杂操作 */
        ...
    }
    ...
}
```

thin的map函数需要查询一棵B+树来判断虚拟块是否已映射到物理块。如果是首次写入，还需要从存储池中分配物理块并更新元数据。这使得thin target的写延迟会有偶发的毛刺（首次分配时），但稳态下的读性能接近linear。

#### cache——分层缓存

`cache` target在慢速设备（如HDD）前面放一个快速设备（如SSD）作为缓存层，类似CPU的L1/L2缓存思想：

```c
/* dm-cache-target.c */
static int cache_map(struct dm_target *ti, struct bio *bio)
{
    struct cache *cache = ti->private;
    /* 查询bio对应的块是否在缓存中 */
    /* 命中 → 从SSD读取/写入 */
    /* 未命中 → 从HDD读取，同时可能触发缓存填充 */
    /* 脏数据回写、缓存替换策略等 */
    ...
}
```

cache的map函数涉及缓存查找、命中/未命中处理、数据迁移和替换策略，是所有target中逻辑最复杂的之一。

### 1.4 各target的IO路径对比

把上述所有target的IO路径画在一张图中，可以清楚地看到它们的复杂度差异：

```
                    用户进程 write()
                         │
                         ▼
                    VFS / 页缓存
                         │
                         ▼
                   通用块层 submit_bio()
                         │
                         ▼
                   dm_submit_bio()
                ┌────────┼────────────────────────────┐
                │        │                            │
          ┌─────┴─────┐  │  ┌──────────────────────┐  │  ┌────────────────┐
          │  linear    │  │  │   striped            │  │  │  raid          │
          │            │  │  │                      │  │  │                │
          │ 地址+偏移  │  │  │ 取模→选盘→算偏移    │  │  │ md_handle_     │
          │ (1次加法)  │  │  │ (取模+加法)          │  │  │ request()      │
          │            │  │  │                      │  │  │     │          │
          │   ○        │  │  │   ○                  │  │  │     ▼          │
          └───┼────────┘  │  └───┼──────────────────┘  │  │ stripe cache  │
              │           │      │                     │  │ 查找/分配      │
              │           │      │                     │  │     │          │
              │           │      │                     │  │     ▼          │
              │           │      │                     │  │ RMW/RCW/FSW   │
              │           │      │                     │  │ 策略选择       │
              │           │      │                     │  │     │          │
              │           │      │                     │  │     ▼          │
              │           │      │                     │  │ 奇偶校验计算  │
              │           │      │                     │  │ (XOR/P+Q)     │
              │           │      │                     │  │     │          │
              │           │      │                     │  │     ▼          │
              │           │      │                     │  │ 多个bio       │
              │           │      │                     │  │ (数据+校验)   │
              └──────┐    │   ┌──┘                     │  └──┬─────────┬──┘
                     │    │   │                        │     │         │
                     ▼    │   ▼                        │     ▼         ▼
                   1个bio │ 1个bio                     │  多个bio(数据+P)
                     │    │   │                        │     │         │
                     ▼    ▼   ▼                        ▼     ▼         ▼
                   ┌───────────────────────────────────────────────────────┐
                   │              底层块设备驱动（NVMe/virtio/SCSI）       │
                   └───────────────────────────────────────────────────────┘

  IO路径复杂度：linear < striped <<< raid
  map函数开销：  ~10ns    ~20ns       ~数百ns到数μs（不含实际磁盘IO）
```

这张图清楚地展示了关键差异：

- **linear和striped**：map函数只做地址转换，输入1个bio输出1个bio，开销是纯CPU计算（纳秒级）
- **raid**：map函数触发整个md子系统的处理流程，可能产生额外的读IO（RMW）、计算开销（XOR）、以及多个输出bio（数据+校验），开销是CPU计算+可能的额外磁盘IO

### 1.5 DM框架的处理流程详解

接下来深入看`dm_submit_bio()`的完整处理流程。一个bio从进入DM到离开，经历了以下步骤：

```c
/* dm.c: DM的bio入口——完整流程 */
static void dm_submit_bio(struct bio *bio)
{
    struct mapped_device *md = bio->bi_bdev->bd_disk->private_data;
    struct dm_table *map;

    /* Step 1: 获取映射表（RCU读锁，极低开销） */
    map = dm_get_live_table_fast(md);
    if (unlikely(!map)) {
        bio_io_error(bio);
        goto out;
    }

    /* Step 2: 根据bio的扇区地址，找到对应的dm_target */
    dm_split_and_process_bio(md, map, bio);
    /* 内部会调用：
     *   ti = dm_table_find_target(map, bio->bi_iter.bi_sector);
     *   → 二分查找或直接索引，找到管理该扇区的target
     */

out:
    dm_put_live_table_fast(md);  /* 释放RCU读锁 */
}
```

Step 2中的`dm_split_and_process_bio()`会做一件很重要的事：**如果bio跨越了target边界或超过了target能处理的最大大小，就把bio拆分成多个小bio，分别交给对应的target处理。**

```c
/* dm.c */
static void dm_split_and_process_bio(struct mapped_device *md,
                                      struct dm_table *map, struct bio *bio)
{
    struct dm_target *ti;
    sector_t len;

    ti = dm_table_find_target(map, bio->bi_iter.bi_sector);

    /* 如果bio超过了这个target管理的范围，需要拆分 */
    len = min(max_io_len(ti, bio), bio->bi_iter.bi_size >> SECTOR_SHIFT);
    if (len < bio_sectors(bio)) {
        /* 拆分bio：前半部分交给当前target，后半部分递归处理 */
        struct bio *split = bio_split(bio, len, GFP_NOIO, &md->queue->bio_split);
        bio_chain(split, bio);
        submit_bio_noacct(bio);  /* 后半部分重新进入DM */
        bio = split;
    }

    /* 调用target的map回调 */
    __map_bio(bio);
}
```

bio拆分机制保证了每个target只需要处理自己地址范围内的IO，不需要关心边界问题。但拆分本身有开销：分配新的bio结构、建立bio chain、可能增加IO完成时的回调层级。因此，**IO大小对齐target边界可以避免不必要的拆分**。

最后是`__map_bio()`——真正调用target的map函数：

```c
/* dm.c */
static void __map_bio(struct bio *bio)
{
    struct dm_target_io *tio = clone_to_tio(bio);
    struct dm_target *ti = tio->ti;
    int r;

    /* 调用target的map回调 */
    r = ti->type->map(ti, bio);

    switch (r) {
    case DM_MAPIO_REMAPPED:
        /* linear/striped等简单target：bio已改好目标，直接提交 */
        submit_bio_noacct(bio);
        break;

    case DM_MAPIO_SUBMITTED:
        /* raid/thin等复杂target：bio已被target接管，不需要DM提交 */
        break;

    case DM_MAPIO_REQUEUE:
        /* target请求重新排队（资源不足等） */
        ...
        break;

    case DM_MAPIO_KILL:
        /* target拒绝该IO */
        bio_io_error(bio);
        break;
    }
}
```

三种返回值对应了三种不同的IO路径模式：

| 返回值 | 含义 | 典型target | DM框架后续动作 |
|-------|------|-----------|-------------|
| `DM_MAPIO_REMAPPED` | target已完成地址转换 | linear, striped | DM提交bio到底层设备 |
| `DM_MAPIO_SUBMITTED` | target自己处理bio | raid, thin | DM不再管这个bio |
| `DM_MAPIO_REQUEUE` | 请求重试 | thin（池满时） | 重新入队 |

### 1.6 DM框架的性能开销分析

DM框架本身的性能开销主要来自以下几个环节：

**1) RCU映射表查找**

`dm_get_live_table_fast()`使用`rcu_read_lock()`保护映射表的访问，这在现代内核中几乎零开销（只是禁止抢占），不需要任何原子操作或内存屏障。映射表在运行时极少变化（只有`lvresize`/`pvmove`等管理操作才会更新），所以RCU的"读多写极少"优势被充分发挥。

**2) target查找**

`dm_table_find_target()`需要根据bio的扇区地址找到对应的target。如果映射表只有一个target（最常见的情况，如单个LV），这就是一次直接索引；如果有多个target，则是一次二分查找。对于典型的单LV场景，这一步的开销可以忽略。

**3) bio克隆**

DM需要为每个bio创建一个`dm_target_io`封装，这涉及slab分配器的分配操作。在高IOPS场景下（百万级），这可能成为可观测的开销。内核通过per-CPU的`bioset`缓存来缓解这个问题。

**4) bio拆分**

如果bio跨越target边界或超过`max_io_len`，需要拆分。拆分的开销类似bio克隆，加上bio chain的建立。合理配置chunk大小和IO对齐可以最小化拆分次数。

综合来看，对于linear和striped这种简单target，DM框架的开销主要是bio克隆的slab分配，大约在**数十纳秒**量级。在云盘（毫秒级延迟）和普通SSD（百微秒级延迟）场景下完全可以忽略。但在使用Intel Optane这类超低延迟设备（微秒级）且IOPS达到百万级的极端场景下，DM框架的开销可能占到总延迟的1%~5%。

### 1.7 各target性能特征总结

| target | map复杂度 | 额外IO | 写放大 | 数据冗余 | 典型延迟开销 | 适用场景 |
|--------|----------|--------|-------|---------|------------|---------|
| `linear` | O(1)加法 | 无 | 1x | 无 | ~10ns | LVM基本卷，通用存储 |
| `striped` | O(1)取模+加法 | 无 | 1x | 无 | ~20ns | 需要高吞吐的临时数据 |
| `raid`(RAID5) | 复杂 | RMW需读 | 最差4x | 有（单盘容错） | ~数百ns+额外IO | 需要冗余的持久数据 |
| `thin` | B+树查找 | 首写需分配 | 1x（稳态） | 取决于底层 | ~100ns（命中）| 多租户、快照场景 |
| `cache` | 缓存查找 | 未命中需读源 | 回写时2x | 取决于底层 | ~50ns（命中）| SSD缓存HDD |

其中"写放大"一列需要特别关注：
- **linear/striped**：写入N字节，底层设备实际写入N字节，写放大为1x
- **RAID5 RMW**：写入N字节，需要先读旧数据+旧P，再写新数据+新P，写放大最差可达4x
- **RAID5 FSW**：写入一整条stripe，只需额外写一个校验块，写放大约为`(N+1)/N`（接近1x）

这也是为什么后续章节要花大量篇幅分析RAID5的FSW触发条件和优化方法——**减少写放大是提升RAID5写性能的核心。**

理解了DM框架和各target的特性，我们就可以深入分析每种target的具体实现了。接下来先看最直观的LVM条带化。

---

## 二、LVM条带化（dm-stripe）的性能分析

上一章我们了解了DM框架的整体架构，知道了`striped` target的map函数只做简单的取模+加法运算，IO路径极短。但这只是"怎么做"——条带化到底是什么？为什么需要它？它在什么情况下有用，什么情况下反而有害？本章将彻底讲清楚这些问题。

### 2.1 什么是条带化？从一个性能瓶颈说起

假设你有4块磁盘，要存储一个大文件。最简单的做法是用LVM的`linear`模式——先写满第一块盘，再写第二块，依次类推：

```
linear（线性拼接）：

  逻辑地址空间：  [    0 ~ 999G     |  1000G ~ 1999G  |  2000G ~ 2999G  | 3000G ~ 3999G  ]
                        │                  │                  │                  │
                        ▼                  ▼                  ▼                  ▼
  物理设备：         Disk 0             Disk 1             Disk 2             Disk 3

  问题：访问 0~999G 的数据时，只有 Disk 0 在工作，其他 3 块盘完全空闲！
```

这就是linear模式的致命缺陷——**同一时刻只有一块盘在服务IO**。如果你的应用在集中读写某个区域的数据（这是大多数真实负载的特征），那其他盘就是白白浪费的。

条带化（Striping）的思路正好相反：**把数据切成小块（chunk），像发牌一样轮流分配到各个磁盘**。这样，无论访问哪个区域的数据，多块盘都能同时工作：

```
striped（条带化）：chunk_size = 128KB

  逻辑地址空间：  [ chunk0 | chunk1 | chunk2 | chunk3 | chunk4 | chunk5 | chunk6 | chunk7 | ... ]
                     │        │        │        │        │        │        │        │
                     ▼        ▼        ▼        ▼        ▼        ▼        ▼        ▼
  物理设备：      Disk 0   Disk 1   Disk 2   Disk 3   Disk 0   Disk 1   Disk 2   Disk 3  ...

                 ┌────────┬────────┬────────┬────────┐
  Stripe 0  →   │chunk 0 │chunk 1 │chunk 2 │chunk 3 │  ← 4个chunk = 1个完整条带（512KB）
                 ├────────┼────────┼────────┼────────┤
  Stripe 1  →   │chunk 4 │chunk 5 │chunk 6 │chunk 7 │
                 ├────────┼────────┼────────┼────────┤
  Stripe 2  →   │chunk 8 │chunk 9 │chunk10 │chunk11 │
                 └────────┴────────┴────────┴────────┘
                  Disk 0    Disk 1   Disk 2   Disk 3
```

这里有两个关键概念需要明确：

- **chunk（块）**：条带化的最小分配单元。一个chunk是连续分配到同一块磁盘的最小数据量。上图中每个chunk是128KB。
- **stripe（条带）**：所有磁盘上同一行的chunk组合。上图中一个stripe = 4个chunk = 512KB。

chunk_size的选择直接影响性能特征，我们后面会详细分析。

### 2.2 条带化为什么能提升性能？

理解了数据布局，性能提升的原理就很直观了。我们从三个维度来分析：

#### 2.2.1 吞吐量：N块盘的带宽叠加

顺序读一个大文件时，linear模式下数据全在一块盘上，吞吐量受限于单盘带宽。条带化后，一次大IO会被自动拆分到N块盘并行读取：

```
顺序读 1MB，chunk=128KB，4盘条带：

  linear模式：
    时间线 ──────────────────────────────────────────────────▶
    Disk 0: [===== 读 1MB ======================================]
    Disk 1: [                    空闲                           ]
    Disk 2: [                    空闲                           ]
    Disk 3: [                    空闲                           ]
    总耗时: T = 1MB ÷ 单盘带宽

  striped模式（4盘）：
    时间线 ──────────────────▶
    Disk 0: [= 读 256KB（chunk 0,4） =]
    Disk 1: [= 读 256KB（chunk 1,5） =]
    Disk 2: [= 读 256KB（chunk 2,6） =]
    Disk 3: [= 读 256KB（chunk 3,7） =]
    总耗时: T = 256KB ÷ 单盘带宽 ≈ 原来的 1/4

  理论吞吐量提升：N倍（N = 条带数）
```

这就是条带化最直观的优势——**带宽叠加**。4块盘条带化，理论顺序吞吐量是单盘的4倍。

#### 2.2.2 IOPS：随机IO的负载分散

对于随机4K小IO，条带化的优势在于**负载分散**。由于chunk是轮流分配的，随机访问的IO会均匀分散到所有磁盘：

```
随机读 4K IO（假设IO均匀分布），4盘条带：

  linear模式（数据集中在 Disk 0）：
    Disk 0: [IO][IO][IO][IO][IO][IO][IO][IO]   ← 所有IO都打到一块盘
    Disk 1: [                              ]
    Disk 2: [                              ]
    Disk 3: [                              ]
    IOPS ≈ 单盘IOPS

  striped模式：
    Disk 0: [IO]    [IO]    [IO]    [IO]        ← 每块盘只承担 1/4 的IO
    Disk 1:    [IO]    [IO]    [IO]    [IO]
    Disk 2: [IO]    [IO]    [IO]    [IO]
    Disk 3:    [IO]    [IO]    [IO]    [IO]
    IOPS ≈ 4 × 单盘IOPS

  理论IOPS提升：N倍（N = 条带数）
```

在我们前面第一章的实验数据中已经验证了这一点：4盘条带的随机读IOPS（2541）约为2盘条带（1217）的2倍，8盘条带（5289）约为4盘条带的2倍——几乎完美的线性扩展。

#### 2.2.3 延迟：对单次IO没有帮助

但条带化有一个常见的误解需要澄清：**它不能降低单次IO的延迟**。

```
单次随机读 4K：

  linear模式：
    请求 → Disk X → 响应     延迟 = T_disk

  striped模式：
    请求 → stripe_map() → Disk Y → 响应     延迟 = T_map + T_disk
                           │
                     只是换了一块盘而已

  T_map ≈ 20ns，T_disk ≈ 数百μs（云盘）或数十μs（NVMe SSD）
  条带化不会让单次IO变快——只是换了目标盘
```

条带化提升的是**聚合性能**（吞吐量和IOPS），而不是**单次延迟**。如果你的应用是单线程、串行IO，那条带化的收益很有限——因为同一时刻只有一个IO在飞行，只有一块盘在工作。

这引出了一个重要的适用条件：**条带化需要足够的IO并发度才能发挥优势**。这个"足够"是多少？我们后面会给出测试方案。

### 2.3 chunk_size的选择：条带化调优的核心

chunk_size是条带化中最关键的参数，它直接影响IO的分布方式和性能特征。选错了chunk_size，条带化不仅不能提升性能，甚至可能带来负面影响。

#### 2.3.1 chunk_size太小会怎样？

如果chunk_size比IO大小还小，一次IO就会跨越多个chunk，被拆分到多个磁盘：

```
chunk_size = 4KB，单次顺序读 128KB：

  逻辑IO：   [============= 128KB =============]
  拆分后：   [4K][4K][4K][4K][4K][4K]...[4K][4K]   ← 32个4K IO
              │   │   │   │   │   │       │   │
              D0  D1  D2  D3  D0  D1      D0  D1

  问题1：32个小IO代替1个大IO，磁盘寻道开销爆炸（HDD场景）
  问题2：每个小IO都要走一次块层提交流程，CPU开销增大
  问题3：磁盘队列被大量小IO填满，排队延迟上升
```

#### 2.3.2 chunk_size太大会怎样？

如果chunk_size远大于IO大小，小IO就无法分散到多块盘，失去了条带化的意义：

```
chunk_size = 1MB，随机写 4KB：

  写请求 A（逻辑地址 0x100）  → 落入 chunk 0 → Disk 0
  写请求 B（逻辑地址 0x200）  → 落入 chunk 0 → Disk 0
  写请求 C（逻辑地址 0x300）  → 落入 chunk 0 → Disk 0
  ...
  写请求 X（逻辑地址 0xFFFFF）→ 落入 chunk 0 → Disk 0
  写请求 Y（逻辑地址 0x100000）→ 终于到 chunk 1 → Disk 1 ！

  如果数据集中在一个 1MB 的范围内，所有IO都打到同一块盘
  条带化完全退化为 linear
```

#### 2.3.3 最优chunk_size的选择原则

最优的chunk_size取决于你的IO模式。核心原则是：

**对于顺序IO**：chunk_size应该足够大，让一次IO尽量落在一块盘上，避免跨盘拆分开销。同时又不能太大，否则多块盘无法同时工作。一个好的平衡点是让chunk_size等于或略大于典型的IO大小。

**对于随机小IO**：chunk_size应该较小，这样不同地址的IO更容易分散到不同盘上。但不能小于IO大小，否则会引起IO拆分。

```
chunk_size 选择速查表：

  工作负载               推荐chunk_size       原因
  ─────────────────────────────────────────────────────────────
  数据库（8K/16K随机IO） 64KB ~ 128KB         足够小以分散IO，
                                              不会拆分单次IO

  大文件顺序读写         256KB ~ 1MB          减少跨盘拆分，
                                              利用磁盘顺序带宽

  混合负载               128KB ~ 256KB        折中选择

  OLTP（4K随机为主）     32KB ~ 64KB          最大化IO分散度

  流媒体/视频处理        512KB ~ 1MB          大块顺序IO为主
```

一个实用的经验法则：**chunk_size = 典型IO大小 × (2~4)**。对于大多数通用场景，128KB是一个安全的默认值。

> **建议1：chunk_size应匹配应用IO模式，而非盲目使用默认值。**
>
> LVM默认的chunk大小为64KB（`lvcreate -i 4 -I 64k`）。对于大多数数据库和通用服务器负载，这是合理的。但如果你的应用以大块顺序IO为主（如视频处理、数据仓库），应增大到256KB甚至更大。可以用`iostat -x 1`观察应用的avgqu-sz和avgrq-sz来判断典型IO大小。

### 2.4 stripe_map_sector：内核如何实现地址映射

理解了条带化的概念，让我们深入看内核的具体实现。LVM条带卷对应的target是`striped`，实现在`drivers/md/dm-stripe.c`。其核心数据结构是：

```c
struct stripe_c {
    uint32_t stripes;       /* 条带数量（底层设备数） */
    int stripes_shift;      /* stripes的log2，用于快速取模 */
    sector_t stripe_width;  /* 每个底层设备分配的扇区数 */
    uint32_t chunk_size;    /* chunk大小，单位：扇区 */
    int chunk_size_shift;   /* chunk_size的log2，用于快速计算 */
    ...
    struct stripe stripe[]; /* 每个底层设备的描述 */
};
```

当一个bio到来时，`stripe_map()`通过`stripe_map_sector()`完成扇区到底层设备的映射：

```c
static void stripe_map_sector(struct stripe_c *sc, sector_t sector,
                               uint32_t *stripe, sector_t *result)
{
    sector_t chunk = dm_target_offset(sc->ti, sector);
    sector_t chunk_offset;

    /* 计算chunk内偏移 */
    if (sc->chunk_size_shift < 0)
        chunk_offset = sector_div(chunk, sc->chunk_size);
    else {
        chunk_offset = chunk & (sc->chunk_size - 1); /* 位运算，快速路径 */
        chunk >>= sc->chunk_size_shift;
    }

    /* 计算目标stripe编号 */
    if (sc->stripes_shift < 0)
        *stripe = sector_div(chunk, sc->stripes);
    else {
        *stripe = chunk & (sc->stripes - 1);         /* 位运算，快速路径 */
        chunk >>= sc->stripes_shift;
    }
    ...
}
```

这段代码的核心逻辑用一句话概括就是：**给定逻辑扇区号，通过两次"除法+取模"运算，算出该扇区应该映射到哪块物理磁盘的哪个位置**。让我们用图来展示这个过程：

```
stripe_map_sector 的地址映射过程（4盘条带，chunk=128KB=256扇区）：

  输入：逻辑扇区号 = 1234

  Step 1: 计算chunk编号和chunk内偏移
          chunk编号   = 1234 ÷ 256 = 4      （第5个chunk）
          chunk内偏移 = 1234 % 256 = 210     （chunk内的第210个扇区）

  Step 2: 计算目标磁盘和磁盘内chunk编号
          目标磁盘      = 4 % 4 = 0          （映射到 Disk 0）
          磁盘内chunk号 = 4 ÷ 4 = 1          （Disk 0 上的第2个chunk）

  Step 3: 计算物理扇区号
          物理扇区 = 磁盘内chunk号 × chunk_size + chunk内偏移
                   = 1 × 256 + 210 = 466

  输出：Disk 0，物理扇区 466

  验证（在数据布局图中定位）：
                 Disk 0     Disk 1     Disk 2     Disk 3
                ┌──────────┬──────────┬──────────┬──────────┐
  Stripe 0  →  │ chunk 0  │ chunk 1  │ chunk 2  │ chunk 3  │
                ├──────────┼──────────┼──────────┼──────────┤
  Stripe 1  →  │ chunk 4  │ chunk 5  │ chunk 6  │ chunk 7  │  ← 逻辑chunk 4
                │  ▲       │          │          │          │     在Disk 0
                │  └─扇区466          │          │          │     的第2个chunk内
                └──────────┴──────────┴──────────┴──────────┘
```

这里有一个很有意思的优化细节：**当chunk_size和stripes数量都是2的幂时，内核用位运算（`&`和`>>`）代替除法（`sector_div`）来完成取模和除法运算**。这就是`chunk_size_shift`和`stripes_shift`这两个字段的意义——在条带化配置时预计算好移位量，在IO热路径中就不需要做昂贵的除法了。

那么实际场景中，位运算 vs 除法的性能差异到底有多大？

#### 实验验证：位运算 vs 除法到底差多少？

上面的分析是从源码角度给出的优化建议，那么问题来了——这个优化在实际IO场景中，到底能带来多大的性能差异？是显著差异还是理论上的差异？我们需要用数据说话。

为了回答这个问题，我们设计了一组对照实验。由于标准的`lvcreate`命令不支持创建非2的幂的chunk大小，我们通过`dmsetup`直接创建striped target来绕过这个限制，这样就能精确控制stripes和chunk_size两个变量，分别测试2的幂与非2的幂配置的性能差异。

测试环境为腾讯云CVM：

- CPU：AMD EPYC 7K83 32核
- 内存：121GB
- 磁盘：12块10.9TB云硬盘（virtio-blk）
- 内核：4.18.0-348.7.1.el8_5.x86_64

fio测试参数：runtime=60s，size=10G，每种负载跑3轮取平均值。随机IO使用bs=4k/iodepth=128/numjobs=4，顺序IO使用bs=1M/iodepth=16/numjobs=1，全部direct IO。

**实验A：固定4盘4条带，只改变chunk大小**

这组实验的关键是：条带数固定为4（2的幂），只在chunk这一个维度上对比2的幂与非2的幂的差异。

| 测试组 | chunk大小 | chunk是否2^n | 随机读4K IOPS | 随机写4K IOPS | 顺序读1M (MB/s) |
|--------|----------|:----------:|-------------:|-------------:|---------------:|
| A1 | 64K | ✅ | 2,484 | 2,732 | 939 |
| A2 | 96K | ❌ | 2,541 | 2,781 | 937 |
| A3 | 128K | ✅ | 2,539 | 2,809 | 940 |
| A4 | 192K | ❌ | 2,544 | 2,745 | 939 |
| A5 | 256K | ✅ | 2,535 | 2,629 | 939 |
| A6 | 384K | ❌ | 2,545 | 2,721 | 937 |

我们可以看到，2的幂（A1/A3/A5）和非2的幂（A2/A4/A6）之间的IOPS差异在1%以内，完全在正常波动范围。顺序读带宽也几乎一致。chunk是否为2的幂，在实际IO场景中**没有产生可观测的性能差异**。

**实验B和C：变条带数，观察真正的性能影响因素**

接下来，我们在4盘和8盘两种配置下改变条带数，看看条带数对性能的影响：

| 测试组 | 磁盘数 | 条带数 | 条带2^n | 随机读4K IOPS | 顺序读1M (MB/s) |
|--------|-------|-------|:------:|-------------:|---------------:|
| B1 | 4 | 2 | ✅ | 1,217 | 490 |
| B2 | 4 | 3 | ❌ | 1,893 | 718 |
| B3 | 4 | 4 | ✅ | 2,541 | 938 |
| C1 | 8 | 4 | ✅ | 2,540 | 938 |
| C2 | 8 | 5 | ❌ | 3,299 | 1,165 |
| C3 | 8 | 6 | ❌ | 3,932 | 1,378 |
| C4 | 8 | 7 | ❌ | 4,601 | 1,589 |
| C5 | 8 | 8 | ✅ | 5,289 | 1,810 |

这组数据很有意思。从B1到B3，随机读IOPS从1217涨到2541，翻了一倍——这是条带并行度从2增加到4的效果。从C1到C5同样如此，IOPS几乎与条带数成线性关系。而更关键的是，**非2的幂的条带数（如5、6、7条带）的性能完全符合线性增长预期，没有因为走了除法路径而出现任何性能下降**。

**实验D：四种代码路径的直接对比**

最后，我们设计了四种组合来独立验证两个维度的影响。这是最有说服力的一组对比：

| 测试组 | 条带数 | chunk | stripes 2^n | chunk 2^n | 代码路径 | 随机读4K IOPS | 顺序读1M (MB/s) | CPU sys% |
|--------|-------|-------|:----------:|:--------:|---------|-------------:|---------------:|--------:|
| D1 | 4 | 128K | ✅ | ✅ | 双位运算 | 2,540 | 938 | 13.76 |
| D2 | 4 | 192K | ✅ | ❌ | chunk除法 | 2,540 | 937 | 10.75 |
| D3 | 3 | 128K | ❌ | ✅ | stripes除法 | 1,888 | 717 | 10.57 |
| D4 | 3 | 192K | ❌ | ❌ | 双除法 | 1,899 | 718 | 8.73 |

我们来看D1和D2的对比：同样是4条带，D1用128K（2的幂，走位运算），D2用192K（非2的幂，走除法），两者的IOPS完全一致——2540 vs 2540。再看D3和D4：同样是3条带，D3的128K和D4的192K，IOPS也几乎没有区别——1888 vs 1899。

那D1和D3之间25%的IOPS差距是怎么来的？答案很简单：不是位运算 vs 除法的差异，而是4条带 vs 3条带的**并行度差异**。4块盘并行处理IO，自然比3块盘快。

还有一个有趣的发现：CPU sys%方面，双位运算的D1（13.76%）反而比双除法的D4（8.73%）更高。这是因为4条带的并行度更高、IOPS更大，CPU需要处理更多的IO完成中断和上下文切换，而不是说位运算比除法更耗CPU。

**实验结论**

在实际IO场景中，`stripe_map_sector`中的地址计算只是整个IO路径上微不足道的一环。一次磁盘IO的延迟通常在百微秒到毫秒级别，而位运算与整数除法的差异是纳秒级的——差了三到四个数量级。在我们的全部测试中，相同条带数下，2的幂与非2的幂配置在IOPS、吞吐量和延迟上的差异均在1%以内，属于正常测量波动。

> **对建议1的修正：使用2的幂主要是一种良好的工程实践（代码路径更优、配置更规范），但在实际IO性能上的影响微乎其微。真正影响条带化性能的关键因素是条带数（并行度）——应尽量使用所有可用磁盘作为条带成员来最大化IO并行度，以及chunk大小与应用IO模式的匹配关系。**

需要注意的是，本文的测试是在云盘（virtio-blk）环境下进行的，单盘IO延迟本身就在毫秒级。对于使用NVMe SSD或Intel Optane这类超低延迟设备的场景，单次IO延迟可以低至微秒级，此时CPU路径上的开销占比会更大一些。如果你的环境是这类设备且IOPS非常高（百万级），建议自行验证。但即便如此，2的幂的建议仍然是合理的工程实践——它不会带来坏处，而且内核代码确实为此做了优化。

### 2.5 io_hints：告诉上层最优IO大小

`dm-stripe.c`实现了`stripe_io_hints()`，这个函数的作用是向上层（文件系统、IO调度器）通告条带设备的最优IO参数：

```c
static void stripe_io_hints(struct dm_target *ti,
                             struct queue_limits *limits)
{
    struct stripe_c *sc = ti->private;
    unsigned int chunk_size = sc->chunk_size << SECTOR_SHIFT;

    blk_limits_io_min(limits, chunk_size);          /* 最小IO = 1个chunk */
    blk_limits_io_opt(limits, chunk_size * sc->stripes); /* 最优IO = 1个完整条带 */
}
```

这里设置了两个关键参数：

- **`io_min`（最小IO大小）= chunk_size**：告诉上层，小于这个大小的IO无法被进一步优化。文件系统的allocator应避免产生小于一个chunk的IO。
- **`io_opt`（最优IO大小）= chunk_size × 条带数**：告诉上层，每次IO的最优大小是一个完整stripe。如果IO大小等于这个值，每个底层设备都能被均匀地访问一次，并发度最高，效率最优。

这些参数会通过sysfs暴露给用户空间：

```bash
# 查看条带设备的IO参数
$ cat /sys/block/dm-0/queue/minimum_io_size    # io_min
131072                                          # = 128KB (1个chunk)
$ cat /sys/block/dm-0/queue/optimal_io_size     # io_opt
524288                                          # = 512KB (4 × 128KB，完整条带)
```

文件系统在格式化时会读取这些参数来自动对齐。例如ext4的`mke2fs`会自动计算stride和stripe_width。但自动探测并不总是可靠的（特别是在多层DM堆叠的场景下），所以手动指定是更保险的做法。

> **建议2：创建文件系统时，`stride`和`stripe_width`参数应匹配LVM条带配置。**
>
> 以ext4为例，如果创建了4块盘、chunk=128KB的条带卷：
> ```bash
> # chunk_size=128K, stripes=4, stripe_width=512K
> mkfs.ext4 -E stride=32,stripe_width=128 /dev/vg/lv_stripe
> # stride = chunk_size / block_size = 128K / 4K = 32
> # stripe_width = stride * 条带数 = 32 * 4 = 128
> ```
> 这样ext4在分配block group和做预读时就会按条带对齐，避免跨条带边界的IO拆分。XFS也有类似的参数：
> ```bash
> mkfs.xfs -d su=128k,sw=4 /dev/vg/lv_stripe
> # su = stripe unit = chunk_size
> # sw = stripe width = 条带数
> ```

### 2.6 条带化的适用场景与局限性

#### 2.6.1 适合使用条带化的场景

由于`dm-stripe`本身没有奇偶校验计算，IO路径极短（纯地址映射，不产生额外IO），适合以下场景：

**场景一：临时数据/缓存——追求极致吞吐，不需要冗余**

```bash
# 4块盘条带化，用于数据库临时表空间
lvcreate -i 4 -I 128k -L 200G -n lv_temp vg_data
mkfs.xfs -d su=128k,sw=4 /dev/vg_data/lv_temp
mount /dev/vg_data/lv_temp /tmp/db_temp
```

临时数据丢了就丢了——数据库重启后会重建。条带化让临时表的排序、join操作速度提升N倍。

**场景二：分布式存储底层——上层已有冗余保障**

如果你在Ceph OSD或HDFS DataNode之上已经有了副本机制（3副本或纠删码），底层的单机存储就不需要RAID冗余了。这时条带化是最好的选择——获得多盘的性能，冗余交给上层。

**场景三：已有硬件RAID保护的场景**

如果底层已经有硬件RAID控制器提供了磁盘冗余，在LVM层再做软RAID是多余的。条带化可以把多个硬件RAID组的带宽叠加起来。

#### 2.6.2 不适合使用条带化的场景

**场景一：单线程、低并发IO**

前面分析过，条带化的优势来自并行度。如果应用是单线程串行IO（例如某些备份程序），同一时刻只有一个IO在飞行，那条带化几乎没有收益：

```
单线程串行IO：
  时间线 ─────────────────────────────────────────────▶
  线程:    [submit IO1][等...][submit IO2][等...][submit IO3]
  Disk 0:  [IO1]                          [IO3]
  Disk 1:              [IO2]
  Disk 2:                                         
  Disk 3:

  每个时刻只有1块盘在工作——条带化的4块盘中有3块在空闲
  性能 ≈ 单盘性能（条带化白费了）
```

**场景二：需要数据保护的持久化存储**

条带化没有任何冗余——任何一块盘故障，整个卷的数据全部丢失。而且由于数据分散在所有盘上，一块盘坏意味着每个文件都可能损坏。这比linear模式还糟糕（linear模式下一块盘坏只影响该盘上的数据）。

```
磁盘故障的影响：

  linear模式（4盘拼接）：
    Disk 0: [数据A]    Disk 1: [数据B]    Disk 2: [数据C☠]    Disk 3: [数据D]
    Disk 2 坏了 → 只丢失数据C，其他完好

  striped模式（4盘条带）：
    Disk 0: [A₀][B₀][C₀]    Disk 1: [A₁][B₁][C₁]    Disk 2: [A₂☠][B₂☠][C₂☠]    Disk 3: [A₃][B₃][C₃]
    Disk 2 坏了 → 文件A、B、C的每1/4数据都丢了 → 所有文件全部损坏！
```

如果需要数据保护，请使用RAID1/5/6（承受性能代价换取冗余），这正是我们第三章要深入分析的内容。

#### 2.6.3 条带化 vs RAID5：性能与安全的取舍

这两种方案的核心取舍可以用一张表总结：

| 特性 | dm-stripe（条带化） | dm-raid（RAID5） |
|------|:------------------:|:----------------:|
| **写路径** | 纯地址映射（~20ns） | RMW/FSW（额外读+XOR计算） |
| **写放大** | 1x | RMW最差4x，FSW约1.2x |
| **额外IO** | 无 | 需读旧数据/旧校验 |
| **CPU开销** | 几乎为零 | XOR/P+Q校验计算 |
| **随机写性能** | ≈ N × 单盘 | ≈ (0.25~0.5) × N × 单盘 |
| **顺序写性能** | ≈ N × 单盘 | ≈ (0.5~0.8) × N × 单盘 |
| **数据冗余** | ❌ 无（一盘坏全丢） | ✅ 容忍1盘故障（RAID5） |
| **容量利用率** | 100% | (N-1)/N |
| **适用场景** | 临时数据、上层有冗余 | 需要持久化保护的数据 |

**性能差距在随机写上最为显著**。条带化的随机写就是简单地分发到目标盘，而RAID5的随机写需要走RMW路径——每个写IO产生3-4个额外IO（读旧数据、读旧校验、写新数据、写新校验）。这就是为什么我们在第三章要花大量篇幅分析如何触发FSW来减少RAID5的写放大。

### 2.7 条带化的性能测试方案

要验证条带化的性能特征，需要从多个维度进行测试。以下是完整的测试方案：

#### 方法1：条带数扩展性测试——验证线性扩展

```bash
#!/bin/bash
# 测试不同条带数的性能扩展性
# 预期结果：IOPS和吞吐量应与条带数成线性关系

DISKS="/dev/vdb /dev/vdc /dev/vdd /dev/vde /dev/vdf /dev/vdg /dev/vdh /dev/vdi"

for n in 1 2 4 6 8; do
    # 取前n个磁盘
    selected=$(echo $DISKS | tr ' ' '\n' | head -n $n | tr '\n' ' ')
    
    # 创建条带卷
    pvcreate $selected 2>/dev/null
    vgcreate vg_test $selected 2>/dev/null
    lvcreate -i $n -I 128k -l 100%FREE -n lv_test vg_test
    
    echo "=== 条带数: $n ==="
    
    # 随机读4K
    fio --name=randread --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=4k --iodepth=128 \
        --numjobs=4 --runtime=60 --rw=randread \
        --group_reporting --output-format=json \
        > stripe_${n}_randread.json
    
    # 顺序读1M
    fio --name=seqread --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=1M --iodepth=16 \
        --numjobs=1 --runtime=60 --rw=read \
        --group_reporting --output-format=json \
        > stripe_${n}_seqread.json
    
    # 随机写4K
    fio --name=randwrite --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=4k --iodepth=128 \
        --numjobs=4 --runtime=60 --rw=randwrite \
        --group_reporting --output-format=json \
        > stripe_${n}_randwrite.json
    
    # 清理
    lvremove -f vg_test/lv_test
    vgremove -f vg_test
    pvremove $selected 2>/dev/null
done
```

#### 方法2：chunk_size影响测试——找到最优chunk

```bash
#!/bin/bash
# 固定4盘条带，改变chunk大小，观察对不同IO模式的影响
# 预期：小IO随机负载对chunk不敏感，大IO顺序负载对chunk敏感

for chunk in 32k 64k 128k 256k 512k 1024k; do
    lvcreate -i 4 -I $chunk -l 100%FREE -n lv_test vg_test
    
    echo "=== chunk_size: $chunk ==="
    
    # 4K随机读：应该对chunk不敏感
    fio --name=rand4k --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=4k --iodepth=128 \
        --numjobs=4 --runtime=60 --rw=randread \
        --group_reporting --output-format=json \
        > chunk_${chunk}_rand4k.json
    
    # 128K顺序读：chunk越大越有利（减少跨盘拆分）
    fio --name=seq128k --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=128k --iodepth=16 \
        --numjobs=1 --runtime=60 --rw=read \
        --group_reporting --output-format=json \
        > chunk_${chunk}_seq128k.json
    
    # 混合读写（70/30）：模拟真实负载
    fio --name=mixed --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=8k --iodepth=64 \
        --numjobs=4 --runtime=60 --rw=randrw --rwmixread=70 \
        --group_reporting --output-format=json \
        > chunk_${chunk}_mixed.json
    
    lvremove -f vg_test/lv_test
done
```

#### 方法3：IO并发度影响测试——验证条带化需要并发

```bash
#!/bin/bash
# 固定4盘条带 vs 单盘，改变iodepth，观察并发度的影响
# 预期：低并发时条带化优势不明显，高并发时优势显著

for depth in 1 2 4 8 16 32 64 128; do
    echo "=== iodepth: $depth ==="
    
    # 单盘
    fio --name=single --filename=/dev/vdb \
        --ioengine=libaio --direct=1 --bs=4k --iodepth=$depth \
        --numjobs=1 --runtime=60 --rw=randread \
        --group_reporting --output-format=json \
        > depth_${depth}_single.json
    
    # 4盘条带
    fio --name=stripe4 --filename=/dev/vg_test/lv_test \
        --ioengine=libaio --direct=1 --bs=4k --iodepth=$depth \
        --numjobs=1 --runtime=60 --rw=randread \
        --group_reporting --output-format=json \
        > depth_${depth}_stripe4.json
done

# 分析：绘制 IOPS vs iodepth 曲线
# 单盘在某个iodepth后饱和，条带卷的饱和点应该是单盘的N倍
```

#### 方法4：条带化 vs RAID5 写性能对比

```bash
#!/bin/bash
# 条带化 vs RAID5 在不同写负载下的性能对比
# 预期：随机写差距最大，顺序大IO写差距较小（RAID5走FSW路径）

# 准备：分别创建4盘条带和4盘RAID5（3数据+1校验）
# lvcreate -i 4 -I 128k ...   → stripe卷
# lvcreate --type raid5 -i 3 -I 128k ...  → RAID5卷

for rw in randwrite write; do
    for bs in 4k 64k 256k 1M; do
        echo "=== $rw bs=$bs ==="
        
        # 条带化
        fio --name=stripe --filename=/dev/vg_test/lv_stripe \
            --ioengine=libaio --direct=1 --bs=$bs --iodepth=64 \
            --numjobs=4 --runtime=60 --rw=$rw \
            --group_reporting --output-format=json \
            > vs_stripe_${rw}_${bs}.json
        
        # RAID5
        fio --name=raid5 --filename=/dev/vg_test/lv_raid5 \
            --ioengine=libaio --direct=1 --bs=$bs --iodepth=64 \
            --numjobs=4 --runtime=60 --rw=$rw \
            --group_reporting --output-format=json \
            > vs_raid5_${rw}_${bs}.json
    done
done

# 关注点：
# 1. randwrite 4k：RAID5走RMW，差距最大（可能4-8倍）
# 2. write 1M：RAID5可能走FSW，差距缩小（可能1.5-2倍）
# 3. 对比CPU使用率：RAID5的XOR计算会消耗额外CPU
```

理解了条带化的原理、优势和局限性，我们就可以进入更复杂的领域了——下一章将深入分析软RAID5/6，看看当引入奇偶校验和数据冗余后，IO路径会变得多么复杂，以及有哪些方法可以优化它的写性能。

---

## 三、软RAID5/6的性能深度分析

软RAID是Linux存储性能优化中最复杂的话题，因为它涉及奇偶校验计算、stripe cache管理、RMW/RCW两种写策略的选择，以及多线程并发控制。

### 3.1 Stripe Cache：一切性能问题的核心

理解软RAID5性能，必须先理解**stripe cache**。

RAID5的写操作有一个"写洞"问题：如果一次写操作不是整条stripe的完整写（Full Stripe Write），就必须先读出老数据（Read-Modify-Write，简称RMW），或者读出其他数据块计算新奇偶校验（Reconstruct Write，简称RCW），然后才能写入新数据和更新后的奇偶校验。无论哪种方式，都引入了额外的读IO，这正是RAID5写性能差的根本原因。

为了减少这种额外的读操作，内核实现了**stripe cache**。其核心数据结构是`stripe_head`：

```c
/* raid5.h中的stripe_head，描述一个stripe的缓存状态 */
struct stripe_head {
    struct hlist_node   hash;       /* 在hash表中的位置 */
    struct list_head    lru;        /* LRU链表 */
    struct r5conf       *raid_conf;
    short               generation;
    sector_t            sector;     /* 这个stripe对应的起始扇区 */
    short               pd_idx;     /* P盘（奇偶校验盘）编号 */
    short               qd_idx;     /* Q盘（RAID6第二奇偶校验盘）编号 */
    ...
    atomic_t            count;      /* 引用计数 */
    spinlock_t          stripe_lock;
    ...
    struct r5dev        dev[];      /* 每个成员盘的状态和数据buffer */
};
```

每个`stripe_head`缓存一个完整stripe的数据，包含所有成员盘（数据盘+奇偶校验盘）对应的内存页面。当多个写请求命中同一个stripe时，它们可以在内存中合并，最终只需一次完整条带写，避免了RMW开销。

stripe cache的大小通过`/sys/block/md0/md/stripe_cache_size`控制，对应的内核实现：

```c
/* raid5.c */
static ssize_t
raid5_show_stripe_cache_size(struct mddev *mddev, char *page)
{
    ...
    return sprintf(page, "%d\n", conf->max_nr_stripes);
}

static ssize_t
raid5_store_stripe_cache_size(struct mddev *mddev, const char *page, size_t len)
{
    ...
    /* 每个stripe_head需要为每个成员盘分配一个内存页 */
    /* 对于N盘RAID5，每个stripe_head消耗N个page */
}
```

**stripe cache大小的影响：**
- 太小：并发写请求无法合并，大量RMW操作，写放大严重
- 太大：内存占用过大，可能引发内存压力，同时影响stripe查找效率

> **建议3：根据工作负载调整stripe_cache_size。**
>
> 默认值通常为256个stripe。对于写密集型工作负载（如数据库WAL、日志写入），建议增大到4096甚至8192：
> ```bash
> echo 4096 > /sys/block/md0/md/stripe_cache_size
> ```
> 注意这会消耗内存：每个stripe_head约消耗`成员盘数 × PAGE_SIZE`的内存。5盘RAID5，8192个stripe需要约160MB内存。

### 3.2 Full Stripe Write：RAID5的理想写路径

当一次写操作覆盖整个stripe的所有数据块时，就是**Full Stripe Write（FSW）**。内核中的判断逻辑：

```c
static bool is_full_stripe_write(struct stripe_head *sh)
{
    /* 所有数据盘都被覆盖 = 完整条带写 */
    BUG_ON(sh->overwrite_disks > (sh->disks - sh->raid_conf->max_degraded));
    return sh->overwrite_disks == (sh->disks - sh->raid_conf->max_degraded);
}
```

FSW的特点是：不需要读取任何旧数据，直接计算新的奇偶校验值，然后写入所有盘。这是RAID5的最优写路径，写性能接近无RAID的情况。

内核使用`pending_full_writes`计数器跟踪正在进行中的FSW，并在调度器中给予优先处理：

```c
/* raid5.c中的调度逻辑 */
if (atomic_read(&conf->pending_full_writes) == 0))
    /* 没有FSW时，可以处理其他类型的写请求 */
```

> **建议4：尽量让应用程序以stripe大小的倍数进行写操作，触发Full Stripe Write。**
>
> stripe大小 = chunk_size × 数据盘数。例如5盘RAID5（4数据+1奇偶），chunk=512KB，则stripe大小=2MB。数据库的写入单位（checkpoint、redo log块）若能对齐到stripe大小，将极大提升RAID5写性能。

### 3.3 RMW vs RCW：两种非全条带写策略

上一节讲到，Full Stripe Write是RAID5的理想写路径——所有数据盘都被覆盖，直接算出新奇偶校验写入即可。但现实中，大部分写操作只修改stripe中的一部分数据块。这时RAID5面临一个核心难题：**怎样用最少的额外IO算出新的奇偶校验？**

内核提供了两种策略：RMW（Read-Modify-Write）和RCW（Reconstruct Write）。它们解决的是同一个问题，但"读什么、算什么"完全不同。

#### 3.3.1 RMW：读旧值，差分更新奇偶校验

RMW的核心思想是**差分计算**——利用XOR的自消性质，只需要知道"变化了什么"就能更新奇偶校验：

```
XOR的关键性质：A XOR A = 0（任何值与自身异或得零）

因此：
  new_parity = old_parity XOR old_data XOR new_data

证明：
  设原始数据为 D0, D1, D2, D3，校验为 P = D0 XOR D1 XOR D2 XOR D3
  现在只修改 D1 → D1'，新校验 P' = D0 XOR D1' XOR D2 XOR D3
  
  P XOR D1 XOR D1' 
  = (D0 XOR D1 XOR D2 XOR D3) XOR D1 XOR D1'
  = D0 XOR (D1 XOR D1) XOR D2 XOR D3 XOR D1'    ← D1 XOR D1 = 0
  = D0 XOR D1' XOR D2 XOR D3
  = P'   ✓
```

所以RMW只需要读2样东西：**被修改块的旧值**（old_data）和**旧校验**（old_parity）。

用一个具体例子展示完整流程（5盘RAID5，修改D1）：

```
【RMW流程：5盘RAID5，只修改D1】

  Disk0    Disk1    Disk2    Disk3    Disk4(P)
  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
  │ D0 │  │ D1 │  │ D2 │  │ D3 │  │ P  │
  └────┘  └────┘  └────┘  └────┘  └────┘

  Step 1: 读取旧值
  ─────────────────────────────────────────
          ┌────┐                    ┌────┐
    READ  │oldD1│              READ │oldP│
          └──┬─┘                    └──┬─┘
             │                         │
  Step 2: XOR差分计算（CPU操作，不涉及磁盘IO）
  ─────────────────────────────────────────
     new_P = old_P  XOR  old_D1  XOR  new_D1
             ~~~~~~~~~~~~~~~~~~~~~~~~~~~
             只需3个值就能算出新校验

  Step 3: 写入新值
  ─────────────────────────────────────────
          ┌────┐                    ┌────┐
   WRITE  │newD1│             WRITE │newP│
          └────┘                    └────┘

  总代价：2次读 + 2次写 = 4次磁盘IO
  不管stripe有多少数据盘，修改1块的代价始终是4次IO
```

RMW在内核中的实现分为两个阶段：**prexor**（预异或）和**reconstruct**（重建）。prexor阶段先用旧数据"减去"变化前的值，reconstruct阶段再"加上"变化后的值：

```c
/* raid5.c: ops_run_prexor5 —— RMW的第一步：从旧parity中"减去"旧数据 */
static struct dma_async_tx_descriptor *
ops_run_prexor5(struct stripe_head *sh, struct raid5_percpu *percpu,
        struct dma_async_tx_descriptor *tx)
{
    int disks = sh->disks;
    struct page **xor_srcs = to_addr_page(percpu, 0);
    int count = 0, pd_idx = sh->pd_idx, i;

    /* P盘（校验盘）既是源也是目标 */
    struct page *xor_dest = xor_srcs[count++] = sh->dev[pd_idx].page;

    for (i = disks; i--; ) {
        struct r5dev *dev = &sh->dev[i];
        /* 只处理即将被写入的块（Wantdrain标志） */
        if (test_bit(R5_Wantdrain, &dev->flags)) {
            xor_srcs[count++] = dev->page;  /* 此时page中是旧数据 */
        }
    }

    /* 执行异步XOR：P = P XOR old_D1
     * 即：P' = old_P XOR old_D1（"减去"旧值）
     * ASYNC_TX_XOR_DROP_DST 表示目标页也参与XOR运算 */
    tx = async_xor_offs(xor_dest, ..., xor_srcs, ..., count, ...);
    return tx;
}
```

prexor完成后，stripe_head中的P盘page已经变成了`old_P XOR old_D1`。接下来`ops_run_reconstruct5`会把新数据（new_D1）XOR进去：

```c
/* raid5.c: ops_run_reconstruct5 —— RMW的第二步：XOR新数据完成parity更新 */
static void ops_run_reconstruct5(...)
{
    int prexor = 0;
    ...
    /* 检测是否走prexor路径（即RMW） */
    if (head_sh->reconstruct_state == reconstruct_state_prexor_drain_run) {
        prexor = 1;
        /* P盘是源也是目标（已经被prexor处理过） */
        xor_dest = xor_srcs[count++] = sh->dev[pd_idx].page;
        for (i = disks; i--; ) {
            struct r5dev *dev = &sh->dev[i];
            if (head_sh->dev[i].written) {
                /* 此时page中是新数据（已经drain进来） */
                xor_srcs[count++] = dev->page;
            }
        }
        /* XOR结果：(old_P XOR old_D1) XOR new_D1 = new_P ✓ */
    } else {
        /* RCW路径：直接XOR所有数据盘，从零开始计算新P */
        xor_dest = sh->dev[pd_idx].page;
        for (i = disks; i--; ) {
            if (i != pd_idx)
                xor_srcs[count++] = sh->dev[i].page;
        }
        /* XOR结果：D0 XOR new_D1 XOR D2 XOR D3 = new_P */
    }
    /* prexor路径用 XOR_DROP_DST（目标参与运算）
     * 非prexor路径用 XOR_ZERO_DST（目标先清零再XOR） */
    flags = prexor ? ASYNC_TX_XOR_DROP_DST : ASYNC_TX_XOR_ZERO_DST;
    ...
}
```

内核用`reconstruct_state`枚举来追踪当前stripe_head处于写流程的哪个阶段：

```c
/* raid5.h */
enum reconstruct_states {
    reconstruct_state_idle = 0,
    reconstruct_state_prexor_drain_run,   /* RMW: prexor + drain新数据 */
    reconstruct_state_drain_run,          /* RCW: 直接drain新数据 */
    reconstruct_state_run,                /* expand扩容 */
    reconstruct_state_prexor_drain_result,
    reconstruct_state_drain_result,
    reconstruct_state_result,
};
```

RMW路径走的是`prexor_drain_run`→`prexor_drain_result`——先prexor减去旧值，drain新数据，再reconstruct加上新值。

#### 3.3.2 RCW：读其他盘，从头重算奇偶校验

RCW的思路完全不同——不做差分，直接**收齐所有数据盘的最新值，从零开始计算新的奇偶校验**：

```
【RCW流程：5盘RAID5，修改D1和D2】

  Disk0    Disk1    Disk2    Disk3    Disk4(P)
  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
  │ D0 │  │ D1 │  │ D2 │  │ D3 │  │ P  │
  └────┘  └────┘  └────┘  └────┘  └────┘

  Step 1: 读取未被修改的数据块
  ─────────────────────────────────────────
  ┌────┐                    ┌────┐
  │ D0 │  READ              │ D3 │  READ
  └──┬─┘                    └──┬─┘
     │  new_D1和new_D2已在内存中  │

  Step 2: 用所有数据盘重新计算P（CPU操作）
  ─────────────────────────────────────────
     new_P = D0  XOR  new_D1  XOR  new_D2  XOR  D3
             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
             需要全部4个数据盘的值

  Step 3: 写入新值
  ─────────────────────────────────────────
          ┌────┐  ┌────┐              ┌────┐
   WRITE  │newD1│ │newD2│        WRITE │newP│
          └────┘  └────┘              └────┘

  总代价：2次读（D0和D3）+ 3次写（D1, D2, P）= 5次磁盘IO
```

注意RCW**不需要读旧的P盘和旧的D1、D2**——它只关心"最终所有数据盘上的值是什么"，然后直接算出新P。

RCW在内核中对应`schedule_reconstruction`函数的`rcw`分支：

```c
/* raid5.c: schedule_reconstruction —— 调度写重建操作 */
schedule_reconstruction(struct stripe_head *sh, struct stripe_head_state *s,
                        int rcw, int expand)
{
    int i, pd_idx = sh->pd_idx, disks = sh->disks;

    if (rcw) {
        /* === RCW路径 === */
        /* 释放prexor可能分配的额外页面（不需要prexor了） */
        r5c_release_extra_page(sh);

        for (i = disks; i--; ) {
            struct r5dev *dev = &sh->dev[i];
            if (dev->towrite && !delay_towrite(conf, dev, s)) {
                set_bit(R5_LOCKED, &dev->flags);
                set_bit(R5_Wantdrain, &dev->flags); /* 标记：需要drain新数据 */
                if (!expand)
                    clear_bit(R5_UPTODATE, &dev->flags);
                s->locked++;
            }
        }
        /* 状态设为 drain_run：直接drain新数据，然后XOR所有盘算新P */
        sh->reconstruct_state = reconstruct_state_drain_run;
        set_bit(STRIPE_OP_BIODRAIN, &s->ops_request);
        set_bit(STRIPE_OP_RECONSTRUCT, &s->ops_request);

        /* 如果锁住的盘数 + 最大降级数 == 总盘数，说明是FSW */
        if (s->locked + conf->max_degraded == disks)
            if (!test_and_set_bit(STRIPE_FULL_WRITE, &sh->state))
                atomic_inc(&conf->pending_full_writes);

    } else {
        /* === RMW路径 === */
        /* 前置条件：P盘（和Q盘）的旧值必须已经在内存中 */
        BUG_ON(!(test_bit(R5_UPTODATE, &sh->dev[pd_idx].flags) ||
                 test_bit(R5_Wantcompute, &sh->dev[pd_idx].flags)));

        for (i = disks; i--; ) {
            struct r5dev *dev = &sh->dev[i];
            if (i == pd_idx || i == sh->qd_idx)
                continue;
            if (dev->towrite &&
                (test_bit(R5_UPTODATE, &dev->flags) ||
                 test_bit(R5_Wantcompute, &dev->flags))) {
                set_bit(R5_Wantdrain, &dev->flags);
                set_bit(R5_LOCKED, &dev->flags);
                clear_bit(R5_UPTODATE, &dev->flags);
                s->locked++;
            }
        }
        /* 状态设为 prexor_drain_run：先prexor减旧值，再drain新数据 */
        sh->reconstruct_state = reconstruct_state_prexor_drain_run;
        set_bit(STRIPE_OP_PREXOR, &s->ops_request);     /* 第一步：prexor */
        set_bit(STRIPE_OP_BIODRAIN, &s->ops_request);   /* 第二步：drain */
        set_bit(STRIPE_OP_RECONSTRUCT, &s->ops_request); /* 第三步：XOR */
    }
    ...
}
```

两条路径的区别一目了然：
- **RCW**：`drain_run` → 直接drain新数据 → `RECONSTRUCT`从零XOR所有盘
- **RMW**：`prexor_drain_run` → 先`PREXOR`减旧值 → drain新数据 → `RECONSTRUCT`加新值

#### 3.3.3 内核如何选择策略：代价计数器

理解了两种策略的原理，关键问题来了：**内核怎么决定用哪一种？**

答案藏在`handle_stripe_dirtying`函数中。内核不是简单地比较"修改了多少块"，而是精确计算两种策略各自**还需要多少次额外读IO**，然后选代价低的：

```c
/* raid5.c: handle_stripe_dirtying —— 策略选择的核心逻辑 */
static int handle_stripe_dirtying(struct r5conf *conf,
                                  struct stripe_head *sh,
                                  struct stripe_head_state *s,
                                  int disks)
{
    int rmw = 0, rcw = 0, i;
    sector_t recovery_cp = conf->mddev->recovery_cp;

    /* 特殊情况1：如果rmw_level配置为禁用RMW，或者阵列正在恢复中
     * 且当前stripe在恢复点之后（parity可能不一致），强制使用RCW。
     * 
     * 为什么？因为RMW依赖旧parity的正确性来做差分计算。
     * 如果parity本身就是错的（比如异常断电后还没同步完），
     * 差分算出来的新parity也是错的——错上加错。
     * RCW不依赖旧parity，从头重算，所以更安全。
     */
    if (conf->rmw_level == PARITY_DISABLE_RMW ||
        (recovery_cp < MaxSector && sh->sector >= recovery_cp &&
         s->failed == 0)) {
        rcw = 1; rmw = 2;  /* 人为让rmw更贵，强制走RCW */
        pr_debug("force RCW rmw_level=%u, recovery_cp=%llu sh->sector=%llu\n",
                 conf->rmw_level, ...);
    } else for (i = disks; i--; ) {
```

注意第一个分支：**恢复期间强制RCW**。这是一个重要的安全策略——RMW的差分计算`new_P = old_P XOR old_D XOR new_D`成立的前提是old_P必须正确。如果阵列正在resync，意味着部分stripe的parity可能不一致，此时只有RCW（从零重算）才能保证正确性。

然后是正常情况下的代价计算：

```c
    } else for (i = disks; i--; ) {
        /* === 计算RMW代价 === */
        struct r5dev *dev = &sh->dev[i];
        /* RMW需要读：被修改的数据块（如果旧值不在cache中）+ P/Q盘 */
        if (((dev->towrite && !delay_towrite(conf, dev, s)) ||
             i == sh->pd_idx || i == sh->qd_idx ||
             test_bit(R5_InJournal, &dev->flags)) &&
            !test_bit(R5_LOCKED, &dev->flags) &&
            !(uptodate_for_rmw(dev) ||
              test_bit(R5_Wantcompute, &dev->flags))) {
            if (test_bit(R5_Insync, &dev->flags))
                rmw++;                /* 盘正常：需要1次读 */
            else
                rmw += 2*disks;       /* 盘故障：代价设为极大值 */
        }

        /* === 计算RCW代价 === */
        /* RCW需要读：所有未被覆盖写(OVERWRITE)的数据块 */
        if (!test_bit(R5_OVERWRITE, &dev->flags) &&    /* 没有被整块覆盖 */
            i != sh->pd_idx && i != sh->qd_idx &&      /* 不是校验盘 */
            !test_bit(R5_LOCKED, &dev->flags) &&
            !(test_bit(R5_UPTODATE, &dev->flags) ||     /* 也不在cache中 */
              test_bit(R5_Wantcompute, &dev->flags))) {
            if (test_bit(R5_Insync, &dev->flags))
                rcw++;                /* 盘正常：需要1次读 */
            else
                rcw += 2*disks;       /* 盘故障：代价设为极大值 */
        }
    }
```

这段代码的关键在于：**它不是在数"修改了几块"，而是在数"还需要从磁盘读几块"**。已经在stripe cache中的数据（`R5_UPTODATE`标志）不需要读，所以不计入代价。这意味着策略选择是**动态的**——同样修改2块，如果stripe cache中碰巧缓存了其他块的数据，RCW可能变得更便宜。

计算完两个代价后，比较选择：

```c
    /* 选择代价更低的策略 */
    if ((rmw < rcw || (rmw == rcw && conf->rmw_level == PARITY_PREFER_RMW))
        && rmw > 0) {
        /* RMW更便宜（或代价相同但配置偏好RMW） */
        for (i = disks; i--; ) {
            struct r5dev *dev = &sh->dev[i];
            /* 对需要读旧值的块，设置Wantread标志 */
            ...
            if (...需要读旧数据...) {
                pr_debug("Read_old block %d for r-m-w\n", i);
                set_bit(R5_LOCKED, &dev->flags);
                set_bit(R5_Wantread, &dev->flags);  /* 触发读IO */
                s->locked++;
            }
        }
    }

    if ((rcw < rmw || (rcw == rmw && conf->rmw_level != PARITY_PREFER_RMW))
        && rcw > 0) {
        /* RCW更便宜 */
        rcw = 0;  /* 重新精确计算 */
        for (i = disks; i--; ) {
            struct r5dev *dev = &sh->dev[i];
            if (!test_bit(R5_OVERWRITE, &dev->flags) &&
                i != sh->pd_idx && i != sh->qd_idx && ...) {
                rcw++;
                if (test_bit(R5_Insync, &dev->flags) &&
                    test_bit(STRIPE_PREREAD_ACTIVE, &sh->state)) {
                    pr_debug("Read_old block %d for Reconstruct\n", i);
                    set_bit(R5_LOCKED, &dev->flags);
                    set_bit(R5_Wantread, &dev->flags);  /* 触发读IO */
                    s->locked++;
                }
            }
        }
    }

    /* 所有需要读的数据都到齐后（rmw==0或rcw==0），启动写操作 */
    if (s->locked == 0 && (rcw == 0 || rmw == 0))
        schedule_reconstruction(sh, s, rcw == 0, 0);
        /* 注意：rcw==0 表示RCW不需要额外读了，传给schedule_reconstruction
         * 的参数rcw=1表示走RCW路径；rcw!=0表示走RMW路径 */
}
```

注意`rmw_level`配置项允许管理员手动干预策略偏好：

```c
/* raid5.h */
enum {
    PARITY_DISABLE_RMW = 0,   /* 禁用RMW，强制RCW */
    PARITY_ENABLE_RMW,        /* 允许RMW（默认） */
    PARITY_PREFER_RMW,        /* 代价相同时偏好RMW */
};
```

通过sysfs可以查看和修改：

```bash
# 查看当前策略
cat /sys/block/md0/md/rmw_level
# 0 = 禁用RMW，1 = 允许RMW（默认），2 = 偏好RMW
```

#### 3.3.4 代价分析：什么时候RMW更好，什么时候RCW更好？

用一个5盘RAID5（4个数据盘 + 1个P盘）来量化分析——假设stripe cache中没有任何缓存数据（最坏情况）：

```
修改 k 个数据块（共4个数据盘）时：

  RMW需要读：k个旧数据块 + 1个旧P = (k+1) 次读
  RCW需要读：(4-k)个未修改的数据块   = (4-k) 次读

  ┌───────────────────────────────────────────────────┐
  │ 修改块数k │ RMW读次数 │ RCW读次数 │ 更优策略    │
  ├───────────┼──────────┼──────────┼─────────────┤
  │     1     │    2     │    3     │ RMW ✓       │
  │     2     │    3     │    2     │ RCW ✓       │
  │     3     │    4     │    1     │ RCW ✓       │
  │     4     │    5     │    0     │ FSW（无需读）│
  └───────────────────────────────────────────────────┘

  交叉点：当 k+1 == 4-k，即 k = 1.5 时两者代价相同。
  所以：修改1块用RMW，修改2块及以上用RCW（或FSW）。
```

推广到N个数据盘的一般情况：

```
  RMW读次数 = k + 1（k个旧数据 + 旧P）
  RCW读次数 = N - k（N个数据盘 - k个已在内存中的新数据）

  交叉点：k + 1 = N - k  →  k = (N-1)/2

  即：修改的块数 < (数据盘数-1)/2 时，RMW更优
      修改的块数 > (数据盘数-1)/2 时，RCW更优
```

但这只是**无缓存时的理论分析**。实际上内核的计数器还会考虑stripe cache中已有的数据——如果某个数据块恰好还在cache中（`R5_UPTODATE`），它就不需要从磁盘读了，对应的rmw或rcw计数就不会增加。这就是为什么内核不是简单比较"修改了几块"，而是要逐个磁盘遍历统计实际需要读取的IO数——**缓存命中可以改变最优策略**。

举一个cache影响策略的例子：

```
【5盘RAID5，修改D1，D0恰好在stripe cache中】

  不考虑cache：
    RMW需要读：旧D1 + 旧P = 2次读
    RCW需要读：D0 + D2 + D3 = 3次读  →  选RMW

  D0在cache中：
    RMW需要读：旧D1 + 旧P = 2次读（D0的cache对RMW没用）
    RCW需要读：D2 + D3 = 2次读（D0已在cache中不用读了）
    →  两者代价相同！如果rmw_level == PARITY_PREFER_RMW，选RMW；否则选RCW
```

#### 3.3.5 为什么内核不直接比较"修改块数"？

看到这里可能会有一个疑问：既然理论上交叉点就是`k = (N-1)/2`，为什么内核不直接数修改块数然后比较，而要这么复杂地遍历所有盘计算rmw/rcw？

原因有三：

**第一，stripe cache让"实际代价"和"理论代价"不同。** 理论分析假设所有旧数据都要从磁盘读。但stripe cache可能缓存了某些块，实际需要读的次数更少。

**第二，磁盘故障让某些读操作变得不可能。** 如果一个需要读的盘故障了（`!R5_Insync`），该读操作的代价被设为`2*disks`（一个极大值），相当于"无穷大"。这巧妙地让内核自动避开需要读故障盘的策略：

```c
if (test_bit(R5_Insync, &dev->flags))
    rmw++;                /* 正常盘：代价+1 */
else
    rmw += 2*disks;       /* 故障盘：代价设为极大值，几乎不可能被选中 */
```

例如，如果P盘故障了，RMW需要读旧P但读不到（代价暴涨），内核就自动选RCW（不需要读P盘）。

**第三，部分写（非OVERWRITE）需要特殊处理。** 如果一个4KB的写请求只修改了chunk中的一部分（比如chunk是512KB但只写了4KB），`R5_OVERWRITE`标志不会被设置。这意味着RCW也需要把这个块读出来（因为未被覆盖的部分需要参与重算），而不是简单地将它视为"已修改"。

这三个因素叠加在一起，让策略选择变成了一个需要**逐盘遍历**才能精确计算的问题，而不是简单的数学比较。

#### 3.3.6 RMW vs RCW的性能影响

理解了两种策略后，回到性能优化的话题。对于RAID5的写性能，关键结论是：

**1. 小IO随机写是RAID5的噩梦。** 4KB随机写几乎必然走RMW（只修改stripe中1个chunk），每次写都带来额外的2次读——这就是"RAID5写惩罚"（write penalty）的根源。

**2. stripe cache大小影响策略选择。** 更大的stripe cache意味着更多旧数据留在内存中，减少RMW/RCW的实际读次数，甚至可能让多个小写合并成FSW。

**3. rmw_level参数可以调优。** 对于RAID6（有P和Q两个校验盘），RMW需要同时更新P和Q，计算量翻倍。此时可以通过设置`rmw_level=0`强制使用RCW，避免复杂的双校验差分计算：

```bash
# RAID6场景下禁用RMW，强制RCW
echo 0 > /sys/block/md0/md/rmw_level

# RAID5场景下偏好RMW（减少读取量）
echo 2 > /sys/block/md0/md/rmw_level
```

> **建议5：理解RMW/RCW的代价模型是优化RAID5写性能的基础。**
>
> 对于RAID5：保持默认的`rmw_level=1`通常是最优的，让内核动态选择。如果工作负载以大块顺序写为主（容易触发FSW或RCW），可以试试`rmw_level=0`（禁用RMW）简化代码路径。
>
> 对于RAID6：建议设置`rmw_level=0`（禁用RMW），因为RAID6的RMW需要同时读写P和Q两个校验盘（4次读+4次写），而RCW只需要读未修改的数据盘再重算P和Q。
>
> 无论哪种策略，最有效的优化始终是**让写操作对齐到stripe大小，直接触发FSW**——彻底绕过RMW/RCW。

### 3.4 Batch Write：合并多个stripe的写操作

为了减少写开销，内核实现了**批量写（batch write）**机制，将来自同一个"stripe组"中连续stripe的FSW合并处理：

```c
/* raid5.c */
/* 只有全条带写且没有任何其他操作的新stripe才可以加入batch */
static bool stripe_can_batch(struct stripe_head *sh)
{
    return test_bit(STRIPE_BATCH_READY, &sh->state) &&
        !test_bit(STRIPE_BITMAP_PENDING, &sh->state) &&
        is_full_stripe_write(sh);
}

static void stripe_add_to_batch_list(struct r5conf *conf,
        struct stripe_head *sh, struct stripe_head *last_sh)
{
    ...
    /* 只有sector地址与chunk边界对齐时才能batch */
    if (!sector_div(tmp_sec, conf->chunk_sectors))
        ...
}
```

batch write将多个连续stripe的写操作合并为一个大的写操作，减少奇偶校验计算的次数和磁盘写操作的次数，对顺序写性能提升显著。

### 3.5 Plug/Unplug机制：积攒IO再批量提交

#### 块层plug/unplug的基本原理

Linux块层有一个plug/unplug机制——先"塞住"IO不提交，积攒一批后再一起提交（unplug），目的是增加IO合并机会。这个概念类似浴缸的塞子：plug（塞住）时水（IO请求）不断积攒，unplug（拔掉）时一次性放出。

在通用块层，plug/unplug的基本流程是这样的：

```c
/* block/blk-core.c */
void blk_start_plug(struct blk_plug *plug)
{
    /* 将plug挂到当前进程的task_struct上 */
    INIT_LIST_HEAD(&plug->mq_list);
    ...
    current->plug = plug;
}

void blk_finish_plug(struct blk_plug *plug)
{
    if (plug == current->plug) {
        /* 拔掉塞子，提交所有积攒的IO */
        __blk_flush_plug(plug, false);
        current->plug = NULL;
    }
}
```

关键点在于：**plug是per-task的**——每个进程（线程）有自己独立的plug链表，在`blk_start_plug()`和`blk_finish_plug()`之间提交的所有IO都会被暂存，直到unplug时才真正下发到设备驱动。

#### 没有plug会怎样？——逐个提交的问题

要理解plug的价值，先看看没有plug时RAID5写IO的处理流程。假设应用层连续写入4个4KB块，恰好覆盖同一个stripe的全部数据盘：

```
【无plug——逐个提交模式】

  应用连续写4个块（D0~D3），它们属于同一个stripe

  时间线：
  ─────────────────────────────────────────────────────>

  D0到达RAID5层：
    → 只有1/4数据，无法FSW
    → 触发RMW：读旧D0 + 读旧P → 计算新P → 写D0 + 写P
    → 代价：2读 + 2写

  D1到达RAID5层：
    → stripe_head上已有D0在处理中，但D0的RMW可能已经提交了
    → D1又触发一次RMW：读旧D1 + 读旧P → 计算新P → 写D1 + 写P
    → 代价：2读 + 2写

  D2到达、D3到达... 同样的悲剧重复

  总代价：4次RMW = 8读 + 8写 = 16次磁盘IO
```

每个写请求到达时，RAID5层看到stripe上只有部分数据，只能走RMW路径。即使应用层的意图是顺序写满整个stripe，由于请求一个一个提交，RAID5层无法感知到"后面还有更多数据要来"。

#### 有plug时——批量到达，一次FSW

现在看有plug时的情况：

```
【有plug——批量提交模式】

  应用连续写4个块（D0~D3），它们属于同一个stripe

  时间线：
  ─────────────────────────────────────────────────────>

  blk_start_plug()
    │
    ├── D0提交 → 暂存在plug链表
    ├── D1提交 → 暂存在plug链表
    ├── D2提交 → 暂存在plug链表
    ├── D3提交 → 暂存在plug链表
    │
  blk_finish_plug() → unplug，4个请求一起下发到RAID5层
    │
    ├── D0~D3同时到达同一个stripe_head
    ├── stripe_head发现所有数据盘都被覆盖
    ├── 触发FSW（Full Stripe Write）！
    └── 直接计算新P → 写D0+D1+D2+D3+P

  总代价：0读 + 5写 = 5次磁盘IO
```

**从16次磁盘IO降到5次——这就是plug对RAID5写性能的核心价值。**

关键差异不仅仅是IO次数的减少。FSW路径还有一个重要优势：**不需要等待读操作完成**。RMW的"先读后写"意味着写操作必须等读完成后才能开始计算奇偶校验，而FSW的所有数据都已经在内存中，可以立即计算并提交写操作。

#### RAID5的专属plug实现

通用块层的plug机制已经能积攒IO，但RAID5在此基础上实现了**自己的plug回调**，进一步优化了stripe级别的IO合并：

```c
/* raid5.c */
struct raid5_plug_cb {
    struct blk_plug_cb  cb;
    struct list_head    list;   /* 积攒的stripe_head列表 */
    struct list_head    temp_inactive_list[NR_STRIPE_HASH_LOCKS];
};

static void
raid5_make_request(struct mddev *mddev, struct bio *bi)
{
    ...
    /* 尝试获取当前进程的RAID5 plug */
    plug = container_of(cb, struct raid5_plug_cb, cb);
    if (...) {
        /* 如果当前进程有活跃的plug，把stripe_head
         * 挂到plug的链表里，暂不提交处理 */
        list_add_tail(&sh->lru, &plug->list);
    } else {
        /* 没有plug，直接提交处理 */
        release_stripe_plug(mddev, sh);
    }
    ...
}
```

这段代码的含义是：当一个bio到达RAID5层时，内核先找到对应的stripe_head，然后检查当前进程是否有活跃的plug。如果有，就把stripe_head暂存到plug的链表里而不立即处理；如果没有plug或者plug链表已满，就直接提交。

unplug时的处理是这样的：

```c
/* raid5.c */
static void raid5_unplug(struct blk_plug_cb *blk_cb, bool from_schedule)
{
    struct raid5_plug_cb *cb = container_of(
        blk_cb, struct raid5_plug_cb, cb);
    struct stripe_head *sh;
    struct mddev *mddev = cb->cb.data;
    struct r5conf *conf = mddev->private;
    int cnt = 0;

    if (cb->list.next && !list_empty(&cb->list)) {
        spin_lock_irq(&conf->device_lock);
        while (!list_empty(&cb->list)) {
            sh = list_first_entry(&cb->list,
                struct stripe_head, lru);
            list_del_init(&sh->lru);
            /*
             * 避免对每个stripe_head单独调用raid5_release_stripe，
             * 而是批量处理——减少锁获取/释放的次数
             */
            __release_stripe(conf, sh, &cb->temp_inactive_list[...]);
            cnt++;
        }
        spin_unlock_irq(&conf->device_lock);
    }
    /* 批量唤醒raid5d守护线程处理所有积攒的stripe */
    release_inactive_stripe_list(conf, cb->temp_inactive_list, ...);
    ...
}
```

注意这里的关键优化：**unplug时不是逐个处理stripe_head，而是先批量收集到`temp_inactive_list`中，最后一次性唤醒`raid5d`守护线程**。这避免了反复获取/释放锁、反复唤醒守护线程的开销。

#### 从全局视角看plug/unplug的IO路径

把整个流程画出来：

```
【RAID5 plug/unplug 完整IO路径】

  用户进程                      内核RAID5层                raid5d守护线程
  ═══════                      ═══════════               ═══════════════

  blk_start_plug()
      │
  write(D0)───────→ raid5_make_request()
      │               ├ 找到stripe_head #100
      │               └ 挂到plug->list ─→ [sh#100]
  write(D1)───────→ raid5_make_request()
      │               ├ stripe_head #100已在list
      │               └ 更新sh#100的数据 ─→ [sh#100(D0,D1)]
  write(D2)───────→ raid5_make_request()
      │               ├ 同上
      │               └ 更新sh#100 ─→ [sh#100(D0,D1,D2)]
  write(D3)───────→ raid5_make_request()
      │               ├ 同上
      │               └ 更新sh#100 ─→ [sh#100(D0,D1,D2,D3)]
      │
  blk_finish_plug()
      │
      └──────────→ raid5_unplug()
                      ├ 遍历plug->list
                      ├ sh#100：所有数据盘已覆盖 → 标记FSW
                      └ 批量提交 ──────────────→ handle_stripe()
                                                   ├ is_full_stripe_write() = true
                                                   ├ 直接计算新parity
                                                   └ 提交5个写bio（D0+D1+D2+D3+P）
```

这里有一个细节值得注意：在plug期间，同一个stripe的多次写不会创建多个stripe_head，而是**累积到同一个stripe_head上**。这意味着plug不仅减少了IO次数，还减少了stripe_head的分配和回收开销。

#### plug/unplug的触发时机

了解plug在什么时候被启用和释放，对理解实际性能影响非常重要：

| 场景 | plug行为 | 对RAID5的影响 |
|------|---------|-------------|
| **同步写（sync，如`write`+`fsync`）** | 内核VFS层自动plug/unplug | 单次write期间的多个bio可以合并 |
| **buffered写** | 回写线程（writeback）会plug | 大批脏页一起提交，合并效果好 |
| **direct IO + libaio/io_uring** | `io_submit()`批量提交时plug | 一次系统调用中的多个IO可以合并 |
| **direct IO + 同步逐个write** | 每次write独立plug/unplug | plug效果有限，每个请求独立处理 |
| **fio numjobs=1, iodepth=1** | 几乎无plug效果 | 每个IO独立处理，最差情况 |
| **fio numjobs=1, iodepth=N** | libaio批量提交有plug | iodepth越大，plug积攒越多 |

从这个表可以看到，**应用层的IO提交方式直接决定了plug能积攒多少IO**。如果应用每次只写一个块然后等完成再写下一个（典型的同步逐块写），plug几乎帮不上忙；而如果应用使用`io_uring`或`libaio`一次提交大量IO，plug就能充分发挥作用。

#### plug积攒量与FSW触发率的关系

plug的核心价值是增加FSW的触发率。我们可以从数学角度来理解这种关系。

假设一个5盘RAID5（4数据盘），随机写4KB块（假设均匀分布在不同stripe上）：

| plug积攒的IO数 | 同一stripe恰好收集满4块的概率 | 预期写路径 |
|:--:|:--:|:--|
| 1 | 0% | 必然RMW |
| 4 | 约0.03%（4块恰好在同一stripe） | 几乎全是RMW |
| 16 | 有限（取决于stripe_cache_size） | 少量合并 |
| 1000+ | 显著（writeback场景） | 大量FSW |

对于**随机写**，plug能积攒的IO数量通常有限（取决于iodepth），且分散在不同stripe上，FSW的触发率很低。但对于**顺序写**，情况完全不同——连续的写请求天然会命中相邻的stripe，plug积攒一小批就足够触发FSW。

这也解释了一个常见现象：**RAID5的顺序写性能远好于随机写，且差距比单盘时更大**。在单盘上，顺序写和随机写的差异主要来自磁盘寻道和缓存命中率；而在RAID5上，还叠加了FSW vs RMW的路径差异——这个差异可以是数倍的。

#### 与batch write机制的协同

3.4节介绍了batch write——将多个连续stripe的FSW合并处理。plug机制是batch write能生效的**前提条件**：

```
【plug + batch write 协同】

  没有plug时：
    stripe#0 FSW → 提交 → stripe#1 FSW → 提交 → stripe#2 FSW → 提交
    （每个stripe独立提交，无法batch）

  有plug时：
    plug期间积攒 → [stripe#0, stripe#1, stripe#2] → unplug
    → stripe#0~#2连续地址 → stripe_can_batch() = true
    → 合并为一个大的写操作 → 一次提交
    （减少了2次dispatch开销和raid5d唤醒次数）
```

plug保证了多个stripe的请求**同时可见**，batch write才有机会检测到它们是连续的并合并处理。如果没有plug，每个stripe请求到达时前一个可能已经提交了，batch就无从谈起。

> **建议5：应用层的IO提交方式直接决定RAID5能否充分利用plug机制。**
>
> - 使用`io_uring`或`libaio`的批量提交接口（`io_submit()`一次提交多个IO），可以最大化plug积攒量
> - `fio`测试时增大`iodepth`（如32或64），让libaio引擎有足够的IO可以批量提交
> - buffered IO场景下，调整`/proc/sys/vm/dirty_writeback_centisecs`和`dirty_expire_centisecs`控制回写批量大小
> - 避免"写一个块等一个块"的同步IO模式——这会让plug完全失效，每个IO都走RMW

#### 性能测试方案

plug/unplug机制的效果可以通过对比不同IO提交方式的性能差异来间接观测：

**方法1：对比不同iodepth的FSW触发率**

iodepth直接影响plug能积攒多少IO。通过对比不同iodepth下的顺序写性能，可以观察plug的效果：

```bash
# 测试脚本：固定顺序写，改变iodepth
for depth in 1 2 4 8 16 32 64 128; do
  fio --name=plug_depth_test --filename=/dev/md0 \
      --rw=write --bs=4k --iodepth=$depth --numjobs=1 \
      --ioengine=libaio --direct=1 --runtime=60 --time_based \
      --group_reporting --output-format=json \
      --output=plug_depth_${depth}.json
done
```

预期结果：
- `iodepth=1`：每次只提交一个IO，plug几乎无效，性能最差
- `iodepth=4~8`：开始有足够的IO填满stripe，FSW比例上升，性能跳跃式增长
- `iodepth=32+`：FSW比例趋近饱和，性能增长趋缓

关键指标：对比`iodepth=1`和`iodepth=32`的IOPS差异。如果差异超过2倍，说明plug带来的IO合并/FSW效果显著。

**方法2：对比sync IO vs libaio批量提交**

```bash
# 同步IO引擎（psync）——每次写操作独立提交，plug效果最小
fio --name=sync_test --filename=/dev/md0 \
    --rw=write --bs=4k --iodepth=1 --numjobs=1 \
    --ioengine=psync --direct=1 --runtime=60 --time_based \
    --group_reporting --output-format=json \
    --output=plug_psync.json

# 异步IO引擎（libaio）——批量提交，充分利用plug
fio --name=aio_test --filename=/dev/md0 \
    --rw=write --bs=4k --iodepth=32 --numjobs=1 \
    --ioengine=libaio --direct=1 --runtime=60 --time_based \
    --group_reporting --output-format=json \
    --output=plug_libaio.json

# io_uring引擎——更高效的批量提交
fio --name=uring_test --filename=/dev/md0 \
    --rw=write --bs=4k --iodepth=32 --numjobs=1 \
    --ioengine=io_uring --direct=1 --runtime=60 --time_based \
    --group_reporting --output-format=json \
    --output=plug_iouring.json
```

这组测试直接对比了三种IO引擎在RAID5上的写性能。`psync`（同步逐个写）是plug效果最差的情况，`libaio`和`io_uring`则能充分利用plug批量提交。

**方法3：用blktrace直接观察IO合并效果**

```bash
# 在一个终端启动blktrace
blktrace -d /dev/md0 -o plug_trace &

# 在另一个终端运行写测试
fio --name=trace_test --filename=/dev/md0 \
    --rw=write --bs=4k --iodepth=32 --numjobs=1 \
    --ioengine=libaio --direct=1 --runtime=10 --time_based

# 停止blktrace后分析
kill %1
blkparse -i plug_trace -d plug_trace.bin
btt -i plug_trace.bin -o plug_analysis
```

`btt`的输出会包含`Q2G`（请求排队到合并）和`D2C`（下发到完成）的延迟分布。如果plug工作良好，`Q2G`时间会较长（IO在plug中等待合并），但`D2C`时间会较短（合并后的IO更高效）。

**方法4：通过/proc/mdstat观察stripe cache命中率**

```bash
# 运行测试时每秒采样stripe cache状态
while true; do
  echo "=== $(date +%H:%M:%S) ==="
  cat /proc/mdstat | grep -A5 md0
  cat /sys/block/md0/md/stripe_cache_active
  sleep 1
done
```

关注`stripe_cache_active`的变化。在plug有效的高iodepth场景下，活跃stripe_head数量会更稳定（同一个stripe_head被反复命中）；而在iodepth=1的逐个提交场景下，活跃stripe_head数量会频繁波动（每次分配新的stripe_head）。

### 3.6 stripe_cache的并发控制：hash锁

为了在多核环境下提高stripe cache的并发访问性能，内核使用了分段哈希锁（sharded hash lock）：

```c
/* raid5.h */
#define NR_STRIPE_HASH_LOCKS 8
#define STRIPE_HASH_LOCKS_MASK (NR_STRIPE_HASH_LOCKS - 1)

struct r5conf {
    ...
    struct hlist_head   *stripe_hashtbl;
    spinlock_t          hash_locks[NR_STRIPE_HASH_LOCKS];
    ...
};
```

#### 为什么需要hash锁？——从单锁瓶颈说起

在早期的内核实现中（3.13之前），整个stripe cache只有**一把全局自旋锁**（`device_lock`）。每次查找、插入、释放stripe_head都要先获取这把锁。在单核或低并发场景下这没问题，但随着多核CPU和高速NVMe设备的普及，这把全局锁成了严重的性能瓶颈：

```
【单锁模型——所有CPU争抢一把锁】

    CPU0         CPU1         CPU2         CPU3
      |            |            |            |
      v            v            v            v
  +--------------------------------------------+
  |           device_lock (全局锁)              |
  |  ┌──────────────────────────────────────┐  |
  |  │     stripe cache hash table          │  |
  |  │  bucket[0] → sh → sh → ...          │  |
  |  │  bucket[1] → sh → sh → ...          │  |
  |  │  bucket[2] → sh → sh → ...          │  |
  |  │  ...                                 │  |
  |  └──────────────────────────────────────┘  |
  +--------------------------------------------+

  问题：4个CPU同时处理不同stripe的IO请求，
  但每次操作都必须排队获取 device_lock。
  在32核/64核系统上，锁争用导致大量CPU空转。
```

当多个CPU核心同时处理IO请求时（比如32路并发随机写），每个请求都需要在stripe hash table中查找或分配stripe_head。如果所有CPU都在争抢同一把锁，实际的并发度被锁序列化为1——即使有32个核心，同一时刻也只有一个核心能操作stripe cache，其余31个核心都在自旋等待。

#### hash锁如何解决？——分段锁（Lock Striping）

内核的解决方案是**分段锁**（Lock Striping）：将stripe hash table按hash值分成`NR_STRIPE_HASH_LOCKS`（8）个分段，每段有自己独立的自旋锁。不同分段的操作可以完全并行。

```c
/* raid5.c: stripe_head的hash计算 */
static inline struct hlist_head *stripe_hash(struct r5conf *conf, sector_t sect)
{
    /* 通过sector地址计算hash桶编号 */
    int hash = (sect >> RAID5_STRIPE_SHIFT(conf)) & STRIPE_HASH_LOCKS_MASK;
    return &conf->stripe_hashtbl[hash];
}
```

核心思想是：**不同sector地址的stripe_head大概率落在不同的hash桶中，因此只需要锁住对应的桶，而不是整个hash表。**

```
【分段锁模型——每个桶有独立的锁】

  CPU0         CPU1         CPU2         CPU3
    |            |            |            |
    v            v            v            v
  hash_lock[0]  hash_lock[1]  hash_lock[2]  hash_lock[3]
    |            |            |            |
    v            v            v            v
  bucket[0,8,16..]  bucket[1,9,17..]  bucket[2,10,18..]  bucket[3,11,19..]
  → sh → sh        → sh → sh          → sh → sh          → sh → sh

  关键改进：4个CPU访问不同桶时，完全无锁争用，真正并行！
  只有两个CPU恰好访问同一个桶时才需要等待。
```

具体来说，当一个IO请求进入RAID5层时，处理流程变成：

1. **计算hash**：根据请求的sector地址，计算出该stripe应该在哪个hash桶（`sect >> STRIPE_SHIFT & 0x7`）
2. **只锁对应桶**：获取`hash_locks[bucket_id]`，而不是全局锁
3. **查找/操作**：在该桶的链表中查找stripe_head，执行插入/引用/释放
4. **释放桶锁**：操作完成后释放

由于8个桶的锁是完全独立的，最多可以有8个CPU核心同时操作stripe cache而完全无锁争用。

#### 并发度提升的量化分析

假设有N个CPU核心同时发起IO请求，每个请求需要操作stripe cache，锁的持有时间为t：

| 模型 | 最大并行度 | N核争用概率 | 有效吞吐 |
|------|-----------|-----------|---------|
| 单全局锁 | 1 | 100%（N>1时必争用） | 受限于1/t |
| 8分段锁 | 8 | N≤8时约 1-(7/8)^(N-1) | 受限于8/t |

用生日悖论类比：8个桶，N个请求随机分配，发生冲突（两个请求落在同一个桶）的概率：
- 2个并发请求：冲突概率 ≈ 12.5%（1/8）
- 4个并发请求：冲突概率 ≈ 41%
- 8个并发请求：冲突概率 ≈ 75%

但即使在8个并发请求、75%概率有某对冲突的情况下，平均也有5~6个请求可以同时进行——这远好于单锁模型下的"只有1个能执行"。

#### 为什么是8个锁？

`NR_STRIPE_HASH_LOCKS=8`是内核开发者经过权衡选择的经验值：

- **太少（如2个）**：锁争用改善有限，多核场景下仍然是瓶颈
- **太多（如256个）**：每个锁对应的stripe太少，锁本身的内存开销和cache line占用增大；且unplug时需要遍历更多的分段，增加延迟
- **8个**：在主流服务器CPU（8~64核）上，锁争用已显著降低，同时内存开销仅8个`spinlock_t`（约256~512字节），cache友好

值得注意的是，`NR_STRIPE_HASH_LOCKS`是编译时常量（`#define`），**不可运行时调优**。如果你的场景确实需要更高的并行度（比如128核CPU + 高速NVMe组RAID5），可以通过修改内核头文件重新编译。

#### 与plug/unplug机制的协同

回顾3.5节的plug/unplug机制，`raid5_plug_cb`中也使用了按hash锁分段的临时链表：

```c
struct raid5_plug_cb {
    struct blk_plug_cb  cb;
    struct list_head    list;
    struct list_head    temp_inactive_list[NR_STRIPE_HASH_LOCKS];  /* 每个锁桶一条链表 */
};
```

unplug时，每个桶的stripe可以独立处理，进一步减少了锁的持有时间。这是一个精心设计的**lock-per-bucket**全局一致方案——从IO进入（plug时按hash分桶积攒）到IO处理（unplug时按桶独立提交），锁粒度始终保持在桶级别。

#### 性能测试方案

由于`NR_STRIPE_HASH_LOCKS`是编译时常量，要严格A/B测试需要用不同的内核。但我们可以通过**间接方法**观察hash锁的效果：

**方法1：对比不同并发度下的扩展性曲线**

固定RAID5配置，逐步增加fio的`numjobs`（1→2→4→8→16→32），观察IOPS是否线性增长：

```bash
# 测试脚本示例
for jobs in 1 2 4 8 16 32; do
  fio --name=hash_lock_test --filename=/dev/md0 \
      --rw=randwrite --bs=4k --iodepth=32 --numjobs=$jobs \
      --ioengine=libaio --direct=1 --runtime=60 --time_based \
      --group_reporting --output-format=json \
      --output=hashlock_jobs${jobs}.json
done
```

如果hash锁工作良好：
- 1→8 jobs：IOPS应接近线性增长（受益于8个独立锁桶）
- 8→16→32 jobs：增长曲线开始变平（8个锁桶成为新瓶颈）

如果hash锁不起作用（退化为单锁）：
- 1→2 jobs时IOPS增长就会明显放缓

**方法2：使用perf/lockstat观察锁争用**

```bash
# 开启锁统计（需要内核编译时开启CONFIG_LOCK_STAT）
echo 1 > /proc/sys/kernel/lock_stat

# 运行高并发写测试
fio --name=test --filename=/dev/md0 --rw=randwrite --bs=4k \
    --iodepth=64 --numjobs=16 --ioengine=libaio --direct=1 \
    --runtime=30 --time_based

# 查看RAID5相关的锁争用统计
grep -A5 'hash_lock\|stripe_lock\|device_lock' /proc/lock_stat
```

关注指标：
- `contentions`：锁争用次数
- `waittime-avg`：平均等待时间
- `holdtime-avg`：平均持有时间

**方法3：通过ebpf/bpftrace追踪**

```bash
# 追踪hash_lock的获取和释放
bpftrace -e '
  kprobe:raid5_get_active_stripe {
    @start[tid] = nsecs;
  }
  kretprobe:raid5_get_active_stripe /@start[tid]/ {
    @latency_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
  }
'
```

在不同`numjobs`下对比`raid5_get_active_stripe`的延迟分布：并发越高，如果锁争用严重，该函数的延迟尾部（P99/P999）会显著增大。

### 3.7 RAID算法选择（RAID5布局）

RAID5有多种奇偶校验分布算法，通过`/sys/block/md0/md/layout`查看：

```c
/* raid5.c: 算法选择 */
int algorithm = previous ? conf->prev_algo : conf->algorithm;
switch (algorithm) {
case ALGORITHM_LEFT_ASYMMETRIC:    /* 左不对称，奇偶校验从左轮转 */
case ALGORITHM_RIGHT_ASYMMETRIC:   /* 右不对称 */
case ALGORITHM_LEFT_SYMMETRIC:     /* 左对称，Linux默认 */
case ALGORITHM_RIGHT_SYMMETRIC:    /* 右对称 */
case ALGORITHM_PARITY_0:           /* 奇偶校验固定在盘0 */
case ALGORITHM_PARITY_N:           /* 奇偶校验固定在最后一盘 */
}
```


为了更直观地理解这些布局算法的差异，下面以**4块磁盘（3数据+1校验）**为例，展示每种布局下Stripe 0 ~ 7中数据块（D0, D1, D2, ...）和校验块（P）在各磁盘上的分布。这些图表通过模拟内核`raid5_compute_sector()`函数的算法逻辑生成，保证与实际行为一致。

#### 理解"左/右"和"对称/不对称"

在看具体图之前，先理解命名规则：

- **"左"vs"右"**：指校验块（P）的**起始位置和旋转方向**。"左"表示P从最右列开始、向左移动（每个stripe中P的位置向左移一列）；"右"表示P从最左列开始、向右移动。
- **"对称"vs"不对称"**：指数据块的**编号方式**。"不对称"（asymmetric）时数据块总是从Disk0开始、按固定列顺序填充（跳过P所在列）；"对称"（symmetric）时数据块从P块的下一列开始循环填充（即"绕回来"），使得连续数据块尽量分布在不同磁盘上。

#### 1. Left-Asymmetric（左不对称）

P从右侧开始向左旋转，数据块按固定列顺序填充（跳过P所在列）：

```
              Disk0    Disk1    Disk2    Disk3
  Stripe 0:    D0       D1       D2       P0
  Stripe 1:    D3       D4       P1       D5
  Stripe 2:    D6       P2       D7       D8
  Stripe 3:    P3       D9       D10      D11
  Stripe 4:    D12      D13      D14      P4
  Stripe 5:    D15      D16      P5       D17
  Stripe 6:    D18      P6       D19      D20
  Stripe 7:    P7       D21      D22      D23
```

特点：P向左轮转；数据块总是"跳过P、从Disk0到Disk3"顺序填充。注意看Stripe 0→1的过渡：D2在Disk2，D3又回到了Disk0，顺序读时Disk3空闲了一个周期。

#### 2. Left-Symmetric（左对称）⭐ Linux默认

P从右侧开始向左旋转，数据块从**P的下一列位置开始**循环填充：

```
              Disk0    Disk1    Disk2    Disk3
  Stripe 0:    D0       D1       D2       P0
  Stripe 1:    D4       D5       P1       D3
  Stripe 2:    D8       P2       D6       D7
  Stripe 3:    P3       D9       D10      D11
  Stripe 4:    D12      D13      D14      P4
  Stripe 5:    D16      D17      P5       D15
  Stripe 6:    D20      P6       D18      D19
  Stripe 7:    P7       D21      D22      D23
```

关键区别在于：看Stripe 0→1的过渡，D2在Disk2，**D3被放在了Disk3**（P1的下一列位置），D4→Disk0，D5→Disk1。这意味着连续读取D0→D5时，**每块盘都参与了数据供给**：D0(Disk0)→D1(Disk1)→D2(Disk2)→D3(Disk3)→D4(Disk0)→D5(Disk1)。**顺序读时四块盘完全并行，带宽利用率最大化**。这正是Linux默认选择它的原因。

#### 3. Right-Asymmetric（右不对称）

P从左侧开始向右旋转，数据块按固定列顺序填充：

```
              Disk0    Disk1    Disk2    Disk3
  Stripe 0:    P0       D0       D1       D2
  Stripe 1:    D3       P1       D4       D5
  Stripe 2:    D6       D7       P2       D8
  Stripe 3:    D9       D10      D11      P3
  Stripe 4:    P4       D12      D13      D14
  Stripe 5:    D15      P5       D16      D17
  Stripe 6:    D18      D19      P6       D20
  Stripe 7:    D21      D22      D23      P7
```

特点：P向右轮转；数据跳过P后从左到右顺序填充。与Left-Asymmetric互为镜像。

#### 4. Right-Symmetric（右对称）

P从左侧开始向右旋转，数据从P的下一列开始循环填充：

```
              Disk0    Disk1    Disk2    Disk3
  Stripe 0:    P0       D0       D1       D2
  Stripe 1:    D5       P1       D3       D4
  Stripe 2:    D7       D8       P2       D6
  Stripe 3:    D9       D10      D11      P3
  Stripe 4:    P4       D12      D13      D14
  Stripe 5:    D17      P5       D15      D16
  Stripe 6:    D19      D20      P6       D18
  Stripe 7:    D21      D22      D23      P7
```

特点：P向右轮转；数据环绕填充。与Left-Symmetric互为镜像，顺序读并发度同样高。

#### 5. Parity-0（校验固定在盘0）

P始终在Disk0，数据在Disk1~3顺序填充：

```
              Disk0    Disk1    Disk2    Disk3
  Stripe 0:    P0       D0       D1       D2
  Stripe 1:    P1       D3       D4       D5
  Stripe 2:    P2       D6       D7       D8
  Stripe 3:    P3       D9       D10      D11
```

特点：本质上就是RAID4的布局方式。Disk0成为校验瓶颈——**所有写操作都需要更新Disk0的校验块**，该盘的写负载是其他盘的3倍，成为整个阵列的写性能瓶颈。不推荐使用。

#### 6. Parity-N（校验固定在最后一盘）

P始终在最后一个盘，与Parity-0类似：

```
              Disk0    Disk1    Disk2    Disk3
  Stripe 0:    D0       D1       D2       P0
  Stripe 1:    D3       D4       D5       P1
  Stripe 2:    D6       D7       D8       P2
  Stripe 3:    D9       D10      D11      P3
```

特点：同样是RAID4风格，最后一个盘成为校验瓶颈。

#### 布局对比总结

| 布局 | P起始与方向 | 数据填充方式 | 顺序读并发度 | 写负载均衡 |
|------|-----------|-------------|-------------|-----------|
| **Left-Symmetric** ⭐ | Disk3→左移 | 从P下一列环绕 | ★★★★★ 最优 | ★★★★★ |
| Right-Symmetric | Disk0→右移 | 从P下一列环绕 | ★★★★★ 最优 | ★★★★★ |
| Left-Asymmetric | Disk3→左移 | 跳过P、固定顺序 | ★★★☆☆ | ★★★★★ |
| Right-Asymmetric | Disk0→右移 | 跳过P、固定顺序 | ★★★☆☆ | ★★★★★ |
| Parity-0 | 固定Disk0 | 固定顺序 | ★★★☆☆ | ★☆☆☆☆ 最差 |
| Parity-N | 固定末盘 | 固定顺序 | ★★★☆☆ | ★☆☆☆☆ 最差 |

**关键差异在于顺序读场景**：Symmetric（对称）布局中，跨stripe的连续数据块（如D2→D3）分布在不同磁盘上，顺序读可以充分利用所有磁盘的并行带宽。而Asymmetric（不对称）布局中，跨stripe时数据块回到Disk0重新开始，导致某些磁盘在stripe交界处空闲一拍，略微降低了并行度。不过在实际测试中，这种差异通常在5%以内，远不如chunk size和stripe cache size的影响大。

Linux默认使用`left-symmetric`（左对称）布局，这种布局下每个stripe的数据块在多个成员盘上的分布更均匀，顺序读时可以最大化磁盘并发度。

> **建议6：保持默认的`left-symmetric`算法，这是多年实践证明在大多数场景下性能最优的选择。**

---

## 四、md层的关键调优参数

上面分析了RAID5的内部机制，这里汇总所有可调参数。

### 4.1 stripe_cache_size

```bash
# 查看
cat /sys/block/md0/md/stripe_cache_size
# 修改（需要根据内存和工作负载调整）
echo 4096 > /sys/block/md0/md/stripe_cache_size
```

### 4.2 sync_speed（重建速度限制）

RAID重建（sync）和正常IO共享相同的IO带宽。可以通过以下参数控制重建速度，避免在业务高峰期重建占用过多IO：

```bash
# 最低重建速度（KB/s）
cat /proc/sys/dev/raid/speed_limit_min

# 最高重建速度（KB/s）
cat /proc/sys/dev/raid/speed_limit_max

# 临时降低重建速度，保证正常业务
echo 10000 > /proc/sys/dev/raid/speed_limit_max    # 限制为10MB/s
```

### 4.3 chunk_size的选择

chunk_size是RAID5性能调优中最重要的参数，在创建时确定，不可在线修改：

- **顺序读写密集（如大文件、流媒体）**：较大chunk（512KB~2MB），减少跨stripe的元数据操作
- **随机IO密集（如数据库）**：较小chunk（64KB~256KB），单次IO尽量不跨chunk，减少RMW

```bash
# 创建RAID5时指定chunk大小（512KB）
mdadm --create /dev/md0 --level=5 --chunk=512 \
      --raid-devices=5 /dev/sd{b,c,d,e,f}
```

### 4.4 read_ahead（预读）

顺序读场景下，增大预读可以显著提升吞吐量：

```bash
# 查看当前预读设置（单位：512字节扇区）
blockdev --getra /dev/md0

# 设置预读为8MB
blockdev --setra 16384 /dev/md0
```

---

## 五、LVM和软RAID组合使用的注意事项

在实践中，经常会看到"LVM on RAID"或"RAID on LVM"两种组合方式，它们的性能特征有所不同。

### 5.1 RAID on LVM（LVM层在上）

```
文件系统
   ↓
LVM（提供逻辑卷）
   ↓
md软RAID（提供冗余）
   ↓
物理磁盘
```

这种方式的好处是LVM可以自由管理RAID阵列上的空间。但需要注意**LVM的chunk_size要与RAID的chunk_size对齐**，否则LVM的条带化会打乱RAID层的IO对齐，引发大量RMW。

### 5.2 LVM on RAID（RAID层在上）

```
文件系统
   ↓
md软RAID（内嵌于dm-raid）
   ↓
LVM PV
   ↓
物理磁盘
```

使用`dm-raid`（LVM RAID）时，LVM直接管理RAID配置，两层之间不存在对齐问题，推荐这种方式。

创建LVM RAID5卷：
```bash
# 创建RAID5逻辑卷（4数据盘+1奇偶校验盘）
lvcreate --type raid5 -i 4 -I 512k -L 100G -n lv_raid5 vg_name
```

---

## 六、性能基准测试方法

在做任何调优前，先建立基准线：

```bash
# 1. 顺序写测试（验证FSW效率）
fio --name=seq_write --rw=write --bs=4M --size=10G \
    --filename=/dev/md0 --direct=1 --numjobs=1 \
    --ioengine=libaio --iodepth=32

# 2. 随机写测试（最能体现RAID5痛点）
fio --name=rand_write --rw=randwrite --bs=4K --size=10G \
    --filename=/dev/md0 --direct=1 --numjobs=8 \
    --ioengine=libaio --iodepth=64

# 3. 混合读写（模拟数据库场景）
fio --name=mixed --rw=randrw --rwmixread=70 --bs=8K --size=10G \
    --filename=/dev/md0 --direct=1 --numjobs=4 \
    --ioengine=libaio --iodepth=32
```

通过`/proc/mdstat`和`/sys/block/md0/md/`下的计数器监控RAID5的内部状态：

```bash
# 查看RAID状态
cat /proc/mdstat

# 查看stripe cache命中情况
cat /sys/block/md0/md/stripe_cache_size

# 监控raid5线程状态
cat /proc/sys/dev/raid/speed_limit_max
```

---

## 七、总结

本文从源码角度梳理了Linux软RAID和LVM的核心IO路径和性能关键点，主要结论如下：

1. **DM框架开销极小**，使用RCU和位运算优化了热路径，框架本身不是性能瓶颈

2. **LVM条带化**性能好，chunk_size和条带数建议使用2的幂（良好的工程实践），但实测表明位运算 vs 除法的差异在实际IO场景中可忽略不计（详见2.1节实验验证）。真正影响条带化性能的是条带数（并行度）和chunk大小与IO模式的匹配，应尽量让所有可用磁盘都参与条带化，并让文件系统参数与之对齐

3. **RAID5写性能的核心**是尽量触发Full Stripe Write，避免RMW：
   - 增大stripe_cache_size，提高写合并概率
   - 应用层以stripe大小对齐写入
   - 利用plug机制批量提交IO

4. **RAID5适合顺序写**，不适合随机小写——这不是配置问题，而是RAID5算法的本质决定的。对于随机小写密集的场景，应考虑使用硬件RAID（带写缓存）、RAID10，或上层软件RAID（Ceph、ZFS）

5. **重建期间注意限速**，通过`speed_limit_max`保护正常业务IO

希望本文能帮助大家建立起从源码到实践的理解框架，在遇到具体问题时知道从哪里入手分析。

---

大家好，我是Zorro！

如果你喜欢本文，欢迎在微博上搜索"orroz"关注我，地址是：https://weibo.com/orroz

大家也可以在微信上搜索：**Linux系统技术** 关注我的公众号。

我的所有文章都会沉淀在我的个人博客上，地址是：https://zorrozou.github.io/

欢迎使用以上各种方式一起探讨学习，共同进步。
