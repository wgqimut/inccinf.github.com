---
layout: post
title: "进程的创建与可执行程序的加载"
description: ""
category: "Linux Class Experiments"
tags: [实验, linux]
---
{% include JB/setup %}

---

本文作为“linux操作系统分析 of USTC”实验二的报告    
姓名：王磊  
学号：SA12226224  

---

#实验总结
  本次实验的目的是分析fork和exec的执行过程以及进程地址空间和ELF可执行文件格式的表现形式。我们编程使用fork、exec等库函数时，其实是使用了glibc对于fork系统调用的封装（详见附录1），系统调用通过`int 0x80`汇编指令以及`eax`中相应的系统调用编号陷入内核，内核在进程上下文中代替用户进程工作。在最新版本的linux 3.9.2内核代码中，系统调用号和对应的处理过程通过`arch/x86/syscalls/syscall_32.tbl`文件进行关联，所以陷入内核后，内核会查询到相应的处理过程并执行。  
  对于fork系统调用，其对应的处理过程是`int 0x80->sys_fork()->do_fork()`。`do_fork()`首先对父进程的一些标志位进行检查，然后调用`copy_process()`函数为新建的子进程创建进程描述符，之后寻找一个可用的PID分配给子进程，最后将子进程放入就绪队列中（如果不需要阻塞），由进程调度器调度运行，父进程返回子进程的PID。在整个执行过程中`copy_process()`最为重要，它的主要工作就是为子进程产生一个与父进程几乎一致的进程描述符，为了避免大多数情况下复制的浪费（如fork后马上exec），所以linux使用了“copy-on-write”技术，初始时子进程与父进程只读共享进程描述符的大部分内容，仅在需要写入时父进程或子进程才生成副本。其余技术细节见附录2。  
  对于exec系统调用，其对应的处理函数是`sys_execve`，整个exec过程可以分为两个阶段：准备阶段和载入阶段。在准备阶段需要构建`linux_binprm`这种数据结构，该结构将可执行文件所需的一些信息组织到一起，之后需要在`linux_binfmt`链表上寻找匹配的可执行文件加载模块，该模块包含了载入程序所需的`load_binary`函数指针（如`load_elf_binary`）。到载入阶段后，繁杂的任务可以分为5个步骤：__检查ELF头->预分配地址空间->新进程空间代替旧进程空间->载入程序信息段并建立虚存映射->修改系统调用返回地址，使得exec返回到用户态时的EIP指向新载入的ELF入口地址。__完成这些步骤后，exec调用返回，开始全新的历程。其余技术细节见附录3。 
  ELF可执行文件载入后与进程的地址空间形成了某种对应关系，这种关系可以表示为下图：  
  ![lab2-1](https://raw.github.com/inccinf/inccinf.github.com/master/images/forkandexec1.jpg)





##附录 1 — 最新版本glibc-2.17对系统调用的封装  
glibc最新版本2.17对于系统调用封装文件的组织方式发生了一些变化，借助cscope的帮助终于找到了系统调用封装的一些线索。  
对于fork函数，其定义的位置是`glibc/nptl/sysdeps/unix/sysv/linux/fork.c`中，该文件大概130行处调用了`ARCH_FORK()`宏，该宏与体系结构相关，在i386上被展开为`INLINE_SYSCALL()`宏，该宏定义也与体系结构相关，对于i386，其定义在`glibc/sysdeps/unix/sysv/linux/i386/sysdep.h`中：  


	# define INTERNAL_SYSCALL(name, err, nr, args...) \
  	({									      	\
    	register unsigned int resultvar;		\
    	EXTRAVAR_##nr							\
    	asm volatile (							\
    	LOADARGS_##nr							\
    	"movl %1, %%eax\n\t"					\
    	"int $0x80\n\t"							\
    	RESTOREARGS_##nr						\
    	: "=a" (resultvar)						\
    	: "i" (__NR_##name) ASMFMT_##nr(args) : "memory", "cc");		      						\
    	(int) resultvar; })

到这里，就可以看到glibc对于fork的封装其实就是使用\__NR_fork对应的系统调用号使用int 0x80真正地进行系统调用完成的，当然还有其他的一些辅助操作。  
而对于execve函数，其定义位置为`glibc/sysdeps/unix/sysv/linux/execve.c`，其调用了宏`INLINE_SYSCALL (execve, 3, file, argv, envp)`，而该宏的定义与使用方法与fork等系统调用是一样的，所以可以看出glibc对于execve的封装形式就是通过\__NR_execve对应的系统调用号使用int 0x80软中断陷入内核完成真正的系统调用。

##附录2 — fork系统调用的执行过程分析  
当调用fork()函数时，glibc帮我们使用int 0x80进行系统调用，通过查询`/arch/x86/syscalls/syscall_32.tbl`表可以看到，fork系统调用对应的处理函数是`sys_fork`，所以内核会调用`sys_fork()`函数进行处理，该函数最终调用`do_fork()`函数，`do_fork`函数的定义在`linux/kernel/fork.c`大约1558行处。所以，分析do_fork成为了分析fork过程的重点。  
do_fork的分析：  
1、开始时，分配了一个指针p指向即将为子进程分配的进程描述符，还有一个long类型的nr，代表子进程的PID。


	long do_fork(unsigned long clone_flags,
		   unsigned long stack_start,
	   	   unsigned long stack_size,
	   	   int __user *parent_tidptr,
	   	   int __user *child_tidptr)
	{
		struct task_struct *p;
		int trace = 0;
		long nr;

  
2、进行一些标志位的检查，如果出错则返回。  


	if (clone_flags & (CLONE_NEWUSER | CLONE_NEWPID)) {
			if (clone_flags & (CLONE_THREAD|CLONE_PARENT))
				return -EINVAL;
		}

3、检测父进程的CLONE_UNTREACED标志位，看父进程是否要进行跟踪，如果需要跟踪，则进行一些设置。跟踪的常见例子是进程被debugger跟踪，除此之外一般情况下跟踪很少发生。  
4、创建子进程的进程描述符和其他数据结构，是进程创建的核心部分


	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace);

  
copy_process的主要工作就是创建子进程的进程描述符，由于是由fork产生，所以子进程的进程描述符绝大部分内容应该与父进程一样，而且由于LINUX使用了copy-on-write技术，此时并不需要将父进程的task_struct内容完全拷贝一份到子进程中，而只需要使用指针的方式让父子进行暂时共享大部分的内容，只有当后续父进程或者子进程需要对某处进行写入时，再产生真正的拷贝，这样可以省去绝大多数不必要的内存拷贝操作（因为大部分情况下fork完成后会马上调用exec）。  
应该注意到，由于在调用fork前，父进程的执行状态（寄存器状态、栈状态等）被保存到了它的task_struct中，所以在新task_struct构造后的子进程具有与父进程几号一致的状态，所以子进程在开始运行后从堆栈返回EIP的位置与父进程一样，这也就是fork后父子进程从同一位置开始进行运行的原因。

5、如果上述过程执行成功，则寻找一个可用的进程号填入子进程描述符，并赋值给nr，进行其他相关设置后，唤醒子进程将其放入就绪队列中，可以被调度运行。


		struct completion vfork;

		trace_sched_process_fork(current, p);

		nr = task_pid_vnr(p);

		if (clone_flags & CLONE_PARENT_SETTID)
			put_user(nr, parent_tidptr);

		if (clone_flags & CLONE_VFORK) {
			p->vfork_done = &vfork;
			init_completion(&vfork);
			get_task_struct(p);
		}

6、返回nr，即子进程的进程PID，这也就是fork函数执行后父进程得到的返回值。

##附录3 — exec系统调用的执行过程分析  
使用glibc库的exec簇函数与fork类似，使用int 0x80进行\__NR_exec系统调用，该系统调用的处理过程是`sys_execve`，该过程调用`do_execve()`函数对新可执行文件的载入进行处理，在`linux/fs/exec.c`的1582行左右，在3.9.2内核中，该函数最终调用`do_execve_common()`函数进行内存分配、数据载入、旧进程回收等工作。  
do_execve对可执行文件的载入分成两个阶段，第一个阶段是准备阶段，该阶段完成对进程参数的预读（读入内核空间）和对于可执行文件格式的判断，并为对应格式的可执行文件选取加载器（如ELF）。第二个阶段就是载入阶段，完成对新进程数据段、代码段、bss等等信息的载入。  
1、准备阶段  
1.1 初始化`linux_binprm`数据结构，该数据结构将可执行文件时所需的信息组织在一起。  
1.2 接下来准备将可执行文件加载到内核中，会调用`search_binary_handler`函数，寻找匹配的可执行文件加载模块，这些模块使用数据结构`linux_binfmt`表示并构成一个链表，linux_binfmt有三个函数指针load_binary、load_shlib以及core_dump，其中load_binary就是具体的装载程序，对于ELF格式而言，其装载函数是`load_elf_binary`，位于`linux/fs/binfmt_elf.c`中。  


	static struct linux_binfmt elf_format = {
		.module		= THIS_MODULE,
		.load_binary	= load_elf_binary,
		.load_shlib	= load_elf_library,
		.core_dump	= elf_core_dump,
		.min_coredump	= ELF_EXEC_PAGESIZE,
	};

2、载入阶段  
这里分析ELF格式的可执行文件载入过程  
2.1 从准备阶段binprm中读取elf文件头，对文件头进行一些检查。  
2.2 初始化bss段、代码段和数据段的起始地址和终止地址，相当于为新进程预分配了适合的内存空间。  
2.3 使用新进程地址空间代替旧进程地址空间，需要对旧进程进行空间的释放，由`flush_old_exec`函数完成。  
2.4 对程序信息段载入，还需要通过`elf_map`建立用户虚拟地址空间与目标文件的地址映射，在`elf_map`内部，调用`do_mmap`完成程序段到虚存空间的映射工作。  
2.5 修改系统调用的返回地址，使得系统调用返回到用户态时EIP指向新载入的ELF可执行文件的入口地址，这也是为什么调用exec函数，如果成功后不会得到返回的原因，子进程已经开始了新的旅程了！

##附录4 — 动态链接库与ELF文件格式以及进程地址空间关系  
ELF的动态连接库是内存位置无关的，ELF中是将解释器（动态连接器）与普通模块一样映射到进程地址空间，解释器的段类型为PT_INTERP，而后运行时控制权先交到解释器，由解释器加载动态链接库，然后才会回到用户程序运行。解释器会将每一个依赖的动态库都加载到内存，并形成一个链表，后面的符号解析过程主要就是在这个链表中搜索符号的定义。  
Global Offset Table（GOT）  
当在程序中引用某个共享库中的符号时，编译链接阶段并不知道这个符号的具体位置，只有等到动态链接器将所需要的共享库加载时进内存后，也就是在运行阶段，符号的地址才会最终确定。因此，需要有一个数据结构来保存符号的绝对地址，这就是GOT表的作用，GOT表中每项保存程序中引用其它符号的绝对地址。这样，程序就可以通过引用GOT表来获得某个符号的地址。  
Procedure Linkage Table（PLT）  
过程链接表（PLT）的作用就是将位置无关的函数调用转移到绝对地址。在编译链接时，链接器并不能控制执行从一个可执行文件或者共享文件中转移到另一个中（如前所说，这时候函数的地址还不能确定），因此，链接器将控制转移到PLT中的某一项。而PLT通过引用GOT表中的函数的绝对地址，来把控制转移到实际的函数。

在实际的可执行程序或者共享目标文件中，GOT表在名称为.got.plt的段中，PLT表在名称为.plt的段中。