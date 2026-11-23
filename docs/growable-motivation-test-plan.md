# 可增长远端对象：真机 Motivation 测试方案

> 目的：用最少的实现量，在真实 RDMA 环境中证明 **"resize（扩容搬迁）会明显损害尾延迟，并在带宽受限时损害吞吐"**，为 extent 方案提供动机。
> 本方案 **不实现完整的 extent 方案**，而是用一个 "永不 resize" 的理想上界（Oracle）衡量 resize 的代价有多大。
> 相关文档：`growable-remote-object-design.md`（设计）、`growable-remote-object-quick-experiment.md`（成本模型与模拟结果）。

---

## 1. 要回答的问题

| 编号 | 问题 | 对应实验 |
|---|---|---|
| Q0 | 模型参数（RTT、带宽、消息速率、批量读）在真机上是多少？ | M0 |
| Q1 | 单次 resize 的停顿有多长？是否随对象大小线性增长？ | M1 |
| Q2 | resize 是否让 append 尾延迟出现明显尖峰？ | M2 |
| Q3 | 带宽受限时，resize 让吞吐下降多少？ | M3 |
| Q4 | 为什么不用链式结构？逐跳追指针的读代价有多大？ | M4 |

Q1–Q3 证明 "resize 有代价"，Q4 证明 "简单的链式方案不可取"。两者合起来就是设计文档第九节中的核心矛盾。

---

## 2. 环境

| 项目 | 要求 |
|---|---|
| 机器 | 2 台：1 台 CN，1 台 MN。M3 打满带宽时，单 CN 不够就加第 2 台 CN |
| 网卡 | ConnectX-5/6，100 Gbps，RC 连接。无本地集群可用 CloudLab（如 `r6525`、`c6525-100g`） |
| 不能用 | SoftRoCE（只能调正确性，不能测性能） |
| 内存 | MN 注册一块 8–32 GB MR，使用 hugepage |
| 绑定 | 线程绑核，绑到网卡所在的 NUMA 节点；关闭 CPU 降频（`performance` governor） |
| RoCE | 确认 PFC/ECN 已配置，否则大流量下丢包会污染尾延迟 |
| 代码 | C++ + libibverbs，全部为单边操作，MN 端 CPU 不参与。可复用 MiTDM 的 RDMA 通信框架 |

---

## 3. 被测方案（只需 3 个）

| 名称 | 行为 | 作用 |
|---|---|---|
| `Contig-2x` | 初始容量 256 B，满了就分配 2 倍新区域，CN 先 READ 旧数据再 WRITE 到新区域，最后 CAS 发布新地址 | 被测对象：resize 的代价 |
| `Contig-2x-pipe` | 同上，但搬迁按 64 KB 分段，READ 和 WRITE 流水线重叠 | **更强的 baseline**，避免审稿人说 baseline 太弱 |
| `Oracle` | 创建时直接按最终大小预分配，永不 resize | 理想上界：去掉 resize 后能达到的性能 |

M4 另外实现一个 `Linked-256`（256 B 块，`next` 指针）。

**所有方案共用：** CN 本地分配（预先划分 MN 地址区，CN 本地 bump 分配，0 RTT），header 中 8 B 的锁（RDMA CAS），读路径一次 RDMA READ。

Append 流程：

```text
CAS(lock)                              # 1 RTT
if size + len > cap:                   # 仅 Contig
    new = local_alloc(2 * cap)
    RDMA READ  old[0:size] -> local buf # 搬迁：读旧
    RDMA WRITE local buf -> new         # 搬迁：写新
    RDMA WRITE index[key] = new         # 发布新地址
WRITE data + new size + unlock          # 1 RTT（doorbell batch）
```

---

## 4. 实验

### M0：硬件参数标定（半天）

直接用 `perftest`，不需要写代码：

| 测试 | 命令 | 记录 |
|---|---|---|
| READ/WRITE 延迟 vs 大小（64 B–1 MB） | `ib_read_lat -a`、`ib_write_lat -a` | RTT、大块传输时延 |
| 带宽 vs 大小 | `ib_read_bw -a`、`ib_write_bw -a` | 单向峰值带宽 |
| 消息速率 | `ib_write_bw -s 64 -q 8 --report_gbits`（换算 Mops） | 小消息上限 |
| CAS 延迟 | `ib_atomic_lat` | 加锁代价 |
| 1 个 64 KB READ vs 64 个 1 KB READ（doorbell batch） | 自写 | extent 批量读是否接近大块读 |

**输出：** 把 RTT、带宽、消息速率填回 `growable-sim/sim.py` 的 `P` 和 `tput_sim.py` 的 `NET`，重跑模拟，确认结论方向不变。

### M1：单次 resize 停顿（半天）

- **方法：** 单线程，把一个对象恰好填到容量边界，然后测触发 resize 的那一次 append。
- **变量：** 对象大小 4 KB、16 KB、64 KB、256 KB、1 MB、4 MB、16 MB；方案为 `Contig-2x`、`Contig-2x-pipe`。
- **指标：** 停顿总时长，以及分解：READ 旧数据 / WRITE 新数据 / 发布。每个点重复 100 次，取中位数和 p99。
- **图 1：** 横轴对象大小（对数），纵轴停顿时长（对数），叠加普通 append 延迟的水平线作参照。
- **预期：** 停顿与大小近似线性，1 MB 约 0.1–0.2 ms，16 MB 约 2–3 ms，比普通 append（约 4 µs）高 1–3 个数量级。

### M2：append 尾延迟（半天）

- **方法：** 单线程，64 B 和 1 KB 两种 append，把对象从 0 增长到 1 MB，记录每一次 append 的延迟。
- **方案：** `Contig-2x`、`Contig-2x-pipe`、`Oracle`。
- **图 2a（时间线）：** 横轴 append 序号，纵轴延迟。预期 `Contig` 在 2 的幂边界出现尖峰，`Oracle` 为平线。这张图最直观，适合作为论文第一张 motivation 图。
- **图 2b（CDF）：** 延迟 CDF，标注 p50、p99、p99.9、max。
- **预期：** p50 三者接近，p99.9 和 max 相差 1–2 个数量级。

### M3：带宽受限下的吞吐（1–2 天）

- **方法：** 多线程闭环 append，每个线程追加自己的私有对象，对象长到上限后换一个新对象，持续 30 s。
- **变量：**
  - append 大小：64 B、256 B、1 KB、4 KB（线程数固定为 32）
  - 线程数：4、8、16、32、64（append 固定为 1 KB）
  - 对象上限：64 KB、1 MB、16 MB（1 KB append，32 线程）
- **方案：** `Contig-2x`、`Contig-2x-pipe`、`Oracle`。
- **指标：** 吞吐（Mops），`Oracle / Contig - 1` 即 resize 造成的吞吐损失；append p99、p99.9；网卡收发实际带宽（用 `ethtool -S` 或 `mlnx_perf` 读计数器），以及 "实际字节 / 逻辑字节"（写放大）。
- **图 3：** 横轴 append 大小或线程数，纵轴吞吐，并标出提升百分比。
- **预期（模拟值）：** 64 B 时约 +5%，256 B 时约 +20%，1 KB 时约 +40%，4 KB 时约 +65%；线程越多，差距越大。

### M4：链式读代价（半天）

- **方法：** 单线程读整个对象，对象大小 256 B–64 KB。
- **方案：** `Linked-256`（逐块追 `next`）vs `Oracle`（一次 READ）。
- **图 4：** 读延迟 vs 对象大小。预期 `Linked` 随块数线性增长（MiTDM 中已测得：1 跳 3.05 µs，20 跳 74.03 µs，可直接引用作为补充）。

---

## 5. 测量规范

- 每组：预热 5 s，测 30 s，重复 3 次，报告中位数和误差范围。
- 延迟：CN 本地 `rdtsc` 或 `clock_gettime(CLOCK_MONOTONIC)`，记入 HdrHistogram（精度 3 位）。
- 每个 append 都等完成事件再计时结束（signaled）；批量中只对最后一个 verb 发 signal。
- 记录并汇报：网卡型号、固件版本、MTU、驱动版本、CPU 型号。

---

## 6. 判定标准

| 标准 | 通过条件 | 不通过时 |
|---|---|---|
| T1 停顿 | M1 中 1 MB 对象的 resize 停顿 ≥ 25× 普通 append | resize 本身不够贵，需重新审视问题 |
| T2 尾延迟 | M2 中 `Contig-2x-pipe` 的 p99.9 ≥ 10× `Oracle` | 尾延迟叙事不成立 |
| T3 吞吐 | M3 中 1 KB append、32 线程时，`Oracle` 比 `Contig-2x-pipe` 高 ≥ 20% | 不宣称吞吐优势，只主打尾延迟 |
| T4 链式 | M4 中 16 KB 对象 `Linked-256` 读延迟 ≥ 5× `Oracle` | 链式 baseline 不够差，需改用更细的链 |

T1、T2 通过就足以支撑 motivation。T3 决定能否把吞吐写进卖点。

**注意：** `Oracle` 是上界，不是 extent 方案的实际结果。extent 方案的读路径和分配有额外开销，实际提升会小于 `Oracle` 与 `Contig` 的差距；这一点在后续完整实验（`growable-remote-object-quick-experiment.md` 的 Phase 1）中测量。

---

## 7. 排期（约 4–5 天）

| 天数 | 内容 |
|---|---|
| 1 | 环境配置、M0（perftest 和批量读）；更新模拟参数 |
| 2 | 实现 `Contig-2x`、`Oracle`、`Linked-256` 的 append/read 循环；M1、M4 |
| 3 | 实现 `Contig-2x-pipe`；M2 |
| 4–5 | M3 参数扫描、画图、对照 T1–T4 下结论 |
