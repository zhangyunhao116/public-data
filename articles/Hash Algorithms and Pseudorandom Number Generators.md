# **Hash Algorithms and Pseudorandom Number Generators**



## 前言

> 本文写于 2021-10

Hash 算法和伪随机数生成器可能是大家在日常实践中大量使用，但是对其中原理不太熟悉的两种算法。由于这两种算法原理类似，本篇文章会解析这些算法的原理。本次我们讨论的算法都是 `Non-Cryptographic algorithms`。



这里给出一些实现 Hash 算法和伪随机数生成器的例子：

- fastrand: https://github.com/bytedance/gopkg/tree/develop/lang/fastrand。相比于 `math/rand` 快 100x 的伪随机数包（GO <= 1.22 时）。
- xxhash3: https://github.com/bytedance/gopkg/tree/develop/util/xxhash3。xxhash3 在 Golang 的实现。



## Hash 算法

Hash 算法大家的第一印象应该是在 hashmap 中的实践，即对于某个 `string` 转化成一个数值，例如 `uint64`。这个数值会作为 index 插入对应的 slot 来存储 key-value。



这种典型场景下，hash 算法有如下性质

- **固定的输入对应固定的输出**（例如 "123" 一定输出 97820173）

优秀的 hash 算法应该具备以下特性

- **hash result 生成质量好**。即不同的输入，其输出应该尽量分布均匀 （例如即便 "123" "1234" 这样类似的 `string` 输出也分布均匀）

- **性能好**。通常最大的性能提升点在于 CPU 是否能提供对应的高效指令集。



近些年来，涌现了很多 hash 算法，这里介绍几种在各方面表现较好，并且在 Go runtime 中也使用过的 hash 算法。

- xxhash，https://github.com/Cyan4973/xxHash。目前有 xxhash64、xxhash32、xxhash3 三个版本。特点是可以利用 CPU 提供的硬件指令来加速（AVX2、SSE），**通常来说对于长度较长的输入性能表现较好**。之前是 memhash 的 fall back 算法。

- wyhash，https://github.com/wangyi-fudan/wyhash。目前有多个版本。特点是实现简单，容易在不同的架构中实现（portable），**通常来说对于长度较短(< 64 bytes)的输入性能表现较好**。目前是 memhash 的 fall back 算法，可以在 https://github.com/golang/go/blob/master/src/runtime/hash64.go 的 `memhashFallback` 中看到。

- memhash，目前是 golang 默认的 hash 算法。特点是利用了 CPU 提供的 AES 的指令，大大加速了 hash 速度。速度和上面两者类似，但是标准库提供的 API 对于 small key 性能不好。我们在高性能基础库的一些数据结构中，对于 map 这样 small key 较多的场景，选择使用 wyhash 而不是 memhash 来提升性能。



### 基本流程

目前来说，Hash 算法的基本流程 (以 wyhash 为例)

- **初始化 internal status**。这里的 internal status 一般是一个或多个值，例如 wyhash 使用的是一个固定的 `uint64`。
- **读取 string 的每个 byte，影响 internal status**。每次读取不同的 bytes 来改变这个 internal status。为了理解方便，可以简单认为是将每个 byte 的值加入这个 `uint64` 中（不是实际的步骤，单纯举例）。

> 从性能方面考虑，一般来说 hash 算法大多会尝试一次读取多个 byte，然后使用这些 bytes 生成一个或者多个值，最后才用这些值来对 internal status 进行改变。
>
> 从 hash 质量方面考虑，这里 bytes -> `uint64 ` 的过程会使用很多的方法，例如 `^` 异或，来使得类似的输入可以得到不同的输出，保证结果不会碰撞。

- **将这个 internal status 即 `uint64`，再进行一些操作，生成最终的结果(同样也是 uint64)**。这一步的目的是，避免上一步的 internal status 不够均匀，例如当这个 `string` 长度较小时，internal status 没有经过足够的迭代，会使得相似的内容容易出现结果碰撞。

大部分 Hash 算法基本都是上面几个流程，区别只在于读取 string 影响 internal status 的方式不同、internal status 可能由多个值组成等。这些方式的目的在于，**使得输入和输出尽可能没有关联**，这样即便用户输入的数据在内容上很相似，但是会得到完全不同的结果。

为了使得输入和输出尽可能没有关联，hash 算法可能会使用很多的步骤，但是这些步骤又会减慢 hash 算法的速度。所以优秀的 hash 算法都需要在 生成质量和生成速度 之前做好足够的平衡。

例如在 step 3 中 wyhash 依赖 `MUM` 来做混淆，并且使用异或来解决原始 `MUM` 易受影响的问题(例如当 a 为 0 时会丢失熵)。混淆的目的还是在于使得输入和输出尽可能没有关联。关于 `MUM` 具体讨论可以参考 wyhash 仓库中的 PDF 原理介绍。

> MUM (A, B) -> C, where A, B, C are 64-bit unsigned integers，对应示例 Go 代码为

```
func _wymix(a, b uint64) uint64 {
	hi, lo := bits.Mul64(a, b)
	return hi ^ lo
}
```



## 伪随机数生成器

伪随机数生成器，英文为 PRNG (pseudo random number generator)，同样也是一种我们经常使用的常见算法。

无论是 `math/rand` 和 `fastrand` 中都是采用了 PRNG 来生成随机数。注意，这里的随机数是 **伪造** 的，意味着我们得到 PRNG 的状态，并且弄清楚了其生成算法，我们**可以人为的计算出下一个随机数**。而真随机数一般需要硬件支持，我们无法从一个随机数推导出下一个随机数的值。



这里先列举一下 Go runtime 中使用过的 PRNG 算法，前两者是 fastrand 中使用的算法，最后是 math/rand 使用的算法。

- XORSHIFT，https://www.jstatsoft.org/article/view/v008i14/xorshift.pdf。
- wyrand，https://github.com/wangyi-fudan/wyhash。实际和 wyhash 的原理基本一致。
- math/rand 算法。https://golang.org/src/math/rand/rng.go。



### 基本流程

一般来说，PRNG 算法的基本流程为 (以 XORSHIFT为例)

- **初始化 internal status**。这里的 internal status 同样也是一个或多个值，例如 XORSHIFT 使用的是 `[2]uint32`。
- **使用 internal status，生成 result，然后更新 internal status**。

> XORSHIFT algorithm，这里的 Uint32() 会生成一个 `uint32` 随机值，来初始化 internal status。一般来说最初的 seed 会来自操作系统提供的伪随机数。

```go
// Step 1.
var tmp [2]uint32 // internal status
tmp[0], tmp[1] = Uint32(), Uint32()
// Step 2.
s1, s0 := tmp[0], tmp[1]
s1 ^= s1 << 17
s1 = s1 ^ s0 ^ s1>>7 ^ s0>>16
tmp[0], tmp[1] = s0, s1 // update internal status
result = s1 + s0
```

我们可以明显的发现，**PRNG 算法其实和 Hash 算法的原理基本一致**，可以理解为 PRNG 只是缺少了读取 `string` 来影响 internal status 的步骤(Hash 算法的 step 2)，此时 Hash 算法的 step 2 作用只是带来了更多的熵。

还记得 math/rand 中我们可以调用一个 `Seed` 方法么？这里其实就是**初始化 PRNG 的 internal status**，根据 PRNG 原理，如果我们确定了 internal status (也就是 seed)，接下来 math/rand 的随机数生成序列是唯一确定的。

为了保持这种生成序列的一致，math/rand 其实使用了一把锁，将 internal status 保护起来，避免 PRNG 算法的并发安全问题（多个 goroutines 调用会使得 internal status 不一致）。生成序列一致有利于一些依赖这种特性的测试。

但是很多情况下，我们并不需要生成序列一致，只需要保证每次过程足够随机即可。fastrand 就是采用了这个想法，在每个 Machine(GMP概念，可以理解为 per-thread) 中保存 internal status，这样每个 goroutine 不需要获得锁就可以直接生成一个伪随机数并且更新 internal status。基于此，随着 CPU 核数的上升，其性能相比 math/rand 会越来越好。

fastrand 会使用操作系统提供的伪随机数生成器，例如 linux 的 /dev/urandom，来初始化一个 fastrandseed uintptr。每次 M 初始化的时候，Go 会使用 fastrandseed 和 cputicks 共同生成一个 seed 作为 M 的 internal status。后续 fastrand 生成随机数的时候，会不断迭代 M 的 internal status 来生成伪随机数。



## 总结

Hash 算法和伪随机数生成器其实原理一致，区别只在于 Hash 算法会使用用户的输入来影响 internal status。伪随机数生成器的 `seed` 的作用在于确定后续的随机数生成序列。如果不需要生成序列的一致性，可以使用 `fastrand` 来代替 `math/rand`。

