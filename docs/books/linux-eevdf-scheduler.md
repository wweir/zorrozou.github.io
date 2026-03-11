# 为什么Linux需要一个新的进程调度算法？

自从2007年开始，CFS进入Linux 2.6.23内核以来，这个调度算法已经陪伴Linux的发展超过15年了。但时间长并不是要替换掉它的理由，CFS虽然并不完美，但是大多数场景下它都可以提供很好的调度效果。随着行业的发展，Linux内核也被用在了各种更复杂的场景需求下。在很多场景下，应用发现有很多针对进程调度延迟的差异需求。比如，很多应用在需要占用CPU的时候，总是想要很快能运行起来，而且它要占用的时间长度很短，所以它对那些需要长时间占用CPU进行运算的进程来说，总体吞吐量影响也不算大，如果可以尽快调度这一类应用，可以给用户提供更好的用户体验。而CFS从一开始就是为了公平而涉及的调度算法，其设计思路是保证在一个特定的预期时间内，所有进程都是平均占用CPU的。虽然我们也可以通过调整这个预期时间的长短，来让新进程尽可能的快的得到响应，但是这依然将破坏其他进程的调度时间长度，从而更加频繁的产生上下文切换，不能让CPU的缓存得到充分的利用。

在这样的需求驱动下，Peter Zijlstra在2023年向内核提交了EEVDF调度算法，其全称为："Earliest Eligible Virtual Deadline First"。强行翻译一下就是，“最早符合资格的虚拟最终时间优先“调度算法。大家先不要被这个名字吓到，实际上它并不是一个很难以理解的设计。这个算法从提交到现在已经发展了一段时间了，从效果上来看，它不仅提供了比CFS调度延迟上更多的优势，它还摆脱了CFS实现中为了解决各种特定条件下一致性的一大堆启发式代码。本文就是基于当前最新的稳定版内核6.10.8对EEVDF调度算法实现的一个介绍。

# EEVDF调度算法简介

我们先来从EEVDF的名字来理解一下这个算法的设计思路。"Earliest Eligible Virtual Deadline First"，“最早符合资格的虚拟最终时间优先“调度算法。简单来说，这个名字中最关键的两个词是Deadline和Eligible。更简单来说，这个调度算法其实就是找到队列中当前预期占用CPU时间最短的（Deadline）那个进程来执行。这样就保证了我们改进的目标，即：占用时间很短的进程，应该更快的响应。当然，除了这个要求外，还要符合资格（Eligible）。这个算法本身并不是一个新提出的算法，早在1995年就已经由Ion Stoica和 Hussein Abdel-Wahab提出了，论文连接：<https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564>

设计目标上跟CFS一样，EEVDF也试图在多个争抢CPU的进程之间公平的分配CPU时间。我们假定这个队列中有5个进程要使用CPU，调度预期时间为1000ms的情况下，我们会给这5个进程分配200ms的“预期CPU占用时间”。此时，就可以理解为这5个进程的预期deadline时间都是200ms之后。但是在执行的过程中，进程并不都是按这个预期来使用CPU的。有些进程可能占用不到200ms就已经主动释放CPU了。于是这里就产生了一个实际上的不公平现象，即对于每个进程来说，它给他分配的预期时间跟它实际占用的时间并不一样。我们可以通过跟踪这个差值（分配时间-实际占用时间）来跟踪一个进程的“滞后”性（lag值）。如果这个值为正，则说明这个进程因为某种原因一致没能用满其该占用的CPU时间，从公平的角度来说，它被不公平的对待了，所以这种进程应该考虑下次调度的时候获得优先占用CPU的权利。

我们把这种进程规定为“符合资格”（Eligible）的进程，即lag值为正的进程。这也是EEVDF调度算法进行调度选择的第一个条件：从lag值为正的进程中选择下一个该调度的进程。对于任何当前调度周期不符合资格的进程，因为本次调度仍然会给它分配预期CPU的占用时间，所以其lag值未来某个时间总会变成正值，就会再次符合资格。

另一个起作用的条件就是虚拟最终时间（Virtual Deadline），这个就是本次分配完时间长度后，进程预计的执行结束时间。这个时间就是当前分配的时间长度+“符合条件”时间（Eligible Time）。这个Eligible Time是当进程不符合条件（lag值为负值）的时候，要等待其成为正值的时间。

EEVDF就是在符合条件（lag值为正）的进程中选择虚拟最终时间最短的那个进程来执行。

# EEVDF调度算法核心代码分析pick_eevdf()

内核在执行中总会找时机进入调度算法选的的过程中，在此我们先不对调度时机进行解释，仅列出其进入调度核心算法的过程路径：

eevdf调度算法核心函数：pick_eevdf()

调用逻辑：调度器schedule()--\>eevdf class: fair --\> \_\_pick_next_task_fair --\> pick_eevdf()

--\> pick_task_fair

为了对多个运行程序提供一种基于不同延迟预期的调度算法，EEVDF基于以下规则在运行队列中选择进程:

1、进程必须符合资格。即eligible，eligible的意思是说，曾经分给进程的可执行时间有剩余，即调度器分给进程的时间片总长度大于等于进程实际使用的时间。

2、在符合条件1的情况下，选择最终期限要求最短的进程执行。

在一个红黑树中，这个选择策略可以在O（log n）时间复杂度的条件下实现。这个红黑树使用进程的预期截止时间deadline时间进行排序。同时维护了一个基于vruntime的堆，来判断进程是否符合资格。vruntime的计算符合如下公式：

se-\>min_vruntime = min(se-\>vruntime, se-\>{left,right}-\>min_vruntime)

这个函数不算长，我们全文注释一下：

算法实现并不复杂，核心就是一句话：在符合资格的进程中，选择deadline最小的那一个进程。从其实现来看，我们也能了解，对于调度器来说，最核心的调度对象就是struct sched_entity这个数据结构。我们也来简单看一下这个数据结构：

从核心函数以及数据结构中我们可以引申出几个细节问题：

1.  entity_eligible()的实现细节中具体如何判断一个entity是eligible的？

2.  什么是RUN_TO_PARITY内核参数？它在哪里配置？

3.  对于每个任务的se结构来说，内核是什么时候对这些数据进行更新的？

下面我们就来分别看一下这些问题。

## 如何判断一个任务是eligible的？

我们可以从代码中看一下entity_eligible()的具体实现：

核心函数是vruntime_eligible()，我们先来看一下这里用到的struct cfs_rq数据结构。这个结构也就是调度队列的结构：

数据结构比较长，我们仅解释一下目前用到的数据含义：

cfs_rq-\>avg_vruntime：队列中所有任务的平均vruntime。

cfs_rq-\>avg_load：队列中队友任务的平均load，这个load用来计算权重。

vruntime_eligible()是一个比较重要的方法，简单来说，他用来判断一个se是不是有资格使用CPU。我们回顾一下之前说过的eligible条件是：lag值为整。即：任务的预期占用时间 \>= 任务的实际占用时间。注释中给了一个计算这个值的公式推导过程：

lag_i = S - s_i = w_i\*(V - v_i)

任务i的lag值 = 任务的应得时间 - 任务的已得时间 = 任务i的权重 \* (任务的应得vruntime - 任务的已得vruntime)

lag_i \>= 0 -\> V \>= v_i

若要lag_i \>= 0，则任务的应得vruntime \>= 任务的已得vruntime

\Sum (v_i - v)\*w_i

V = ------------------ + v

\Sum w_i

这个公式推导起来稍微麻烦点，首先，根据lag的定义，我们可以得知，一个队列中所有进程的lag值的和，即：

\Sum lag_i = 0

从上述公式中也就可以推导出：

\Sum w_i \* V - w_i \* v_i = 0

那么可以推导出：

V = \Sum v_i \* w_i / \Sum w_i

定义v是队列中的最小vruntime，那么就可以推导出上述公式了。即：

\Sum (v_i - v)\*w_i

V = ------------------ + v

\Sum w_i

那么如果

lag_i \>= 0

则需要：

\Sum (v_i - v)\*w_i - (v_i - v)\*(\Sum w_i) \>= 0

即：

\Sum (v_i - v)\*w_i \>= (v_i - v)\*(\Sum w_i)

即队列中的所有进程的（（已分配vruntime - 队列中的最小vruntime） \* 进程权重）之和 \>= （当前进程的vruntime - 队列中的最小vruntime） \* 队列中所有进程的权重之和。于是得出函数中的：

return avg \>= (s64)(vruntime - cfs_rq-\>min_vruntime) \* load;

## 什么是RUN_TO_PARITY内核参数？它在哪里配置？

RUN_TO_PARITY是内核的一个可调整feature。内核提供的可配置feature都放在一个sys文件系统的目录下：

RUN_TO_PARITY：这个特性的目的是减少激进的抢占。一般情况下EEVDF总会选择deadline最小的任务进行调度，在某些场景下如果这个进程频繁更换，会造成很多的上下文切换导致整体执行效率下降。开启了这个特性后，内核会在符合条件的情况下，一直执行当前的任务，直到其不符合资格（non-eligible），或者其获得一个新的slice，在此期间内而不被其他进程抢占。

## 对于每个任务的se结构来说，内核是什么时候对这些数据进行更新的？

及时更新调度对象中的数据，对于调度器能准确达成调度目标来说是非常重要的事情。在内核中，更新这些数据的核心函数是update_curr()

这个函数本身比较简单，主要做的事情基本一目了然。这里需要单独说明的就是进程vruntime的更新，对应使用的方法是calc_delta_fair()，实际上在EEVDF中，这个更新的方法跟CFS中没有变化，核心思路也就是当前时间跟进程对应nice值计算出的权重做一个计算，得到虚拟runtime，即vruntime。

内核中的除法转成乘法和移位运算，细节这里不做多解释了。另外一个比较重要的方法就是如何更新进程的预期执行时间deadline。使用的是update_deadline()这个函数：

这里最核心的是，时间片的赋值用到了sysctl_sched_base_slice。这个值就是EEVDF可以给用户调整的最重要的一个参数了。用户接口定义在：

这也就是EEVDF的时间片设置。进程的预期执行时间也就是当前任务的实际占用时间+时间片长度。EEVDF就看队列中谁的这个时间最小，下一个就调度谁。

这些重要的时间会在哪些时机更新呢？大家可以使用任何一种代码查看器来看内核都在哪些方法里调用了update_curr()即可，比如在我的cscope中：

这里最核心的时机是如下几个：

task_sched_runtime：调度执行

reweight_entity：权重（优先级）调整

enqueue_entity：新进程入队

dequeue_entity：进程出队

put_prev_entity：在put_prev_task_fair和pick_next_task_fair中调用。用来做任务切换前的准备工作。

entity_tick：在task_tick_fair中调用，当有时钟中断来临时，做相关操作。

check_preempt_wakeup_fair：处理进程抢占的时候。

pick_task_fair：选择进程

pick_next_task_fair：选择进程

yield_task_fair：进程让出cpu

task_fork_fair：新建进程

这里需要进一步了解的是新进程创建的时候，除了使用update_curr()更新当前队列外，更重要的是对新进程的相关数据进行初始化的过程。

# 新进程如何初始化EEVDF的数据结构？

新进程创建时，会使用update_curr()和place_entity()初始化整个任务相关数据。其中最重要的是使用place_entity()来初始化整个数据结构。

调用路径：

kernel_clone -\> copy_process

\|--\> sched_cgroup_fork -\> task_fork_fair -\> place_entity(ENQUEUE_INITIAL);

-\> wake_up_new_task

\|--\> activate_task -\> enqueue_task -\> enqueue_task_fair -\> enqueue_entity -\> place_entity();

place_entity()是进程加入队列时最核心的EEVDF相关数据更新的函数。

另外就是进程在出队时，要将出队进程的lag调整成尽量少收到加权平均处理的影响，导致V由于删除一个任务而变大。所以要对出队任务的lag做一个限制。主要更新方法是update_entity_lag()：

# 对CFS保持一致的特殊处理的改进

### 新进程的vruntime值：

上面我们已经看到了EEVDF调度下产生新进程的vruntime值的计算，而CFS通过vruntime最小值来选择需要调度的进程的，那么可以想象，在一个已经有多个进程执行了相对较长的系统中，这个队列中的vruntime时间纪录的数值都会比较长。如果新产生的进程直接将自己的vruntime值设置为0的话，那么它将在执行开始的时间内抢占很多的CPU时间，直到自己的vruntime追赶上其他进程后才可能调度其他进程，这种情况显然是不公平的。所以CFS对每个CPU的执行队列都维护一个min_vruntime值，这个值纪录了这个CPU执行队列中vruntime的最小值，当队列中出现一个新建的进程时，它的初始化vruntime将不会被设置为0，而是根据min_vruntime的值为基础来设置。这样就保证了新建进程的vruntime与老进程的差距在一定范围内，不会因为vruntime设置为0而在进程开始的时候占用过多的CPU。

所以不可避免的，CFS额外对新进程的处理添加很多额外的代码。这还包括了CFS的kernel.sched_child_runs_first的内核参数的功能支持。而EEVDF这部分代码就可以简化了。

参见补丁：https://lwn.net/ml/linux-kernel/20230306141502.630872931@infradead.org/

## IO消耗性进程的处理：

有一类叫做IO消耗类型的进程，它们的特点是基本不占用CPU，主要行为是在S状态等待响应。这类进程典型的是vim，bash等跟人交互的进程，以及一些压力不大的，使用了多进程（线程）的或select、poll、epoll的网络代理程序。如果CFS采用默认的策略处理这些程序的话，相比CPU消耗程序来说，这些应用由于绝大多数时间都处在sleep状态，它们的vruntime时间基本是不变的，一旦它们进入了调度队列，将会很快被选择调度执行。但这样的默认策略也是有问题的，有时CPU消耗型和IO消耗型进程的区分不是那么明显，有些进程可能会等一会，然后调度之后也会长时间占用CPU。这种情况下，如果休眠的时候进程的vruntime保持不变，那么等到休眠被唤醒之后，这个进程的vruntime时间就可能会比别人小很多，从而导致不公平。

对于这样的进程，CFS也会对其进行时间补偿。补偿方式为，如果进程是从sleep状态被唤醒的，而且GENTLE_FAIR_SLEEPERS属性的值为true，则vruntime被设置为sched_latency_ns的一半和当前进程的vruntime值中比较大的那个。

在EEVDF调度器的情况下，这部分代码也是不必要的了。

参见补丁：https://lwn.net/ml/linux-kernel/20230306141502.691294694@infradead.org/

# 参考资料：

[An EEVDF CPU scheduler for Linux \[LWN.net\]](https://lwn.net/Articles/925371/)

https://lwn.net/ml/linux-kernel/20230306132521.968182689@infradead.org/

[CFS的覆灭，Linux新调度器EEVDF详解-CSDN博客](https://blog.csdn.net/qq_41603411/article/details/136277735)
