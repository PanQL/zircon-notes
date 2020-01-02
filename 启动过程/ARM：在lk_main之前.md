# 在lk_main之前的初始化

> 本篇文档记录了zircon在启动过程中从bootloader到主控函数lk_main的过程中所进行的初始化。该部分代码在x64架构和arm64架构上差异较大，当前这篇文档只阐述ARM部分。  

对于ARM64架构来说，种类繁多的ARM开发板在boot同一个内核的过程中，需要给出适合自身开发板的boot代码和驱动配置。在zircon中，这一部分工作主要由boot-shim部分的代码来承担。与x64架构下类似，在如GRUB这样的bootloader眼中，xx-boot-shim.bin是“内核”，包含zircon内核的image.zbi是“INITRD”。

ARM64架构下的boot分为以下步骤（代码）：

* 针对ARM开发板进行区分的xx-boot-shim阶段（//zircon/kernel/target/arm64)

* 由image.S和start.S及部分其他代码组成的初始化阶段

### xx-boot-shim阶段  

1. 代码结构

这一阶段的代码位于//zircon/kernel/target/arm64中，结构如下:

```shell
.
├── board
│   ├── as370
│   ├── c18
│   ├── cleo
│   ├── hikey960
│   ├── kirin970
│   ├── msm8998
│   ├── msm8x53-som
│   ├── mt8167s_ref
│   ├── qemu
│   ├── vim2
│   └── visalia
├── boot-shim
└── dtb
```

board目录下的各个子目录，为zircon支持的板子各自的不同代码。其中内容各有不同，但都以对外提供头文件的形式提供append_board_boot_item函数。这一函数主要通过硬编码的方式，设定板子需要的驱动信息和硬件信息。比如内存大小、中断控制器、串口等等。下面这个面向hikey960开发板的appedn_board_item函数可以作为例子：

```C++
// //zircon/kernel/target/arm64/hikey960/boot-shim.config.h
static void append_board_boot_item(zbi_header_t* bootdata) {
    // add CPU configuration
    append_boot_item(bootdata, ZBI_TYPE_CPU_CONFIG, 0, &cpu_config,
                     sizeof(zbi_cpu_config_t) + sizeof(zbi_cpu_cluster_t) * cpu_config.cluster_count);

   // add memory configuration
   append_boot_item(bootdata, ZBI_TYPE_MEM_CONFIG, 0, &mem_config,
                    sizeof(zbi_mem_range_t) * countof(mem_config));

   // add kernel drivers
   append_boot_item(bootdata, ZBI_TYPE_KERNEL_DRIVER, KDRV_PL011_UART, &uart_driver,
                    sizeof(uart_driver));
   append_boot_item(bootdata, ZBI_TYPE_KERNEL_DRIVER, KDRV_ARM_GIC_V2, &gicv2_driver,sizeof(gicv2_driver));
   append_boot_item(bootdata, ZBI_TYPE_KERNEL_DRIVER, KDRV_ARM_PSCI, &psci_driver,
                    sizeof(psci_driver));
   append_boot_item(bootdata, ZBI_TYPE_KERNEL_DRIVER, KDRV_ARM_GENERIC_TIMER, &timer_driver,sizeof(timer_driver));
   append_boot_item(bootdata, ZBI_TYPE_KERNEL_DRIVER, KDRV_HISILICON_POWER, &power_driver,sizeof(power_driver));

   // add platform ID
   append_boot_item(bootdata, ZBI_TYPE_PLATFORM_ID, 0, &platform_id, sizeof(platform_id));
}
```

2. 运行流程

真正的bootloader将指向Device Tree Block的指针传递给xx-boot-shim，并跳转到xx-boot-shim的第一条代码（boot-shim.S中的_start）。

在boot-shim.S中，主要进行了以下操作：

* 关闭Cache和MMU

* 设置初始栈

* 进入boot_shim函数，为zbi文件添加硬件相关信息，解析内核首地址

* 从boot_shim函数返回，跳转到内核第一条指令

boot_shim函数中主要完成以下操作：

* 首先尝试定位包含内核的zbi文件在内存中的位置（并借此定位kernel所在位置）：

  * 首先，内核所在的zbi文件作为INITRD，可能会作为DeviceTree中的一项，xx-boot-shim将会检查DeviceTree中是否有这一项的相关信息，如果有则认为已找到zbi文件。

  * 如果上一步失败，xx-boot-shim会认为zbi文件紧随在xx-boot-shim镜像之后。

* 调用对应开发板的append_board_item函数：向zbi文件中加入硬编码的硬件驱动信息

* 从解析好的DeviceTree信息中查找Memory相关的硬件信息和cmdline，补充到zbi文件中

* 返回kernel所在位置的首地址

boot_shim返回zbi文件首地址和内核首地址，之后将跳转到内核的第一条指令：

```Assembly
// x0: pointer to device tree
    bl      boot_shim

// x0: bootdata_t* to pass to kernel
// x1: kernel entry point
    br      x1
```

### 内核汇编阶段
