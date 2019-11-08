>这里记录 build 出来的好东西都被放在了什么鬼地方

out目录中的文件夹规律/规则是：按照toolchain来生成文件夹。out文件夹本身对应的是默认的toolchain：stub。然后out中的其他文件夹例如kernel-x64-clang，都代表着编译过程中用到的一种toolchain，在这些文件夹中都有一个toolchain.ninja。里面描述的各种rule就是gn定义的toolchain的内容，然后文件下部分就是调用这些rule来进行build，同时给出了依赖关系。

到ninja这一层就比make清晰了很多。

toolcahin本身是一个gn的关键字，用来定义工具链。zircon中，对这个内置的template进行了几层封装，首先是在public/gn/toolchain/toolchain.gni中进行封装，之后只能通过toolchain_with_tools这个template来创建新的toolchain。然后在public/gn/toolchain/c_toolchain.gni中对toolchain_with_tools也进行了一层封装，封装为c_toolchain。而c_toolchain在public/gn/toolchain/envirionmen.gni中被封装在了environment这个template中，用来帮助定义编译环境。

environment及其相关的template比如standard_environment，就是构建zircon过程中比较常用的template，用于定义一个编译环境


* Q：用户程序在哪儿？

./default.zircon/user-x64-clang/obj/system/uapp/

注意到找不到 echo,ls 等经典程序，怀疑是集成到了 sh 中？

* Rust build for fuchsia 缺失的库
  * Scrt1.o，libzircon.so，libc.so：
    * ./default/sdk/exported/zircon_sysroot/arch/x64/sysroot/lib/
    * ./default/x64-shared/sdk/exported/zircon_sysroot/arch/x64/sysroot/lib/
    * ./default/zircon_toolchain/obj/zircon/public/sysroot/sysroot/lib/
    * 以上三个文件夹好像是一样的？？
  * libfdio.so: 
    * ./default.zircon/user-x64-clang.shlib/obj/system/ulib/fdio/libfdio.so
  * libunwind: 
    * clang内置的
    * ../prebuilt/third_party/clang/linux-x64/lib/x86_64-unknown-fuchsia/c++/libunwind.so

将上述文件复制下来并链接，能够成功编译！

