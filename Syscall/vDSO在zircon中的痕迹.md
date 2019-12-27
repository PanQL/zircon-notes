>如果能够看懂C艹代码在说什么，请无视gn代码;千万别抱有先看懂编译过程再来理解c艹实现的想法。
>如果您的逻辑能力足够强大，请无视上一句话：）

这个文档记录了作者在查看userboot的启动时，发现的一些vDSO的痕迹，也许可以作为vDSO在zircon中实现的一部分说明。但为了完整记录过程，可能会比较冗长。

* vDSO首次亮相

首先，根据我们对LK_INIT_HOOK宏的理解，结合zircon在qemu中运行时的输出。可以定位：userboot的初始化函数（进程创建过程）在kernel/lib/userabi/userboot.cc中：

![图片](https://uploader.shimo.im/f/NHTp2BSrVE8dGdNx.png!thumbnail)
在浏览该文件的过程中，我们很自然地发现了下面这些东西，结合注释，userboot_image应该正是文档中提及的链接进来的vDSO之一——userboot代码。

![图片](https://uploader.shimo.im/f/tpAZy5PrOKcdVLZi.png!thumbnail)
如果采用了编译时生成的compile_commands.json,这个时候应该可以通过跳转，找到userboot-code.h的位置。（同时我们也察觉到了，rodso-asm.h中应该有相关信息）。将两个文件都找到并打开：

![图片](https://uploader.shimo.im/f/yJKgqKyyO0EMOcsM.png!thumbnail)
![图片](https://uploader.shimo.im/f/L9Vjwv0927cJZ6PJ.png!thumbnail)
我们发现，两个文件中正好体现了两个方面：要链接的userboot.so的信息、一个可能与链接相关的宏。（还差一点东西，让他们连贯起来）

但是我们发现的其实不止这两个文件，还有out/kernel-x64-clang/gen/kernel/lib/userabi这个目录，在这个目录下，我们还发现了userboot.image.S这个文件：

![图片](https://uploader.shimo.im/f/SBDnbfNVMfUaUhi3.png!thumbnail)
打开这个文件，可以看到其中正好包含了上面我们发现的两个可能相关的内容：

![图片](https://uploader.shimo.im/f/obPIxlYjnu0jZLN3.png!thumbnail)
那么，userboot.image.S这个文件是build过程中生成的，而这三行代码明显应该是code-gen的产物。我们尝试在除了out目录之外的部分搜索RODSO_IMAGE这个字符串，得到了一些比较好的结果：

![图片](https://uploader.shimo.im/f/KDAUMOtSM24OsLEJ.png!thumbnail)
打开kernel/lib/userabi目录的BUILD.gn文件，可以发现，"RODSO_IMAGE"出现在了一个名为rodso的模板中：）。可以发现rodso这个模板总共在该BUILD.gn中被调用三次，且名称恰好符合我们在zircon.bin.map中发现的三个名称。

![图片](https://uploader.shimo.im/f/6n7BlwD5nbgn3X1b.png!thumbnail)

[这里](https://github.com/PanQL/zircon/blob/master/kernel/lib/userabi/BUILD.gn)是作者尝试对rodso这个template进行的注释，希望有所帮助。（必有不足，请指出）

>如果您纠结于文中某些比较奇怪的目录，除了学习gn似乎没有别的解释办法。
