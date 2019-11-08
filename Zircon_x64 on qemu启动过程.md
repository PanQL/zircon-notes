

> 本文档记录了采用multiboot协议的zircon镜像——multiboot.bin在qemu中的启动过程。
>> qemu中自带了支持multiboot协议的GRUB，能够支持启动multiboot.bin。
>> 下面的内容将与代码中的这些部分有较强关联：
>> ​		$zx/kernel/target/pc/multiboot/*
>> ​		$zx/kernel/arch/x86/{start.S,image.S}
>> ​		$zx/kernel/{kernel.ld,image.ld}
### 启动顺序

---


[这里](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)是multiboot协议的官方说明文档，在继续阅读之前，建议先对multiboot协议有一个简单的了解。

在zircon的启动中，multiboot.bin起到了一个类似`蹦床`的作用：

以我们开展实验的qemu环境为例，对于下面这个命令:

```
	qemu-system-x86_64 \
		-kernel out/multiboot.bin \
		-initrd out/legacy-image-x64.zbi \
		...
```
字面意思来看，这个命令会向qemu指定一个multiboot.bin作为kernel，legacy-image-x64.zbi作为初始的ramfs;然而实际上，真实的启动关系不应该直接按字面意思来理解。因为multiboot.bin的大小(在本文档写作时约为3KB)，实际与真实的zircon内核大小相去甚远，更不用说加上bootfs的大小了。

实际上，从qemu裸机到zircon内核的启动顺序，是下面这样的：

```
legacy-image-x64.zbi中的zircon内核
			 ^
			 |
			 |
			 |
multiboot.bin(由$zx/kernel/target/pc/multiboot中的代码编译得到，支持multiboot协议)
			 ^
			 | 
			 |
			 |
GRUB（qemu自带的bootloader，遵循multiboot协议）
```
因此我们在探究zircon内核镜像的构建/启动的过程中，需要同时了解两个部分，即multiboot目录和kernel目录。这两个部分在编写的时候几乎可以完全没有关联，但是在系统boot的过程中却有紧密的联系。



### multiboot.bin相关部分

---


从上述分析中，我们知道，按照一般顺序，multiboot.bin将是GRUB之后最早执行的代码。所以我们从这里开始。

首先需要确定的一件事情是，multiboot.bin由gn脚本中的哪一个target生成？按照gn脚本的编写习惯，我们在$zx/kernel/target/pc/multiboot/BUILD.gn中发现以下代码：

```
  executable("multiboot") {
    output_extension = "bin"
    output_dir = root_build_dir
    output_path = rebase_path("$output_dir/$target_name.$output_extension",
                              root_build_dir)
    metadata = {
      # For the //:images build_api_module().
      images = []
      foreach(name,
              [
                "multiboot-kernel",
                "qemu-kernel",
              ]) {
        images += [
          {
            label = get_label_info(":$target_name", "label_with_toolchain")
            name = name
            type = "kernel"
            path = output_path
            cpu = current_cpu
          },
        ]
      }
    }
    sources = [
      "multiboot-main.c",
      "multiboot-start.S",
      "paging.c",
      "trampoline.c",
      "util.c",
    ]
    deps = [
      "$zx/kernel/arch/x86/page_tables:headers",
      "$zx/kernel/platform/pc:headers",
      "$zx/system/ulib/libzbi",
    ]
    ldflags = [ "-Wl,-T," + rebase_path("multiboot.ld", root_build_dir) ]
    inputs = [
      "multiboot.ld",
    ]
  }
```
可以明显看出，multiboot.bin是根据multiboot.ld链接得到，并且依赖的源文件中存在名称中带start的汇编代码，可能是分析的突破口。首先查看汇编文件multiboot-start.S，发现确实存在符号_start以及我们关心的multiboot_header头部信息结构体。
![图片](https://uploader.shimo.im/f/4ELl9Fo2M4w6q5RM.png!thumbnail)
在_start中，对bss段进行清零，初始化栈，然后跳转到multiboot_main入口（同时传入用于核查的magic数及指向multiboot信息结构体的指针）。在multiboot_main中，主要完成了这些任务：

* 检查GRUB给入的magic数是否正确
* 解析GRUB给入的内存信息，在zbi文件中创建新的item来保存这些信息。
* 检查zbi的格式、需要的内存大小是否满足
* 查找kernel在zbi文件中的位置
* 建立简单的页表
* 跳转到kernel镜像

更详细的细节可以查看源代码。


### 进入内核

---
跳转到内核之后将执行$zx/kernel/arch/x86/start.S中的代码，入口为_start。

从进入_start开始，到跳转到C语言入口lk_main之前，start.S中主要依次完成以下这些任务：

* 设置临时栈
* 清空bss段（此处涉及KASLR的实现，清空之前进行了复制）
* 初始化boot时期的alloc机制
* 设置新的页表
* 跳转到fixups代码所在位置，修正KASLR带来的偏差
* 设置中断向量表
