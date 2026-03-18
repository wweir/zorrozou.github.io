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
