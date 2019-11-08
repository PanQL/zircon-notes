>本文档将对zircon中syscall的生成作分析，帮助了解如何从syscalls.banjo到libzircon（vDSO）的编译过程。
## 原理阐述：
从之前的文档中，我们可以知道，vDSO被映射到需要系统调用的程序（如userboot）之后。简单来说，这样做可以让用户态的程序在进行系统调用的时候，不直接使用如int 80或syscall这样的指令，而是通过函数调用来实现系统调用的功能。同时，通过编译器的支持，让用户程序在编译的时候为这样的函数生成一个符号，比如我们之前曾在userboot.so中看到的：

![图片](https://uploader.shimo.im/f/rL4XcLpTkYECBO7B.png!thumbnail)
将zx_*这样的函数，定位到本程序所在内存区域之后一定偏移量的位置。两者相结合，可以让所有的系统调用都必须经过函数调用来达成目的。

这样的设计有一个非常灵活的点就是：**用户认识的系统调用和内核提供的系统调用可以有很大差别**。

在用户程序的角度来看，调用的某个zx_*函数就是一个系统调用，功能上与syscall指令完全无差别;

从内核实现的角度来看，内核开发者可以将一些功能简单、不需要内核做很多事情的系统调用（比如读取当前时间），直接在vDSO中完成;也可以将一些功能较为复杂的系统调用，尝试拆分成几个简单的系统调用，然后在vDSO中通过调用其他系统调用来完成操作。

这样，只有一部分用户以为的系统调用真正占据了内核中的一个系统调用号，而内核的系统实现可以减少复杂度。

## 具体实现：
在zircon中，syscall的相关代码主要有以下这些部分：

* $zx/kernel/syscalls：内核中的系统调用实现
* $zx/system/ulib/zircon：
  * 与内核中syscall不一一对应的zx_*函数，可能一个函数对应多次syscall指令，也可能无需进入内核。
  * 一些帮助实现vDSO的宏定义
* $zx/system/public/zircon/syscalls.banjo及abigen工具在编译时根据syscalls.banjo生成的代码：
  * syscalls.banjo：（设计并）描述了所有系统调用
  * 其他生成的代码：与$zx/system/ulib/zircon中的一些代码共同被编译为libzircon.so（即vDSO）

以上这些代码中，我们只关注会在用户态运行的、最终会进入到vDSO的代码，即第2、3部分。对于第一部分，读者可以自行定位内核中的中断入口，然后结合系统调用的定义规范进行阅读。（这部分代码的形式与常规的宏内核的系统调用实现大同小异）

# 构建过程
      由于zircon中各个组件的编译过程高度依赖gn脚本的配置，且在构建vDSO的过程中较多地使用了代码生成的技术，所以我们首先将gn脚本中如何编译得到libzircon.so（也就是vDSO）的过程进行说明。

      libzircon.so作为$zx/kernel/lib/userboot:rodso("vdso")的依赖，被library("userboot")的编译过程所调用，通过一个environment_redirect("userland-vdso")进行编译环境的重定向，将编译环境设定为user，然后进行依赖项的编译，即编译libzircon：

![图片](https://uploader.shimo.im/f/iAMC10pJO3oSyHSq.png!thumbnail)
在$zx/system/ulib/zircon/BUILD.gn中，根据上图在environment_redirect("userland-vdso")中设定的environment_label，我们可以找到目标library("zircon")：

![图片](https://uploader.shimo.im/f/bv8XTeU40UYIqmOl.png!thumbnail)
根据阅读gn脚本的习惯，查看一个target时我们首先查看其依赖项如何被满足。

在这里，":generate"代表的依赖项是一个名为"generate"的abigen模板，该模板的定义在$zx/kernel/syscalls/abigen.gni中，作者添加了一点注释，可以在github上查看。

该模板的主要作用可以概括为：调用abigen工具，根据gen列表中的信息，生成对应的文件。

![图片](https://uploader.shimo.im/f/ozcSwBM43I864npR.png!thumbnail)
从图中可以看出，生成的definition.*文件在out/user-x64-clang.shlib/gen/zircon/syscalls目录下，其余三个文件在out/user-x64-clang.shlib/gen/system/ulib/zircon/zircon下。

![图片](https://uploader.shimo.im/f/PPY5fYXY8O8HGLpb.png!thumbnail)
这里有*容易忽略但对后续理解非常重要*的一点，就是，这几个文件在指定生成的文件名称的时候，都在内置的$root_gen_dir或$target_gen_dir后面多加了一级/两级目录，**这是为了在其他代码中include这些文件时不至于引发混乱。**

生成的文件中，路径包含zircon/syscalls的两个definitions文件，将用于$zx/kernel/syscalls目录下的代码编译，这属于系统调用相关的第一部分内容。而路径包含zircon/的三个文件，将用于$zx/system/ulib/zircon目录下的代码编译。




---


在library("zircon")的第二个依赖":syscall-asm"中，我们可以看到：

![图片](https://uploader.shimo.im/f/qmdtE6caQ7wedAEi.png!thumbnail)
首先syscall-sam这个target存在依赖项$zx/ernel/syscalls:syscall-abi（":generate"在之前已经满足）。查看$zx/kernel/syscalls/BUILD.gn文件，可以发现，"syscall-abi"中又通过abigen生成了几个文件、并且将这些文件所在的地址作为include寻找的地址之一。

![图片](https://uploader.shimo.im/f/nHQfYXG4XKAGhkf4.png!thumbnail)
![图片](https://uploader.shimo.im/f/QEd4YvZtls8SC5Zr.png!thumbnail)
回到"syscall-asm"这个target所在处。config("config")指出，该依赖项在编译三个.S汇编文件的过程中，可以从$target_gen_dir中寻找缺失的头文件。source_set("syscall-asm")的sources中选定的三个汇编文件都在$zx/system/ulib/zircon目录下;**其中如#include <zircon/*>这样的语句包含进来的将是我们在":generate"中生成的文件。**

综合考虑将被编译的三个汇编文件及它们包含的文件，可以得到如下结论：

* zircon_syscall宏封装了x64中的syscall指令
* syscall_entry_begin和syscall_entry_end宏帮助生成具有别名的函数入口。如为名为zx_xxx的函数，生成别名：SYSCALL_xxx、VDSO_xxx、_zx_xxx这样的声明。
* m_syscall宏帮助协调ABI。
* 生成的zircon/syscalls-x86-64.S中调用了m_syscall来生成syscall入口，其中public参数用于表明该系统调用是否暴露给用户。如果暴露给用户则可以在用户态调用函数名称为VDSO_xxx、_zx_xxx、zx_xxx的函数来进行系统调用。

这一部分生成的zircon/syscalls-x86-64.S中，包含的所有系统调用与内核中需要实现的系统调用一一对应。每一个SYSCALL_xxx函数对应封装了一个syscall语句。

zx_futex_wake_handle_close_thread_exit-x86-64.S、zx_vmar_unmap_handle_close_thread_exit-x86-64.S则是在SYSCALL_xxx的基础上再进行了一层封装，然后提供给用户态进行使用。



---
在满足以上两个依赖之后，library("zircon")终于可以开始编译自己的sources中的文件，然后将所有依赖项编译得到的文件和自身得到的文件进行链接。

![图片](https://uploader.shimo.im/f/ZjNyGqsoBqU1uexb.png!thumbnail)
在这里编译的zx_xxx.cc文件，就是vDSO中对内核实现的系统调用进行进一步封装，得到的新的“系统调用”，在用户态可以通过函数访问该服务。例如zx_channel_call.cc的内容：

![图片](https://uploader.shimo.im/f/rktVVDMBjaY7AMd8.png!thumbnail)


## 总结
简单对构建过程进行总结，主要分为以下两个阶段：

1. 将syscalls.banjo编译为syscalls.abigen。这个过程作为template("abigen")的依赖最早进行。
2. 代码生成及编译
  1. 调用abigen工具将syscalls.abigen转化为各个不同的文件，也就是上文中的abigen(xxx)。
  2. 编译$zx/system/ulib/zircon中的代码，得到vDSO中的各个系统调用入口。也就是上文中的library("zircon")、source_set("syscall-asm")。

