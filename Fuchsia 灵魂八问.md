1. Fuchsia 用户程序的运行过程？

    musl for fuchsia: [https://github.com/PanQL/zircon/tree/master/third_party/ulib/musl](https://github.com/PanQL/zircon/tree/master/third_party/ulib/musl)

	1. 入口点在哪里？内核怎样传递参数（args，envp，vdso）？

	2. 在 main 前后，libc 进行哪些初始化/收尾工作？

	3. 🚧vdso 数据布局什么样？如何进行 syscall？

2. 用户程序能看到哪些内核对象？

	1. 内存：section，VMAR，VMO 的关系？

	2. 进程：Job，Process，Thread 的关系？

	3. IPC：Socket，Channel，FIFO 都是啥？

	4. 同步：Futex，和 Linux 是否一样？

	5. 信号：Signal，Port？

	6. 权限：Capability？

最好能配合实际程序解释。

3. Zircon 如何启动第一个用户程序？

	1. ✅userboot 保存在哪里？

        源码位于kernel/lib/userabi/userboot

	    编译时内嵌在内核中，运行时无需解析 ELF 头

	2. userboot 的内存布局是怎样的（链接脚本）？

	    建立了哪些 VMAR，VMO？

	3. Bootstrap channel 如何建立？传递了哪些信息？

	4. 如何进入用户态？

4. 用户程序如何启动其它程序？

	1. 如何创建新进程并获得 channel？

	2. channel 的工作原理是什么？

	3. 通过 channel 传给新进程哪些东西？

	4. 新进程启动后如何处理 channel 传来的对象？

5. 从 userboot 到 sh，中间经历了哪些过程？

	1. bootfs 是什么？存在哪儿？里面有什么？

	2. 🚧userboot 做了什么事情？

        [https://github.com/PanQL/zircon/blob/master/kernel/lib/userabi/userboot/start.cc](https://github.com/PanQL/zircon/blob/master/kernel/lib/userabi/userboot/start.cc)

	3. 🚧启动了哪些服务进程？它们都干啥的？

	    devmgr, devhost, svchost, fshost, appmgr, sysmgr？

        userboot 根据 kernel 传来的 cmdline 中的 `userboot.root` 参数，决定下一步启动哪个进程。它在BUILD.gn中被设置为`bootsvc`。

	4. 这些服务进程用到了哪些 syscall？

6. 驱动程序是如何工作的？

	1. 谁来完成设备探测和发现？

	2. 需要用到哪些内核对象？

	3. 内核如何保证请求映射的物理地址合法？

	4. 如何向内核注册 IRQ？内核如何处理 IRQ？

7. 用户程序如何访问文件？

	1. 都经过哪些服务进程？POSIX API -> VFS -> ?FS -> BlockDriver？

	2. 各个进程之间的通信接口是什么？

	3. 内核如何高效调度这些进程？

8. ZirCore 的设计方案？

	1. rCore 的哪些模块可以重用？

	2. 需要补充实现哪些内核对象？

	3. milestone？时间表？这学期能不能造出来！？

x. Zircon中多核是如何协同工作的？boot时的情况？ipi？

