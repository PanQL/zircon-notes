### build过程生成的out文件夹的内容

* out目录中的文件夹规律/规则是：按照toolchain来生成文件夹。

  * out文件夹本身对应的是默认的toolchain：stub。
  * out中的其他文件夹(例如kernel-x64-clang)，都代表着编译过程中用到的一种toolchain，在这些文件夹中都有一个toolchain.ninja。toolchain.ninja中描述了：
  
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
