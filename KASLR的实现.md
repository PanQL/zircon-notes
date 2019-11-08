>本文档关注zircon中如何实现KASLR。

KASLR(Kernel Address Space Layout Randomization)支持在boot的时候，将内核随机加载在内存中任何位置。zircon通过添加fixups代码的方法来支持KASLR。简单来说，就是让内核在启动之后，主动执行一些代码，修改自己的代码中带有相对地址偏移的代码，对随机位置加载带来的影响进行消除。

zircon中KASLR的实现分为以下几个部分：

1. 生成一个正常的kernel.elf镜像，其中大部分代码允许带相对地址偏移的指令。在该文件链接的过程中，加入"--emit-reloc"选项，使kernel.elf中附带有自身的相对地址偏移指令的相关信息（位置、类型等）。
2. 对这个kernel.elf文件进行分析，得到fixups代码，可用于在运行时修正相对地址偏移代码。
3. 将kernel.elf、fixups代码重新链接成image.elf文件。作为真正的内核镜像

下面是zircon中的相关实现所在的位置：

在$zx/kernel/BUILD.gn中，根据注释容易得到excutable("zircon")为正常的内核ELF镜像，excutable("image")为附加了fixups代码的内核镜像，而最终生成的kernel.zbi是zbi格式的含fixups的内核镜像。

excutable("zircon")这个target生成正常内核ELF镜像的过程与KASLR的实现没有联系，在这里不再赘述。在这里主要需要注意ldflags中加入了--emit-relocs。同时，生成的zircon.bin所在位置将会被写入到zircon_image.h中，用于之后image.S中的链接。（如下图）

![图片](https://uploader.shimo.im/f/H0g3prrLQQA8Buu3.png!thumbnail)
### 生成fixups代码
fixups代码的生成对应的target为$zx/kernel:fixups。

![图片](https://uploader.shimo.im/f/UwwAOM5HgWgBKTht.png!thumbnail)
从这里我们可以发现使用了gen-kaslr-fixups.sh脚本来处理已经编译好的zircon.elf（借助zircon.elf.rsp，zircon.elf.rsp中引用了zircon.elf的实际位置）。生成的fixups代码在out/kernel-x64-clang/gen/kernel/kernel-fixups.inc中：

![图片](https://uploader.shimo.im/f/INMEUX8wtPEN0w1E.png!thumbnail)
很明显地，这些都是对某个名为fixup的宏的调用。

### 构建image
构建image对应的target为excutable("image")：

![图片](https://uploader.shimo.im/f/PBmdM4gn1IkfsofV.png!thumbnail)
可以看到，这个target依赖先前生成的fixups代码和kernel_image.h头文件，编译了image.S汇编文件，链接文件为image.ld。首先查看$/zx/kernel/arch/x86/image.S：

![图片](https://uploader.shimo.im/f/ihjTjx5vDQYQ4c3h.png!thumbnail)
该汇编文件包含有较详细的注释，image.S主要涉及两个方面，包括构建kernel的zbi头、链入fixups代码。在这里我们仅关注“链入fixups代码”这方面，关于kernel的zbi头创建可以查看启动过程文档。

可以看到，该文件中引用了之前生成的kernel_image.h和kernel-fixups.inc。其中，kernel_image.h通过定义KERNEL_IMAGE来帮助指出要链入的zircon.bin内核镜像;kernel-fixups.inc中包含生成的fixups代码，其中调用的fixup宏在image.S中给出了定义。

### Kernel中的相关代码
通过链接fixups代码来实现KASLR，首先需要考虑的问题是，内核启动启动之后，在成功执行fixups之前，不应该执行到相对地址偏移的代码，否则很容易出错。zircon在编译时和运行时对内核的汇编代码做了几点调整：

* 首先是通过修改汇编文件中某些符号的定义，强制在该符号中生成位置无关代码，比如_start：
![图片](https://uploader.shimo.im/f/tVU3pM8SFhcoJxkq.png!thumbnail)
* 然后是对于bss段的占用问题，上图中体现了zircon的做法。kernel.ld在链接生成zircon.elf的过程中，只记录了bss段的大小，但实际上bss段在zircon.elf中是空的。而我们将zircon.elf链接入image中，在zircon.elf之后紧接着链接了fixups的代码，这会导致加载之后fixups的代码占据了本来应该是bss段的位置。zircon在进入内核之后，手动将这部分代码复制到bss段之后，然后才进行bss段的清零。在运行时化解这个冲突。
* fixups代码在start.S的high_entry中被调用，在这之后存放fixups代码的内存可用于内存分配：
![图片](https://uploader.shimo.im/f/6mAo5snsgTkD25nM.png!thumbnail)


