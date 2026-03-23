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

#### 实验验证：文件系统对齐参数到底影响多大？

前面的**建议2**说"stride和stripe_width参数应匹配LVM条带配置"。那么，**如果不匹配，性能到底差多少？**在生产环境中，运维人员经常会遇到这样的疑问：格式化时忘了指定stride参数，或者指定了错误的值，需不需要重新格式化？

为了回答这个问题，我们设计了一组完整的对照实验，分别测试EXT4和XFS在不同对齐配置下的性能差异。

**测试环境**

- 腾讯云CVM，4块云硬盘通过`dmsetup`创建stripe设备（4条带，chunk=128K）
- 操作系统：Ubuntu 22.04，内核 5.15
- 每种配置测试8种负载：随机读/写4K（DIO和Buffered）、顺序读/写1M（DIO和Buffered）
- 每种负载跑3轮取平均，runtime=60s，size=10G
- DIO随机IO：bs=4k, iodepth=128, numjobs=4
- DIO顺序IO：bs=1M, iodepth=16, numjobs=1
- Buffered IO：bs对应负载类型, iodepth=1, numjobs=1

**EXT4测试组设计**

| 组 | 配置说明 | stride | stripe_width |
|---|---------|--------|-------------|
| E1 | 正确对齐 | 32 (128K/4K) | 128 (32×4) |
| E2 | 自动探测（不指定参数） | auto | auto |
| E3 | 禁用对齐（stride=0） | 0 | 0 |
| E4 | stride偏大（对应256K chunk） | 64 | 256 |
| E5 | stripe_width错误（对应3条带而非4） | 32 | 96 |
| E6 | 完全不对齐（随意值） | 17 | 68 |

**XFS测试组设计**

| 组 | 配置说明 | su | sw |
|---|---------|----|----|
| F1 | 正确对齐 | 128k | 4 |
| F2 | 自动探测（不指定参数） | auto | auto |
| F3 | 禁用对齐（noalign） | noalign | 0 |
| F4 | su错误（96k，不匹配128k chunk） | 96k | 4 |
| F5 | sw错误（3条带，不匹配实际4条带） | 128k | 3 |
| F6 | 完全不对齐（随意值） | 80k | 5 |

**EXT4 测试结果**

| 配置 | 随机读4K DIO<br>(IOPS) | 随机写4K DIO<br>(IOPS) | 顺序读1M DIO<br>(MB/s) | 顺序写1M DIO<br>(MB/s) | 顺序读1M BUF<br>(MB/s) | 顺序写1M BUF<br>(MB/s) | 随机读4K BUF<br>(IOPS) | 随机写4K BUF<br>(IOPS) |
|------|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| **E1 正确对齐** | **2920** | **3160** | **1063** | **1046** | **1010** | **767** | **451** | **39468** |
| E2 自动探测 | 2897 | 3030 | 1060 | 1045 | 1019 | 765 | 456 | 39706 |
| E3 禁用对齐 | 2893 | 3163 | 1058 | 1042 | 1021 | 765 | 457 | 40485 |
| E4 stride偏大 | 2906 | 3037 | 1057 | 1043 | 1018 | 764 | 468 | 40230 |
| E5 sw错误 | 2894 | 3163 | 1057 | 1037 | 1023 | 766 | 471 | 39278 |
| E6 完全不对齐 | 2888 | 2940 | 1062 | 1043 | 1014 | 764 | 442 | 39058 |

**EXT4 相对正确对齐（E1）的性能偏差**

| 配置 | 随机读4K DIO | 随机写4K DIO | 顺序读1M DIO | 顺序写1M DIO | 顺序写1M BUF | 随机写4K BUF |
|------|:--:|:--:|:--:|:--:|:--:|:--:|
| E2 自动探测 | -0.8% | **-4.1%** | ≈0% | ≈0% | ≈0% | +0.6% |
| E3 禁用对齐 | -0.9% | ≈0% | ≈0% | ≈0% | ≈0% | +2.6% |
| E4 stride偏大 | ≈0% | -3.9% | -0.5% | ≈0% | ≈0% | +1.9% |
| E5 sw错误 | -0.9% | ≈0% | -0.6% | -0.9% | ≈0% | ≈0% |
| E6 完全不对齐 | -1.1% | **-7.0%** | ≈0% | ≈0% | ≈0% | -1.0% |

EXT4的结果**出乎意料**：无论怎样设置stride/stripe_width——甚至完全不设置——性能差异都非常小。除了E6（完全不对齐）在DIO随机写上有约7%的下降外，其余所有指标的差异都在±4%以内，大部分在±2%以内，属于正常的测量波动范围。

**为什么EXT4对这些参数不敏感？** EXT4的`stride`和`stripe_width`参数主要影响的是**元数据布局**——block group的起始位置、inode table的对齐、预分配策略。当我们用fio测试大文件IO时，IO路径几乎完全是数据区域内的连续读写，不涉及元数据操作。换句话说，**这些参数优化的是"文件创建、目录遍历、元数据密集"的场景**，而不是"对已有大文件的顺序/随机IO"的场景。

在生产环境中，如果你的工作负载是大量小文件的创建和删除（如邮件服务器、编译系统），stride参数的影响可能会大得多。但对于数据库、大文件存储这类负载，EXT4的对齐配置优先级较低。

**XFS 测试结果**

| 配置 | 随机读4K DIO<br>(IOPS) | 随机写4K DIO<br>(IOPS) | 顺序读1M DIO<br>(MB/s) | 顺序写1M DIO<br>(MB/s) | 顺序读1M BUF<br>(MB/s) | 顺序写1M BUF<br>(MB/s) | 随机读4K BUF<br>(IOPS) | 随机写4K BUF<br>(IOPS) |
|------|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| **F1 正确对齐** | **2864** | **3193** | **1061** | **1055** | **1020** | **827** | **463** | **101080** |
| F2 自动探测 | 2683 | 3144 | 1057 | 1053 | 1019 | 819 | 467 | 99662 |
| F3 禁用对齐 | 2611 | 3077 | 1065 | 1060 | 1020 | 807 | 450 | 92452 |
| F4 su错误 | 2631 | 3253 | 1060 | 1052 | 1016 | 824 | 447 | 100508 |
| F5 sw错误 | 2609 | 3187 | 1065 | 1051 | 1020 | 814 | 462 | 100324 |
| F6 完全不对齐 | 2578 | 3289 | 1067 | 1054 | 1016 | 822 | 437 | 100383 |

**XFS 相对正确对齐（F1）的性能偏差**

| 配置 | 随机读4K DIO | 随机写4K DIO | 顺序读/写1M DIO | 顺序写1M BUF | 随机写4K BUF |
|------|:--:|:--:|:--:|:--:|:--:|
| F2 自动探测 | **-6.3%** | -1.5% | ≈0% | -0.9% | -1.4% |
| F3 禁用对齐 | **-8.8%** | **-3.6%** | ≈0% | **-2.4%** | **-8.5%** |
| F4 su错误 | **-8.1%** | +1.9% | ≈0% | ≈0% | -0.6% |
| F5 sw错误 | **-8.9%** | ≈0% | ≈0% | -1.6% | -0.7% |
| F6 完全不对齐 | **-10.0%** | +3.0% | ≈0% | -0.6% | -0.7% |

XFS的表现与EXT4**截然不同**。在随机读4K DIO场景下，所有非正确对齐的配置都出现了**6%~10%的性能下降**，其中F6（完全不对齐）达到了**-10.0%**，是所有组中最大的偏差。F3（禁用对齐）在随机写4K Buffered IO上也有8.5%的下降。值得注意的是，F6（su=80k,sw=5）虽然在随机读上损失最大，但在随机写DIO上反而有+3.0%的"正向"偏差——这并非性能提升，而是由于错误的su参数改变了分配模式，在某些写入路径上碰巧减少了RMW操作，属于不可预测的随机波动。

**XFS为什么更敏感？** 这与XFS的内部结构有关。XFS使用Allocation Group（AG）来组织数据分配，`su`和`sw`参数直接影响AG的大小和数据块的对齐位置：

1. **AG大小对齐**：XFS会根据`sunit`和`swidth`调整AG的大小，使其与条带边界对齐。如果参数错误，AG边界会落在条带中间，导致跨AG操作更容易发生跨条带IO。

2. **分配策略**：XFS的分配器（`xfs_alloc_vextent`）会尝试将数据块对齐到`sunit`边界。如果`sunit`与实际chunk不匹配，对齐反而会制造不对齐——数据块以为自己对齐了（相对于错误的su），实际在底层条带设备上是跨chunk的。

3. **预读和预分配**：XFS使用`swidth`来决定speculative preallocation的大小，错误的`swidth`会让预分配跨越条带边界。

不过值得注意的是，**顺序大IO（1M）在所有配置下都不受影响**（±1%以内）。这是因为1M的IO远大于128K的chunk_size，无论是否对齐，都会被DM层拆分为多个完整的chunk IO，对齐与否对最终的拆分结果影响微乎其微。

**EXT4 vs XFS 正确对齐配置对比**

一个额外的发现是两个文件系统在相同条带设备上的性能差异：

| 负载 | EXT4 (E1) | XFS (F1) | 差异 |
|------|:---------:|:--------:|:----:|
| 随机读4K DIO | 2,920 IOPS | 2,864 IOPS | EXT4 +2% |
| 随机写4K DIO | 3,160 IOPS | 3,193 IOPS | XFS +1% |
| 顺序读1M DIO | 1,063 MB/s | 1,061 MB/s | 持平 |
| 顺序写1M DIO | 1,046 MB/s | 1,055 MB/s | XFS +1% |
| 顺序写1M BUF | 767 MB/s | **827 MB/s** | **XFS +8%** |
| **随机写4K BUF** | **39,468 IOPS** | **101,080 IOPS** | **XFS +156%** 🔥 |

最显著的差异是Buffered随机写4K：XFS是EXT4的**2.56倍**。这并非对齐问题，而是文件系统本身的设计差异——XFS的delayed allocation和预分配策略对高并发小写场景有极大优势。

**实验结论**

1. **EXT4的stride/stripe_width参数对IO性能影响极小**（绝大部分场景<2%），该参数主要优化元数据布局。如果你在生产环境中发现EXT4格式化时忘了指定stride，通常**不需要为此重新格式化**。

2. **XFS的su/sw参数对随机小IO有可观测的影响**（约-6%~-10%），特别是随机读场景。完全不对齐（F6）的随机读下降达10%。在XFS上应该认真配置这些参数，而且要特别注意**自动探测并不可靠**（F2自动探测也下降了6.3%），建议始终手动指定。

3. **顺序大IO完全不受对齐影响**——无论EXT4还是XFS，1M顺序读写在所有配置下性能一致。这是因为大IO天然会跨越多个chunk，对齐参数只影响分配边界，不影响大块连续IO的实际路径。

4. 如果你的工作负载以**随机小IO为主**（如数据库OLTP），且使用XFS，那么正确配置su/sw有约**6%~10%的收益**。如果以**顺序大IO为主**（如大数据处理），对齐参数几乎不重要。

> **更新建议2**：创建文件系统时，`stride`/`stripe_width`（EXT4）或`su`/`sw`（XFS）参数应匹配LVM条带配置。**XFS尤其重要**（可带来约6%~10%的随机IO改善），EXT4影响较小但作为良好实践仍建议配置。如果已经格式化但参数不对，EXT4一般不需要重做，XFS在随机IO场景下建议重做。自动探测（不指定参数）在某些设备拓扑下可能不准确，始终建议手动指定。

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

#### 实验验证：stripe_cache_size 到底影响多大？

为了验证 stripe_cache_size 对不同工作负载的实际影响，我们在5盘RAID5阵列（chunk_size=256K, XFS, su=256k/sw=4正确对齐）上，测试了5个不同的cache值（256/1024/4096/8192/16384）在6种负载下的性能表现。每个测试60秒，每组3轮取平均值。

**测试结果汇总（IOPS）：**

| 负载 | cache=256 | cache=1024 | cache=4096 | cache=8192 | cache=16384 | 偏差范围 |
|:---|:--:|:--:|:--:|:--:|:--:|:--:|
| RandWrite 4K DIO (deep, iodepth=128×4jobs) | 610 | 581 | 585 | 618 | 602 | -4.8%~+1.3% |
| RandWrite 4K DIO (shallow, iodepth=1×1job) | 123 | 124 | 129 | 166 | 201 | 见注 |
| SeqWrite 1M DIO | 216 | 214 | 211 | 215 | 220 | -2.5%~+1.7% |
| RandRead 4K DIO (iodepth=128×4jobs) | 2280 | 2308 | 2302 | 2346 | 2273 | ±3% |
| SeqRead 1M DIO | 1035 | 1034 | 1034 | 1035 | 1034 | ±0.1% |
| RandRW 4K 70/30 (iodepth=64×4jobs) | 835R+362W | 799R+346W | 833R+360W | 820R+355W | 824R+357W | ±5% |

**注：** shallow写（iodepth=1）在大cache值下出现较大波动（cache=16384三轮分别为185/293/126 IOPS），属于低IOPS场景下的统计噪音，不具备参考意义。

**实验结论：**

1. **核心发现：在SATA机械硬盘环境下，stripe_cache_size 对性能的影响远小于理论预期。** 即便是RAID5最痛点场景——高并发随机小写（iodepth=128×4jobs），从默认值256增大到16384，IOPS变化仅在±5%范围内波动，没有出现预期中的显著提升。

2. **原因分析：** 这与测试环境使用的SATA机械硬盘特性有关。机械硬盘的IO延迟在毫秒级，单盘随机IOPS天花板很低（通常100~200 IOPS），底层设备本身就是瓶颈。在这种场景下，IO到达速率受限于设备处理能力，stripe cache中同时存在的待处理stripe数量远低于cache上限——即使默认值256也足够容纳所有活跃stripe。换句话说，**当后端设备足够慢、并发IO数量本身就不高时，增大stripe_cache_size 不会带来额外收益**，因为cache从来就没满过。

3. **什么环境下这个参数才重要？** 理论上的关键条件是：**后端设备足够快，使得IO并发度能够撑满stripe cache**。但我们的 ramdisk 追加实验表明，即便后端设备延迟接近零（纳秒级），cache 同样没有成为瓶颈——此时瓶颈转移到了内核 raid5d 单线程和锁竞争上。实际上，只有在后端设备速度恰好快到让 cache 成为瓶颈、但又没有快到触及 raid5d 处理上限的"甜蜜区间"，增大该参数才可能有效。对于绝大多数场景（SATA HDD、SSD、甚至 NVMe），默认值 256 通常已经足够。如果确实怀疑 cache 是瓶颈，建议先观察 `/proc/slabinfo` 中 `raid5-*` 条目的 active 数量是否接近 cache 上限，再决定是否调大。

#### 追加实验：用 ramdisk 消除设备瓶颈后的验证

为了排除"后端设备太慢导致看不到效果"的干扰因素，我们使用内核 brd 模块创建 5 块 4GB 的内存虚拟块设备（ramdisk），组建 RAID5 阵列（chunk=256K），直接对裸块设备 `/dev/md0` 进行测试（不挂载文件系统）。ramdisk 的延迟在纳秒级，单盘随机写 IOPS 可达 ~448K，理论上可以将 stripe cache 推到极限。

**测试环境**：5×4GB ramdisk (brd), RAID5, chunk=256K, libaio 引擎（ramdisk 不支持 O_DIRECT），每组30秒×3轮

**Slab 内存使用**（证实 cache 参数确实生效）：

| cache 值 | active stripe_head | total |
|:--:|:--:|:--:|
| 256 | 321 | 559 |
| 1024 | 1333 | 2054 |
| 4096 | 4576 | 4576 |
| 8192 | 8619 | 8619 |
| 16384 | 16783 | 16783 |

**测试结果汇总（IOPS，3轮平均）：**

| 负载 | cache=256 | cache=1024 | cache=4096 | cache=8192 | cache=16384 | 最大波动 |
|:---|:--:|:--:|:--:|:--:|:--:|:--:|
| RandWrite 4K (deep, iodepth=128×4jobs) | 674.8K | 670.1K | 645.3K | 631.2K | 662.9K | 6.9% |
| RandWrite 4K (shallow, iodepth=1×1job) | 161.4K | 161.4K | 159.1K | 159.7K | 155.7K | 3.7% |
| SeqWrite 1M | 948 | 940 | 914 | 908 | 931 | 4.5% |
| RandRead 4K (iodepth=128×4jobs) | 1.17M | 1.16M | 1.16M | 1.16M | 1.15M | 1.7% |
| SeqRead 1M | 3430 | 3410 | 3391 | 3359 | 3345 | 2.6% |
| RandRW 4K 70/30 (iodepth=64×4jobs) | 449K R | 450K R | 463K R | 440K R | 436K R | 6.0% |

**ramdisk 实验结论：**

结果出乎意料——**即使在设备延迟接近零的 ramdisk 环境中，stripe_cache_size 仍然没有带来显著的性能提升**。所有负载的最大波动仅 6.9%（高并发随机写），且趋势并非"越大越好"，反而是默认值 cache=256 在多数场景下性能最高。

这揭示了一个更深层的事实：**stripe_cache_size 不是简单的"增大就好"——真正的瓶颈在内核 md/raid5 子系统本身**。具体原因：

- **raid5d 单线程架构**：内核 RAID5 的核心处理逻辑由单个内核线程 `raid5d` 驱动，所有 stripe_head 的状态机转换、校验计算调度、IO 提交都在这个线程中串行完成。无论 cache 多大，处理吞吐量受限于这个单线程的处理速率。
- **stripe_head 锁竞争**：每个 stripe_head 的处理涉及 `sh->lock` 自旋锁和 `conf->device_lock`，高 IOPS 下锁竞争成为瓶颈。增大 cache 意味着更多 stripe_head 参与竞争，并不能提高吞吐。
- **cache 增大的副作用**：更多的 stripe_head 意味着 `raid5d` 线程需要遍历更长的 handle_list，每次唤醒的处理开销增加，可能反而略微降低性能（这解释了 cache=8192/16384 下随机写 IOPS 反而略低于 cache=256 的现象）。

这个实验彻底说明了：**stripe_cache_size 的效果取决于 cache 是否真正成为瓶颈**。在 SATA HDD 上 cache 没满过所以无效；在 ramdisk 上虽然 IO 速率极高，但内核处理能力（raid5d 单线程 + 锁竞争）才是真正的天花板，cache 同样不是瓶颈。理论上只有在后端设备恰好快到让 cache 满、但又没快到让 raid5d 成为瓶颈的"甜蜜区间"，增大这个参数才可能有效——而这在实践中是一个非常窄的窗口。

4. **读操作和顺序IO完全不受影响**，符合理论预期——stripe cache只参与写路径的合并优化，对读路径无作用；顺序大IO天然是FSW，不需要cache合并。

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

#### 实验验证：plug/unplug对RAID5写性能的实际影响

增大iodepth能提升RAID5写性能，但这个提升并不全是plug/FSW的功劳——更大的队列深度本身就带来了设备级并行处理等好处。为了**隔离plug/FSW机制的净贡献**，我们设计了一组关键的对照实验：在同一批磁盘上，分别测试单盘和RAID5在不同iodepth下的4K顺序写性能。

核心逻辑：**单盘没有stripe概念，不存在FSW/RMW路径差异**，因此单盘上iodepth增大带来的提升 = 纯粹的设备级并行处理能力；而RAID5上的提升 = 设备并行 + **plug/FSW效果** + RAID层底层盘IO合并。两者的差值就是plug/FSW（及其间接效果）的净贡献。

**测试环境**：5×10.9TB SATA HDD（virtio-blk，write_cache=write back），RAID5 chunk=256K，libaio引擎（psync作为对照），Direct IO，每组60秒×3轮取平均值。

**实验1：单盘 vs RAID5 iodepth扩展性对比**

| iodepth | 单盘顺序写(IOPS) | 单盘 vs d=1 | RAID5顺序写(IOPS) | RAID5 vs d=1 | **差值（plug/FSW净贡献）** |
|:--:|:--:|:--:|:--:|:--:|:--:|
| 1 | 15,614 | 1.0x | 1,668 | 1.0x | — |
| 2 | 25,313 | 1.6x | 2,370 | 1.4x | -0.2x |
| 4 | 35,914 | 2.3x | 2,830 | 1.7x | -0.6x |
| 8 | 33,095 | 2.1x | 3,532 | 2.1x | 0.0x |
| 16 | 32,967 | 2.1x | 4,621 | 2.8x | +0.7x |
| 32 | 39,072 | 2.5x | 8,175 | **4.9x** | **+2.4x** |
| 64 | 45,281 | 2.9x | 13,195 | **7.9x** | **+5.0x** |
| 128 | 54,372 | 3.5x | 18,736 | **11.2x** | **+7.7x** |

这组对比数据非常有说服力：

1. **单盘顺序写从d=1到d=128提升了3.5倍，而RAID5提升了11.2倍**。两者的差值（11.2x - 3.5x = **7.7倍**）就是plug/FSW机制（及其间接效果）对RAID5的净贡献。换言之，RAID5从iodepth增大中获得的11.2倍提升中，约31%来自通用的设备级并行处理能力，**约69%来自plug积攒触发FSW以及由此引发的底层IO优化**。

2. **plug/FSW的效果在iodepth=16之后才开始显现**。d≤8时，RAID5的提升倍数与单盘几乎相同（都在2.1x左右），说明这个区间的性能改善主要来自设备并行处理能力。但从d=16开始，RAID5的曲线加速上扬——d=32时RAID5已达4.9倍而单盘仅2.5倍，差值2.4倍。这恰好对应着plug积攒的连续IO开始能覆盖一个完整stripe（4个数据块）的阈值。

3. **单盘d=1时IOPS就有15,614**，而RAID5 d=1仅1,668（差了9.4倍）。这个巨大差距直接体现了RMW的代价：写一个4K块到RAID5上，在d=1的逐个提交模式下几乎100%触发RMW，需要额外的读旧数据+读旧校验+计算+写数据+写校验，磁盘IO量是单盘的数倍。

**深入分析：单盘3.5倍提升的真实来源**

为了准确归因性能提升，我们对fio输出的`disk_util`统计进行了详细审计。一个关键发现是：**单盘顺序写在所有iodepth下，`write_merges`都为0**：

| iodepth | 单盘IOPS | disk write_ios | write_merges | clat(μs) |
|:--:|:--:|:--:|:--:|:--:|
| 1 | 13,629 | 817,268 | **0** | 52 |
| 4 | 37,197 | 2,230,882 | **0** | 99 |
| 32 | 35,918 | 2,154,795 | **0** | 879 |
| 128 | 58,735 | 3,521,176 | **0** | 2,172 |

fio报告的`total_ios`与内核`disk_util`的`write_ios`比率为1.00——**每一个4K IO都原封不动地下发到了设备，块层没有发生任何IO合并**。

那单盘3.5倍提升从哪来？看延迟数据就清楚了：d=1时单个IO的`clat`仅52μs，d=128时增长到2,172μs（增长了42倍），但因为队列中同时有128个IO在飞行，吞吐量仍然在涨（`IOPS ≈ iodepth / clat`）。这是典型的**设备级并行处理**——virtio-blk后端（宿主机/CBS存储）可以同时处理多个请求，虽然每个请求的延迟变大了，但总吞吐量随并发数增加而提升。

> **技术要点**：在本测试环境下，底层设备是virtio-blk云盘而非物理SATA盘。云盘的write_cache设置为write back，且mq-deadline调度器在4K粒度的顺序IO下不做合并（因为这些IO已经是最小粒度了）。性能的提升纯粹来自设备后端的并发处理能力，不涉及块层IO合并。

**RAID5底层盘的IO合并——FSW的间接效果**

与单盘形成鲜明对比的是，RAID5测试中底层各盘出现了**大量的IO合并**：

| RAID5 iodepth | vdb write_merges | vdb read_merges | vdb util |
|:--:|:--:|:--:|:--:|
| 1 | 0 | 0 | 31.9% |
| 2 | 27,560 | 13,619 | 31.4% |
| 4 | 35,158 | 15,349 | 29.6% |
| 16 | 89,762 | 39,797 | 34.4% |
| 64 | 242,122 | 111,426 | 44.8% |
| 128 | **331,364** | **195,908** | 44.4% |

d=1时底层盘完全没有merge，而d=128时write_merges高达33万次。这些合并是怎么来的？原因是FSW模式下，RAID5一次性向底层盘提交整个stripe的数据——同一stripe的chunk在同一块底层盘上是**地址连续的**（chunk=256K对齐），多个相邻stripe的chunk操作被底层mq-deadline调度器识别为连续IO并合并。这是**FSW的间接效果**——只有当plug积攒足够多的IO触发FSW后，底层盘才会看到这种连续的大块写模式，进而触发合并。

**这意味着RAID5的11.2倍提升有一个"叠加效应"**：plug积攒 → 触发FSW → FSW向底层盘下发连续地址的chunk写 → 底层盘的调度器将这些连续chunk合并为更大的IO → 进一步提升性能。

**实验2：IO引擎对比——同步逐个提交 vs 异步批量提交**

为了进一步验证，我们固定4K顺序写，对比不同IO引擎和队列深度的组合：

| IO引擎 | iodepth | IOPS | BW(MB/s) | Lat(μs) |
|:--|:--:|:--:|:--:|:--:|
| psync（同步逐个写） | 1 | 1,699 | 6.6 | 591 |
| libaio（异步引擎） | 1 | 1,824 | 7.1 | 545 |
| libaio（异步引擎） | 32 | 8,545 | 33.4 | 3,859 |
| libaio（异步引擎） | 64 | 12,785 | 49.9 | 5,104 |

1. **psync(d=1) vs libaio(d=32)：IOPS差距达5.0倍**（1,699 vs 8,545）。这个提升包含了设备并行化和plug/FSW两方面因素。

2. **同为iodepth=1时，psync和libaio性能几乎相同**（1,699 vs 1,824，差距仅7%）。这证明了**是队列深度而非IO引擎决定了性能**——当两者都只能逐个提交IO时，plug在两种引擎下的表现没有区别。

**实验3：单盘随机写——进一步验证设备并行化特性**

作为额外的对照，我们测试了单盘4K随机写在不同iodepth下的表现：

| iodepth | 单盘随机写(IOPS) | vs d=1 |
|:--:|:--:|:--:|
| 1 | 796 | 1.0x |
| 2 | 739 | 0.9x |
| 4 | 731 | 0.9x |
| 8 | 730 | 0.9x |
| 16 | 733 | 0.9x |
| 32 | 731 | 0.9x |
| 64 | 734 | 0.9x |
| 128 | 711 | 0.9x |

**单盘随机写IOPS从d=1到d=128完全没有增长**，始终停在~730附近。这揭示了一个重要的特性：

1. **virtio-blk云盘的设备级并行化是有选择性的**——顺序写能从并发提交中受益（3.5倍提升），但随机写不能。这可能与CBS后端的写入路径有关：顺序写请求即使不在块层合并，在CBS后端存储层也可以被更高效地批量处理（如写入连续的存储chunk）；而随机写地址分散，后端无法做类似优化。

2. 需要注意的是，**单盘随机写不增长只说明设备并行化对随机写无效**，不能反推"对顺序写也无效"——事实上单盘顺序写在`write_merges=0`的情况下仍获得了3.5倍提升，证明设备级并行处理确实存在，只是对顺序和随机IO表现不同。

**实验4：RAID5 顺序写 vs 随机写——验证FSW的选择性效果**

plug对顺序写和随机写的提升效果有本质差异：顺序写的连续块天然命中同一stripe，plug稍加积攒就能触发FSW；而随机写分散在不同stripe，即使iodepth很大，同一stripe收集满4个数据块的概率仍然很低。

| iodepth | 顺序写 IOPS | 随机写 IOPS | 顺序/随机 | 顺序 vs d=1 | 随机 vs d=1 |
|:--:|:--:|:--:|:--:|:--:|:--:|
| 1 | 1,693 | 88 | **19x** | 1.0x | 1.0x |
| 4 | 3,042 | 369 | 8x | 1.8x | 4.2x |
| 16 | 5,348 | 554 | 10x | 3.2x | 6.3x |
| 32 | 11,665 | 733 | **16x** | 6.9x | 8.3x |
| 64 | 13,613 | 637 | **21x** | 8.0x | 7.2x |
| 128 | 17,945 | 796 | **23x** | 10.6x | 9.0x |

这里有一个值得注意的现象：**RAID5随机写从d=1的88 IOPS到d=128的796 IOPS，提升了9倍**，而单盘随机写在同样的iodepth变化下完全不增长。这说明RAID5随机写的提升来源既不是设备并行化（单盘随机写证伪了），也不是FSW（随机IO几乎不可能命中同一stripe），而是**RAID层的并发处理**——多个RMW操作可以同时在不同stripe上进行，md_raid5d线程可以并行调度这些不相关的stripe操作，提高了RAID层的管道利用率。

**关于数据波动的说明**

需要指出的是，单盘测试中部分iodepth的三轮数据存在较明显的波动：

```
d=8:  35035, 36108, 28141  （最大差28%）
d=16: 37235, 33808, 27857  （最大差34%）
d=64: 49016, 47544, 39282  （最大差25%）
```

这导致平均值出现了**d=4（35,914）> d=8（33,095）> d=16（32,967）**的反常现象。这种波动是云盘环境的固有特征——virtio-blk云盘的后端是共享的CBS存储集群，受宿主机负载和存储后端限速策略影响，中等iodepth区间的性能抖动较大。相比之下RAID5各轮数据非常稳定（CV通常<5%），这是因为RAID5的多盘并行天然有平滑效果。但这种波动不影响整体结论——在低iodepth（d≤4）和高iodepth（d≥64）区间，数据都很一致且趋势清晰。

**实验结论：**

1. **plug/FSW机制对RAID5顺序写性能的贡献已被定量分离**。在d=1到d=128的提升中，约31%（3.5倍）来自通用的设备级并行处理能力，**约69%（7.7倍）来自plug积攒触发FSW及其间接效果（底层盘IO合并）**。RAID5的提升存在一个"叠加效应"：plug积攒 → FSW替代RMW → 底层盘连续写入 → 底层调度器合并IO，形成了多层优化的级联放大。

2. **plug/FSW效果的启动阈值在iodepth=16左右**，在d=32时进入快速增长区，d=64后趋于饱和。这对应着plug积攒的连续IO开始能覆盖一个完整stripe的4个数据块。对于实际应用，**使用libaio引擎配合iodepth≥32**即可充分利用plug机制。

3. **RAID5随机写无法从plug/FSW中受益**，但仍能从RAID层并发处理中获得约9倍提升。不过其绝对性能（d=128时仅796 IOPS）与顺序写（17,945 IOPS）相差23倍，再次印证了**4K随机写是RAID5的性能噩梦**。

4. **IO引擎的选择不如队列深度重要**。psync和libaio在d=1时性能几乎相同；真正拉开差距的是能否一次提交多个IO。

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

# 设置预读为8MB（16384个512字节扇区）
blockdev --setra 16384 /dev/md0

# 也可以直接通过sysfs设置（单位：KB）
cat /sys/block/md0/queue/read_ahead_kb
echo 8192 > /sys/block/md0/queue/read_ahead_kb
```

read_ahead的原理是：当内核检测到应用在做**顺序读**时，会在page cache路径中**提前**把后续尚未请求的数据也从磁盘读入page cache。这样当应用的下一个read系统调用到来时，数据已经在内存中了，不用再等磁盘IO。

这个机制在**小bs顺序读**场景下效果最为显著——如果应用每次只read 4KB，没有预读的话每个4KB都要等一次磁盘寻道和传输；有了预读，内核一次读入大量连续数据，后续的大量4KB read都直接命中page cache，延迟从毫秒级降到微秒级。

但read_ahead到底能带来多大的提升？设多少才合适？继续增大是否还有收益？我们用真实SATA机械盘来做个实验。

#### 4.4.1 实验设计

测试设备为单块SATA机械硬盘（/dev/vdb），**不组RAID**，先验证单盘上read_ahead的纯粹效果。

```bash
# 测试参数
设备: /dev/vdb（SATA HDD）
RA值: 0, 4, 16, 64, 256, 1024, 4096, 16384 (KB)
fio参数: runtime=30s, size=2G, offset=10G（避开头部分区元数据）
每组: 3轮取平均值
每次测试前: echo 3 > /proc/sys/vm/drop_caches  # 清空page cache
```

测试负载精心设计了三类：

| 类别 | 负载 | 目的 |
|------|------|------|
| **重点测试** | 顺序buffered读，bs=4K/16K/64K/256K/1M，iodepth=1 | 预读对不同bs顺序读的影响 |
| **对照组1** | 顺序direct读，bs=4K/1M | Direct IO绕过page cache，不受预读影响 |
| **对照组2** | 随机buffered读，bs=4K | 预读对随机IO应该无效 |
| **深队列测试** | 顺序buffered读，bs=4K，iodepth=32 | 预读在高并发下的表现 |

每次测试前清空page cache（`drop_caches`）是**关键设计**——确保每次都是从磁盘冷读，而非命中缓存。这样测到的性能差异才能归因于read_ahead机制本身。

#### 4.4.2 实验结果

**顺序Buffered读：不同bs下的带宽随RA值变化**

| RA值(KB) | 4K BW | 16K BW | 64K BW | 256K BW | 1M BW |
|----------|-------|--------|--------|---------|-------|
| **0** | **59 MB/s** | 63 MB/s | 60 MB/s | 59 MB/s | 58 MB/s |
| **4** | **181 MB/s** | 217 MB/s | 273 MB/s | 273 MB/s | 273 MB/s |
| **16** | **273 MB/s** | 273 MB/s | 273 MB/s | 273 MB/s | 273 MB/s |
| 64 | 273 MB/s | 273 MB/s | 272 MB/s | 272 MB/s | 272 MB/s |
| 256 | 272 MB/s | 272 MB/s | 273 MB/s | 273 MB/s | 273 MB/s |
| 1024 | 272 MB/s | 272 MB/s | 272 MB/s | 272 MB/s | 272 MB/s |
| 4096 | 272 MB/s | 273 MB/s | 273 MB/s | 272 MB/s | 273 MB/s |
| **16384** | **270 MB/s** | 270 MB/s | 270 MB/s | 270 MB/s | 270 MB/s |

数据非常清晰：

1. **RA=0时**，无论bs多大，带宽都锁定在~59 MB/s——这就是这块SATA盘在没有预读加持、逐次同步读时的原始带宽上限。
2. **RA=16KB是"甜点"**，4K顺序读带宽从59 MB/s跳升到273 MB/s，提升**4.6倍**，已经接近该磁盘的物理带宽上限（~276 MB/s，由Direct IO 1M顺序读测得）。
3. **RA=4KB处于"半生不熟"的中间态**——4K顺序读只有181 MB/s（提升3倍但没到极限），而64K以上的bs由于自身请求就足够大，RA=4KB也已打满带宽。
4. **RA从16KB到4096KB（增大256倍），性能几乎没有变化**——一旦预读窗口足够覆盖磁盘延迟，继续增大没有收益。
5. **RA=16384KB（16MB）时性能反而微降0.9%**——过大的预读窗口可能导致预读的数据超出应用实际需要的范围，造成少量浪费。

**4K顺序Buffered读：IOPS与延迟随RA值变化（最能体现预读效果的场景）**

| RA值(KB) | IOPS | 带宽 | 平均延迟 | 相比RA=0 |
|----------|------|------|----------|----------|
| **0** | 15,181 | 59 MB/s | 65.2 μs | 基准 |
| **4** | 46,304 | 181 MB/s | 21.1 μs | **3.1倍** |
| **16** | 69,817 | 273 MB/s | 13.9 μs | **4.6倍** |
| 64 | 69,831 | 273 MB/s | 14.0 μs | 4.6倍 |
| 256 | 69,701 | 272 MB/s | 14.0 μs | 4.6倍 |
| 1024 | 69,697 | 272 MB/s | 13.9 μs | 4.6倍 |
| 4096 | 69,764 | 272 MB/s | 13.9 μs | 4.6倍 |
| **16384** | 69,196 | 270 MB/s | 14.0 μs | 4.6倍（微降） |

#### 4.4.3 对照组验证

光看顺序buffered读的数据飙升，可能有人会问：**性能提升真的来自read_ahead，还是buffered IO本身的page cache机制带来的？**

这个质疑有道理，但可以从数据中排除。关键证据是：RA=0时buffered 4K顺序读和Direct 4K顺序读的性能**几乎完全一致**：

| 测试 | RA=0时 IOPS | RA=0时带宽 |
|------|-------------|------------|
| SeqRead-4K-**buffered** | 15,181 | 59.3 MB/s |
| SeqRead-4K-**direct** | 15,055 | 58.8 MB/s |

两者只差0.8%，说明**RA=0时page cache路径本身没有任何加速效果**——数据都必须从磁盘读，buffered路径只是多了一次page cache的内存拷贝，并不能减少磁盘IO次数。因此，当RA增大后buffered路径的性能飙升（59→273 MB/s），**只能归功于read_ahead把数据提前读入了page cache**。

**Direct IO对照组（不受read_ahead影响）**

| RA值(KB) | 4K Direct BW | 1M Direct BW |
|----------|-------------|--------------|
| 0 | 59.1 MB/s | 276.2 MB/s |
| 4 | 62.1 MB/s | 275.3 MB/s |
| 16 | 63.7 MB/s | 275.4 MB/s |
| 64 | 65.9 MB/s | 275.3 MB/s |
| 256 | 59.7 MB/s | 275.4 MB/s |
| 1024 | 69.5 MB/s | 275.2 MB/s |
| 4096 | 64.2 MB/s | 275.0 MB/s |
| 16384 | 58.8 MB/s | 275.3 MB/s |

Direct IO绕过page cache直接操作磁盘，不受read_ahead影响。4K Direct读在各RA值下波动约±18%（这是机械盘正常的随机抖动），1M Direct读稳定在~275 MB/s（0.4%波动），**确认了read_ahead只对走page cache的buffered IO生效**。

**随机读对照组（预读对随机IO无效）**

| RA值(KB) | RandRead-4K IOPS | 带宽 | 延迟 |
|----------|-------------------|------|------|
| 0 | 201 | 0.8 MB/s | 4,959 μs |
| 4 | 201 | 0.8 MB/s | 4,957 μs |
| 16 | 201 | 0.8 MB/s | 4,954 μs |
| 64 | 201 | 0.8 MB/s | 4,956 μs |
| 256 | 201 | 0.8 MB/s | 4,964 μs |
| 1024 | 201 | 0.8 MB/s | 4,957 μs |
| 4096 | 201 | 0.8 MB/s | 4,965 μs |
| 16384 | 201 | 0.8 MB/s | 4,964 μs |

8组RA值下随机读IOPS**完全不变（201 IOPS）**，延迟也稳定在~4.96ms——这正是SATA机械盘的单次寻道+传输延迟。内核的预读算法能正确识别随机访问模式并关闭预读，不会因为RA值设置得很大就盲目预读。

**深队列测试（iodepth=32）**

| RA值(KB) | SeqRead-4K-d32 IOPS | 带宽 | 延迟 |
|----------|---------------------|------|------|
| 0 | 14,780 | 57.7 MB/s | 67.2 μs |
| 4 | 45,755 | 178.8 MB/s | 21.3 μs |
| 16 | 69,769 | 272.5 MB/s | 13.9 μs |
| 4096 | 69,768 | 272.5 MB/s | 13.9 μs |
| 16384 | 69,207 | 270.3 MB/s | 14.0 μs |

深队列下的趋势与iodepth=1完全一致——预读效果并不会被高并发"掩盖"或"替代"。这是因为fio的iodepth只是内核IO调度层面的并发，而预读发生在page cache层面，两者是不同层次的优化。

#### 4.4.4 结论与调优建议

从实验数据中可以得出以下结论：

1. **read_ahead对SATA机械盘的顺序读性能有决定性影响**。RA=0到RA=16KB，4K顺序buffered读带宽从59 MB/s飙升到273 MB/s（**4.6倍**），延迟从65μs降到14μs。

2. **RA=16KB即可达到最优**，继续增大到16MB也没有额外收益（反而微降0.9%）。这说明对于单块SATA盘，内核默认的128KB read_ahead已经绑绑有余。

3. **bs越大，预读的边际收益越小**。64K及以上的bs自身请求就足够大，即使RA=4KB也已打满磁盘带宽。预读主要帮助的是小bs场景。

4. **Direct IO和随机IO完全不受read_ahead影响**，对照组数据验证了这一点。内核的预读算法能正确识别访问模式，不会对非顺序场景做无用预读。

5. **buffered IO路径本身不会凭空加速**——RA=0时buffered和direct性能一致（59 vs 59 MB/s），page cache只是数据的"容器"，真正的加速来自read_ahead把数据**提前**放入容器。

对于RAID阵列，read_ahead的实际最优值还需要考虑stripe宽度。一个经验法则是将read_ahead设为**chunk_size × 数据盘数**（即一个完整stripe的大小），这样每次预读恰好覆盖所有数据盘的一个完整条带，各盘可以并行读取。例如4块数据盘、chunk=512KB的RAID5，建议read_ahead设为2048KB（512K×4）。

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

2. **LVM条带化**性能好，chunk_size和条带数建议使用2的幂（良好的工程实践），但实测表明位运算 vs 除法的差异在实际IO场景中可忽略不计（详见2.4节实验验证）。真正影响条带化性能的是条带数（并行度）和chunk大小与IO模式的匹配

3. **文件系统对齐参数的影响因文件系统而异**（详见2.5节实验验证）：
   - **EXT4的stride/stripe_width对IO性能影响极小**（<2%），即使完全不配置也几乎没有性能损失。这些参数主要影响元数据布局，而非数据IO路径。如果格式化时忘了指定，通常不需要重做
   - **XFS的su/sw参数对随机小IO有约6%~10%的影响**，完全不对齐时随机读下降达10%。XFS上应始终手动指定这些参数，不要依赖自动探测（自动探测也有约6%的性能损失）
   - **顺序大IO完全不受对齐影响**——无论EXT4还是XFS，1M顺序IO在所有对齐配置下性能一致

4. **RAID5写性能的核心**是尽量触发Full Stripe Write，避免RMW：
   - 增大stripe_cache_size，提高写合并概率。但实测表明**无论是SATA机械硬盘还是延迟接近零的ramdisk，该参数都未能带来显著提升**（≤7%波动）。SATA HDD上cache本身没满过；ramdisk上虽然IO速率极高，但瓶颈在raid5d单线程和锁竞争，cache同样不是瓶颈。理论上只有后端设备速度恰好落在"cache饱和但raid5d未饱和"的窄窗口才有效。**对绝大多数场景，默认值256已足够**（详见3.1节SATA实验与ramdisk追加实验）
   - 应用层以stripe大小对齐写入
   - **利用plug机制批量提交IO**（详见3.5节对照实验）：通过单盘 vs RAID5的对照实验定量分离了plug/FSW的贡献——从d=1到d=128，RAID5顺序写提升11.2倍，而同一块盘的单盘顺序写仅提升3.5倍，**差值7.7倍即为plug/FSW的净贡献（占总提升的69%）**。单盘随机写在全部iodepth下IOPS恒定（~730），进一步证实提升来自RAID层的FSW而非底层设备并行化。**推荐使用libaio/io_uring引擎 + iodepth≥32**

5. **read_ahead对顺序读有决定性影响**（详见4.4节SATA实验验证）：SATA机械盘上，4K顺序buffered读在RA=0到RA=16KB之间带宽从59 MB/s飙升到273 MB/s（**4.6倍**）。RA=16KB即达最优，继续增大到16MB无额外收益。Direct IO和随机读完全不受影响，对照组验证了性能提升确实来自预读而非page cache本身。对于RAID阵列，建议将read_ahead设为chunk_size × 数据盘数（一个完整stripe大小）

6. **RAID5适合顺序写**，不适合随机小写——这不是配置问题，而是RAID5算法的本质决定的。对于随机小写密集的场景，应考虑使用硬件RAID（带写缓存）、RAID10，或上层软件RAID（Ceph、ZFS）

7. **重建期间注意限速**，通过`speed_limit_max`保护正常业务IO

希望本文能帮助大家建立起从源码到实践的理解框架，在遇到具体问题时知道从哪里入手分析。

---

大家好，我是Zorro！

如果你喜欢本文，欢迎在微博上搜索"orroz"关注我，地址是：https://weibo.com/orroz

大家也可以在微信上搜索：**Linux系统技术** 关注我的公众号。

我的所有文章都会沉淀在我的个人博客上，地址是：https://zorrozou.github.io/

欢迎使用以上各种方式一起探讨学习，共同进步。
