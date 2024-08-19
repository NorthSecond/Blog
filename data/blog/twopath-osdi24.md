---
type: Blog
title: "[Paper Note]  A Tale of Two Paths: Toward a Hybrid Data Plane for Efficient Far-Memory Applications"
date: 2023-07-16 19:00:00 +0800
summary: OSDI 2024
tags:
  - paper
  - Notes
  - system
---
## Basic Information

- Title: A Tale of Two Paths: Toward a Hybrid Data Plane for Efficient Far-Memory Applications
- Venue: 18th USENIX Symposium on Operating Systems Design and Implementation (OSDI 2024)
- Author Information: ![](https://blog-img.yfyang.me/Picgo/osdi24-two-1.png)
- Link:[https://www.usenix.org/conference/osdi24/presentation/chen-lei](https://www.usenix.org/conference/osdi24/presentation/chen-lei)

---

## Problem

流内存分离系统场景下的 RMA 粒度控制

---

## Motivation

- 远程内存已经成为了大规模数据中心的一种解决方案，但是尽管RDMA等技术可以实现快速网络访问，但是每次远程访问还是至少要比本地访问慢一个数量级
    - 现有的解决方案主要包括使用传统页表无粒度访问远程内内存（直接换页）和以对象为粒度访问远程内存。
    - 但是这两种方案各有各的问题：
        - 直接换页对于局部性不强的程序来说会加剧IO放大问题
        - 在运行时定位获取对象粒度的内存资源代价很高，甚至于在严格的时间预算下连LRU都办不到
- 所以应该提出一种两者结合的方式来进行RMA
- 为了识别哪种需要换页，哪种需要对象，需要使用动态profiling的方式进行控制
- 但是存在三个实现上的挑战
    - 如何准确和连续地profile
    - 如何动态切换访问机制
    - 如何同步这两个访问路径所访问的数据

---

- 现实世界内存中有多样化的内存访问模式
    - 一个例子是 Map Reduce 其内存局部 性存在但是在 RMA 场景中并不占优势
    - 对象提取通过获取细粒度对象最小化 I/O 放大，但是优势不大
    - 甚至相同的任务对于不同的负载强度来说每对应的RMA访问特点都有不同

![](https://blog-img.yfyang.me/Picgo/20240819193343.png)

---

## Significance/Contribution

Atlas：一个基于动态profiling的同时使用换页和对象访问两种访问数据路径的混合数据访问平台
- 相比于SOTA 提高了吞吐量降低了尾延迟

---

## Method

> 与AIFM一样，Atlas要求程序使用智能指针（即实现屏障）并为对象声明解引用范围

- Atlas 将页面分割为 Card 实现实时的 Profilling
    - 构建一个 Card Access Table（CAT）（实质上是位图）。
        - 每个 Card 表示对应的 16 个连续字节
        - 对于相邻页面的 CATs，在内存空间中分配连续的位置——作用：可以将虚拟地址映射到 CAT 条目实现高效的按位操作
        - CAT 是一个表示该页自分配或上一次被换入以来内部每个 card 是否被访问过的 bitmap。
    - Atlas为每个页面维护一个1位 Path Selector Flag（PSF）。这个 flag 的取值为 runtime 或 paging，内核将该页换出时设置该 flag，用户态运行时库访问这个页内的对象时根据这个 flag 决定将整个页取回还是仅取回被访问的对象。
    
![](https://blog-img.yfyang.me/Picgo/20240819193702.png)
    
- 页面访问（换入）
    - 首先使用硬件内存事务（Intel TSX）来判断该对象地址是否存在映射
- 页换出
    - Atlas 仅仅以页为单位进行换出
    - 内核在换出一个页时，会根据该页的 CAT 来设置该页的 PSF，如果 CAT 中被设置的 bit 超过某个阈值（Atlas 设定为 80%），内核会将该页的 PSF 设置为 paging，否则设置为 runtime，同时内核会清空该页的 CAT。值得注意的是，在 Atlas 中以页为粒度进行换出不会带来显著的 I/O 放大问题，因为 Atlas 还会使用 object evacuation 技术在运行时提升应用访问对象的时间局部性

### 同步问题

![](https://blog-img.yfyang.me/Picgo/2024/08/064210540a8003ed61c88086cc524fe1.png)

- 使用类似于 C++ 智能指针的方式提供对应的 `UniquePtr` 和 `SharedPtr`
- 共享指针允许别名。 Atlas 将对象的第一个共享指针视为主要指针。 共享指针的布局类似于唯一指针，只是它有额外的 8 字节来链接所有指针

- 提出同步不变式来解决同步问题：
    - （1）防止对象同时从两个路径中检索（对象内 vs. 页内）
        - 限制任何时刻同一页面上的所有数据必须通过 PSF 指导进行访问路径
    - （2）防止包含刚刚运行时检索的对象的页面立即被交换出去（对象内 vs. 页外）
        - Atlas 强制执行这样一个规则：包含正在执行主动解引用范围的对象的页面不能被交换出去。
        - Atlas 通过维护每页的解除计数来实现这一点，当页面上的任何对象进入解除引用范围时递增该计数，在范围完成时递减。
    - （3）防止对象同时被运行时检索和被疏散者移动（对象内 vs. 疏散）。
        - Atlas使用页面的 dereference 引用计数在任务窃取线程和取消引用范围之间进行同步。非零取消引用计数会防止页面被窃取。
        - Atlas使用精细的解引用范围，每个范围与一个单一的智能指针解引用相关联。

### 内存管理

- 对象分配：日志结构的分配器维护线程本地分配缓冲区（TLAB），以减少并行对象分配期间的全局锁争用。TLAB以与页面对齐的日志段粒度进行管理，保证没有对象可以越过页面边界。Atlas将对象在TLAB上连续分配，因为先前的研究[65，70]表明：时间上接近分配的对象表现出类似的使用模式。这样一来，具有时间接近性的对象自然地分组到相同的日志段（页面），增强了局部性。
- 元数据分配：元数据被分配到一个专用的元数据空间中，连续16Byte字段
    - 每个卡表在分配日志段期间分配和初始化。它与日志段一起释放。卡表所需的空间是总内存的1/128。总之，空间开销小于2%。
- 垃圾回收：日志结构分配器通过基于复制的疏散器来支持碎片整理，这是现代 GC 中广泛使用的一种技术。Atlas扩展了疏散器，通过将热门对象分组到连续的日志段（页面）中来改善页面的时间局部性。疏散器与应用程序并行运行以减少碎片化。
    - Atlas 的分配器会**周期性地扫描对象**，将最近一个周期内被访问的对象移动到连续的页中。
    - 通过智能指针中的访问位跟踪对象自上次疏散以来是否已被访问
    - CAT 在页面淘汰时由内核读取和清除，而访问位在撤离时由运行时读取和清除。
- 计算下载
    - 为 Computing Offloading 在堆中保留了一个专门的地址空间
    - 注册为可远程化的对象都被分配到空间中
    - Atlas 要求用户保证可远程化的数据结构不能引用不可远程化的对象。这个属性在远程调用函数时确保地址一致性。

---

## Implementation

- 运行时库：7675行 C/C++代码
- 在 Linux kernel 5.14-rc5 中加入了对于相关页面管理的支持。
- 一个计算节点和一个内存节点
    - 每台服务器配备2个Intel Xeon Gold 6342 CPU（每个具有24个物理核心），256 GB内存和100 Gbps Mellanox ConnectX-5 RDMA 网卡。
    - Ubuntu 18.04
    - 禁用 Tubro Boost，CPU睿频和大页支持

![](https://blog-img.yfyang.me/Picgo/20240819193814.png)

---

## Markable Results

### 吞吐量

![](https://blog-img.yfyang.me/Picgo/2024/08/572158ea40b501560c469bad3585d8a6.png)

### 时延

![](https://blog-img.yfyang.me/Picgo/2024/08/b36d0e84d8e196199d153c0740d82864.png)

设置本地内存比例为 25%，使用 Memcached 应用，Meta CacheLib 作为工作负载时（Zipfian 分布），如上图所示，Atlas 可以达到比 Fastswap 和 AIFM 更低的时延和 90 分位尾时延。

---

## Other Notes