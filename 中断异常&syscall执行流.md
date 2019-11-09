## 中断/异常
设置入口点：[zircon/kernel/arch/x86/idt.cc：idt_setup](https://github.com/PanQL/zircon/blob/master/kernel/arch/x86/idt.cc#L86)，设置中断向量表IDT，第i项的内容是__isr_table[i]，即符号_isr_{i}的地址

* 用户态触发异常或发生中断进入内核
* 入口点 zircon/kernel/arch/x86/exceptions.S: _isr_{i}
* _isr_{i}：保存错误码和中断号，跳转到interrupt_common
* interrupt_common：保存寄存器，跳转到x86_exception_handler
* zircon/kernel/arch/x86/faults.cc: x86_exception_handler
## syscall指令
设置入口点：zircon/kernel/arch/x86/mp.cc：x86_init_percpu，设置X86_MSR_IA32_LSTAR

* 用户态执行syscall指令进入内核
* 入口点 zircon/kernel/arch/x86/syscall.S: x86_syscall
  * 不保存寄存器！（也就意味着没有TrapFrame！因为没有fork，总会原路返回）
  * 跳转到 .Lcall_wrapper_table[syscall_id] 保存的函数地址
  * 不在范围内的直接跳转到 unknown_syscall
* .Lcall_wrapper_table被包装在start_syscall_dispatch宏中
* start_syscall_dispatch实际在以下文件中使用out/default.zircon/kernel-x64-clang/gen/kernel/syscalls/zircon/syscall-kernel-branches.S
* 每个syscall由syscall_dispatch宏声明，它的定义仍然在syscall.S中
* syscall_dispatch：
  * 使用{pre, post}_{nargs}_args宏完成syscall参数寄存器的移动。
  * 内部调用了wrapper_{syscall}函数，它定义在out/default.zircon/kernel-x64-clang/gen/kernel/syscalls/zircon/syscall-kernel-wrappers.inc中
* wrapper_{syscall}里面调用了do_syscall，它定义在zircon/kernel/syscalls/syscalls.cc中
* do_syscall
  * 入口出口处开关中断
  * 检查PC是否来自vDSO
  * 执行传入的make_call闭包，再次回到syscall-kernel-wrappers.inc中
* make_call
  * 检查用户传入的handle等参数
  * 最终调用 sys_* 函数完成不同的功能，它们定义在zircon/kernel/syscalls的各个文件中
