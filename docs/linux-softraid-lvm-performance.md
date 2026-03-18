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

Linux的LVM和软RAID（md）虽然面向用户的工具链不同（lvm2 vs mdadm），但在内核IO路径上都依赖同一个框架：**Device Mapper（DM）**。

DM的核心思想是：将一个虚拟块设备的IO请求，通过"映射表（mapping table）"转换后分发给一个或多个底层真实块设备。这个转换过程由各种**target**（目标模块）实现，常见的有：

| target类型 | 实现文件 | 用途 |
|---|---|---|
| `linear` | `dm-linear.c` | 线性映射，LVM基本块 |
| `striped` | `dm-stripe.c` | 条带化，LVM条带卷 |
| `raid` | `dm-raid.c` | 软RAID，底层封装md |
| `thin` | `dm-thin.c` | 精简置备 |
| `cache` | `dm-cache-target.c` | 块缓存 |

从IO路径的角度看，一个写请求进入虚拟块设备后的流程如下：

```
用户进程 write()
    ↓
VFS / 页缓存
    ↓
通用块层（submit_bio）
    ↓
dm_submit_bio()          ← DM框架入口
    ↓
target->map()            ← 例如 stripe_map()
    ↓
bio重定向到底层设备
    ↓
底层块设备驱动
```

关键函数是`dm_submit_bio()`，它从`dm.c`接收bio，查找映射表，调用对应target的`.map`回调，完成地址转换后将bio提交给底层设备。整个过程是同步的、无锁的（通过RCU保护映射表），这使得DM框架本身的开销非常小。

```c
/* dm.c: DM的bio入口 */
static void dm_submit_bio(struct bio *bio)
{
    struct mapped_device *md = bio->bi_bdev->bd_disk->private_data;
    ...
    map = dm_get_live_table_fast(md); /* RCU读锁，极低开销 */
    ...
    /* 调用target的map函数完成地址转换 */
    r = dm_map_bio(tio);
    ...
}
```

理解了这个框架，我们就可以分别看LVM条带化和软RAID的具体实现。

---

## 二、LVM条带化（dm-stripe）的性能分析

### 2.1 条带化的核心：stripe_map_sector

LVM条带卷对应的target是`striped`，实现在`drivers/md/dm-stripe.c`。其核心数据结构是：

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

这里有一个很有意思的优化细节：**当chunk_size和stripes数量都是2的幂时，内核用位运算（`&`和`>>`）代替除法（`sector_div`）来完成取模和除法运算**。这就是`chunk_size_shift`和`stripes_shift`这两个字段的意义——在条带化配置时预计算好移位量，在IO热路径中就不需要做昂贵的除法了。

这也给我们提供了第一个调优建议：

> **建议1：创建LVM条带卷时，条带数量（`-i`）和chunk大小（`-I`）都应该使用2的幂。**
>
> 例如使用2、4、8块盘条带化，chunk大小使用64KB、128KB、256KB，而不是3块盘、96KB这样的非2的幂配置。非2的幂会导致内核在每次IO时执行除法操作，在极高IOPS场景下可能产生额外开销。

#### 实验验证：位运算 vs 除法到底差多少？

上面的分析是从源码角度给出的建议，那么问题来了——这个优化在实际IO场景中，到底能带来多大的性能差异？是显著差异还是理论上的差异？我们需要用数据说话。

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

### 2.2 io_hints：告诉上层最优IO大小

`dm-stripe.c`实现了`stripe_io_hints()`：

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

`io_opt`（optimal IO size）告诉上层文件系统：每次IO的最优大小是`chunk_size × 条带数`，即一个完整的RAID条带大小。如果IO大小等于这个值，每个底层设备都能被均匀地访问一次，并发度最高，效率最优。

> **建议2：创建文件系统时，`stripe_unit`和`stripe_width`参数应匹配LVM条带配置。**
>
> 以ext4为例，如果创建了4块盘、chunk=128KB的条带卷：
> ```bash
> # chunk_size=128K, stripes=4, stripe_width=512K
> mkfs.ext4 -E stride=32,stripe_width=128 /dev/vg/lv_stripe
> # stride = chunk_size / block_size = 128K / 4K = 32
> # stripe_width = stride * 条带数 = 32 * 4 = 128
> ```
> 这样ext4在分配block group和做预读时就会按条带对齐，避免跨条带的读-改-写。

### 2.3 条带化的适用场景

由于`dm-stripe`本身没有奇偶校验计算，IO路径极短，适合**对写性能要求高、不需要冗余保护**的场景，如：
- 临时数据目录（日志、缓存）
- 已有上层冗余保证的分布式存储底层
- 数据库的临时表空间

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

当写操作不是FSW时，RAID5有两种策略：

**RMW（Read-Modify-Write）**：
1. 读出当前数据块（old data）
2. 读出当前奇偶校验块（old parity）
3. 计算新奇偶：`new_parity = old_parity XOR old_data XOR new_data`
4. 写入新数据和新奇偶
- 代价：2次读（data + parity）+ 2次写
- 适合：修改的数据块占少数时

**RCW（Reconstruct Write）**：
1. 读出所有未被修改的数据块
2. 利用所有新旧数据重新计算奇偶
3. 写入新数据和新奇偶
- 代价：(N-1-修改块数)次读 + 写
- 适合：修改的数据块占多数时（超过一半）

内核根据修改的数据块数量选择策略：

```c
/* raid5.c: 选择RMW还是RCW */
/*
 * 如果修改的数据盘数 <= 数据盘总数/2，使用RMW（读更少的块）
 * 否则使用RCW（重建更高效）
 */
```

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

Linux块层有一个plug/unplug机制——先"塞住"IO不提交，积攒一批后再一起提交（unplug），目的是增加IO合并机会。RAID5专门实现了自己的plug机制：

```c
/* raid5.c */
struct raid5_plug_cb {
    struct blk_plug_cb  cb;
    struct list_head    list;   /* 积攒的stripe列表 */
    struct list_head    temp_inactive_list[NR_STRIPE_HASH_LOCKS];
};

static void raid5_unplug(struct blk_plug_cb *blk_cb, bool from_schedule)
{
    struct raid5_plug_cb *cb = container_of(
        blk_cb, struct raid5_plug_cb, cb);
    /* 将所有积攒的stripe一次性提交处理 */
    ...
    dispatch_bio_list(&tmp);
}
```

当上层以plug方式提交IO时，RAID5会把相关stripe积攒在`raid5_plug_cb`的链表里，等到unplug时一起处理，这样同一个stripe上的多个IO就有机会合并，尽量触发FSW。

> **建议5：应用层使用`fio`测试时加上`--io_submit_mode=offload`或使用libaio/io_uring批量提交，可以给RAID5更多的IO合并机会，提升顺序写性能。**

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

8个哈希锁将stripe按sector地址分散到不同的锁桶，减少多线程并发访问时的锁竞争。对于高并发随机写场景，这是一个重要的可扩展性设计。

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
