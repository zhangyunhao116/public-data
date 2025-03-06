# Scalable MPMC queue in Go language

[TOC]

## 前言

> 本文写于 2021-09

本篇文章将会介绍在研究 Go 并发数据结构和算法的过程中，对于 scalable MPMC (multiple-producer multiple-consumer) queue 在 Go 语言场景下的实践和探索。

最终开发出了一个在 8C16T (Intel 11700k, -cpu=16)  大部分场景下相比于原有的基于全局锁结构的 LinkedQueue 快 **~ 5x** ，相比于内置的 channel 在 EnqueueDequeuePair 的场景快 **~ 7x** 的 unbounded MPMC queue，将其命名为 [LSCQ][1] (Linked Scalable Circular Queue)。

[LSCQ][1] 是一个 scalable、unbounded 的 MPMC queue，其容量无限并且随着 CPU 核数的增加，其性能会越来越好。

Go 并发数据和算法优化工作的背景是，目前 Go 的大部分常用的数据结构都是 unscalable 的，在多核高并发的条件下表现并不优秀并且存在瓶颈，例如 sync.Map、channel 等。我们希望补充 Go 语言在这方面的缺陷。这里的 unscalable 的可以理解为，在高并发的情况下无论我们怎样增加一个实例使用的资源（CPU、内存），我们都无法提升这些数据结构的处理能力，反而，随着操纵这些数据结构的 goroutines 的增多、核数的增加，其处理能力还有可能会降低。

[LSCQ][1] 已经开源，位于 [https://github.com/bytedance/gopkg/tree/develop/collection/lscq](https://github.com/bytedance/gopkg/tree/develop/collection/lscq)

![pair-all](https://github.com/zhangyunhao116/public-data/raw/master/lscq-benchmark-pair-all.png)

> 随着 cores 增加，LSCQ 性能越来越好，而其他的 queue 性能越来越差



## 理论基础

我们最终选择了 [A Scalable, Portable, and Memory-Efficient Lock-Free FIFO Queue][1] 的 CAS2 版本作为我们 MPMC queue 的理论基础，并加入了一些理论上和工程上的优化。本篇文章中，我们将其 bounded (容量有限) 的实现简称为 [SCQ][1]（*scalable circular queue*）。[LSCQ][1] 是 [SCQ][1] 的 unbounded 版本，原理是将 [SCQ][1] 视为黑盒，在外层使用 CAS 的方式不断迭代 [SCQ][1]。

> 这里的 CAS2 指代的是 128-bit 的 CAS ([Compare-And-Swap][2])。我们一般使用的 CAS 都是对两个 64-bit 的数据进行 Compare-And-Swap，CAS2 则可以对两个 128-bit 的数据进行 CAS 操作。在 amd64 平台其汇编实现为 [cmpxhg16b][3]。

我们同时还分析和实现过其他的一些 MPMC queue 算法，但是他们都因为各种各样的原因被放弃。例如基于 combiner 的 [CCQueue][4]，以及基于 FAA ([fetch-and-add][5]) 的 [A Wait-free Queue as Fast as Fetch-and-Add](http://chaoran.me/assets/pdf/wfq-ppopp16.pdf) ，因为他们都依赖 [threadlocal-state][6] 这样的特性，而 Go 原生并没有给我们提供诸如 threadlocal 这样的功能，导致我们无法以一般的方式来实现它（尽管有取巧的方式，但是它们往往带有很多不确定性），所以最终被放弃。

Queue 领域不得不提的一个经典实现是 [MSQueue][7] ，它同时也是 Java [ConcurrentLinkedQueue][8] 的理论基础。尽管 MSQueue 是 "wait-free" 的，但是由于其依赖过多的 CAS 操作，导致其在多核高并发的情况下表现并不好。一个类似的假想是，如果 Go 的 sync.Mutex 只使用 CAS 来实现，尽管它不依赖锁 (linux-futex 等)，但是它的效率依然很低，因为在多核高并发条件下，一个 CAS 的成功会导致同一时间其他所有 CAS 的失败，造成了互相竞争，绝大部分的 CPU 被消耗在重复 CAS 上。 所以并非无锁的算法就是最优的，锁的存在很大程度上就是为了解决这种无止境的数据竞争问题。

近年来学术界的一个新进展是，在高并发环境下，FAA 的效率比 CAS 的效率更高。循环数组在这种情况下也展现出了其优势。[SCQ][1]（我们选择的理论基础），以及速度处于第一梯队的 [LCRQ][9] 都是这两者结合的产物。[LCRQ](9) 经常被其他论文引用为速度对比，并且经常在大部分情况下夺冠。我们将 [LCRQ](9) 的 bounded 版本简称为 CRQ (concurrent ring queue)。

LCRQ 之所以速度很快，是因为其依赖的新的指令集 CAS2。在 [LCRQ](9) 发表的 2013 年，这个指令还是比较前沿的，并不是所有机器都支持。但是在 2021 年，绝大部分机器都支持这个指令，比如我们常见的 amd64/arm64，两者的新型号基本都支持 CAS2。所以我们决定基于此来构建 scalable queue（从实践上的原因来看，我们同时实现了基于 CAS 的版本来进行存储任意指针的测试，相比 CAS2 速度慢了一倍，并且消耗了 4 倍的内存）。

为什么我们不使用 [LCRQ](9) 而是 [SCQ][1] 呢？可以简单的理解为，**SCQ 的 CAS2 版本是优化后的 CRQ**，并且解决了 CRQ 的一些问题，例如

- CRQ 可能导致潜在的 livelock 问题，并且 CRQ 不是 stand-alone 的(SCQ 可以单独使用)。
- CRQ 会消耗更多的内存，[LCRQ](9) 产物堆积的情况下会造成巨大的内存浪费(需要对齐 cacheline，不断迭代 CRQ 等)。

[SCQ][1] 的 unbounded 版本，[LSCQ][1] 也可以达到 [LCRQ](9) 相近的速度。



## SCQ

> 这里描述的是基于 CAS2 的版本。

[SCQ][1] 的主要特点是使用了 循环数组([ring buffer](https://en.wikipedia.org/wiki/Circular_buffer)) + [FAA](https://en.wikipedia.org/wiki/Fetch-and-add) + CAS2 的组合，是一个 bounded、scalable、stand-alone 的 MPMC queue。循环数组的 head 和 tail 分别被 dequeuer 和 enqueuer FAA，此时 dequeue 和 enqueue 完全并行，互不干扰。此外，SCQ 使用 threshold 的方式解决了 CRQ 可能存在的 livelock 的问题，从而使得 SCQ 可以单独被使用。另外由于 CAS2 的特性，flags(64-bit) 和 data(64-bit) 可以处于同一个 entry 然后被原子性替换。**关于具体实现可以看论文 Section 5**，本篇文章会简单介绍大致流程，更多会集中在原理优化和工程优化的方面，以及实践方面的探索。

我们将 SCQ 的容量 scqsize 设定为 1 << 16 (65536)，这同时也是 SCQ 的循环数组的长度。

SCQ 会拥有以下限制

- 操作的线程数目（对于 Go 来说是 goroutines） k <= scqsize
- CPU 支持 CAS2 (arm64 和 amd64 新版本基本都支持)

> 对于限制一，我们模拟了[极端的情况](#SCQ: goroutines 数目是否会超过 scqsize)，验证了其可能性。



### 简介

> 论文中有从头开始演化的完整流程，**请先查看论文中的流程介绍（section 5），本篇文章主要集中在论文基础上的理论拓展和实践优化。**

[SCQ][1] 主要由 ring buffer + FAA + CAS2 组成，这里简述一下它们存在的意义。

其中 [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) 也被称为 circular queue、circular buffer 等。实际是一个定长的 array 加上 head、tail 组成的。在 Go runtime 中也有大量应用，例如 channel 以及 sync.Pool 等。可以把 ring buffer 视为一个特殊的 array，但是这个 array 的第一个元素不一定位于 0，而是可以处于 array 的任意位置，这对于经常增加/删除元素的场景优势较大。所以很多 queue 的实现都是基于 ring buffer。一般来说，Enqueue 的时候，使用 head++ 的值作为 index 定位一个 slot 来插入新元素。Dequeue 的时候使用 tail++ 的值作为 index 定位一个 slot 来删除一个元素。

那么为什么我们需要使用 FAA 呢? 例如每次 queue 增加元素时，我们需要划分一个 slot 给新的元素使用，最简单的方法就是对 ring buffer 的 head++，取新 head 的值为 slot 的 index。在多核的情况下，因为会遇到 data race 的问题，我们需要原子性地对 head++，FAA 此时对应了 Go 的 `atomic.AddUint64(&head, 1)` 。

CAS2 作用又是什么呢？在前面的 FAA 中，我们每次会对 head++，但是如何判断我们拿到的 slot 没有被其他的 goroutines 占用呢，例如元素插入很快，迅速使得 head 循环一次，此时同一个 slot 被两个 goroutines 使用。这时候我们就需要在 slot 中加入至少一个状态位，来记录这个 slot 的状态（因为 MPMC，所以 head 和 tail 都会随时改变，不能用 head == tail 来判断是否 array 被占满）。因为状态位是需要和数据一起被原子性替换的，所以只有两种解决方式 1) 存储的数据少于 64bit，预留一些状态位 2) 使用 CAS2，可以原子性替换 128bit 数据。我们最终选择了方式 2，这样用户使用层面对数据无要求。CAS2 可以一次性地将状态位和数据一起替换掉。

基本特性列举，想要推导原理的同学可以参考一下

> - head 和 tail 只会单向增长，初始值为 scqsize。head == tail 时其为空队列，正常情况下 head >= tail 始终成立。
>   - cyclehead 的计算方式为 head/scqsize，其当前所代表的 index 计算方式为 head % scqsize。tail 同理。
>
> - threshold 初始为 -1，执行任一 enqueue 后，threshold 的值都会变为 2 * scqsize -1。
>
>
> - 每个 entry 初始的 cycle 为 0，isempty 和 issafe 都为 true。
> - Enqueue 会在尝试追上 head 后(保证 head >= tail)，将 data 插入当前的 entry 中。
> - Dequeue 会在发现当前 Entry 的 cycle 和 headcycle 一致的之后，直接返回这个值。
> - Enqueue 和 Dequeue 都可能在 entry 中不断移动
>   - Enqueue 在发现 head > tail 时会进行移动，每次移动时 tail++
>   - Dequeue 移动的情况较为罕见，每次移动时 head++，需要满足以下条件
>     - cycleEnt < cycleH
>     - NewEntry 置换成功
>     - head > tail，即 queue 里面有 data
>     - threshold > 0
>



### Optimizations

#### 改造 fixstate 流程，优化 Dequeue 较多的场景

fixstate(也就是论文中的 catchup，这里沿用 LCRQ 的表述) 的作用是，当 Dequeue 较多的时候，head 的值会超过 tail，从而造成整个 queue 处于非法状态。此时我们需要修复这个状态，解决的方法是将 tail 的值替换为最新的 head 的值，fixstate 由造成这种非法状态的 dequeuer 完成，当 dequeuer 发现 head > tail 的时候会尝试 fixstate，将 tail 的值成功替换为 head 的值后直接返回(因为发生 fixstate 的状态说明 queue 中已经不存在 data)。

fixstate 的意义在于尽可能地将 tail CAS 成 head，使得两者的距离尽量接近。但是论文中的实现有如下问题，造成了在 Dequeue 更多的情况下性能下滑比较严重

- **CAS 成功也不一定能保证 fixstate 完成后 head >= tail**。在 head 增长较快的时候，CAS 成功后的 tail 也有可能小于 head，即仍然处于非法状态。例如在 atomic.Load head 的值后，有另外一个 Dequeuer 进入，将 head ++，此时即便 CAS 成功仍然没法脱离非法状态。

- **Dequeue 占比稳定大于 Enqueue 的时候，会出现大量 Dequeuer 不断 CAS 失败的问题**。发生的原因是，每个 Dequeue 结束时都发现 head 大于 tail，于是都尝试 fixstate。任意 Dequeuer 的 fixstate 或者 enqueuer 的进入都更新 tail 的值，导致其他 Dequeuer 的 CAS 失败，不断重复尝试这个循环。其他 dequeuer 的 CAS 失败的无限重试策略也会加剧这种情况的发生。

据此，我们改造了 fixstate 的流程，如果发现最新的 head 已经超越了 dequeuer 进行 fixstate 的 head（也就是发现有 dequeuer 处于现有的 dequeuer 前方），则我们直接跳过 fixstate，直接返回，而不是无限次尝试 fixstate。此时，**fixstate 的任务会交给最后的造成这种非法状态的 dequeuer**。

这个改动理论上会造成如下结果：

- **每个 Dequeuer 会在发现非法状态后更快地返回，由于减小 threshold 的操作更快了，可能会造成 Enqueuer 插入新值后的 threshold 变大（插入一个元素会重置 threshold）**。Dequeuer 发现非法状态后会尝试 fixstate 然后直接返回。由于本身 fixstate 的行为就是非原子性的（两个 atomic.Load 分别加载 head 和 tail），提前返回在原来的流程也是完全可能发生的行为（例如在 atomic.Load head 之后，一个新的 Dequeuer 到来将 head ++），新的改动会加剧这个问题。
- **改动可能会造成 enqueuer 更多的循环**。Enqueuer 因为发现 entry.cycle == tailcycle (这个 cycle 是 dequeuer 在移动时 CAS 赋值的)，也会不断循环将 tail ++，因为新的机制 CAS 只由最后一个 Dequeuer 负责，tail 想要追上这个值可能会更难。此条存疑，因为尽管 CAS 的 Deqeueur 大大减少，但是 CAS 成功后的距离大大增加了，这种情况下应该是有优势也有劣势。
- **大大减少了 fixstate 不断重复 CAS 失败的问题**。因为只要 CAS 成功，就会使得 tail 的值和 head 非常接近（因为总是最新的 dequeuer 负责 fixstate）。

从 benchmark 的结果来看，这个改动整体利大于弊，对于 4 core 及以上 30Enqueue70Dequeue 的场景 有 **15% ~ 30%** 的提升，对于其他的场景没有观测到影响。

> 此改动还有一种方式是，先尝试 CAS 然后再判断自己是否是最后一个 Dequeuer，如果是则返回。这种方式相比于目前采用的改动，会使得在 30Enqueue70Dequeue 4 core 以上的场景性能减少 ~ 8%，在 70Enqueue30Dequeue 4core 和 8core 的场景性能增加 ~ 5%。由于 30Enqueue70Dequeue 是 queue 的弱势场景，我们希望提升这方面的性能，所以没有采用这种方式。

#### cache remap

此项是论文中描述的优化，目的是将相邻的 entry 分配到不同的 cache line，使得 CPU cache 尽可能被命中。这项优化给单核的大部分场景带来了 ~ 5% 的性能减少，但是给 2 core 以上的其他大部分场景带来了 **10% ~ 40%** 的性能提升。由于实际场景中单核场景较少，最终采用了这项优化。

#### fastpath

我们发现论文 Figure 8 中 Line 33 - 35 会定义一个 NewEntry。但仅当 Cycle(Ent) < Cycle(H) 时，我们才会使用到这个变量。所以我们将 Line 33 - 35 的逻辑移动到 Line 36 的 if 中，在大部分单核的场景获得了 **~ 5 %** 的性能提升。

#### dequeue retry

论文中关于 dequeue 增加 retry 机制的优化，我们发现在将重试次数设置为 3 时，在 benchmark/30Enqueue70Dequeue 中对于 -cpu=16 有 3% 的提升，但是对于 -cpu=1,2,4 的情况有 **10% ~ 30%** 的下降。对于 benchmark/Pair 在 -cpu=2 的情况下有 **20%** 的提升。

综合考虑，这项优化从 benchmark 上看并不高效，最后放弃了这项优化。



## LSCQ

LSCQ 是 SCQ 的 unbounded 版本，其原理是使用类似 MSQueue 的方式，将多个 SCQ 串联起来，相当于 SCQ 组成的链表。其原理是，当位于前面的 SCQ 满了以后，执行 Enqueue 的 goroutiens 会创建新的 SCQ，然后通过 CAS 插入到末尾的 SCQ 之后。成功则向新的 SCQ 插入当前元素，失败则再次查看末尾的 SCQ。当位于前面的 SCQ 没有数据之后，Go GC 便将其从链表中摘除。

我们对论文中的 LSCQ 有如下改动：

- 将 free_SCQ 替换为放入 sync.Pool。**由于 Go 自带 GC，并且没有提供显式的回收接口，我们只能使用 sync.Pool 来解决**。这个改动会提高内存的复用率，缺点是在 sync.Pool 中也会经过两次 GC 才会被回收，可能会造成内存占用较高的问题。
- 我们移除了 Figure 9 的 Line 15。由于 LSCQ 本身忽略了 memory reclamation 问题，我们无法保证 cq 被放入 sync.Pool 的时候没有其他 goroutine 正在使用它。hazard pointer 等方式代价比较高昂，依靠 Go 本身 GC 解决这个问题是更合理的方式。benchmark 表明，GC 可以较好地回收这块地址，不会造成内存泄露问题。
- 在 Figure 9 的 Line 24 加锁，确保在创建新的 SCQ 时只有一个 Enqueuer 能够成功创建新的 SCQ 并且 CAS 成功。这个改动的背景是，我们在 benchmark 的时候发现 LSCQ 在极端场景会造成内存泄露问题，问题的原因是当 LSCQ 迭代创建 SCQ 时，所有 Enqueuer 都发现上个 SCQ 满了，然后都会先尝试创建一个新的 SCQ，但是只有一个 Enqueuer 会成功将自己创建的 SCQ CAS 到链表中，可能会造成内存泄露的问题。例如有 10000 个 Enqueuer 就会造成 9999 个 SCQ 没有被 free （Go 没有显式的归回内存操作），即便放到 sync.Pool 里面也要经历两次 GC，并且 GC 在高压力情况下来不及释放这些内存，就会造成 OOM。**使用锁的方式基本对性能没有影响，但是会导致 LSCQ 理论上无法做到 lock-free**。





## 附录一：不同情况的理论推导



### SCQ: Dequeue 在 Enqueue 之前，并且发现当前 entry 为非空

（论文证明位于 Section 5.2 : A dequeuer arrives prior to its enqueuer countpart, but the corresponding entry is already occupied）

这种情况的发生同样也是 [unsafe 发生的条件](#SCQ: unsafe 发生的条件)。



### SCQ: unsafe 发生的条件

这里模拟 entry 的 issafe 将会被设置为 false 的情况。简单来说，就是某个 Dequeuer blocking 在某个已经有 data 的 entry 上，直到后续的 Dequeuer 遍历 all entry 再次到达同个位置时，entry 的 isunsafe 会被设置为 true。unsafe 状态只能由 Enqueuer 恢复，并且由于 threshold 的限制，Dequeuer 的距离是被限制的，会因为 threshold 为 0 而直接返回。

init: scqsize = 4, head = 4, tail = 4, all entry (issafe:true,isempty:true,cycle=0)

- e1 Enqueue 一个数据 1，成功并结束

  - head = 4, tail = 5
  - (issafe:true,isempty:false,cycle=1,data=1)
  - (issafe:true,isempty:true,cycle=0)
  - (issafe:true,isempty:true,cycle=0)
  - (issafe:true,isempty:true,cycle=0)

- d1,d2,d3,d4 同时出现，假设 d1 在将 head ++ 之后立即被 block。d2 d3 d4 由于发现entry 为空，此时它们会将其所占领的 entry 的 cycle 都改成 cycleH(1)，然后尝试 fixstate，最终将 tail 的值 CAS 为 head 的值。

  - head = 8, tail = 8
  - (issafe:true,isempty:false,cycle=1,data=1)    [d1 blocking]
  - (issafe:true,isempty:true,cycle=1)
  - (issafe:true,isempty:true,cycle=1)
  - (issafe:true,isempty:true,cycle=1)

- d5 出现，占领第一个 entry，此时会发现 cycleH(2) > entry.cycle(1)，出现 unsafe 的情况。此时 d1、d5 同时占领着第一个 entry。对于 d5，它会尝试把 entry 的 issafe 设置为 false。对于 d1 是一个正常的 Enqueue。

  - head = 9, tail = 8
  - (issafe:true,isempty:false,cycle=1,data=1)    [d1、d5 blocking]
  - (issafe:true,isempty:true,cycle=1)
  - (issafe:true,isempty:true,cycle=1)
  - (issafe:true,isempty:true,cycle=1)

  ##### 假设 d5 CAS 成功，其将第一个 entry 设置为 unsafe 后，会使得 head ++ 并移动到下一个 entry，对于 d1 一切正常。

  - head = 9, tail = 8
  - (issafe:false,isempty:false,cycle=1,data=1)   [d1 blocking]
  - (issafe:true,isempty:true,cycle=1)  [d5]
  - (issafe:true,isempty:true,cycle=1)
  - (issafe:true,isempty:true,cycle=1)

  ##### 假设 d1 CAS 成功，正常将此 entry dequeue 后，d5 将第一个 entry 的 cycle 变成 2 后进行 head++ 移动到下个 entry。

  - head = 9, tail = 8
  - (issafe:true,isempty:true,cycle=1)   
  - (issafe:true,isempty:true,cycle=1)  [d5]
  - (issafe:true,isempty:true,cycle=1)
  - (issafe:true,isempty:true,cycle=1)

两种情况均符合预期。



### SCQ: 验证一般情况下是否可能有 entry 遗留问题

这里讨论在什么情况下会出现，Enqueue 的 entry 会被临近的 Dequeue 跳过，从而遗留到 head 之前，造成 FIFO 失效的问题。这里构造了简单的情景来验证这种情况。（Unsafe 情况也是之一，可以看前一部分的流程推导）

- Enqueue blocking 在 CAS 之前一个语句，此时 tail 已经被 ++。
- Dequeue 将 head ++ 后，发现自己的 cycle(1) > entry.cycle(0)，并且此时此 entry 为空，则 Dequeuer 会尝试将此 entry 的 cycle CAS 为自己的。

假设此时 Dequeuer blocking 在 CAS 之前，一共两种情况。

1. Enqueue 先恢复，成功将 entry 赋值。此时 Dequeue 的 CAS 失败，进行 dequeue-retry，成功拿到刚刚放进去的值（dequeue-retry 不会使得 head++）。
2. Dequeue 先恢复，成功将 entry.cycle 赋值为 1。此时 Enqueue CAS 失败，进行 enqueue-retry，此时因为发现 entry.cycle 为 1，会进行下次循环，将 tail ++，尝试在下个位置插入数据。

两种情况都能满足，所以一般情况不会造成 entry 遗留问题。



### SCQ: goroutines 数目是否会超过 scqsize

这种情况的发生必须基于以下条件：

- runtime 存在超过 scqsize 的 goroutines
- 超过 scqsize 的 goroutines 在同一时刻都在执行 dequeue 或者 enqueue

首先对于第一种情况，因为目前的 scqsize 值为 1 << 16 (65536)，如果 runtime 中 goroutines 大于 scqsize 时，大部分业务很可能处于一个不太稳定的状态，很大概率已经不能正常服务。

其次，由于 SCQ 的 enqueue 和 dequeue 耗时非常短，绝大多数 < 50 ns，大约在 20 ns 左右，处于这个量级的操作，runtime 中有大量 goroutine blocking 在这个方法上是非常难的。想要同时将如此多的 goroutines blocking 在 SCQ 上必须在一些极端的情况。

为此，我们使用了一台内存为 32 G 的台式机进行极端情况的模拟实验，我们尽可能创造 goroutines 来执行 Enqueue 和 Dequeue，并且 Enqueue 和 Dequeue 之间没有间隔。

通过各种方法比对，可测得的 blocking goroutines 最大值为 18000 左右，此时占用 95% 内存（测试的 goroutine 栈只有 2k，一般业务应用占用的内存应该更多，因为部分栈会增长）。构造方法是，尽量启动 goroutine 占满现有内存，每个 goroutine 都会 time.Sleep 一个固定时间，然后等待 runtime 唤醒，然后不断执行 Enqueue-Dequeue-Pair。使用 atomic.Add 来进行测量，最终取最大值作为结果。

实际应用基本不会达到这个情况，因为当我们稍微加入了一点点执行时间在 Enqueue 和 Dequeue 之间，此时可测得的 blocking goroutines 只有 ~ 15 左右。

因为在最极端的 goroutines 占满 32G 内存的情况下也没有发生突破限制的情况，并且可测得最大值还不到 scqsize 的一半。据此，我们认为 goroutines 不会突破 scqsize 这个限制。



### LSCQ: SCQ 迭代是否会导致 data 遗留

LSCQ 会在 Enqueue 填满一个 SCQ 的时候尝试创建一个新的 SCQ 来存储新的 data，从而到达无限容量的特性。我们假设 LSCQ 将 q1 填满后，创建了 q2 并且把 q1 close 掉。q1 处于 close 状态时，任何 Enqueue 都会失败，只接受 Dequeue。

我们使用反证法，尝试证明在现有机制下，**一个 Enqueue 可能在所有 Dequeue 都执行结束后还能将 data 注入一个已经被 closed 的 queue (q1)**。如果命题成立，则 Enqueuer 会将 data 遗留在 被 closed 的 queue 上（因为没有新的 Deueuer 会进入）。

现有的代码逻辑是，将论文的 Figure 8 加上 Figure 10 的改动（因为我们实现的是 CAS2 版本），并且在 Figure 8 的 Line 13 中加入对 closed 状态的判定。这个判定的原理是在 tail 预留 1-bit 的标志位，如果为 1 则表示这个 queue 已经被 closed，此时 Enqueue 会直接返回 false。

- 如果 Enqueuer 在 Line [0,13] 被 blocking。由于我们在 Line 13 加入对于 closed 状态的判定，所以其会被直接返回，此种情况不会造成 data 遗留。
- 如果 Enqueuer 在 Line (13,15] 被 blocking，由于此时我们已经执行了 Line 13，所以此时 tail 必然会被增加 1，所以最终总会有一个 Dequeue 到达这个位置，所以一共两种情况
  - 此位置 Dequeue 先到达，则会将 entry 的 cycle 设置为和 tailcycle 相等，此时 Line 16 不成立。进行下次循环的时候会经过 Line 13，此时会发现 queue 已经被 closed，所以不会造成 data 泄露。
  - 此位置 Enqueuer 先结束 blocking，则 Enqueuer 成功将 data 注入，由最后的 Dequeuer 取出。
- 如果 Enqueuer 在 Line [16,18) 被 blocking，同上个情况类似，最终一定有一个 Dequeue 到达这个位置，我们以 Line 18 是否成功来分析两种情况
  - Enqueuer CAS 成功的情况，此时如果 Dequeuer 后到达，则 Dequeuer CAS 成功，拿到 data；如果 Dequeue 先到达，此时会进行 dequeue-retry，再次 Load 这个 entry，成功拿到 data。
  - Enqueuer CAS 失败的情况，此时必然是因为 Dequeue 先到达导致 entry 变化造成的，这种情况下 Enqueuer 会进行 dequeue-retry，回到 Line 15，此时由于发现 entry 的 cycle 和 tailcycle 一致（Dequeuer 造成的），所以会进行下次重试，然后在下次重试的 Line 13 结束后发现 queue 已经被 closed，于是返回 false。
- Enqueuer 在 Line 18 以后被 blocking 的情况不会向 queue 中插入新元素。

据此，我们逐段分析也无法构造出 data 遗留的情况，**所以 SCQ 迭代不会导致 data 遗留问题**。



## 附录二：Benchmark

### 名词解释

- LinkedQueue，使用一把锁和双向链表构成的 Queue，每次 enqueue 就在链表尾部添加数据，每次 dequeue 就从链表头部移除数据，使用一个锁来解决并发安全问题。
- MSQueue，即 [Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms][7] 论文中的第一种实现。



### 测试平台1

- Go version: go1.16.2 linux/amd64 
- OS: ubuntu 18.04
- CPU: AMD 3700x(8C16T), running at 3.6 GHZ (disable CPU turbo boost)
- MEMORY: 16G x 2 DDR4 memory, running at 3200 MHZ



#### CPU=100

```bash
go test -bench=. -cpu=100 -run=NOTEST -benchtime=1000000x
```

![benchmarkcpu100](https://raw.githubusercontent.com/zhangyunhao116/public-data/master/lscq-benchmark-cpu100.png)

```
Default/EnqueueOnly/LSCQ-100            38.9ns ±14%
Default/EnqueueOnly/linkedQ-100          209ns ± 3%
Default/EnqueueOnly/msqueue-100          379ns ± 2%
Default/DequeueOnlyEmpty/LSCQ-100       10.0ns ±31%
Default/DequeueOnlyEmpty/linkedQ-100    79.2ns ± 4%
Default/DequeueOnlyEmpty/msqueue-100    7.59ns ±44%
Default/Pair/LSCQ-100                   58.7ns ± 7%
Default/Pair/linkedQ-100                 324ns ± 5%
Default/Pair/msqueue-100                 393ns ± 2%
Default/50Enqueue50Dequeue/LSCQ-100     34.9ns ± 8%
Default/50Enqueue50Dequeue/linkedQ-100   183ns ± 7%
Default/50Enqueue50Dequeue/msqueue-100   191ns ± 3%
Default/30Enqueue70Dequeue/LSCQ-100     78.5ns ± 4%
Default/30Enqueue70Dequeue/linkedQ-100   148ns ± 8%
Default/30Enqueue70Dequeue/msqueue-100   136ns ± 4%
Default/70Enqueue30Dequeue/LSCQ-100     36.2ns ±13%
Default/70Enqueue30Dequeue/linkedQ-100   195ns ± 4%
Default/70Enqueue30Dequeue/msqueue-100   267ns ± 2%
```



#### CPU=16

```bash
go test -bench=. -cpu=16 -run=NOTEST -benchtime=1000000x
```

![benchmarkcpu16](https://raw.githubusercontent.com/zhangyunhao116/public-data/master/lscq-benchmark-cpu16.png)

```
Default/EnqueueOnly/LSCQ-16             33.7ns ± 5%
Default/EnqueueOnly/linkedQ-16           177ns ± 2%
Default/EnqueueOnly/msqueue-16           370ns ± 1%
Default/DequeueOnlyEmpty/LSCQ-16        3.27ns ±47%
Default/DequeueOnlyEmpty/linkedQ-16     91.1ns ± 2%
Default/DequeueOnlyEmpty/msqueue-16     3.23ns ±46%
Default/Pair/LSCQ-16                    56.1ns ± 3%
Default/Pair/linkedQ-16                  290ns ± 1%
Default/Pair/msqueue-16                  367ns ± 1%
Default/50Enqueue50Dequeue/LSCQ-16      31.8ns ± 3%
Default/50Enqueue50Dequeue/linkedQ-16    157ns ± 8%
Default/50Enqueue50Dequeue/msqueue-16    188ns ± 4%
Default/30Enqueue70Dequeue/LSCQ-16      73.8ns ± 2%
Default/30Enqueue70Dequeue/linkedQ-16    149ns ± 5%
Default/30Enqueue70Dequeue/msqueue-16    123ns ± 2%
Default/70Enqueue30Dequeue/LSCQ-16      28.8ns ± 4%
Default/70Enqueue30Dequeue/linkedQ-16    176ns ± 3%
Default/70Enqueue30Dequeue/msqueue-16    261ns ± 2%
```



#### CPU=1

```bash
go test -bench=. -cpu=1 -run=NOTEST -benchtime=1000000x
```

![benchmarkcpu1](https://raw.githubusercontent.com/zhangyunhao116/public-data/master/lscq-benchmark-cpu1.png)

```
name                                    time/op
Default/EnqueueOnly/LSCQ                17.3ns ± 1%
Default/EnqueueOnly/linkedQ             59.9ns ± 6%
Default/EnqueueOnly/msqueue             67.1ns ± 2%
Default/DequeueOnlyEmpty/LSCQ           4.77ns ± 1%
Default/DequeueOnlyEmpty/linkedQ        11.3ns ± 2%
Default/DequeueOnlyEmpty/msqueue        3.14ns ± 1%
Default/Pair/LSCQ                       36.7ns ± 0%
Default/Pair/linkedQ                    56.2ns ± 6%
Default/Pair/msqueue                    60.2ns ± 2%
Default/50Enqueue50Dequeue/LSCQ         23.1ns ± 2%
Default/50Enqueue50Dequeue/linkedQ      34.1ns ± 3%
Default/50Enqueue50Dequeue/msqueue      40.8ns ± 9%
Default/30Enqueue70Dequeue/LSCQ         26.5ns ± 2%
Default/30Enqueue70Dequeue/linkedQ      27.0ns ±28%
Default/30Enqueue70Dequeue/msqueue      26.7ns ± 7%
Default/70Enqueue30Dequeue/LSCQ         25.2ns ± 5%
Default/70Enqueue30Dequeue/linkedQ      47.3ns ± 5%
Default/70Enqueue30Dequeue/msqueue      55.2ns ± 8%
```





## 附录三：应用案例

### gopool

> 实际任务队列使用 MPMC queue 并不是最优的，采用 per-P queue 的方式更加合适，这里使用 MPMC queue 相比于 linkedQueue  性能更好（因为 runtime 外无法使用 per-P）。

[gopool](https://github.com/bytedance/gopkg/tree/develop/util/gopool) 是由字节跳动框架团队开发的 goroutines pool，其中有一个组件是"任务队列"，作用是将不同的任务交给池中的 goroutines，然后让这些 goroutines 来完成对应的任务。gopool 同时也是 kitex 和 netpoll 依赖的基础组件。

原本的任务队列是由一把锁加上双向链表组成的（即对比的 LinkedQueue），这种结构在少核低负载的时候表现不错，但是在多核高负载的情况下表现不好（因为锁的存在，只能利用一个 CPU 的资源），我们在和框架团队的交流中也发现了这个问题。一个简单的比喻是，如果整个小区的快递只有一个快递员来配送，那么在平常快递员能够很好的完成任务，小区的人也可以及时收到快递。但是到了快递较多的双十一期间，如果还是只有一个快递员，那么尽管外面的快递包裹堆积如山（任务堆积），但是小区内收到的快递总是恒定的（就是那个快递员的极限），所以会造成小区的人不能及时收到快递。而 scalable queue 则有一个特殊的机制，即外面的快递较多的时候，可以让小区的人自己去取快递，这样整个小区能够收到快递的数量就会多很多。

我们将 LSCQ 应用到 gopool 后，使得 gopool 的任务队列效率大大提升，并且变成 scalable queue。以下列出了在不同核数情况下的吞吐提升。

```
name     old time/op    new time/op    delta
Pool-4     4.15ms ± 7%    3.88ms ± 2%   -6.40%  (p=0.000 n=10+10)
Pool-8     5.18ms ± 5%    4.60ms ± 2%  -11.16%  (p=0.000 n=10+10)
Pool-16    12.1ms ± 1%     9.6ms ± 1%  -20.93%  (p=0.000 n=9+10)
```

将优化后的 gopool 在 kitex 中集成 benchmark，在 4 core、QPS=5000 的测试中，提升了  ~ 1 W 的 avg QPS (95968 -> 105643)。

总的来说，此次优化主要目的是提升资源的利用率（任务的收发更快了，所以有更多的 CPU 可以参与其中），对于多核(>=4)、高负载的情况下有很大的提升，对于少核、低负载的情况下，由于此时 LinkedQueue 也表现不错，可能只能在时延上看到收益。



## References

[1]:https://arxiv.org/abs/1908.04511	"SCQ"

[2]:https://en.wikipedia.org/wiki/Compare-and-swap	"Compare-And-Swap"
[3]:https://www.felixcloutier.com/x86/cmpxchg8b:cmpxchg16b	"cmpxchg16b"
[4]:https://www.i3s.unice.fr/~jplozi/prog_concurrente_2016/tps/tp2_C31rNqzu/p257-fatourou.pdf	"CCQueue"
[5]:https://en.wikipedia.org/wiki/Fetch-and-add	"Fetch-And-Add"
[6]:https://en.wikipedia.org/wiki/Thread-local_storage	"Thread-local storage"
[7]:https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf	"MSQueue"
[8]:https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentLinkedQueue.html	"ConcurrentLinkedQueue"
[9]:https://www.cs.tau.ac.il/~mad/publications/ppopp2013-x86queues.pdf	"CRQ"

