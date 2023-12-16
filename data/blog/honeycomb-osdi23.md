---
type: Blog
title: "[Paper Note] Honeycomb: Secure and Efficient GPU Executions via Static Validation"
date: 2023-11-23 16:00:00 +0800
summary: 17th USENIX Symposium on Operating Systems Design and Implementation(OSDI 2023)
tags: ['paper', 'Notes', 'system', 'HPC']
---

## Basic Information

- Title: Honeycomb: Secure and Efficient GPU Executions via Static Validation
- Venue: 17th USENIX Symposium on Operating Systems Design and Implementation(OSDI 2023)
- Author Information: 计算所 SKLP 牵头，清华等实验室参与
- Link: 
	- [Honeycomb: Secure and Efficient GPU Executions via Static Validation](https://www.usenix.org/conference/osdi23/presentation/mai)
	- [paper pdf](https://www.usenix.org/system/files/osdi23-mai.pdf)

---
	
## Problem

- Software-based Trusted Execution Environments (TEEs) on GPU Apps

---

## Motivation

- 此前没有纯软件实现的 GPU TEE
- 论文中有提到“三个挑战”：
    - 平衡验证器的功能性与复杂性（Z3有525K行的代码，可信基很大，一个简单的验证器可能有无法在加载时候验证）
    - Honeycomb must minimize the end-to-end TCB to provide high confidence in security.
    - must provide system-level support for secure and efficient IPC

---

## Significance/Contribution

1. 通过静态分析+结合CPU TEE构造了一个纯软件的 GPU TEE
2. 更加高效的实现，将运行时的检查 移到了加载时候，从而将开销从关键路径上移除了。
3. 安全不变式提供了一种高效以及安全的enclave间通信的方式。

---

## Method

### 威胁模型

- The adversary controls the entire software stack, including the compiler toolchains, the operating system, the hypervisor, and the device drivers. It also has physical access to the server hardware and may sniff the PCIe traffic.
- The host machine CPU features TEE capability, and the GPU features a hardware random number generator or performance counters to collect entropy for cryptographic uses.
- Users have the specifications of the server hardware and how it is connected.
- Honeycomb is able to establish a trusted MMIO path with the GPU.
- 不考虑side channel攻击

### 整体实现

![](https://zbtmdnr2iv.feishu.cn/space/api/box/stream/download/asynccode/?code=OTZiODA0OTYxMTg1ZjMxMWI4OGU0NzEzYjgzM2NjY2VfMnRTa09NWUxLQ1VnQm05azFLckxhUTgxQ1J1RHRjT2JfVG9rZW46QVJ0VWJIZXpwbzBRc1Z4M2E2N2NIc3htbnFkXzE3MDEwODM2Mzc6MTcwMTA4NzIzN19WNA)

- SEV-SNP的虚拟机的VMPL0中运行了Secure VM Service Module（SVSM），SVSM能够监控所有的应用和GPU之间的通信。应用和GPU之间需要分配GPU访问内存，这部内存只能够由SVSM访问。
- Honey将GPU隔离在一个单独的isolated sandox中，sandbox中运行了secure monitor， SM能够监控GPU和驱动之间的行为，能够保证GPU初始化的过程，以及对设备内存访问权限的控制。
- 当需要运行一个GPU kernel时候，应用会加载包含GPU kernel的二进制到 GPU 设备内存中，validator对kernel的二进制做检查，保证内存不越界。
    - 对于一些地址在静态编译的时候无法确定（依赖参数），validator根据预先定义的参数约束，生成boundary，在运行的时候进行检查。对于一些间接跳转，需要手动annotation，跳过检查。

### Validator

三个安全不变式：

- No dangling accesses.
- All memory accesses reside in their regions.
- Control flow integrity.（The execution must start at the designated entry point of the GPU kernel. The kernel can only transfer its control to the entry points of its basic blocks.）

##### Checking un initialized uses of values.

构建链路上的 SSA ，在此基础上用 CFG 做检查。

##### Checking memory accesses

通过符号执行对kernel 所有的内存访问都落在合法的内存区间中，算法采用了 scalar evolution analysis 和 polyhedral models

![](https://zbtmdnr2iv.feishu.cn/space/api/box/stream/download/asynccode/?code=YTdkZjRiYzEyMjZlZmY4OGI2NzEyZDc1NWRkY2JlNWZfaVFyUU5OZGEwT2YxM3FkWTlEUHdJejhCc09za2h3Sk5fVG9rZW46TkE0SmJreWJ0b0pXQVB4VGNzS2NpaEhYbndnXzE3MDEwODM2Mzc6MTcwMTA4NzIzN19WNA)

##### Enforcing control flow integrity

Ban掉or改写所有的间接跳转就可以了  

### Security Monitors

**SVSM 和 SM：**

- The SVSM regulates the interactions between the application and the GPU. 
- The security monitor (SM) in the sandbox VM regulates the interactions between the driver and the GPU.
1. initialization：因为没有对GPU 驱动初始化行为的规范，所以作者trace了自己机器上GPU的行为，并且让SM截获了所有MMIO的交易，GPU的行为和trace的行为一致。
2. Launching GPU kernels：提供了两条新的接口：`hipModuleLoadData` 将KERNEL拷贝到SVSM buffer中并且有validator检查；`hipLaunchKernel` 运行kernel，在运行前会对kernel实际的参数做检查
3. Isolating address spaces：GPU内的内存隔离是由RMP table保证的，SM会跟踪每个页的owner，截获对MMIO请求，并依据此对RMP的页表做更新。

### Secure and efficient IPC

- 在validation阶段，验证器划分出四种不同权限的内存（protected, read-only (RO), read-write (RW), and private, each of which has different access policies.）
- 基于不同权限的内存区间可以实现 enclave间的安全send 和 recv接口
    - 例如让send的数据处在protected 的内存区间，recv的数据处在RO内存区间，这样消息就能够通过明文传递了。

---

## Implementation

- About 32000 行 Rust
- **没明说开源**
- 实现基于 AMD 的 TEE，跨平台要重新实现

## Markable Results

作者在24-core 2.85 GHz AMD EPYC 7443 CPUs, 128 GB DDR4 memory, 480 GB SAMSUNG PM893 SSD的服务器上，以及AMD RX6900XT GPU，16 GB of device memory的GPU上做测试。

- TCB的比较：Honeycomb因为移除了Linux内核以及GPU的驱动，所以TCB较小
    - ![](https://zbtmdnr2iv.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjAzMTkzN2Q5MzY5OGU2OTJmZjBmZTM5ZjU0ZTI0OTFfY2tJOG96c0FrR1E4R013bUJaTG5EajFhcVFwUkR4M3RfVG9rZW46UTFMZWIybTJvb0k0NUh4dko2OGNjRTdPbnM5XzE3MDEwODM3MjA6MTcwMTA4NzMyMF9WNA)
- 时间开销：
    - ![](https://zbtmdnr2iv.feishu.cn/space/api/box/stream/download/asynccode/?code=MjZjNDgxZjE1ODc4MDcxNDgwODUzZGJhMmVkOWU2ZWFfQjN6b3c4RjN6UktNWmZGTGExTnV5V2Q4dXRJRng5Z09fVG9rZW46SGpnQ2J1c2F2bzBiYnh4UGx4RGNyVTBtbkhmXzE3MDEwODM3MjA6MTcwMTA4NzMyMF9WNA)
    - 相对执行时间在0.89和1.27之间
    - 有时间消融的结果
- 其他开销：
    - 验证GPU kernel的开销：最大的一个kernel在30ms
    - 加密传输数据的开销：明文10.6GB/s，密文2.20GB/s
    - IPC 开销：共享明文内存:233 GB/s，加密内存共享：411MB/s    
    - ![](https://zbtmdnr2iv.feishu.cn/space/api/box/stream/download/asynccode/?code=NGMxODU5ZDMyNzJmMzczYzQ4MjczMjU2YTdhN2M5ZjdfU1huQlMzMEVTYXVjb3U0T0VlbkNkeWdtRjRWRThSN0pfVG9rZW46V1U0dWJnUDdab3Jqbkp4VTFvVmM1YUQzbndnXzE3MDEwODM3MjA6MTcwMTA4NzMyMF9WNA)

## Other Notes

- 没开源很伤
- 这就是 OSDI 的工作量吗……
- 参考了 IPADS 对论文的解读内容。