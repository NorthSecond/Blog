---
type: Blog
title: "[Paper Note] Automated Mapping of Task-Based Programs onto Distributed and Heterogeneous Machines"
date: 2024-05-14 20:00:00 +0800
summary: "SC '23: Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis"
tags:
  - paper
  - Notes
  - system
  - HPC
---

## Basic Information

- Title: Automated Mapping of Task-Based Programs onto Distributed and Heterogeneous Machines
- Venue: SC 2023
- Author Information: Stanford legion team.
- Link: 
	- Paper Link: [Automated Mapping of Task-Based Programs onto Distributed and Heterogeneous Machines | Proceedings of the International Conference for High Performance Computing, Networking, Storage and Analysis (acm.org)](https://dl.acm.org/doi/abs/10.1145/3581784.3607079)
	- Code Link: 
		- 算法代码：[Thiago S F X Teixeira / Automap · GitLab](https://gitlab.com/thiagotei/automap)
		- 实验代码：[Thiago S F X Teixeira / automap-py-exps · GitLab](https://gitlab.com/thiagotei/automap-py-exps)

---

## Problem

异构分布式高性能计算系统的自动处理器任务映射和内存分配方案。

---

## Motivation

- 当前绝大多数映射算法都是贪心based的启发式算法
- 一些运行时系统提供了手动自定义映射器的接口，但是手动编写成本过高

---

## Significance/Contribution

提出一种基于离线搜索的自动化并行和映射方案。搜索算法是约束坐标下降（CCD）

- formalize the mapping problem for task-based programming models on machines where the mapping of tasks does not fully determine the mapping of data.
- describe *AutoMap*, a system that optimizes an application’s mapping by dynamically searching the space of possible mapping decisions
- design a search algorithm tailored to finding high-performance mappings for task-based programs called *constrained coordinatewise descent*
- implement and evaluate AutoMap

在论文中提到，Automap 并不是映射的最佳解决方案，只是自动化优化的一种可能方案。

---

## Method

### Machine Model

Machine $M$ : A graph
- Nodes: processors & memorys
	- processors: kind
	- memorys:  a kind and a capacity in bytes
- Edges: 
	- between a processor $p$ and a memory $m$: $m$ is addressable by $p$
	- between two memories: there is a communication channel between the two memories

Task 建模：
- 基于任务的编程模型在映射上具有共同点：程序表示并被翻译为在运行时执行的无环依赖图，
	- 节点是task，边表示task依赖关系。
	- Tasks are functions of named **data collections** that they may read, write, or both.
	- Data Collections 又是多维数组的某种变体。
- To run on a processor kind, a task must have a variant for that processor kind

![](/static/images/5bb1354807abb61805bb654552cbc176.png)

### AutoMap

![](/static/images/0882b8d714164c8ecca5245be06924e4.png)
- 主要组成部分： Mapper 和 Driver
	- Mapper 主要负责与运行时系统交互，获取性能perf
	- Driver 主要包含搜索算法实现和 pref 数据库。
- 使用 Legion
	- 如果使用其他的，要满足
		1. 任务参数为单个数据集的集合
		2. 可以显式建模任务依赖
	- 理论上讲 StarPU 和 PaRSEC 也可以做到，但是没有提供对应的实现

#### 任务建模
- 所有任务全部建模为组任务，其中单个任务被建模为大小为 1 的组
- 如果需求内存 > 物理内存剩余，允许映射失败
- 将映射器 $f(t, c)$ 扩展为一个记忆优先列表

如果强行搜索的话，搜索空间会过大，将搜索分为两步，降低搜索空间：A search over the kinds of processors/memories to use and runtime logic to select specific processors/memories of the appropriate kind.
- Specifically, the driver invokes a search algorithm to choose the kind of processor for each task, and the mapper distributes tasks in a deterministic, blocked fashion among processors of the selected kind.
- Similarly to mapping tasks, the driver invokes a search algorithm to choose the kind of memory for each collection, and the mapper instantiates each collection in the the memory of the desired kind that is closest to the selected processor. Finally, tasks in a group task are assigned the same mapping of processor kind and collection kinds.

AutoMap searches to find a mapping function with the signature $tasks \times collections \rightarrow bool \times processor kind \times memory kind$. Then $f (t, c) = (d, k_p, k_m)$ where the boolean $d$ indicates whether the group task is distributed or not, $k_p$ is the processor kind for $t$, and $k_m$ is the selected memory kind for $c$.
- 总的来说，AutoMap：
	- 输入：搜索空间，机器模型（包括所有目标应用程序的所有或者部分的任务和数据集）的包含文件
		- The file representing the search space is generated automatically by running and profiling the application once.
	- 离线搜索+自动调用应用程序进行潜在的性能评估
	- 当超过当前最佳时取消测试（时间限制）

### 搜索算法

创新点+1：we introduce constrained *coordinate-wise descent*, a novel algorithm tailored to finding solutions for the mapping problem.

![](/static/images/275e32379d421ab162d6e08e115b3116.png)

- CD 一次只改变一个映射决策
- 对于一组任务：
	- 首先贪心优化其节点分发，然后是处理器类型，然后是每一个集合的内存类型
	- 所以：CD的运行时间和任务集合的数量正相关
- 循环方法：从时间最长的任务到时间最短的任务，集合从任务最多到任务最少
	- 昂贵任务很少受到其他任务的影响
- 算法思路：首先对于数据移动提供高惩罚，然后又逐渐放宽，保证数据移动和计算任务的 trade off
- 在算法过程中维护任务的依赖图 $G$
	- 对于任务的集合，构建图 $C = (V, E)$,对于每一个集合$c \in V$，有 $(c_1, c_2)\in E$ iff $c_1 \cap c_2 \neq 0$  
		- The weight of the edge is $|c_1 \cap c_2|$.
	- ![](/static/images/f87b34ac4b023fada882679815de23b7.png)
	- The second constraint is the co-location constraint on overlapping collections to minimize data movement, described in Algorithm 2.
	- ![](/static/images/73ac461b979b2574d993363cb985c5e2.png)

- 在实现过程中 CCD 表现最佳，是因为：
	- 确保了对搜索空间中每一个可能维度的探索；
	- 优化了任务集合相同地址空间限制
---

## Implementation
- 
![](/static/images/e59100189dc65e2cda7f34767fd39ab6.png)
![](/static/images/f5767260807ab7d4fb102f6760a1e38e.png)
- 内存包含显存 Zero-Copy 还有CPU的内存
- 进行 5 次循环（trade-off）

---

## Markable Results

![](/static/images/1fef815870f7a70cae13683bfc1d5f7c.png)

![](/static/images/441675d44452fac0c1c2649ead50652d.png)

---

## Other Notes

- 感觉实现起来还是不太美，但是算法还是很符合直觉的
- 感觉还是得看看 HEFT、MCT和FCP相关的实现。