# 2019OSTrain第一组结题报告  

### 项目概述  

> 本组选题为“zircon内核的rust设计与实现”。Fuchsia是Google最新推出的微内核实时操作系统，zircon作为支撑整个Fuchsia运行的内核，具有相当的探究价值。同时，经过rCore项目的实践与验证，我们积累了一定的使用rust开发OS的经验，具有一定的信心用rust实现一个微内核。
>
> 所以本组的目标是：希望能够对zircon内核进行调研分析，理解zircon的设计思路和实现方法，用rust语言重新实现zircon内核（原本为C++实现）。

### 相关（已有）工作

#### 关于rust开发操作系统的已有工作：

* [rCore](https://github.com/rcore-os/rCore)
* [redox](https://www.redox-os.org/zh/)

#### 关于微内核的已有工作：  

* JOS

### 主要完成情况

经过半学期的分析与动手实践，主要完成了以下内容：

* 熟悉Fuchsia项目的结构和工具链，对Fuchsia内容进行拆分，分离出[zircon](https://github.com/PanQL/zircon)。

* 基于上一步得到的zircon代码，基于QEMU环境对zircon进行运行和代码分析，留下[分析过程](https://github.com/PanQL/zircon-notes)。

* 使用rust进行实践上的[尝试](https://github.com/PanQL/zircon-rs)

* 尝试往树莓派上移植zircon。

### 主要完成过程  

#### 从Fuchsia分离zircon  

本过程基于Google在2019年7月15日放出的Fucsia代码。借助国内镜像的支持，得到Fuchsia的源代码，并首先根据官方文档在qemu上运行Fuchsia。

能够成功运行Fuchsia之后，对Fuchsia的整个项目组织结构进行分析；首当其冲是整个项目的构建过程，为此我学习了gn和ninja的相关内容：

* [Gn文档](https://gn.googlesource.com/gn/+/master/docs/reference.md)
* [Ninja文档](https://ninja-build.org/manual.html)
* [如何更好地使用Gn](http://os.cs.tsinghua.edu.cn/oscourse/OsTrain2019/g1?action=AttachFile&do=view&target=Using+GN+build.pdf)

并阅读了Fuchsia项目中部分Gn代码。在对Fuchsia项目的构建过程具有一定了解之后，动手分离zircon部分的代码。

#### zircon代码分析  

在上一步的基础上，得到一份可以正常编译、在qemu上启动的zircon项目代码。对这份代码进行分析，主要分析过程如下：

* 分析zircon x86 for qemu启动过程
* 结合部分官方文档内容，分析zircon中KernelObject的设计与实现

主要遇到的困难是zircon代码具有相当的代码数量，并且在x86架构下的zircon启动过程很复杂，对分析造成了一定程度的阻碍。

分析文档可访问[github链接](https://github.com/PanQL/zircon-notes)查看

#### zcore的实践尝试  

以下以zcore来指代zircon的rust版本。

经过对zircon一定的分析，将zcore的实现确定为如下架构（从上到下依次为软件栈自底向上）：

* rCore中原有的部分代码：物理内存管理、线程调度器、页表基本操作。

* KernelObject的rust实现：调用底层接口，实现KernelOjbect

* syscall层：这层与zircon的vDSO兼容，主要负责基于已经实现的KernelObject提供系统调用服务

* userboot层：这层负责从内核态进行初始化，创建第一个用户程序并启动

[当前代码](https://github.com/PanQL/zircon-rs)

注：该部分代码不是由我单独完成，当前的主要代码为王润基完成，我仅提供了微小的帮助，并且目前在尽量follow他并在现有进度上继续开发。

#### zircon在树莓派上的移植

zircon当前官方支持多款使用了ARM cpu的板子，但缺少对树莓派的支持。为了更深入理解zircon的启动过程以及硬件平台相关代码的实现，我尝试在树莓派上运行zircon。

遇到的主要问题有：

* 缺少串口驱动
* 缺少对树莓派对应硬件情况的配置

我进行的工作主要包括：

* 基于网上已经有的[mini uart驱动](https://github.com/s-matyukevich/raspberry-pi-os/tree/master/src/lesson01),在树莓派上验证可用后，移植到zircon的Gn编译体系中
* 查阅树莓派的硬件手册，调整zircon的树莓派启动过程[硬件配置](https://github.com/PanQL/zircon/blob/rpi3/kernel/target/arm64/board/rpi3/boot-shim-config.h)

Port的结果代码已上传到[github](https://github.com/PanQL/zircon/tree/rpi3)

### 后续计划

* 进一步完善zircon的树莓派支持

  当前的树莓派移植显得较为简单。缺少对多核和用户态shell的完整支持，应当进一步打开多核，并尝试使用zircon管理树莓派上的丰富外设。

* zcore的后续开发

  基于已有的rust代码作进一步开发，补全所有的syscall支持和KernelOjbect实现。使其支持用户态的zircon程序。

* zircon的持续分析

  此前对zircon的分析仍有需要补充的地方，之后在zcore的开发过程中需要对zircon进行持续的分析。比如用户态驱动的实现。