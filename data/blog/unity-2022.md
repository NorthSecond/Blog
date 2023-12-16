---
type: Blog
title: "[Paper Note] Unity: Accelerating DNN Training Through Joint Optimization of Algebraic Transformations and Parallelization"
date: 2023-10-30 16:00:00 +0800
tags: [paper, Notes, system, HPC]
summary: 16th USENIX Symposium on Operating Systems Design and Implementation(OSDI 2022)
---

## Basic Information

- Title: Unity: Accelerating DNN Training Through Joint Optimization of Algebraic Transformations and Parallelization
- Venue: OSDI 2022
- Author Information: 
	- ![](/img/3f214ddae419a271abeacabc2aae942b.png)
- Link:
	- Paper: [Unity: Accelerating DNN training through joint optimization of algebraic transformations and parallelization](https://www.usenix.org/conference/osdi22/presentation/unger)
	- Slides: [Unity: Accelerating DNN Training Through Joint Optimization of Algebraic Transformations and Parallelization](https://www.usenix.org/system/files/osdi22_slides_unger.pdf)

## Problem

大型神经网络的训练加速优化

## Motivation

**Algebraic Transformation 和 Parallelization 是两种常用的 DNN 训练优化方法**

- Aligebraic Transformation 主要介绍了 OP 的一些操作，包括 Operator Fusion、Operator Splitting 和 Operator Reordering 等
- 作者给 Parallelization 分了六类：
	1. *Data parallelism*： 数据并行，并行的最直观形式。在每个设备上保留整个DNN模型的副本，并为每个设备分配训练数据的子集。
	2. *Model parallelism*：将DNN模型划分为不相交的子模型，并在特定的设备上训练每个子模型。
	3. *Attribute parallelism*：对 tensor 的空间维度（如图像的高/宽）进行划分。可能需要设备间的通信。
	4. *Reduction parallelism*：利用 OP 的线性性质划分 tensor 。注意每个结果 tensor 的 shape 都和最终 tensor 相同，因此只需相加进行规约；
	5. *Pipeline Parallelism*： parallelize across different training iterations
	6. *Operator-specific parallelism*：OP 特定的并行化优化方式。
> Most parallelizations are not pure performance optimizations, but are instead trade-offs among different cost metrics.

- 但是这两种方式目前没有一个好的办法组合起来结合使用。串行使用会优化不完全or错误。

## Significance/Contribution

Present the first system that jointly optimizes algebraic transformations and parallelization in distributed DNN training.

## Method

### Parallel Computation Graph（PCG）

- Node: 
	- Changes in parallelization
	- tensor OP
- Edges:
	- Tensor Data Movement
	- Data Dependence

![](/img/641d61da63cf56ddd4c588c9e1c69880.png)

#### OPs is mapped to machine
- n 维数组来表示每个 OP 每份运算在哪个设备上运行，n 是并行维度的数目。例如，若 n=3，则将设备组织为三维阵列并序列化，以供该 OP 划分后任务到设备的映射。
- 维度也和设备的拓扑结构相关，可以手动显式地调整 Machine Mapping.

![](/img/7b3f6cf103ac7ce1b7a758c2faf2d468.png)

#### Parallelization Operators

考虑六类并行策略，两两分为三组

![](/img/d0feabadea7321b71181484aff9f1c83.png)

- `Partition` 和 `Combine`：改变 tensor 除 replica 以外维度的并行度；
- `Replication` 和 `Reduce`：改变 replica （复制份数）；
- `Pipeline` 和 `Batch`：`Pipeline`将 tensor 沿某个维度等大小地划分为多个子 batch，`Batch` （并未更改 tensor 的并行度，weight 需要和子 batch 相同 size）将多代的 tensors 聚合在一起。

### Graph Substitutions

As in TASO, Unity discovers substitutions in two steps:

![](/img/687e41482c20b0606da52bb07b38598b.png)

1. it uses a fast heuristic to identify candidate substitutions.
	- Unity 在固定图 size 内枚举所有可能的 PCG。对每个生成的 PCG，计算一个Fingerprint Hash，并拓展以包含每个维度的并行 degree。若一对 PCG 有相同的指纹，则是一个 candidate substitutions
2.  it uses a more expensive formal verification to ensure correctness.
	- 使用自动理论证明器 Z3
	- 只在新加入 OP 时运行即可，因此可离线

- Example：![](/img/e5bb67e89bc4f4d00e7dc11d6d156981.png)


### Joint Optimization

Unity uses a three-level hierarchical search algorithm. 将 PCG 输入划分为多个子图，为每个子图选择优化后的替代序列和设备映射，再将这些子方案组合为最终的输出。

![](/img/53bd918123019137ccf521b953eb95f6.png)

- PCG 替代序列：使用 TASO 基于 cost 的回溯搜索算法来计算 PCG 的替代序列，以最小化执行时间。
- PCG 图分裂及设备映射优化：基于 DP 的思想分别分裂序列图和并行图  
	- **序列图分裂**：在输入 PCG 中找最优的支配节点。
	- **并行图分裂**：将 PCG 划分为可并行计算的独立子图 ，包含串行（全部硬件资源）和并行（共享硬件资源）两类运行方案，Unity 选择执行时间更短的哪个。
		- 迭代遍历所有可能的资源量分配方案，忽略是仅 GPU 序号不同（而不是数量和位置不同）的方案，将指数搜索优化为二项式复杂度的搜索；

## Implementation

- Unity is implemented on top of FlexFlow, a distributed multi-GPU runtime for DNN training. 
- 代码开源但不独立（这篇文章主要贡献在 FlexFlow 上）
- 测试用例：
	- ![](/img/fe4fd016bbeefcfe62453ba2fc491b8f.png)

## Markable Results
- ![](/img/4ad4c777bce29c4d20a1cfc3136b8c02.png)

- 消融试验：
	- ![](/img/bb8076aea205a5c516d32b98c538a385.png)
	- ![](/img/3c11632b97871269bd00ca8881cc8479.png)

- 搜索算法用时：
	- ![](/img/cb2d3a62ad70b3b32beccfb0d2856af7.png)

## Other Notes

- 这边文章不太 well-written，有些东西没说明白感觉。