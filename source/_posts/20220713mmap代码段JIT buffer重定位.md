---
title: mmap代码段JIT buffer重定位
date: 2022-06-13 22:51:17
mathjax: true
tags:
- PHP-SRC
- 性能优化
- 内存布局
categories:
- 性能优化

---

### JIT buffer重定位

#### 问题背景

##### PHP8 JIT

Just-In-Time compilation is a hybrid model of interpreter and Ahead-of-Time compilation, that some or all of the code is compiled, often at run-time, without requiring the developer to manually compile it. 

JIT也就是边解释执行代码，找到热点代码将其编译，以后就直接去执行编译成机器码的热点代码，提高性能。JIT



> JavaScript Performance Penalty. Function calls between embedded builtins and JIT compiled code can come at a considerable performance penalty. With the x86-64 instruction set,  we can’t use direct calls. Instead, we need to rely on indirect calls through a register or memory operand. Such calls rely more heavily on prediction since it’s not immediately apparent from the call instruction itself what the target of the call is.

> For 64-bit applications, branch prediction performance can be negatively impacted when the target of a branch is more than 4 GB away from the branch.  

JIT生成机器码之后，从编译好的code跳转到解释执行的code，这之间是存在性能损失的，因为Intel对这种远距离的分支跳转，分支预测得不准（似乎是有对分支跳转的距离假设，比如函数A跳转到函数B的虚拟地址一般是在4GB以内）。

思考：那PHP里的JIT buffer的位置是在哪呢？ 跳转到PHP的解释器执行的代码的情况如何呢？ 是否也可以对这个远距离跳转去优化呢？



#### JIT buffer内存布局

我们跑了wordpress的workload，php.ini的配置中如下两项配置了opcache的最大值和jit buffer的最大值

```
opcache.memory_consumption=1024
; The size of the shared memory storage used by OPcache, in megabytes.

opcache.jit_buffer_size=128M
; The amount of shared memory to reserve for compiled JIT code. A zero value disables the JIT.

opcache.huge_code_pages=1
; Enables or disables copying of PHP code (text segment) into HUGE PAGES. This should improve performance
```

跑workload 用户发出请求由php处理的过程中，查看任意php-fpm 的进程的内存使用maps文件。

```bash
$ cat /proc/1888419/maps
# open hugepage， /proc/sys/vm/nr_hugepages = 20000

# start addr-end addr     perms offset  dev   inode                  pathname
555555400000-5555554f9000 r--p 00000000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
# PHP .text is put into the huge page
555555600000-555555a00000 r-xp 00000000 00:0f 285761873                  /anon_hugepage (deleted)
555555a00000-555556231000 r--p 00600000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
55555655f000-555556600000 r--p 00f5f000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556600000-555556604000 rw-p 01000000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556604000-555556624000 rw-p 00000000 00:00 0
555556800000-555556ca4000 r-xp 01400000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556ca4000-555556ebe000 rw-p 00000000 00:00 0                          [heap]
555556ebe000-555556f19000 rw-p 00000000 00:00 0                          [heap]
7fffad3f0000-7fffad400000 rwxp 00000000 00:00 0
7fffad400000-7fffad600000 rw-p 00000000 00:0f 285734811                  /anon_hugepage (deleted)
# here 1024 MB Opcache
7fffad600000-7fffed600000 rw-s 00000000 00:0f 285761874                  /anon_hugepage (deleted)
# here 128M JIT Buffer
7fffed600000-7ffff5600000 r-xs 40000000 00:0f 285761874                  /anon_hugepage (deleted)
7ffff560b000-7ffff565b000 rwxp 00000000 00:00 0
7ffff565b000-7ffff56dc000 rw-p 00000000 00:00 0
7ffff56e0000-7ffff56f0000 rwxp 00000000 00:00 0
7ffff56f0000-7ffff5705000 rw-s 00000000 00:01 135173                     /dev/zero (deleted)
7ffff5705000-7ffff571a000 r--p 00000000 08:02 6166503                    /opt/pkb/git/hhvm-perf/opcache-wp5.8-php8.1.4-jit.so
7ffff571a000-7ffff57cb000 r-xp 00015000 08:02 6166503                    /opt/pkb/git/hhvm-perf/opcache-wp5.8-php8.1.4-jit.so
7ffff57cb000-7ffff57e4000 r--p 000c6000 08:02 6166503                    /opt/pkb/git/hhvm-perf/opcache-wp5.8-php8.1.4-jit.so
7ffff57e4000-7ffff57e7000 r--p 000de000 08:02 6166503                    /opt/pkb/git/hhvm-perf/opcache-wp5.8-php8.1.4-jit.so
7ffff57e7000-7ffff57f7000 rw-p 000e1000 08:02 6166503                    /opt/pkb/git/hhvm-perf/opcache-wp5.8-php8.1.4-jit.so
```

可以看到 7fffed600000 这里JIT 跳转进入到555555600000 php的.text 段，距离是很远的，far jump会有性能损失。

注意，当 opcache.jit_buffer_size=16M , opcache.memory_consumption=128M 的时候，mmap会分配到＜2GB的位置上去。

```bash
# opcache
40000000-48000000 rw-s 00000000 00:0f 286759728                          /anon_hugepage (deleted)
# jit buffer
48000000-49000000 r-xs 08000000 00:0f 286759728                          /anon_hugepage (deleted)
555555400000-5555554f9000 r--p 00000000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555555600000-555555a00000 r-xp 00000000 00:0f 286759727                  /anon_hugepage (deleted)
555555a00000-555556231000 r--p 00600000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
55555655f000-555556600000 r--p 00f5f000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556600000-555556604000 rw-p 01000000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556604000-555556624000 rw-p 00000000 00:00 0
555556800000-555556ca4000 r-xp 01400000 08:02 6166492                    /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556ca4000-555556ebe000 rw-p 00000000 00:00 0                          [heap]
555556ebe000-555556f19000 rw-p 00000000 00:00 0                          [heap]
```

#### **代码实现**

代码思路在代码注释里写的很清晰了，就是在分配JIT buffer之前（函数create_segments），找一段空闲的内存空间，距离 PHP .text 段在4GB以内的都可以。

https://github.com/stkeke/php-src/commit/7ccc2a9209af156a9da0619b22e0be12f0adc2ab

```c
/* 
			We will search for any candidates BEFORE and AFTER PHP .text segment.
        E.g., [hole0], [hole1], [hole2], and [hole3] all might be good
        candidates. We have verified that using [hole3] as jit buffer will not
        block or affect heap growth for later memory allocation.
*/

//......
#if defined(MAP_HUGETLB)
/* Do jit buffer relocation, only if requested buffer size is
       greater than at least one huge page and takes up multiple huge pages.
    */
    if (requested_size >= huge_page_size &&
        requested_size % huge_page_size == 0)
    {
        /* Try mmap all candidate address, return if any one succeeds. */
        for(int i = 0; i < candidate_count; i++) {
            res = mmap((void*)candidates[i], requested_size, flags,
                       MAP_SHARED|MAP_ANONYMOUS|MAP_HUGETLB|MAP_FIXED, fd, 0);
            if (MAP_FAILED != res) {
                return res;
            }
        }
    }
#endif /* MAP_HUGETLB */
```

后续maintainer修改了这个patch，更简洁一些。

https://github.com/php/php-src/commit/17aa81a5e22d4b8d1ffd7c89cb641939b4f6b7db

最后JIT Buffer的位置被移动到了heap之后，或者一开始的位置。

```bash
555513e00000-555553e00000 rw-s 00000000 00:0f 287680313                  /anon_hugepage (deleted)
555553e00000-555555200000 r-xs 40000000 00:0f 287680313                  /anon_hugepage (deleted)
555555400000-5555554f9000 r--p 00000000 08:02 11403879                   /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555555600000-555555a00000 r-xp 00000000 00:0f 287680312                  /anon_hugepage (deleted)
555555a00000-555556231000 r--p 00600000 08:02 11403879                   /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
55555655f000-555556600000 r--p 00f5f000 08:02 11403879                   /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556600000-555556604000 rw-p 01000000 08:02 11403879                   /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556604000-555556624000 rw-p 00000000 00:00 0
555556800000-555556ca4000 r-xp 01400000 08:02 11403879                   /opt/pkb/git/hhvm-perf/php-fpm-wp5.8-php8.1.4-jit.bolt
555556ca4000-555556ebe000 rw-p 00000000 00:00 0                          [heap]
555556ebe000-555556f19000 rw-p 00000000 00:00 0                          [heap]
7ffff53f0000-7ffff5400000 rwxp 00000000 00:00 0
7ffff5400000-7ffff5600000 rw-p 00000000 00:0f 287683296                  /anon_hugepage (deleted)
```

代码已经合并到php-src社区，好久没写C了，这里总结一些写代码时遇到的思考和问题。

1. 用函数指针可以获取代码段所在的位置。void* addr = php_text_lighthouse;
2. 注意ifdef的使用中，不存在的代码最后不会被编译进binary，要注意到hugepage 变量的定义，避免出现未定义的问题。
3. 注意区分数据类型，uintptr_t 表示地址，任何指向void的合法指针都可以转化为这个类型。 ptrdiff_t 有符号整数类型，两个指针相减结果的类型。表示地址之间的距离。 size_t 表示大小,是无符号整数类型，这是sizeof操作符结果的类型。
4. 函数参数是数组指针的时候，最好还传一个数组的max_size，避免越界
5. 减少if的嵌套
6. 用中间变量保存一些计算步骤，让代码更易读
7. 提PR的时候写好注释，所有关键地方都写上，方便maintainer阅读



#### Reference

1. [Short builtin calls](https://v8.dev/blog/short-builtin-calls)
2. [Intel Optimization Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-optimization-manual.pdf)

3. [PHP: Runtime Configuration - Manual](https://www.php.net/manual/en/opcache.configuration.php#ini.opcache.jit-buffer-size)