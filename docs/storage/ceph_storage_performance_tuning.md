# Ceph 存储性能优化实战

-------------------------------------------------------------------------------------

版权声明：

本文章内容在非商业使用前提下可无需授权任意转载、发布。

转载、发布请务必注明作者和其微博、微信公众号地址，以便读者询问问题和甄误反馈，共同进步。

微博ID：**orroz**

微信公众号：**Linux系统技术**

-------------------------------------------------------------------------------------

## 前言

本文记录了一次在云环境中为某客户进行 Ceph 存储性能优化的实战过程。测试场景涵盖 Rados 和 JuiceFS 两大场景，优化涉及 CPU、网络、虚拟化等多个层面。希望这些优化经验可以为类似场景提供参考。

## 测试用例一：Rados 场景

主要对标数据（包括优化前后的数据）：

| Rados bench | 块大小 | 操作类型 | 优化后 | 优化前 | 基准性能 |
| --- | --- | --- | --- | --- | --- |
| | 1M测试 | 顺序读 | 56784.87 MB/s | 30489.95 MB/s | 48044.32 MB/s |
| | | 顺序写 | 18832.97 MB/s | 13661.08 MB/s | 21062.63 MB/s |
| | 256K测试 | 顺序读 | 50299.99 MB/s | 29633.50 MB/s | 48044.32 MB/s |
| | | 顺序写 | 18440.75 MB/s | 12107.61 MB/s | 20083.13 MB/s |

测试集群为 3 台大规格云服务器作为 Ceph 存储集群，3 台作为 Rados 客户端集群。开始发压后，读操作是 Rados 客户端收包，Ceph 发包。写操作是 Rados 客户端发包，Ceph 因为三台数据要保持强一致性同步，所以既要收包也要发包。

以下是各项优化要点。

### CPU 收包性能瓶颈分析

无论在压测写还是读的时候，从收包方的负载来看，中断的消耗会把单核打满。测试机型有 48 个网卡 IRQ 队列，所有对应核心的 CPU 消耗都达到 100%，明显存在性能瓶颈。

除了 CVM 和裸金属的机型差异外，CVM 环境用的是客户自定义的 Ubuntu 24.04 操作系统，裸金属用的是定制 Linux + 4.x 内核。

经 perf 分析发现，CVM 的主要消耗在内存清零操作上，而裸金属没有这个现象。

### 优化点：init_on_alloc=0

经确认发现，客户的 Ubuntu 镜像的内核 `CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y` 是开着的。在 CVM 的测试环境上，这会导致每次申请内存时都对内存清零，使 IRQ 中断占用 CPU 特别高。这可以通过内核参数 `init_on_alloc=0` 来关闭。关闭后 CVM 环境的收包性能提升明显。而裸金属用的内核 `CONFIG_INIT_ON_ALLOC_DEFAULT_ON is not set`，这个选项本身就是关闭的。

### 优化点：关闭网卡大页分配

另外，可以调整：

```bash
echo 1 > /proc/sys/net/core/high_order_alloc_disable
```

关闭网卡大页分配。在带宽较高时，这会减少内核 buddy allocator 的 spinlock 竞争。实际测试中 CPU 消耗略有下降，但 perf 热点变化不大。

### 优化点：收包校验 Offload

在 perf 分析中，我们还发现 `csum_partial` 占用比率较高。

`ethtool -k` 查看网卡对应的 rx csum、gso 等选项都是开启的。经确认发现，底层智能网卡之前有一个 bug，会造成 csum 本来是错的，但底层未进行计算就认为是对的，传递给子机标记为已校验，然后出现静默错误。之前研发没有修复这个问题，而是把这个标记功能关闭了，所以需要在宿主机上将 csum 选项打开。

修复后 csum 相关处理占比已经消失。

### 优化点：关闭 pvspinlock

另外 perf 中看到 pvspinlock 也是一个优化点，它会影响 `__pv_queued_spin_lock_slowpath` 的处理占比。需要在宿主机上编辑 CVM 的配置文件来关闭它：

```xml
  <features>
    <acpi/>
    <apic/>
    <pae/>
    <hap state='on'/>
    <pvspinlock state='off'/> <!-- 关闭pvspinlock -->
    <ioapic driver='qemu'/>
    <pvsendipi state='on'/>
  </features>
```

另外，虚拟化层面的优化还包括关闭超线程、hlt 透传。pvspinlock 禁用以后 guest 会自动使能 hlt 透传。

### 优化点：IRQ 处理核心隔离以及网络架构相关优化

因为要测试的网络流量比较大，IRQ 处理可能比较繁忙，为了进一步优化 IRQ 处理性能，还将 IRQ 中断处理的核心从调度器中隔离开，方法为添加内核参数：

```bash
isolcpus=<irq_bindcpu_list> nohz_full=<irq_bindcpu_list> rcu_nocbs=<irq_bindcpu_list>
```

其中 `<irq_bindcpu_list>` 为根据实际 NUMA 拓扑选择的 CPU 核心列表。

然后将所有网卡队列的 IRQ 中断绑定到这些核心上：

```bash
#!/bin/bash

# 将网卡队列中断绑定到指定 CPU 核心
# 根据 NUMA 拓扑和网络架构，将 IRQ 分布到两个 NUMA 节点

for dev in $(seq 0 0)
do
    for q in $(seq 0 47)
    do
        input_irq=$(cat /proc/interrupts | grep -w virtio${dev}-input.${q} | cut -d : -f 1)
        input_irq=$(echo $input_irq | sed -e 's/^[ ]*//g')

        # 根据队列号分配到不同 NUMA 节点的核心
        if (( $q < 24 )); then
            irq_core=$((base_core - 128 - $q))
        else
            irq_core=$((base_core - ($q - 24)))
        fi

        echo $irq_core > /proc/irq/$input_irq/smp_affinity_list
        echo "Bind virtio${dev} queue${q} rx irq $input_irq to cpu core${irq_core}"

        output_irq=$(cat /proc/interrupts | grep -w virtio${dev}-output.${q} | cut -d : -f 1)
        output_irq=$(echo $output_irq | sed -e 's/^[ ]*//g')
        echo $irq_core > /proc/irq/$output_irq/smp_affinity_list
        echo "Bind virtio${dev} queue${q} tx irq $output_irq to cpu core${irq_core}"
    done
done
```

之所以这样绑定是因为需要充分利用服务器底层网络架构。网络层面 LAG rx mode 设为 1，根据 rx 方向收包的物理口选择用哪个 PCIe 发送给子机，通过分配会话，避免出现两路 MAC 口打一路 PCIe 的情况。网络每一路 MAC 口绑定一个 NUMA 节点，所以中断分别绑定到两个 NUMA 节点，原则上会更充分利用底层网络资源。

NUMA 信息示例：

```
numactl -H
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 ... 127
node 0 size: ~1TB
node 1 cpus: 128 129 130 ... 255
node 1 size: ~1TB
node distances:
node   0   1
  0:  10  20
  1:  20  10
```

### 优化点：关闭 RPS、XPS

中断处理相关的其他优化还包括：

关闭 RPS、XPS。其中客户环境中的 RPS 本身就是关闭的，但 XPS 是打开的。

经测试，XPS 在当前 CVM 环境下会严重影响收发包性能，将其关闭：

```bash
for i in /sys/class/net/eth0/queues/tx-*/xps_cpus; do echo 0 > $i; done
```

网络层面其他优化还包括，解除 meter 限速，打开 LAG 配置。

### 优化效果

以上优化完后，进行 iperf 打流测试的结果：双向 iperf 打流，290-300Gbps 左右。

业务 Rados 1M write 的测试结果基本符合预期。跟网络团队确认，限速是分开各自能到 200Gbps。

最后交付客户的测试结果，Ceph 性能压测基本可以贴近基准值。

## 测试用例二：JuiceFS 场景

同时客户反馈，测试用例二在如上已经调优的环境下，仍然跑不出很好的效果。

测试用例使用 3 台 Ceph 存储集群不变，发压端变成了 JuiceFS vdbench 测试用例。跟之前的 Rados 不同，这次测试增加了单路连接压测和更多 block 大小的压测。

### 性能瓶颈分析

从数据对比，我们发现客户的标准数据和我们环境的压测数据相比，单路小包压测明显性能更差。多路大包压测相比稍好，但也只达到客户的 60% 左右。

1. 从数据上看，各项测试的最终吞吐结果都跟平均延迟强相关。延迟越低，吞吐量越高。

2. 平均延迟除了跟网络延迟相关，应该还跟文件的 open 耗时和 close 耗时相关。文件的 open 和 close 也呈现出跟网络耗时强相关的特征。应该是对不同特征的文件，open 和 close 在底层网络上使用了不同个数的数据交互来实现，导致不同特征的文件 open 和 close 耗时不同。

3. 整个数据中有个特例是"超大文件 IO 写多并发"，这个例子中吞吐量两边差不多。看起来是对标环境在这个特例中，close 耗时突然变得很大。导致整体平均延迟跟测试环境变得差不多，导致最后吞吐量变得也差不多。

目前从数据分析看，当前测试整体性能跟网络延迟相关性最大。

### 网络延迟瓶颈分析

通过抓包分析相关请求行为，在部分压力下观察单个文件的写入交互过程可以发现：JuiceFS 在进行数据写入时，首先要跟 JuiceFS 的其他成员进行元数据的交互。

简单分析来看：每一次元信息读写涉及 1.5 个 RTT ≈ 75us。最少的交互次数为 3 次直接可见的元数据操作，3 × 75 = 225us。Ceph 一个 RTT ≈ 50us，元数据如果是分布式同步的话，估计再加 2 个 RTT + 100us，Ceph 的主从同步也加 2 个 RTT + 100us。总体单次写入的纯网络消耗：225 + 50 + 100 + 100 = 475us。

在一些场景下，整个数据包的交互过程会更长。跟踪一个 JuiceFS 上的程序打开文件的过程可以看到，部分请求是新建连接，会进行 3 次握手，第一次握手延迟就达到 200us+。这跟网络架构中 ping flood 首包延迟较高的特性相关。另外整个交互过程要经过多轮数据交互，导致 open 耗时达到 3.5ms，close 耗时达到 2.7ms，其中同步数据过程占 1.5ms，网络延迟导致的时间消耗在整个请求中占比更大。

这一数据印证了压测数据与客户环境对比的差异：open 时间方面，客户平均 240us，测试环境为 548us；close 时间方面，客户平均 793us，测试环境为 1994us。

**网络延迟瓶颈确认：**

从 floodping 的平均延迟看，我们的环境达到 48us，客户环境为 14us。

### CPU 性能瓶颈分析

另外观察高并发大 IO 的服务器负载状态发现，超大文件并发 IO 时，顺序写性能从数据看起来是没问题的，但顺序读看起来低于预期，并且在压测时读端会有单核跑满现象，而且主要集中在 3-4 个核心上，服务器多核心并没有充分利用起来。

perf top 看，CPU 瓶颈也主要在内存 copy 和 libc 的 `__memmove_avx512_unaligned_erms` 上。

另外从抓包的角度来看，Ceph 端的写入时间也达到 800-900us 左右。综合判断，JuiceFS 压测的主要瓶颈是网络延迟和单核 CPU 性能的综合影响。经确认，网络延迟在当前架构下已没有进一步提升的潜力，因此需要从 CPU 侧寻找优化空间来提升整体吞吐。

基于以上 CPU 性能层面的分析，主要瓶颈在内存 copy 上，提升 CPU 访问内存以及缓存的局部性应该可以有效提升读写性能。

### 优化点：CPU 绑核

基于以上分析，我们将 CPU 绑核到 0-15，即前 16 个核心上进行测试对比。从 CPU 架构上看，前 16 个核心是共享 L3 Cache 的。

跟之前的数据对比发现，绑核之后写性能基本没有提升，部分大 IO 用例还有下降，但读性能有较大提升。这跟我们之前在读端看到的现象一致，说明缓存局部性的提升对读性能确实有帮助。

写性能下降的原因，应该是 Ceph 端绑核之后不能充分发挥网络结构的最大吞吐性能，且无法充分利用更多核心。因此进一步调整策略：JuiceFS 端绑核到单独的 NUMA 节点上，而 Ceph 存储端采用 IRQ 双路 NUMA 绑核。

调整后，读写两端的性能均有提升，尤其是高并发大 IO 场景，已经可以达到客户标准性能。但小 IO 单路并发场景性能提升不明显。

### 优化点：关闭内核安全补丁

进一步提升 CPU 性能的潜力有限，我们最后尝试将内核中针对 CPU 安全漏洞的相关补丁关闭后再测试性能。相关内核参数包括：

```bash
nospec_store_bypass_disable noibrs noibpb spectre_v2_user=off spectre_v2=off \
nopti l1tf=off kvm-intel.vmentry_l1d_flush=never mitigations=off \
spec_rstack_overflow=off
```

关闭安全补丁后单核性能有所提升，单路小 IO 场景性能也有一定改善，但距离客户标准仍有较大差距。不过高并发大 IO 场景基本都已能达到客户标准。

## 结论

本次 Ceph 存储性能优化场景，经过性能调整，在 Rados 的高并发性能场景下，基本能达到客户设定的标准。在 JuiceFS 端到端测试场景下，经过进一步基于读写特征进行 CPU 性能优化后，在高并发大 IO 场景下也能达到客户设定的标准。但是小 IO 单路并发场景下，由于网络延迟和 CPU 性能距离客户环境还有进一步差距，导致性能无法进一步提升到客户要求的水平。

从系统和虚拟化能力角度来看，虽然若干性能提升手段可以一定程度地提升性能，但在部分场景下，最核心的瓶颈还是网络延迟和 CPU 单核性能等产品架构及硬件层面的差异所导致的。从云产品的角度来看，产品层面应该提供网络延迟更低、CPU 单核性能更高的产品，以满足此类场景下的性能对标要求。

### 优化点汇总

| 优化项 | 操作 | 效果 |
| --- | --- | --- |
| `init_on_alloc=0` | 内核参数关闭内存申请清零 | 收包 CPU 显著降低 |
| 关闭网卡大页分配 | `echo 1 > /proc/sys/net/core/high_order_alloc_disable` | 减少 buddy spinlock |
| 收包校验 Offload | 宿主机开启网卡 rx csum offload | 消除 csum_partial 热点 |
| 关闭 pvspinlock | 宿主机 XML 配置 `pvspinlock state='off'` | 消除 pv_queued_spin_lock 热点 |
| IRQ 核心隔离 | `isolcpus` + `nohz_full` + `rcu_nocbs` | IRQ 处理更高效 |
| 关闭 XPS | `echo 0 > xps_cpus` | 消除收发包性能影响 |
| LAG 网络优化 | 解除 meter 限速，开启 LAG | 充分利用底层网络带宽 |
| CPU 绑核 | JuiceFS 端绑单 NUMA，Ceph 端双 NUMA | 读性能显著提升 |
| 关闭安全补丁 | `mitigations=off` 等内核参数 | 单核性能提升 |
