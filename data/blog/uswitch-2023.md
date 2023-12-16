---
type: Blog
title: "[Paper Note] μSWITCH: Fast Kernel Context Isolation with Implicit Context Switches"
date: 2023-10-25 20:30:00 +0800
tags: ['paper', 'Notes', 'system', 'security']
summary: 2023 IEEE Symposium on Security and Privacy (S&P)
---

## Basic Information

- Title: μSwitch_Fast_Kernel_Context_Isolation_with_Implicit_Context_Switches
- Topic: Kernel isolation
- Venue: S&P 2023
- Author Information: 
	- Purdue 和 Intel 联合
	- Purdue 的 Pedro Fonseca 组[Reliable and Secure Systems Lab](https://www.cs.purdue.edu/homes/pfonseca/). 刚发了 OSDI 和 SOSP
- Link:https://congyu-liu.github.io/assets/pdf/uswitch_sp23.pdf

--- 
## Problem

- 上下文隔离问题
- 现有的隔离方式不是粒度太粗就是太慢
- 基于进程的隔离由于有两次下陷所以性能太差

## Motivation

- 内核可信域的抽象和隔离
- 基于进程隔离的代价主要在上下文切换上面，所以实现一个不怎么需要下陷的 Sandbox 和对应的上下文切换方法
- lwC 相关实现可以放过来用

## Significance/Contribution

To address the limitations of prior techniques, we propose μSWITCH, a novel approach to efficiently isolate selected kernel resources of a process and application memory into multiple fine-grained domains (μContexts). Isolating kernel resources with μSWITCH ensures security guarantees similar to those of process isolation but with a significantly lower performance overhead. In particular, μSWITCH achieves sandboxed function invocation in 129ns, which is 55× faster than process isolation.

## Method

### 设计

**核心思路：减少下陷次数**

![](/img/6be070b091048ec509ad70eef9ad16ce.png)
- 四个重要概念
	- ![](/img/b0a3ebb6de7cce5e4bcc1e2733f91b03.png)
- API
	- ![](/img/5407073815ef8963e6e603c80cc460b1.png)
	- 创新点：`privCall` 和 `sandboxCall`

### 系统抽象

- A μSWITCH-enabled program consists of privileged functions and sandboxed functions.
	- The program always begins with the privileged functions in the privileged context
	- 使用 `uContext` 定义这些 function 的上下文 The privileged functions dynamically create an isolated context (μContext) for each sandbox.
	- This divides the program into multiple isolated contexts, each of which has a set of kernel resources and a memory domain.
	- privileged 白盒，sandbox 黑盒
- 安全隐患：调用点
	- A trusted entrypoint marks the single entry gate into the privileged context. All entries into the privileged context including privcalls from the unprivileged contexts and returns from sandboxcalls are mediated through this trusted entrypoint.

### 资源隔离

- 使用 Intel MPK 进行内存保护和资源隔离
	- 在页表上有个 4-bit的标志位
	- `PKRU` 寄存器标志
- 安全问题：恶意系统调用，和不合时宜的二进制指令
	- 可信调用点的注册
	- ![](/img/269240256b174054ae1420f0adf7ced7.png)
	- 保证执行内存页不可写
	- 代码的二进制检查，扫描到不安全的指令之后直接改写指令或终止程序；
- 新的函数和寄存器栈
	- ![](/img/4f64ce2bbe0286bf407424e6b3510320.png)
	- 和 `$fs` 固定偏移量，禁用 WRFSBASE 防止 $fs 被篡改

### 上下文切换设计

- **核心思路：在切换后的第一次 syscall 时进行下陷和内核上下文的切换**
	- ![](/img/b8a3b5942f4a153b37190823264106e2.png)
- 用设计中的两个共享描述符来判断要不要进行内核上下文切换
	- ![](/img/033f7eea9b4fd87d05014a8030d17f2c.png)
- 在平均系统调用次数低的上下文切换的场景中可以有效降低切换时延
- ![](/img/376dc2550f728c9af17cd391b96e2561.png)

## Implementation

- 更改了 872 行 kernel 代码、 4493 行用户层链接库代码、 147行 Musl 代码和 227 行 RLBox 代码；
- 使用自旋锁和原子计数来进行多线程控制（还是老一套）
- syscall 劫持：在所有支持 µSWITCH 的进程源码中设置宏 `SYS_CALL_WORK_ENTER` 为 1，通过更改 `syscall_trace_enter()` 函数内容完成对系统调用的劫持。
- 跨上下文的文件调用：对于 privileged context 来说，其具有跨上下文进行文件获取的权限，因此暴露新的 API 接口 `dupFileDescriptor` 调用相关的 Intel MPK syscall。
- Libc 的隔离保护：为每一个沙箱复制一份 libc 资源，使用 dlmopen 加载到沙箱自己的 namespace 中
	- 使用 MUSL libc
	- 但是借用了glibc的内存函数实现

## Markable Results

- ![](/img/27eadefb23bb4afde661c64a30ffda87.png)
- ![](/img/33162930de9cf090f3f03c5efcd4b366.png)
- ![](/img/63d715cb522e404bae60be4b0b76cfec.png)
	- 区分 Per-Origin 隔离和 Per-Component 隔离。
-  ![](/img/33db0cb329f13c556234276fe4852b99.png)

### 原因分析

![](/img/89c0a0b575f793b555be97db21bcca54.png)


## Other Notes