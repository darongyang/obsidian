==kdump允许获取内核崩溃时的dump stack和故障现场，通常dump包括调用栈、寄存器和内存分析等信息，并且需要结合当前版本代码进行详细分析==

- 双内核：主内核、捕获内核（接管系统，轻量级，负责转储）
- 转储流程：
	- （1）主内核预留内存`crashkernel`；
	- （2）主内核崩溃通过`kexec`触发捕获内核；
	- （3）捕获内核收集内存信息，进行转储（`/proc/vmcore`文件， ELF format）;
	- （4）可将文件导出，使用GDB或Crash utility等进行分析。
- 崩溃处理/信息采集：
	- （1）保存cpu寄存器状态（函数`crash_setup_regs()`，保存通用寄存器`EAX`，`EBX`等、程序计数器`EIP`和栈指针`ESP`）
	- （2）生成转储文件元数据（函数`crash_save_vmcoreinfo()`，记录关键结构：内核符号表、内存布局、当前进程信息等）
	- （3）关闭非崩溃cpu（函数`machine_crash_shutdown()`和`crash_save_cpu()`），禁用中断（函数`disable_IO_APIC()`），避免造成干扰。
	- （4）启动捕获内核（函数`machine_kexec()`）
- vmcore的ELF文件格式
	- NOTE段、LOAD段
	- TODO
- 一些数据来源：Bugzilla，缺陷收集（提交、修复、关闭）数据库。

==通过理解对应版本的kernel代码，完成自动化dump信息的分析，给出初步故障诊断建议，简化故障定位流程==
- ==理解常规kdump中，支撑问题分析所需要的信息量。通常这个过程由具有大量经验的专家来完成==
- ==通过prompt engineering，使语言模型能理解故障场景和意图==
- ==通过RAG，使语言模型能够精准定位故障代码==



Q1：偏功能性，评估方法，测试指标？
Q2：

sys caller？
## Reference
- https://chat.deepseek.com/
- https://www.kernel.org/doc/Documentation/kdump/kdump.txt