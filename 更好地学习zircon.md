这里是一些在学习zircon过程中注意到的点，**希望**能够帮助在学习过程中提高效率。

* 关于查看编译结果的**内存布局:**

一般来说，如果希望了解xx.bin或xx.so的内存布局，可以在相同目录下寻找xx.so.map这样的文件，名称相同说明是对应的。比如顶层目录下的multiboot.bin和multiboot.bin.map

* 关于内核镜像的存放位置（x64）：

未含有kaslr预处理的kernel镜像：

out/kernel-x64-clang/obj/kernel/zircon.bin

kernel/kernel.ld

含有kaslr fixup代码的kernel镜像：

out/kernel-x64-clang/obj/kernel/image

kernel/image.ld

* 在build过程中，会在结果生成目录得到一个compile_commands.json。该文件能够被vs-code、vim等编辑器或者clion等ide识别，请使用它来帮助阅读代码。（因为gn一个非常令人头疼的点就是：头文件包含关系在编译时才完全确定，信息隐藏在各种gn脚本中。）
