---
title: 20221018 Indirect call to Direct call
date: 2022-10-18 22:51:17
mathjax: true
tags:
- PHP-SRC
- 性能优化
- JIT
categories:
- 性能优化

---



## Background

### 优化原理

参考JS的V8里头的优化说明：

> With the [x86-64](https://en.wikipedia.org/wiki/X86-64) instruction set, this means we can’t use direct calls: they take a 32-bit signed immediate that’s used as an offset to the address of the call, and the target may be more than 2 GiB away. Instead, we need to rely on indirect calls through a register or memory operand. Such calls rely more heavily on prediction since it’s not immediately apparent from the call instruction itself what the target of the call is.
> 

> It should reduce strain on the indirect branch predictor as the target is known after the instruction is decoded, but it also doesn't require the target to be loaded into a register from a constant or memory.
> 

并且前文我们在看JIT Buffer relocation的性能分析（[https://saruagithub.github.io/2022/06/13/20220713mmap代码段JIT buffer重定位/](https://saruagithub.github.io/2022/06/13/20220713mmap%E4%BB%A3%E7%A0%81%E6%AE%B5JIT%20buffer%E9%87%8D%E5%AE%9A%E4%BD%8D/)）的时候就已经发现了减少indirect call是可以大大提高预测准确率的，因此我们进一步来分析当前JIT里头的indirect call

### 问题调研

get indirect and direct branch statistics before and after JIT buffer relocation patch to see whether indirect branch still exists

我们看JITted code生成的模板里的indirect call还有多少，ext/opcache/jit/zend_jit_x86.dasc 原始实现

```c
#define IS_32BIT(addr) (((uintptr_t)(addr)) <= 0x7fffffff)

|.macro EXT_CALL, func, tmp_reg
|	.if X64
||		if (IS_32BIT(dasm_end) && IS_32BIT(func)) {
|			call qword &func /* direct call */
||		} else {
|			LOAD_ADDR tmp_reg, func
|			call tmp_reg /* indirect call */
||		}
|	.else
|		call dword &func
|	.endif
|.endmacro
```

这个模板文件里如果从当前位置call func，跳转距离在IS_32BIT的话，就是direct call 。否则就生成indirect call，通过寄存器来访问地址。这样就得需要分支预测器BPU进行预测了。

```bash
# this JITTed stub func call func by indirect call
JIT$$leave_function: ; (unknown)
    555596c00170:       mov 0x28(%r14), %edi
    555596c00174:       test $0x20000, %edi
        jnz .L1
    555596c0017c:       mov $zend_jit_leave_nested_func_helper, %rax
    555596c00186:       call *%rax   
    555596c00188:       jmp (%r15)
.L1:
    555596c0018b:       mov $zend_jit_leave_top_func_helper, %rax
    555596c00195:       call *%rax
    555596c00197:       jmp (%r15)
```

所以我们尝试去掉用寄存器访问地址。

### 代码实现

```c
// easy implement
#define IS_SIGNED_32BIT(val) ((((intptr_t)(val)) <= 0x7fffffff) && \
 (((intptr_t)(val)) >= (-2147483647 - 1)))

#define MAY_USE_32BIT_ADDR(addr) \
	(IS_SIGNED_32BIT((char*)(addr) - (char*)dasm_buf) && \
	IS_SIGNED_32BIT((char*)(addr) - (char*)dasm_end))
```

[Indirect call reduction for Jit code by wxue1 · Pull Request #9579 · php/php-src](https://github.com/php/php-src/pull/9579)

这里需要注意特殊的情况，距离是前后2GB以及一些特殊case。

```c
#define IS_NEAR(end_addr, func) ((((uintptr_t)(end_addr)^(uintptr_t)(func)) >> 31) == 0)
// 这里有个corner case
// XOR之后，得到FFFF0000；如果把FFFF0000看作64位无符号数，变成了00000000-FFFF0000，
// 这样即便右移31位，还剩个0000 0000 1， 不对
```

```bash
--- I change the EXT_CALL code and use direct call ---

JIT$$leave_function: ; (unknown)
    555596e00150:       mov 0x28(%r14), %edi
    555596e00154:       test $0x20000, %edi
        jnz .L1
    555596e0015c:       call 0x5555f54d46a1
    555596e00161:       jmp (%r15)
.L1:
    555596e00164:       call 0x5555f54d47e3
    555596e00169:       jmp (%r15)
```

对runtime生成的JITTed code去验证字节码：

```bash
	 0x0000555596e00610 <+0>:     66 c7 02 13 7f  mov    WORD PTR [rdx],0x7f13
   0x0000555596e00615 <+5>:     4c 89 f7        mov    rdi,r14
   0x0000555596e00618 <+8>:     4c 89 fe        mov    rsi,r15
   0x0000555596e0061b <+11>:    e8 93 dc 6c 5e  call   0x5555f54ce2b3  #opcode is e3 now
# gdb command: x/64bx func_address
```

我们去看intel的SDM手册，可以看到第二行 E8这个操作码可以call  rel32 （rel指的是relocation， 是an offset to the address of the call），这是相对近调用，相对于下一指令的位移

![IndirectCall](/images/20221018indirect.png)

### Emon性能分析

我们再对优化后的workload压测跑分，收集emon数据进行分析，看看具体的优化效果如何。

这里的数据我是从JIT buffer relocation，到64bit packing 到 direct call 每个优化一步步收集起来的。我们直接看最后的direct call的分析。数据是wordpress，SPR D0 56core上跑的

1. TPS 整体提高了1%
2. code cache miss整体变好了，但其实是因为分支预测更好了，加载指令更快也更准确了。
3. 最根本的优化是对Branch prediction的，表现类似于relocation优化。总的分支执行更快了，误预测的更少
4. 具体去分析branch类型和误预测的类型，也可以看出来整体的Direct call更多，Indirect call更少
5. 从误预测的情况来看，indirect call预测失败也下降了很多，几乎75%
6. 并且indirect call的预测已经比较准了，误预测率占2.2%，很少了。反而看起来现在是条件分支的差一些 8.7%算比较高的误预测率了。
7. 可以估计下一次indirect call会多消耗多少cycle数

$$
indirect\_call\_cycle = reduced\_cycle / reduced\_indirect\_call
$$

in excel, =(P9/P76) 就是计算的这个，可以看到大概每transaction，一次indirect call func大概会消耗44.9 cycles。这个计算在relocation patch那里的emon数据和direct call的数据是基本一致的。

## Summary

### 通用优化思路

indirect call转为direct call是非常常见的优化。

### 遗留问题

1. dynasm 生成字节码的原理
2. 还有其他地方的indirect call么
3. 条件分支的预测优化？？

### Reference

[Short builtin calls](https://v8.dev/blog/short-builtin-calls)

[Indirect vs Direct Function Call Overhead in C/C++](https://gist.github.com/rianhunter/0be8dc116b120ad5fdd4)