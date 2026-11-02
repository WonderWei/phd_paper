# 池化内存中的可增长远端对象设计

> **设计目标：** 在池化内存（pooled/disaggregated memory）中，高效支持持续增长的远端对象，既避免连续布局在扩容时的整体搬迁，又避免链式布局带来的多次依赖式远程访问。

可以把这个项目定义成一个非常清晰的系统问题：

> **在池化内存（pooled/disaggregated memory）中，如何高效支持会持续增长的远端对象，使其既避免连续布局在扩容时的整体搬迁，又避免链式布局带来的多次依赖式远程访问。**

这个项目的关键不在“variable-size object”本身，而在 **dynamically growing remote object**。如果对象创建时大小就已经确定，那么 size-class/slab allocation 已经是很成熟、也很合理的方案；真正产生结构性矛盾的是：对象在运行过程中不断 append，并逐渐跨越原来的分配边界。

---

## 一、Background：为什么池化内存里会出现这个问题

现代云系统越来越倾向于将计算资源和内存资源解耦。Compute Node 负责执行应用逻辑，而大量 DRAM 集中部署在独立的 Memory Pool 中，通过 RDMA 或 CXL 等高速互连访问。

从逻辑上看，Memory Pool 给上层系统提供的是一个大的远端地址空间：

```text
Compute Nodes
     |
     | RDMA / CXL
     |
     v
+-------------------------+
|    Pooled Memory        |
|                         |
|   Object A              |
|   Object B              |
|   Object C              |
|   ...                   |
+-------------------------+
```

但应用并不是直接操作“内存字节”，而是操作各种逻辑对象，例如：

```text
posting list
event history
time-series sequence
adjacency list
metadata list
remote vector
```

这些对象有两个非常常见的特点。

第一，**对象大小具有明显异质性**。

例如某些对象只有几十或几百字节，而另一些对象可能达到几 KB 甚至几十 KB：

```text
Object A    128 B
Object B    192 B
Object C      8 KB
Object D    256 B
Object E     32 KB
```

第二，也是这个项目真正关注的地方：

> **对象的大小并不是固定不变的。**

例如一个事件历史：

```text
t0:
[event1]

t1:
[event1, event2]

t2:
[event1, event2, event3]

...

tn:
[event1, ..., event10000]
```

或者 posting list：

```text
"database"
    ↓
[doc3, doc17]
    ↓
[doc3, doc17, doc25, doc91]
    ↓
[doc3, doc17, ..., doc10000]
```

这种对象从本质上就是：

> **append-growing object**

也就是说，对象创建时很小，之后会随着 workload 持续增长。

---

## 二、为什么 local memory 中不是特别严重的问题

如果这些对象全部存放在本地 DRAM 中，一个最直接的方案就是动态数组。

比如类似 `std::vector`：

```text
capacity = 256 B

        ↓ full

capacity = 512 B

        ↓ full

capacity = 1 KB

        ↓

capacity = 2 KB
```

每次容量不够，就：

```text
allocate larger region
        ↓
copy existing data
        ↓
update pointer
        ↓
free old region
```

本地系统里，这通常是一个可以接受的设计。

因为：

```text
CPU ───── Local DRAM
```

`memcpy` 使用本地 memory bandwidth，而且内存访问延迟很低。

所以传统软件大量采用这种：

> **contiguous + reallocation**

的设计。

---

## 三、到了池化内存之后，成本模型发生变化

在 pooled memory 中：

```text
CN
 |
 | network
 |
 v
MN
```

对象搬迁已经不再是简单的本地 `memcpy()`。

假设对象当前是 4 KB，现在 append 100 B。

如果原来的 region 放不下，可能需要：

```text
Old remote object: 4 KB
        |
        | allocate new remote region
        v
New remote region: 8 KB
        |
        | copy / RDMA transfer existing 4 KB
        |
        | append 100 B
        |
        | publish new object pointer
        |
        | reclaim old region
```

于是：

> 逻辑上只写了 100 B，物理上却可能搬运几 KB 数据。

这就是第一个重要变化。

第二个变化则来自远程访问。

如果为了避免搬迁，我们把 object 拆成链：

```text
Chunk 1
   |
Chunk 2
   |
Chunk 3
   |
Chunk 4
```

在 local DRAM 上，pointer chasing 虽然不好，但还未必不可接受。

而在 pooled memory 中：

```text
RDMA READ Chunk1
      ↓
wait for completion
      ↓
obtain ptr2
      ↓
RDMA READ Chunk2
      ↓
wait
      ↓
obtain ptr3
```

于是一次逻辑读取可能变成多个依赖式 network round trip。

所以 pooled memory 把两个原本普通的软件工程问题放大成了两个系统问题：

> **Data Relocation**
> 和
> **Remote Pointer Chasing**

---

## 四、核心问题：连续布局和链式布局存在结构性 trade-off

这就是整个项目最核心的一层。

对于 growable object，最自然的两种设计实际上分别站在两个极端。

### 方案 A：Contiguous Object

例如按照 size class 分配：

```text
256 B
512 B
1 KB
2 KB
4 KB
8 KB
...
```

一个 700 B object：

```text
+--------------------+
|     1 KB slot      |
|   object = 700 B   |
+--------------------+
```

它最大的优势是：

> **读取效率极高。**

整个 object 连续存在 remote memory 中。

知道地址后：

```text
RDMA READ(addr, size)
```

即可完成读取。

没有 pointer chasing。

但是问题出现在 object growth。

假设：

```text
700 B
 ↓ append
1.1 KB
```

原来的 1 KB slot 不够了。

于是：

```text
1 KB slot
    ↓
allocate 2 KB
    ↓
copy 700 B / 1 KB existing object
    ↓
append new data
    ↓
publish new pointer
```

所以：

> **Contiguous layout optimizes reads by sacrificing growth efficiency.**

---

## 五、核心问题一：Remote Relocation Amplification

这里建议把问题定义得非常准确。

不是说 reallocation 在 asymptotic complexity 上一定很差。

如果采用 doubling：

$$
256B\rightarrow512B\rightarrow1KB\rightarrow2KB\rightarrow4KB
$$

累计搬迁的数据仍然是几何级数。

真正的问题是：

> **一次很小的逻辑 append，会偶尔触发一次与当前 object size 成比例的大规模远程搬迁。**

定义：

$$
A_{\text{reloc}}
=
\frac{\text{physical bytes moved}}
{\text{logical bytes appended}}
$$

假设当前 object 是 8 KB。

append：

$$
64B
$$

却触发扩容。

可能需要搬：

$$
8KB
$$

于是：

$$
A_{\text{reloc}}
\approx
\frac{8192}{64}
=
128
$$

也就是说：

> 写入 64B，却引入超过百倍的数据搬迁。

实际系统不会每次 append 都这么严重，但在 capacity boundary 会发生明显 spike。

所以你应该把这个问题叫：

> **Remote relocation amplification**

而不只是“allocation overhead”。

---

## 六、Relocation 还有第二层问题：地址稳定性

object relocation 后：

```text
old addr
    ↓
new addr
```

地址发生变化。

这会进一步带来：

```text
metadata update
pointer publication
concurrent reader synchronization
old copy reclamation
cached-address invalidation
```

假设 CN1 正在读：

```text
old_addr
```

CN2 恰好把 object 搬到了：

```text
new_addr
```

就必须决定：

```text
old copy 什么时候可以 free？
```

所以 relocation 不仅消耗 bandwidth，也使 concurrency 和 reclamation 更复杂。

不过在这个项目里，这个问题可以作为 secondary motivation，不需要把它发展成主要贡献。

你的核心还是性能：

> **small incremental updates occasionally trigger large remote data movements.**

---

## 七、那为什么不用链式结构？

非常自然的解决方法就是：

> 不搬旧数据。

例如 fixed-size chunk：

```text
Object Header
     |
     v
+----------+
| Chunk 1  |
+----------+
     |
     v
+----------+
| Chunk 2  |
+----------+
     |
     v
+----------+
| Chunk 3  |
+----------+
```

object append 时：

```text
Chunk 3 full
     ↓
allocate Chunk 4
     ↓
Chunk3.next = Chunk4
```

过去的数据完全不用动。

所以它很好地解决了 relocation：

$$
\text{append cost}
\approx
O(\text{new data})
$$

而不是：

$$
O(\text{current object size})
$$

---

## 八、核心问题二：Fine-grained Remote Pointer Chasing

链式结构的问题在读取路径上。

假设 chunk = 256B，而 object = 8KB：

$$
8192/256=32
$$

需要 32 个 chunk。

如果每个 chunk 只保存：

```text
next_ptr
```

那么地址存在依赖：

```text
Chunk1
  ↓ obtain ptr2
Chunk2
  ↓ obtain ptr3
Chunk3
...
```

从 CN 看：

```text
READ C1
wait
READ C2
wait
READ C3
wait
...
```

假设一个 remote RTT 为 \(T_{RTT}\)，粗略地：

$$
T_{\text{read}}
\approx
N_{\text{chunk}}
\times
T_{\text{RTT}}
$$

所以 object 越大，读取 latency 越差。

于是：

> **Linked layout optimizes growth by sacrificing read efficiency.**

---

## 九、项目真正的核心矛盾

因此你不是在解决一个简单的 allocator 问题，而是在解决一个非常具体的数据布局 trade-off：

$$
\boxed{
\text{Relocation Efficiency}
\quad\leftrightarrow\quad
\text{Remote Read Efficiency}
}
$$

Contiguous：

```text
            Read
             ★★★★★

Contiguous ─────────

Growth
★★
```

Linked：

```text
             Read
              ★★

Linked ─────────────

Growth
★★★★★
```

你的目标是：

```text
             Read
             ★★★★☆

Ours ───────────────

Growth
★★★★☆
```

所以核心研究问题可以写成：

> **How can pooled-memory systems efficiently support dynamically growing objects without repeatedly relocating existing data or introducing fine-grained dependent remote accesses?**

这句话基本可以作为整个项目的 Problem Statement。

---

## 十、第三个隐含问题：对象大小和增长速度高度不均匀

如果所有对象最终都是 8KB，那么事情其实没那么复杂。

直接给每个对象 8KB 就行。

真正让 static provisioning 困难的是：

```text
Object A   128B
Object B   256B
Object C   512B
Object D   16KB
Object E   256B
Object F   64KB
```

而且：

```text
A: never grows
B: slowly grows
C: rapidly grows
D: already stable
...
```

也就是说：

> **growth is heterogeneous and workload-dependent.**

因此不能简单预留：

```text
每个 object 64KB
```

因为会造成巨大的 internal fragmentation。

这给你的设计提出另一个要求：

> 小对象必须保持便宜；只有真正增长的对象才付扩展成本。

这就是为什么你的设计应该有 **Inline Region**。

---

## 十一、核心设计：Block-Extensible Remote Object

整个方案可以非常简单。

暂时叫：

> **Block-Extensible Object Layout**

或者：

> **Inline-Extent Object**

基本结构：

```text
+----------------------------------+
| Object Header                    |
|                                  |
| logical_size                     |
| extent_count                     |
| extent_ptr[0]                    |
| extent_ptr[1]                    |
| extent_ptr[2]                    |
| ...                              |
+----------------------------------+
| Inline Payload                   |
| e.g., 256 B                      |
+----------------------------------+

             |
             |
        +----+----+
        |         |
        v         v

+-------------+   +-------------+
| Extent 0    |   | Extent 1    |
| 1 KB        |   | 1 KB        |
+-------------+   +-------------+
```

这个结构有三个核心设计原则。

---

## 十二、设计原则一：Inline Common Case

对于小 object：

$$
size\le256B
$$

全部数据直接存在：

```text
Header + Inline Payload
```

所以读路径：

```text
CN
 |
 | one RDMA READ
 v
Header + Payload
```

直接结束。

这样保证：

> **支持 dynamic growth 不应该让普通小 object 为此付额外成本。**

这是非常重要的一点。

因为实际 workload 里可能 70%–90% object 都很小。

你的系统不能为了那 10% 会增长的大 object，让另外 90% 都多一次 pointer dereference。

所以：

> **Small objects remain contiguous.**

---

## 十三、设计原则二：Coarse-grained Extent Expansion

当 object 超过 inline：

```text
256B
 ↓
300B
```

不重新分配整个 object。

而是：

```text
Inline
  |
  +---- Extent 0
```

例如：

```text
Inline = 256B
Extent = 1KB
```

继续 append：

```text
Object size = 900B
```

仍然：

```text
Inline + E0
```

当超过：

$$
256+1024
$$

再分配：

```text
E1
```

于是：

```text
Inline
  |
  +-- E0
  +-- E1
```

关键是：

> **已有 E0 永远不搬。**

对象：

```text
1.2KB → 1.5KB
```

只需要：

```text
write into free space of E0/E1
```

如果没有空间：

```text
allocate new extent
write new payload
publish extent metadata
```

而不是：

```text
allocate larger 2KB/4KB object
copy everything
```

因此：

$$
\text{physical append traffic}
\approx
\text{new data}
$$

---

## 十四、为什么叫 coarse-grained extent，而不是 chunk

这是你和 linked chunks baseline 真正的区别。

假设：

```text
Linked chunk = 128B
```

一个 8KB object 需要：

$$
64
$$

个节点。

如果 extent = 2KB：

只需要：

$$
4
$$

个 extent。

所以即使最坏情况下还需要 sequential traversal：

$$
64 RTT
\rightarrow
4 RTT
$$

已经大幅下降。

因此核心思想就是：

> **Turn fine-grained pointer chasing into coarse-grained object expansion.**

这是这个设计最重要的第一层优化。

---

## 十五、设计原则三：Extent Directory 消除 dependent traversal

但如果你的布局仍然是：

```text
E0 -> E1 -> E2 -> E3
```

其实并没有彻底解决 pointer chasing。

所以最好加入一个很轻量的：

> **Extent Directory**

Header 直接保存：

```text
extent_count = 4

extent_ptr[0]
extent_ptr[1]
extent_ptr[2]
extent_ptr[3]
```

这样读取流程是：

```text
         +--> E0
         |
Header --+--> E1
         |
         +--> E2
         |
         +--> E3
```

而不是：

```text
Header -> E0 -> E1 -> E2 -> E3
```

CN 读到 Header 后，已经知道：

```text
addr(E0)
addr(E1)
addr(E2)
addr(E3)
```

于是可以连续 post：

```text
RDMA READ E0
RDMA READ E1
RDMA READ E2
RDMA READ E3
```

NIC 可以同时处理这些 request。

这样：

$$
T
$$

不再近似：

$$
4\times RTT
$$

而更接近：

$$
RTT_{\text{header}}
+
RTT_{\text{parallel extents}}
+
\text{data transfer}
$$

这就是你的第二个真正重要的 design point。

---

## 十六、完整 Read Path

### Small Object

例如 128B：

```text
READ Header + Inline
        ↓
      done
```

一次远程访问。

---

### Medium Object

例如 1KB：

```text
READ Header
     ↓
get E0

READ E0
```

大约两阶段。

---

### Large Object

例如 8KB：

```text
READ Header
     ↓

E0 E1 E2 E3 addresses known

     ↓

parallel RDMA READs

E0  E1  E2  E3
```

不需要：

```text
E0
 ↓
E1
 ↓
E2
 ↓
E3
```

这就是读取性能能够接近 contiguous baseline 的关键。

---

## 十七、完整 Append Path

假设 object 当前：

```text
Inline + E0 + E1
```

而 E1 还有空间。

Append：

```text
RDMA WRITE new data → E1
RDMA WRITE new logical_size
```

即可。

---

如果 E1 满了：

```text
allocate E2
     ↓
RDMA WRITE new payload → E2
     ↓
publish E2 pointer
     ↓
update extent_count / logical_size
```

最关键的一句话：

> **Existing payload is never relocated during normal growth.**

这句话应该反复出现在你的设计描述里。

---

## 十八、为什么不直接预留很大的 virtual region

一个很自然的问题是：

> 为什么不一开始给 object 预留 64KB 地址空间，需要的时候再慢慢用？

如果完全有虚拟地址管理和 page-level demand allocation，这确实是一个 baseline。

但问题可能变成：

```text
page granularity
page table / translation
remote physical allocation
4KB minimum allocation
```

而对于大量：

```text
100B / 200B / 500B
```

对象，page-level allocation 可能又过粗。

不过因为你不追求发表，我建议不要把这个问题展开太大。

直接把 scope 限定为：

> fine-grained application-managed objects over a remote memory allocator.

就足够了。

---

## 十九、Concurrency 不要做复杂

你的项目不是事务协议。

所以 header 放：

```text
Version
Lock
```

即可。

Append：

```text
CAS lock
    ↓
allocate/write extent
    ↓
publish pointer
    ↓
update size/version
    ↓
unlock
```

Read 可以：

```text
read version
     ↓
read payload/extents
     ↓
read version again
     ↓
if changed → retry
```

这种 seqlock-style 方案已经足够 prototype 使用。

不要再做分布式 transaction。

---

## 二十、Allocator 也不要做复杂

Memory Pool 可以直接提前切好：

```text
1KB extent pool
```

例如：

```text
+------+------+
| E0   | E1   | E2 ...
+------+------+
```

CN 用 bitmap / free list 申请。

项目里不要研究：

```text
remote allocator scalability
```

那会把研究问题带跑。

Allocator 只是 supporting mechanism。

---

## 二十一、最适合这个设计的数据模型

实际上你可以直接暴露：

```text
Create(key)
Append(key, data)
Read(key)
ReadTail(key, n)
```

底层 object：

```text
key
 ↓
Growable Remote Object
```

这样已经足够。

上层可以解释成：

```text
event history
posting list
time-series sequence
adjacency list
metadata sequence
```

但 prototype 不需要真正实现这些应用。

---

## 二十二、整个项目真正需要证明的不是“extent 是最快的”

这一点非常重要。

你不是要证明：

> Extent 比任何 layout 都快。

而是证明：

### Static / read-only object

```text
Contiguous
≈
Ours
>
Linked
```

### Rapidly growing / append-heavy object

```text
Linked
≈
Ours
>
Contiguous
```

### Mixed workload

```text
Ours
>
Contiguous / Linked
```

也就是说，你提供的是一个：

> **better trade-off**

这反而比声称“全面碾压”更合理。

---

## 二十三、可以把整个项目凝练成三层逻辑

最后如果按照“背景 → 核心问题 → 核心方案”来浓缩，我建议就是下面这套。

### Background

Pooled memory allows compute nodes to access a large shared remote memory pool through RDMA/CXL. Many applications store variable-sized objects whose sizes evolve during execution, such as event histories, posting lists, time-series sequences, and adjacency lists.

### Core Problem

Existing layouts naturally fall into two extremes. Contiguous size-class allocation provides efficient remote reads but requires relocation when a growing object exceeds its allocated capacity, turning small appends into large remote data movements. Fine-grained linked layouts avoid relocation but introduce dependent remote accesses during reads. Thus, pooled-memory systems face a fundamental trade-off between **growth efficiency and remote-read efficiency**.

### Core Design

We organize each object as a small **inline region** followed by dynamically allocated **coarse-grained extents**. Small objects remain contiguous and require only one remote access. Growing objects append new extents without relocating existing payload. A compact **extent directory** stored in the object header exposes multiple extent addresses at once, allowing compute nodes to issue parallel/batched RDMA reads instead of traversing a pointer chain.

最终可以浓缩成这一张图：

```text
                 Growable Remote Objects
                          |
                          v

        +-----------------+-----------------+
        |                                   |
        v                                   v

  Contiguous Layout                 Linked Layout

  Fast remote reads                 Cheap growth
  One large region                  No relocation

        |                                   |
        v                                   v

  Remote relocation                 Pointer chasing
  on capacity overflow              multiple dependent RTTs

        \                                   /
         \                                 /
          +-------------+-----------------+
                        |
                        v

             Block-Extensible Object

              Header + Inline
                     |
             +-------+-------+
             |       |       |
            E0      E1      E2

          No relocation on growth
                     +
          Parallel extent fetching
```

**这就是这个项目最完整、也最稳的逻辑闭环：不是为了“发明一种新的 object layout”，而是针对 pooled memory 改变后的成本模型，在 contiguous relocation 和 linked traversal 两个极端之间设计一个更合适的 growable-object layout。**
