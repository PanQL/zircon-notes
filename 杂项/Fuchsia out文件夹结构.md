### build过程生成的out文件夹的内容

* out目录中的文件夹规律/规则是：按照toolchain来生成文件夹。

  * out文件夹本身对应的是默认的toolchain：stub。
  * out中的其他文件夹(例如kernel-x64-clang)，都代表着编译过程中用到的一种toolchain[^1]，在这些文件夹中都有一个toolchain.ninja。toolchain.ninja中描述了：
  
    * 各种rule，即gn配置中定义的对应toolchain的内容。
    
    * 调用rule来进行build的gn语句（同时给出了需要的依赖关系）。

> 使用gn写的配置，翻译到ninja这一层之后，就比make脚本要清晰很多了。

* Rust build for fuchsia 缺失的库
  * Scrt1.o，libzircon.so，libc.so：
    * ./default/sdk/exported/zircon_sysroot/arch/x64/sysroot/lib/
    * ./default/x64-shared/sdk/exported/zircon_sysroot/arch/x64/sysroot/lib/
    * ./default/zircon_toolchain/obj/zircon/public/sysroot/sysroot/lib/
  * libfdio.so: 
    * ./default.zircon/user-x64-clang.shlib/obj/system/ulib/fdio/libfdio.so
  * libunwind: 
    * clang内置的
    * ../prebuilt/third_party/clang/linux-x64/lib/x86_64-unknown-fuchsia/c++/libunwind.so

将上述文件复制下来并链接，能够成功编译！

[^1]:toolcahin本身是一个gn的关键字，用来定义工具链。zircon中，对这个内置的template进行了几层封装，首先是在public/gn/toolchain/toolchain.gni中进行封装，之后只能通过toolchain_with_tools这个template来创建新的toolchain。然后在public/gn/toolchain/c_toolchain.gni中对toolchain_with_tools也进行了一层封装，封装为c_toolchain。而c_toolchain在public/gn/toolchain/envirionmen.gni中被封装在了environment这个template中，environment用来帮助定义编译环境。environment及其相关的template比如standard_environment，就是构建zircon过程中可以拿来直接使用的的template了，用于针对某一类软件的编译，定义一个编译环境。