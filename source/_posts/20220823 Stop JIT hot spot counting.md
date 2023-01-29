---
title: Stop JIT Hot Spot Counting
date: 2022-10-22 22:51:17
mathjax: true
tags:
- PHP-SRC
- 性能优化
- JIT
categories:
- 性能优化
---



### Introduction

#### PHP Tracing  JIT

PHP 的tracing JIT是PHP中加速效果比较好的，JIT开启的默认配置就是tracing模式。tracing模式会跟踪经常执行的loop或者function，然后将执行路径（trace）编译成机器码，以后再执行此段代码的时候会直接执行机器码，这样更快。

> A trace is a linear sequence of instructions with a single entry point and one or more exit points. A trace does not contain any inner control-flow join points execution either continues on the trace or it exits the trace.  

其中进行JIT编译的主要的代码类型有loop，function，return，side_exit. 相应的用户可调整的参数是opcache.jit_hot_loop， opcache.jit_hot_func， opcache.jit_hot_return，opcache.jit_hot_side_exit，这些参数的初始值及初始值见官方文档 [opcache configuration](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit-hot-func) 。比如hot_func，当PHP解释执行n次之后 达到了opcache.jit_hot_func 设置的阈值，则触发JIT编译。

本文关心的则是PHP怎么实现的热点代码的计数，怎么判断达到了阈值，以及我做的一点点停掉热点计数的优化。



### Original Code

#### call stack

先简单写个函数调用的PHP脚本add.php，然后去GDB调试，`gdb ./sapi/cli/php`,  `run -c php.ini add.php` 先看看这个call stack

```
bt
#0  zend_jit_setup_hot_trace_counters (op_array=0x7fffb4cdb4e0) at ext/opcache/jit/zend_jit_trace.c:8069
#1  0x00007ffff54d1c96 in zend_jit_op_array (op_array=0x7fffb4cdb4e0, script=0x7fffb4cdb280)
    at /home/wxue/wxue/php-src/ext/opcache/jit/zend_jit.c:4512
#2  0x00007ffff53bec59 in zend_persist_op_array (zv=0x7fffb4cdb4c0) at /home/wxue/wxue/php-src/ext/opcache/zend_persist.c:700
#3  0x00007ffff53c3ac0 in zend_accel_script_persist (script=0x7fffb4cdb280, for_shm=0x1)
    at /home/wxue/wxue/php-src/ext/opcache/zend_persist.c:1320
#4  0x00007ffff53acc50 in cache_script_in_shared_memory (new_persistent_script=0x7ffff5879000,
    key=0x7ffff55100b8 <accel_globals+408>, from_shared_memory=0x7fffffffa9a0)
    at /home/wxue/wxue/php-src/ext/opcache/ZendAccelerator.c:1600
#5  0x00007ffff53aea8f in persistent_compile_file (file_handle=0x7fffffffd000, type=0x8)
    at /home/wxue/wxue/php-src/ext/opcache/ZendAccelerator.c:2169
#6  0x0000555555c21c7c in zend_execute_scripts (type=0x8, retval=0x0, file_count=0x3) at /home/wxue/wxue/php-src/Zend/zend.c:1754
#7  0x0000555555b7ec84 in php_execute_script (primary_file=0x7fffffffd000) at /home/wxue/wxue/php-src/main/main.c:2538
#8  0x0000555555d97b5d in do_cli (argc=0x4, argv=0x555556b8dc10) at /home/wxue/wxue/php-src/sapi/cli/php_cli.c:965
#9  0x0000555555d98c65 in main (argc=0x4, argv=0x555556b8dc10) at /home/wxue/wxue/php-src/sapi/cli/php_cli.c:1367
```

以#4 cache_script_in_shared_memory 为界，前面的函数主要是去将PHP源码解析抽象语法树，最后创建一个包含 op_array （一条条的opline）的script结构 （相关的PHP虚拟机的博客可以参考这个 [PHP7 Virtua Machine](https://www.npopov.com/2017/04/14/PHP-7-Virtual-machine.html)。每次遇到一个.php 的文件就会走一遍这个过程。

cache_script_in_shared_memory  做的事情呢主要是将编译出来的op_array这些缓存到opcache里去，这是个共享内存，在PHP的worker之间共享。

后续的函数呢就是去加速脚本执行，JIT也是在做加速，相关的热点计数的逻辑在zend_jit_setup_hot_trace_counters里。其中zend_jit_setup_hot_trace_counters就是对op_array里的那些loop，func等起始位置设置上计数功能。



#### zend_jit_setup_hot_trace_counters()

```
	if (JIT_G(hot_loop)) {
		zend_cfg cfg;
		ZEND_ASSERT(zend_jit_loop_trace_counter_handler != NULL);
		
		// 构建控制流图，分析基本块(basic block 指令顺序执行的一个整块)，因为loop的识别是基于基本块识别的
		if (zend_jit_build_cfg(op_array, &cfg) != SUCCESS) {
			return FAILURE;
		}
		
		// 对所有的基本块遍历，识别出loop_header (ZEND_BB_LOOP_HEADER)
		for (i = 0; i < cfg.blocks_count; i++) {
			if (cfg.blocks[i].flags & ZEND_BB_REACHABLE) {
				if (cfg.blocks[i].flags & ZEND_BB_LOOP_HEADER) {
					/* loop header */
					opline = op_array->opcodes + cfg.blocks[i].start;
					if (!(ZEND_OP_TRACE_INFO(opline, jit_extension->offset)->trace_flags & ZEND_JIT_TRACE_UNSUPPORTED)) {
						// 关键一步，修改opline的handler，原来 的opline->handler 是 (const void *) 0x555555cd4d39 <execute_ex+7817> ,
						// 被替换为 (const void *) 0x7ffff3edc6b0 <JIT$$hybrid_func_trace_counter> 这个函数就是对counter进行操作的逻辑
						opline->handler = (const void*)zend_jit_loop_trace_counter_handler;
						
						if (!ZEND_OP_TRACE_INFO(opline, jit_extension->offset)->counter) {
							// 这个counter其实是保存的地址，zend_jit_hot_counters数组的第ZEND_JIT_COUNTER_NUM个位置
							// 整个zend_jit_hot_counters数组size是 ZEND_HOT_COUNTERS_COUNT=128, 
							// 也就是说所有的loop 和 func的计数器都保存在这个数组里，
							// 那么这也限制了在整个.php脚本里最多支持对128个loop func 进行计数，
							// 检测是否达到阈值成为热点代码.
							ZEND_OP_TRACE_INFO(opline, jit_extension->offset)->counter =
								&zend_jit_hot_counters[ZEND_JIT_COUNTER_NUM];
							ZEND_JIT_COUNTER_NUM = (ZEND_JIT_COUNTER_NUM + 1) % ZEND_HOT_COUNTERS_COUNT;
						}
						// 设置trace_flag，表示这条handler是tracing JIT loop 起始的位置。
						ZEND_OP_TRACE_INFO(opline, jit_extension->offset)->trace_flags |=
							ZEND_JIT_TRACE_START_LOOP;
					}
				}
			}
		}
	}
	// if (JIT_G(hot_func)) 类似
```

后续当再次执行到设置了计数handler（zend_jit_loop_trace_counter_handler）的opline的时候，就会触发对counter的计算来判断是否满足hot。而这个handler在不同的zend_jit_vm_kind模式下对应的函数不太一样。dasm_labels 里的最终会由 DynASM 去生成汇编码执行，比如对func的计数handler会被替换成JIT$$hybrid_func_trace_counter，这个handler因为被经常执行，已经被当作桩函数给提前JIT生成了汇编码。不过它的逻辑和下面的zend_jit_func_trace_helper是一样的，我们可以之间看zend_jit_func_trace_helper的C代码，更方便一些。

```
static void zend_jit_init_handlers(void)
{
	if (zend_jit_vm_kind == ZEND_VM_KIND_HYBRID) {
		// ...
		zend_jit_func_trace_counter_handler = dasm_labels[zend_lbhybrid_func_trace_counter];
		zend_jit_ret_trace_counter_handler = dasm_labels[zend_lbhybrid_ret_trace_counter];
		zend_jit_loop_trace_counter_handler = dasm_labels[zend_lbhybrid_loop_trace_counter];
	} else {
		// ...
		zend_jit_func_trace_counter_handler = (const void*)zend_jit_func_trace_helper;
		zend_jit_ret_trace_counter_handler = (const void*)zend_jit_ret_trace_helper;
		zend_jit_loop_trace_counter_handler = (const void*)zend_jit_loop_trace_helper;
	}
}

static zend_always_inline ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL zend_jit_trace_counter_helper(uint32_t cost ZEND_OPCODE_HANDLER_ARGS_DC)
{
...
	// PHP VM每次执行到带了tracing计数handler的opline，这里的counter就会每次减cost，
	// 这个cost是个定值，即((ZEND_JIT_COUNTER_INIT + JIT_G(hot_func) - 1) / JIT_G(hot_func)))
	*(ZEND_OP_TRACE_INFO(opline, offset)->counter) -= cost;

	// 当小于等于0的时候表示达到了阈值，即变成了热点代码，
	// 需要对其进行JIT编译，调用zend_jit_trace_hot_root进入录制执行路径的过程。
	if (UNEXPECTED(*(ZEND_OP_TRACE_INFO(opline, offset)->counter) <= 0)) {
		*(ZEND_OP_TRACE_INFO(opline, offset)->counter) = ZEND_JIT_COUNTER_INIT;
		if (UNEXPECTED(zend_jit_trace_hot_root(execute_data, opline) < 0)) {
		...
		}
...

}
// zend_jit_vm_helpers.h 最后调用到zend_jit_trace_counter_helper
ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL zend_jit_func_trace_helper(ZEND_OPCODE_HANDLER_ARGS)
{
	ZEND_OPCODE_TAIL_CALL_EX(zend_jit_trace_counter_helper,
		((ZEND_JIT_COUNTER_INIT + JIT_G(hot_func) - 1) / JIT_G(hot_func)));
}
```



### Optimization

上面基本梳理了计数的代码，接下来是我在测试过程中发现的一个小问题，即当JIT编译了足够多的代码之后，JIT停止以后这些计数功能还在继续执行，带来了一点overhead。毕竟多执行了几个函数，还有counter占了一些cache，还导致了一些错误的分支预测。

所以我想将这些计数功能停止，关键思路其实就是将那些带有计数handler（JIT$$hybrid_func_trace_counter）的opline的handler再设置回原始的由VM 执行的 handler （<execute_ex+7817>）。



优化的代码见GitHub [PR9343](https://github.com/php/php-src/pull/9343) , 相关的注释和讨论都有。

整个实现逻辑里关键的一步就是怎么将所有带有计数handler的那些opline给找出来，需要去遍历存在opcache里的所有script。这个逻辑其实是参考的前面对编译好的脚本缓存进opcache的逻辑。

其实在PHP里opcache将编译的脚本缓存是个非常重要的功能，这部分的实现后续有机会会再写一篇博客。



### Summary

1. 一些汇编码看起来还是有点痛苦，有时候可以先看简单的类似的逻辑。比如zend_jit_func_trace_helper这里。
2. GDB的命令很强大，内存里的数值也可以看  `x/hd 0x7ffff5519524`, 0x7ffff5519524 <zend_jit_hot_counters+4>:       32531
3. 数据结构体很多的时候，注意梳理数据结构



### Reference

1. [jit-in-depth](https://php.watch/articles/jit-in-depth)
2. [opcache.jit](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit)
3. [PR](https://github.com/php/php-src/pull/9343/files)