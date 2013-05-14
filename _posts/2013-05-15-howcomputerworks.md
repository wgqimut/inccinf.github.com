---
layout: post
title: "计算机是怎样工作的？"
description: ""
category: "Linux Class Experiments"
tags: [实验, C/C++, linux]
---
{% include JB/setup %}

---

本文作为“linux操作系统分析 of USTC”实验一的报告    
姓名：王磊  
学号：SA12226224  

---

###一、C程序编译过程及GCC的简单用法
当我们编译一段写好的C程序时，其实整个编译过程可以分成几个重要的步骤，相信学过编译方面知识的都很熟悉这几个步骤，它们大致是：  
>（源代码） &gt; 预处理器 &gt; （预处理后的CPP文件） &gt; 编译器 &gt; （汇编文件） &gt; 汇编器 &gt; （目标文件） &gt; 链接器 &gt; （二进制可执行文件）  

其中，  

 1. 预处理：包括了进行宏替换等步骤。   
 2. 编译：经过词法分析、语法分析、优化等步骤生成汇编代码。   
 3. 汇编：将汇编代码翻译为机器相关的二进制机器码，称为目标文件。   
 4. 链接：对各目标文件进行链接，生成最终的可执行文件（linux下为ELF），链接的目的是由于一个程序很可能由多个源代码组成，这些源代码编译成目标文件时，使用一些占位符来替代外部函数、数据成员的地址，在链接时，需要使用真正的代码和数据地址来替换这些占位符。 
 
使用GCC可以分别生成编译过程各个步骤产生的输出，有以下一些用法：

* _gcc -E -o \*.cpp \*.c_  该命令用于输出C源文件经过预处理器处理后的文件。
* _gcc -x cpp-output -S -o \*.s \*.cpp_  该命令是将预处理后的cpp文件编译为汇编代码。  
也可以使用 _gcc -S -o \*.s \*.c_ 命令直接从源文件生成汇编文件。
* _gcc -x assembler -c \*.s -o \*.o_ 该命令对汇编文件进行汇编，生成目标文件。  
也可以使用 _gcc -c \*.c -o \*.o_ 直接从源文件生成目标文件
* _gcc -o \* \*.o_ 该命令是链接器对目标文件进行链接，生成最终的可执行文件。   
当然也可以使用 _gcc -o \* \*.c_ 来直接从源文件生成可执行文件。

###二、实验源码及预处理
为了解释程序如何在单任务计算机上运行，本实验使用了一段简单的C代码进行分析，代码如下：

	int g(int x) {
   		return x + 3;
	}

	int f(int x) {
    	return g(x);
	}

	int main(void) {
    	return f(8) + 1;
	}
	
注意到代码里没有使用任何的宏定义以及include指示。使用GCC生成预处理后的cpp文件如下：

	# 1 "example.c"
	# 1 "<built-in>"
	# 1 "<command-line>"
	# 1 "example.c"
	int g(int x) {
    	return x + 3;
	}

	int f(int x) {
    	return g(x);
	}

	int main(void) {
    	return f(8) + 1;
	}
	
可以看到经过预处理器处理后的代码并没有发生太大变化，这是由于我们的代码没有使用任何预处理指示，也没有注释。而如果我们代码加入了预处理指示后，预处理器会执行那些指示，比如include等，并进行宏替换，之后这些预处理指令所在行会被替换为空行；而注释则会被空格代替。不过，可以看到其实预处理器在我们的代码前加入了4行“注释”一样的东西，它们被称为linemarkers，作用是指示文件名等后续可能需要的信息，更详细的解释可以看这里 <http://gcc.gnu.org/onlinedocs/cpp/Preprocessor-Output.html>

###三、编译为汇编代码
使用 _gcc -x cpp-output -S -o \*.s \*.cpp_ 编译出汇编代码如下：

		.file	"example.c"
		.text
		.globl	g
		.type	g, @function
	g:
	.LFB0:
		.cfi_startproc
		pushl	%ebp
		.cfi_def_cfa_offset 8
		.cfi_offset 5, -8
		movl	%esp, %ebp
		.cfi_def_cfa_register 5
		movl	8(%ebp), %eax
		addl	$3, %eax
		popl	%ebp
		.cfi_def_cfa 4, 4
		.cfi_restore 5
		ret
		.cfi_endproc
	.LFE0:
		.size	g, .-g
		.globl	f
		.type	f, @function
	f:
	.LFB1:
		.cfi_startproc
		pushl	%ebp
		.cfi_def_cfa_offset 8
		.cfi_offset 5, -8
		movl	%esp, %ebp
		.cfi_def_cfa_register 5
		subl	$4, %esp
		movl	8(%ebp), %eax
		movl	%eax, (%esp)
		call	g
		leave
		.cfi_restore 5
		.cfi_def_cfa 4, 4
		ret
		.cfi_endproc
	.LFE1:
		.size	f, .-f
		.globl	main
		.type	main, @function
	main:
	.LFB2:
		.cfi_startproc
		pushl	%ebp
		.cfi_def_cfa_offset 8
		.cfi_offset 5, -8
		movl	%esp, %ebp
		.cfi_def_cfa_register 5
		subl	$4, %esp
		movl	$8, (%esp)
		call	f
		addl	$1, %eax
		leave
		.cfi_restore 5
		.cfi_def_cfa 4, 4
		ret
		.cfi_endproc
	.LFE2:
		.size	main, .-main
		.ident	"GCC: (Ubuntu/Linaro 4.6.3-1ubuntu5) 4.6.3"
		.section	.note.GNU-stack,"",@progbits
		
看着有点不对劲？有点头大？其实那些“.”开头的语句并非真正的执行语句，而是汇编指示（Assembler Directive），就像C语言的预处理指示一样。上述汇编代码中，以“.cfi_”开头的汇编指示其实是用来支持Dwarf-2 CFI的，[Dwarf-2](http://en.wikipedia.org/wiki/DWARF)其实是一种特殊的数据格式，它用来方便人们进行调试，具体的含义可以参看这里<http://www.logix.cz/michal/devel/gas-cfi/>。  

这些汇编指示虽然并不影响我们进行程序分析，但是看起来还是挺烦的，这时候可以对gcc加上“-fno-asynchronous-unwind-tables“参数来编译出更简洁的汇编代码，使用命令 _gcc -S -fno-asynchronous-unwind-tables -o \*.s \*.c_ 编译出的汇编代码如下（已加注释）：

		.file	"example.c"	#开始一个逻辑文件
		.text					
		.globl	g		#g是一个全局符号，会被链接器用到
		.type	g, @function	#g的类型是一个函数
	g:							
		pushl	%ebp		#12.保持旧栈基址
		movl	%esp, %ebp	#13.开启新栈
		movl	8(%ebp), %eax	#14.将参数x的值放到eax中
		addl	$3, %eax	#15.eax的值加3（8+3=11）
		popl	%ebp		#16.栈基址指向旧栈底，那里存储的是函数的返回地址
		ret			#17.返回f，返回地址出栈到EIP，相当于mov %esp, %eip
		.size	g, .-g
		.globl	f
		.type	f, @function
	f:
		pushl	%ebp		#6.保存旧栈基址
		movl	%esp, %ebp	#7.开启新栈
		subl	$4, %esp	#8.栈顶向下4字节，为参数留出空间
		movl	8(%ebp), %eax	#9.将参数x的值放到eax中
		movl	%eax, (%esp)	#10.将eax的值存到栈顶作为参数
		call	g		#11.调用函数g，将EIP压栈，并修改EIP为g的起始地址
		leave			#18.相当于mov %ebp, %esp和pop %ebp
		ret			#19.返回main，返回地址出栈到EIP，相当于mov %esp, %eip
		.size	f, .-f
		.globl	main
		.type	main, @function
	main:				#0.程序从这里开始
		pushl	%ebp		#1.保存程序开始前的栈基址
		movl	%esp, %ebp	#2.开启一个新栈，新栈底在上一个栈顶之后
		subl	$4, %esp	#3.栈顶向下4字节，为函数参数留出空间
		movl	$8, (%esp)	#4.将参数8放入栈顶位置
		call	f		#5.调用函数f，将EIP压栈并修改EIP为f的起始地址
		addl	$1, %eax	#20.将eax中的值加1（11+1=12）
		leave			#21.相当于mov %ebp, %esp和pop %ebp
		ret			#22.返回到程序开始前地址，返回地址出栈到EIP，相当于mov %ebp, %eip
		.size	main, .-main
		.ident	"GCC: (Ubuntu/Linaro 4.6.3-1ubuntu5) 4.6.3"
		.section	.note.GNU-stack,"",@progbit
		
###四、程序运行过程分析
##1、理论分析
结合C代码和汇编代码来分析程序的运行过程，尤其是栈的变化情况，如图：(标号为上述汇编代码分析中的语句执行标号)  
![lab1-1]()  
##2、GDB跟踪
使用GDB跟踪运行程序，在进入函数的每条语句都插入断点，使用GDB指令查看寄存器状态、栈状态、汇编指令执行情况等。

	int g(int x) {
   		return x + 3;		// 断点3
	}

	int f(int x) {
    	return g(x);		// 断点2
	}

	int main(void) {
    	return f(8) + 1; 	// 断点1
	}
##2.1 程序在断点1停止时，查看其汇编代码执行情况：

	(gdb) disassemble
	Dump of assembler code for function main:
   		0x080483d2 <+0>:	push   %ebp
   		0x080483d3 <+1>:	mov    %esp,%ebp
   		0x080483d5 <+3>:	sub    $0x4,%esp
	=> 	0x080483d8 <+6>:	movl   $0x8,(%esp)
   		0x080483df <+13>:	call   0x80483bf <f>
   		0x080483e4 <+18>:	add    $0x1,%eax
   		0x080483e7 <+21>:	leave
   		0x080483e8 <+22>:	ret
	End of assembler dump.

可以看到此时main函数已经完成了保存旧栈基址和开启新栈，并为参数8留出了空间，但是还没有将参数8放入栈顶，我们查看一下寄存器和栈的状态：  

	寄存器：
	eax            0x1	1
	esp            0xbffff714	0xbffff714	//留出了4字节
	ebp            0xbffff718	0xbffff718	//main的栈基址

  	栈内存：
	(gdb) x 0xbffff714
	0xbffff714:	0x00000000	//目前还未存入参数8

##2.2 我们继续执行到断点2：


	(gdb) disassemble
	Dump of assembler code for function f:
   		0x080483bf <+0>:	push   %ebp
   		0x080483c0 <+1>:	mov    %esp,%ebp
   		0x080483c2 <+3>:	sub    $0x4,%esp
	=> 	0x080483c5 <+6>:	mov    0x8(%ebp),%eax
   		0x080483c8 <+9>:	mov    %eax,(%esp)
   		0x080483cb <+12>:	call   0x80483b4 <g>
   		0x080483d0 <+17>:	leave
   		0x080483d1 <+18>:	ret
	End of assembler dump.

当前的寄存器状态：

	eax            0x1	1
	esp            0xbffff708	0xbffff708	//留出4字节
	ebp            0xbffff70c	0xbffff70c	//f的栈基址
	
看一下main的栈都存了些什么：（main的栈基址为0xbffff718）

	(gdb) x/w 0xbffff714
	0xbffff714:	0x00000008	//参数8
	(gdb) x/w 0xbffff710
	0xbffff710:	0x080483e4	//返回main后的下一条语句地址，从断点1的汇编指令可以看到，就是add指令的地址

再看一下现在f的栈存了什么：（f的栈基址为0xbffff70c）

	(gdb) x/w 0xbffff70c
	0xbffff70c:	0xbffff718	//是main的栈基址

##2.3 继续运行，到断点3:

	(gdb) disassemble
	Dump of assembler code for function g:
   		0x080483b4 <+0>:	push   %ebp
   		0x080483b5 <+1>:	mov    %esp,%ebp
	=> 	0x080483b7 <+3>:	mov    0x8(%ebp),%eax
   		0x080483ba <+6>:	add    $0x3,%eax
   		0x080483bd <+9>:	pop    %ebp
   		0x080483be <+10>:	ret
	End of assembler dump.

寄存器状态：

	eax            0x8	8	//这是f的传参结果
	esp            0xbffff700	0xbffff700
	ebp            0xbffff700	0xbffff700	//g的栈基址

现在可以看看f栈的全部内容了：

	(gdb) x/w 0xbffff70c
	0xbffff70c:	0xbffff718	//f的栈基址
	(gdb) x/w 0xbffff708
	0xbffff708:	0x00000008	//参数值
	(gdb) x/w 0xbffff704
	0xbffff704:	0x080483d0	//返回f后的下一条语句地址，即leave语句

g的栈内容是：（栈基址0xbffff700）

	(gdb) x/w 0xbffff700
	0xbffff700:	0xbffff70c	//是f的栈基址

##2.4 现在执行到最后，看看最后的结果吧  
寄存器状态：

	eax            0xc	12		//最后结果
	esp            0xbffff714	0xbffff714
	ebp            0xbffff718	0xbffff718	//main的栈基址回来了

***使用GDB跟踪一遍程序后可以看出，理论分析结果和实际运行情况基本相符，栈及寄存器的变化是在理论预期中的，说明该程序确实是按照分析的那样进行工作。***

###五、结论
##1、单任务计算机的工作过程
其实，现在所有的计算机都是图灵机的实现，所以计算机的根本执行原理也就是图灵机的执行原理，也就是模拟图灵机卡带的进退并进行基本的运算，模拟卡带进退就是利用栈、跳转等形式来完成的。所以即使如今计算机已经发展到了很辉煌的程度，但是其根本和核心都没有变。  
在冯诺依曼体系结构计算机上，核心的思想是存储执行，而且代码和数据本质上没有任何区别，都是二进制的串。为了运行程序需要将代码和数据全部加载到计算机内存中，为代码和数据分配各自的地址，处理器不断地按照地址取指令、取操作数然后执行。而为了维护一些重要的状态，CPU内部有数个寄存器，EIP寄存器永远指向下一条指令的地址，EBP和ESP寄存器指向当前函数栈的基址和栈顶，而EAX/EBX/ECX等寄存器用来进行运算和返回值传递等操作。  
在函数调用时，***调用者***要完成两步操作，第一步要传递参数，方法是为参数在栈上分配一定的空间，然后将参数压到栈中；第二步是保持返回地址，即把call指令后第一条指令地址压入栈中。***被调用者***首先将调用者的栈基址ebp压栈，然后从栈中取出参数；返回时从栈中恢复调用者的ebp，然后从栈中取出返回地址，放入eip中继续执行，而函数返回值一般是通过eax寄存器来存储的。  
综上，单任务计算机的执行就是栈的增增减减，处理器永远按照程序的预定执行顺序一步一步执行，也就是卡带的来来回回。  
##2、多任务计算机的工作过程
计算机执行多任务是通过进程/线程切换来完成的，操作系统将CPU时间分块，为同时进行的多个任务分配时间块，在单核处理器上，同一时间块内依然只有一个任务在运行，它的运行方式与单任务的模式一致，但是当它的时间块用完后，操作系统会产生中断，切换到另外一个任务运行，整个系统就这样来来回回切换各个进程，由于操作系统分配的时间块非常短，人们是无法察觉的，所以看起来多个任务在同时执行。当然，在多核处理器上，每个核心都可以运行一个任务，所以这时候的确是有多个任务在真正并行地执行的，但是这时候执行的情况会稍微复杂一些，因为真正的并行会产生诸如同步与互斥、缓存一致性等问题。  
在任务切换时，需要保护现场和现场恢复的过程，操作系统为每个进程维护了一个进程控制块（PCB），PCB存储者该进程的堆栈、寄存器等信息，中断发生时，操作系统需要将当前的栈状态和寄存器状态存储，并载入要执行进程的栈和寄存器。  
多任务计算机从本质上来说依然是图灵机，只不过将一条纸带换成了多条纸带，每条纸带运行一会就切换，切换时需要保持纸带的状态（堆栈状态）和计算状态（寄存器状态）。