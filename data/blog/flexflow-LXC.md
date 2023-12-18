---
type: Blog
date: 2023-12-18 18:00:00 +0800
summary: 在有 NVIDIA 显卡的 LXC 虚拟机上配置并测试 FlexFLow 环境
title: 在 LXC 虚拟机上使用 CMake 的 Flexflow 源码编译与测试
tags:
  - HPC
  - codes
  - Notes
---

## 概述

- 虽然 FlexFlow 提供了较为完善的 Docker 接口，但是由于LXC 虚拟机环境的配置，实际上是没有办法直接跑Docker的
- 所以我们需要从源码开始对于 Flexflow 进行编译和测试。
- 以下流程在服务器的 LXC 虚拟机上进行测试，请根据实际情况进行调整。
- 主要参考资料是 [官方文档](https://flexflow.readthedocs.io/en/latest/installation.html)

## 准备工作

如果不对于代码和相关仓库文件进行修改的话，在代码拉取和 `config` 过程中都需要访问 Github，由于网络现实限制，请先做好网络相关准备。

选择一个合适的目录，然后将 FlexFlow 代码从 Github 仓库（https://github.com/flexflow/FlexFlow.git）中拉到本地。

```Bash
git clone --recursive https://github.com/flexflow/FlexFlow.git
cd Flexflow
```

> **注意：**由于 Flexflow 有一些依赖的开源项目的源代码（比如 Legion）是以 `git submodule` 的形式保存的，所以拉的时候要加上 `--recursive` ，如果忘了加的话要补充一句：
> 
> ```Bash
> git submodule update --init --resursive
> ```

如无特别说明，本文相关命令行操作均是在 `PWD=path-to-flexflow/FlexFLow/` 文件夹下进行的。

## 环境配置

### CUDA 相关

Flexflow 配置需要 CUDA 和 CUDNN 的支持。 LXC 虚拟机上默认带的版本和物理机版本其实不匹配，因此如果直接用的话会在编译的过程中报如下所示的错误：

![](/static/images/65cedaba5248129e7bf5b991d47299c8.png)

确认得到服务器物理机的 CUDA 版本为 12.0，那虚拟机上的 CUDA 版本就应该保持 12.0 不变。我在这里选择了 `cuda_12.0.0_525.60.13` 和 `cudnn-local-repo-ubuntu2004-8.8.0.121_1.0-1_amd64` 确认可行。

### CMake

编译 FLexflow 的过程中有个不小的坑是 CMake 的版本问题。如果使用 Ubuntu 20.04 的 apt 下载的最新版本的 CMake 是无法通过编译检查的，需要自己去 CMake 官网下载一个 >3.10 的 Cmake 安装到环境变量或者添加 CMake 的 `PPA` 源。

### Python dependencies

我是用 `Miniconda` 来进行 Python 依赖管理的（安装minicoda和配置教育网内镜像源等可参考 [tuna 镜像站相关说明](https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/)）

> 从官方文档来看的话直接裸机 `pip` 似乎也有 `requirements.txt` 支持，问题应该不大，但是我没试过。

在配置好 conda 后直接运行

```Bash
conda env create -f conda/environment.yml
conda activate flexflow
```

便可以创建一个名为 `flexflow` 的 conda 环境，包含了编译所需的各种包。

## 配置

环境依赖配置好之后，可以开始进行 Flexflow 安装前的配置准备工作了。

- 对于 Linux 而言，其配置文件主要在 `config/config.linux` 中，由于不涉及 HIP 和交叉编译这些问题，加上如果 CUDA 和 CUDNN 等相关环境变量配置完善的话，其实就不需要进行配置了。
- 我们有一个可能可以改的地方就是如果我们本地已经存在编译好的 Legion 的话，可以设置 `FF_USE_LEGION_PRECOMPILED_LIBRARY` 为 `ON` 重用编译好的 Legion，加快编译速度，否则 FlexFlow 会在编译过程中自动地从源码编译一份 legion.

在配置文件检查完成之后，我们创建一个 `build` 文件夹并进入，这就是 flexflow 编译产物存放的地方了。

```Bash
mkdir build
cd ./build
```

在 `build` 文件夹下，在检查网络配置可以访问 Github，`conda` 环境为 `flexflow` 后，便可以运行

```Bash
../config/config.linux
```

进行配置了。

## 编译&安装

### 编译

如果配置过程没有遇到什么问题的话，其实编译也就是一句命令的事情。

```Bash
make -j 14
```

由于 38 服务器的 CPU 不太好使，开满的话怕给 LXC 玩崩了，因此只开14个进程进行并行编译。编译时间比较长，可以考虑使用 `tmux` 或者 `screen` 来防止因为 SSH 断开而炸掉。

实测从 0 开始编译的情况下 `-j 14` 大约用时 40 min，take a coffee and have a break~

![](/static/images/647b2557b72390b586d8162cc65783bb.png)

### 安装（可选）

其实不常用的话也没太大必要安装，每次现场配就好了，如果要安装的话，在build文件夹下 `make install` 就行。

## 测试

### FlexFlow 运行时环境配置

- FlexFlow 运行需要一个环境变量 `FF_HOME` 指向对应的文件夹，可以使用
```Bash
export FF_HOME=/path-to/FlexFlow
```

### Python 接口测试

Flexflow 的 Python examples 在 [examples/python](https://github.com/flexflow/FlexFlow/tree/master/examples/python) 文件夹下，提供了`native`, `keras` and `pytorch` 相关的测试。当然，如果要运行 `keras` 和 `pytorch` 相关的测试的话，需要在环境中安装符合依赖版本的对应框架。

#### 解释器设置

测试 Python 接口有两种方式:

- 第一种是 FlexFlow 自己提供了一个编译好的 python解释器 `flexflow_python`，位置在 `./build` 文件夹下。
	- ![](/static/images/742d025984ab2d59bb4b5acc37ad0904.png)
- 第二种方式是使用原生的 python 解释器，但是要提前按照设置脚本进行设置。可以在 flexflow 文件夹下运行
```Bash
source ./build/set_python_envs.sh
```
- 我在这里使用第一种方式

#### 测试

由于这个conda环境中没有 `pytorch` 或者 `tensorflow.keras` ，我运行 `native` 的原生示例。

- 使用第一种方式的运行命令为：
```Bash
cd "$FF_HOME"
./build/flexflow_python examples/python/native/mnist_mlp.py -ll:py 1 -ll:gpu 1 -ll:fsize <size of gpu buffer> -ll:zsize <size of zero buffer>
```
- Legion runtime flags:（详见 [官方文档说明](https://flexflow.readthedocs.io/en/latest/train_overview.html)）   
	- -ll:gpu: number of GPU processors to use on each node (default: 0)
	- `-ll:fsize`: size of device memory on each GPU (in MB) 
	- `-ll:zsize`: size of zero-copy memory (pinned DRAM with direct GPU access) on each node (in MB). This is used for prefecthing training images from disk.   
	- -ll:cpu: number of data loading workers (default: 4)
    - -ll:util: number of utility threads to create per process (default: 1)
	- -ll:bgwork: number of background worker threads to create per process (default: 1)
	 - 根据当前显卡情况和现存相关的设置（GPU 0 上有人在跑一个大概 2G 显存的任务，所以我开在卡6上，开20G显存和20G内存），运行的指令为   
```Bash
export CUDA_VISIBLE_DEVICES=6 
cd "$FF_HOME"
./build/flexflow_python examples/python/native/mnist_mlp.py -ll:py 1 -ll:gpu 1 -ll:fsize 20000 -ll:zsize 20000
```
可以使用 export CUDA_VISIBLE_DEVICES= 来对于当前运行的 GPU 来进行指定	
- 测试可以成功运行，大概需要32s：
	- ![](/static/images/37b7a7c03c8323b822120623ab06c353.png)
	- ![](/static/images/45851ef1812fe48de16b126f55b74979.png)
- 同理，第二种方法只需要在执行配置脚本后把 `./build/flexflow_python` 改成 `python` 就可以了。
### C++ 接口测试

需要编译时在项目根目录的 `CMakeLists.txt` 中按需更改设置：

![](/static/images/851e8a38440af465c42063e2514dd917.png)

The C++ examples are in the [examples/cpp](https://github.com/flexflow/FlexFlow/tree/master/examples/cpp). For example, AlexNet can be run as:

```Bash
./alexnet -ll:gpu 1 -ll:fsize <size of gpu buffer> -ll:zsize <size of zero buffer>
```

这些参数的使用方法和上面 Python 的相同。

## 一些坑

- 在进行代码更改或者拉取之后增量编译只要 `make` 就好了，如果不幸重新 `config` 的话，那就删掉 `build` 文件夹从头来过吧；
- 这个编译反正不太稳定，出现莫名其妙的报错就从头来吧（）
- `-ll:fsize <size of gpu buffer>` 如果小了，模型跑不出来->段错误；如果大了，分配不了这么多->段错误。