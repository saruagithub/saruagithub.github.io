---
title: 20230515 Trace-based Just-in-Time Type Specialization for Dynamic Languages
date: 2023-05-21 22:35:17
mathjax: false
tags:
- PHP-SRC
- 性能优化
- JIT
categories:
- 性能优化


---



### 1 Abstract

为loop产生类型专有化的机器码，identifies frequently executed loop traces at run-time and then generates machine code on the fly that is specialized for the actual dynamic types occurring on each path through the loop.

对嵌套循环的处理 the way of incrementally compiling lazily discovered alternative paths through nested loops.

### 2 Introduction

#### JIT compilation and a Trace

传统静态编译的成本高，且需要处理所有的变量类型。traditional static analysis is very expensive and hence not well suited for the highly interactive environment of a web browser

JIT编译和一次trace。 the system starts running JavaScript in a fast-starting bytecode interpreter. As the program runs, the system identifies hot (frequently executed) bytecode sequences, records them, and compiles them to fast native code. We call such a sequence of instructions a trace.

- we expect hot loops to be mostly type-stable, meaning that the types of values are invariant
- Each compiled trace covers one path through the program with one mapping of values to types.
- 两种情况下会side exit,  Every compiled trace contains all the guards (checks) required to validate the speculation. If one of the guards fails (if control flow is different, or a value of a different type is generated), the trace exits.
- 热点侧退出点，形成跟踪树。If an exit becomes hot, the VM can record a branch trace starting at the exit to cover the new path. the VM records a trace tree covering all the hot paths through the loop

#### Nested loop tracing

在跟踪内部循环的时候如果exit到外部循环的话，放弃trace。Alternatively, the VM could simply stop tracing, and give up on ever tracing outer loops

We solve the nested loop problem by recording nested trace trees.

避免tail duplication的问题，可以在外部循环重启启动trace。 The system stops extending the inner tree when it reaches an outer loop, but then it starts a new trace at the outer loop header. When the outer loop reaches the inner loop header, the system tries to call the trace tree for the inner loop.

trace是一次执行下来的，没有控制流节点（no PHI）。所以可以去做一些优化。Because traces have no internal control-flow joins, they can be optimized in linear time by a simple compiler

#### Example

```jsx
for (var i = 2; i < 100; ++i) {
	if (!primes[i])
		continue;
	for (var k = i + 1; i < 100; k += i)
		primes[k] = false;
}
# primes is initialized to an array of 100 false values on entry to this code
```

![20230515_workflow](/images/20230515_flow.png)

![20230515_twoTraces](/images/20230515_twotraces.png)

TraceMonkey stops recording when execution returns to the loop header or exits the loop

充分满足条件才会进入机器码执行。This trace can be entered if the PC is at line 4, i and k are integers, and primes is an object.

### 3 Trace Tree

#### Trace

一串程序执行流，A trace is a program path

- 确定的变量类型 A typed trace is a trace annotated with a type for every variable (including temporaries) on the trace.
- LIR中间表示被用于录制。 The LIR operations are generic enough that the backend compiler is language independent
- 录制中间值到小的激活活动区中，trace直接读，trace退出到VM时拷贝会解释器里去。  A trace records all its intermediate values in a small activation record area. To make variable accesses fast on trace. Thus, the trace can read and write these variables with simple loads and stores from a native activation recording, independently of the boxing mechanism used by the interpreter. When the trace exits, the VM boxes the values from this native storage location and copies them back to the interpreter structures.
- guard指令，遇到不同的分支或不同的变量类型会跳出trace。For every control-flow branch in the source program, the recorder generates conditional exit LIR instructions. These instructions exit from the trace if required control flow is different from what it was at trace recording.   When the VM observes a side exit along such a type guard, a new typed trace is recorded originating at the side exit location
- 类型专有化对object的处理 type specialization（共享hash表跟vector）.  TraceMonkey can simply observe the result of the search process and record the simplest possible LIR to access the property value. Then the recorded can generate LIR that reads o.x with just two or three loads: one to get the prototype, possibly one to get the property value vector, and one more to get slot 2 from the vector.
- trace可以函数内联，拷贝传参，返回值处理，记录调用栈帧。LIR traces can cross function boundaries in either direction, achieving function inlining. Move instructions need to be recorded for function entry and exit to copy arguments in and return values out. The frame entry and exit LIR saves just enough information to allow the intepreter call stack to be restored later and is much simpler than the interpreter’s standard call code.
- 侧出口，恢复解释器的执行状态 The exit branches to a side exit, a small off-trace piece of LIR that returns a pointer to a structure that describes the reason for the exit along with the interpreter PC at the exit point and any other data needed to restore the interpreter’s state structures.

#### Trace Trees

多侧出口形成了trace树。 When a side exit becomes hot, TraceMonkey starts a new branch trace from that point and patches the side exit to jump directly to that trace. In this way, a single trace expands on demand to a single-entry, multiple-exit trace tree. 

- 在loop的开头处开始录制，root trace。Starting a tree just means starting recording a trace for the current point and type map and marking the trace as the root of a tree.
- 录制loop结束点，跳转到loop header处。Ideally, the trace reaches the loop header where it started with the same type map as on entry. This is called a type-stable loop iteration.
- loop过程中类型变化导致exit的话需要检查type map。 Every time an additional type unstable trace is added to a region, its exit type map is compared to the entry map of all existing traces in case they complement each other.
- trace中对类型的误预测，提出oracle检查。This represents a mis-speculation, since at trace entry we specialized the Number-typed value to an integer, assuming that at the loop edge we would again find an integer value in the variable, allowing us to close the loop. Speculation towards integers is performed only if no adverse information is known to the oracle about that particular variable.
- side exits: Side exits lead to different paths through the loop, or paths with different types or representations.
- 限制内循环的侧出口trace只在内循环。It extends only if the side exit is for a control-flow branch, and only if the side exit does not leave the loop.  In particular we do not want to extend a trace tree along a path that leads to an outer loop, because we want to cover such paths in an outer tree through tree nesting.

#### Blacklisting

对trace失败的处理 we would suddenly incur a punishing runtime overhead if we repeatedly try to record a trace for this path and repeatedly fail to do so, since we abort tracing every time we observe an exception being thrown

尝试trace但失败了几次之后就进黑名单，When the VM fails to finish a trace starting at a given point, the VM records that a failure has occurred. The VM also sets a counter so that it will not try to record a trace starting at that point until it is passed a few more times

blacklisted, which means the VM will never again start recording at that point.

用no-op指令来避免进入trace monitor。  We define an extra no-op bytecode that indicates a loop header.  To blacklist a fragment, we simply replace the loop header no-op with a regular no-op. Thus, the interpreter will never again even call into the trace monitor.

特殊的outer loop录制退出情况特殊的黑名单处理。The problem is that outer loop traces often abort during startup (because the inner tree is not available or takes a side exit), which would lead to their being quickly blacklisted by the basic algorithm。 we should not count such aborts toward blacklisting as long as we are able to build up more traces for the inner tree.

### 4 Nested Trace tree

Usually, the inner loop (with header at i2) becomes hot first, and a trace tree is rooted at that point.

避免跟踪相同的执行流  In order to execute programs with nested loops efficiently, a tracing system needs a technique for covering the nested loops with native code without exponential trace duplication.

![20230515_typeunstable](/images/20230515_typeunstable.png)

构建嵌套树的算法

- 已编译的loop直接调用 recording a trace for loop R and reach the header of Loop O,  If LO has a type-matching compiled trace tree, we call LO as a nested trace tree.
- 编译Inner loop。we simply abort recording the first trace. The trace monitor will see the inner loop header, and will immediately start recording the inner loop.
- 内外loop各自独立。The goal of nesting is to make inner and outer loops independent;
- 尽量退出到外部loop的相同处。 thus when the inner tree is called, it must exit to the same point in the outer tree every time with the same type map.
- 内部树继续编译增长。If an inner tree side exit happens during execution of a compiled trace for the outer tree, we simply exit the outer trace and start recording a new branch in the inner tree.

### 5 Trace Tree Optimization （better native code）

管道过滤优化 we chose a small set of optimizations. We implemented the optimizations as pipelined filters so that they can be turned on and off independently, and yet all run in just two loop passes over the trace: one forward and one backward.

前向过滤  Every time the trace recorder emits a LIR instruction, the instruction is immediately passed to the first filter in the forward pipeline

- forward filter optimizations are performed as the trace is recorded. Each filter may pass each instruction to the next filter unchanged, write a different instruction to the next filter, or write no instruction at all.
- 当前四种前向过滤  a soft-float filter converts floating-point LIR instructions to sequences of integer instructions.
- constant subexpression elimination
- expression simplification, including constant folding and a few algebraic identities
- source language semantic-specific expression simplification

后向过滤 When trace recording is completed, nanojit runs the backward optimization filters.

- When running the backward filters, nanojit reads one LIR instruction at a time, and the reads are passed through the pipeline.
- Dead data-stack store elimination. The LIR trace encodes many stores to locations in the interpreter stack. But these values are never read back before exiting the trace (by the interpreter or another trace).
- Dead call-stack store elimination.
- Dead code elimination. This eliminates any operation that stores to a value that is never used

### 6 Register Allocation

贪心，后向pass。 a simple greedy register allocator that makes a single backward pass over the trace (it is integrated with the code generator).

启发式的最老寄存器 We use a class heuristic that selects the “oldest” register carried value

### Implementation

Compiled traces are stored in a trace cache, indexed by intepreter PC and type map

trace活动区 To execute a trace, the monitor must build a trace activation record containing imported local and global variables, temporary stack space, and space for arguments to native calls.

- The local and global values are then copied from the interpreter state to the trace activation record. Then, the trace is called like a normal C function pointer.
- 活动区变量拷贝 When a trace call returns, the monitor restores the interpreter state. First, the monitor checks the reason for the trace exit and applies blacklisting if needed. Then, it pops or synthesizes interpreter JavaScript call stack frames as needed. Finally, it copies the imported variables back from the trace activation record to the interpreter state
- 减少解释器到JIT引擎的转换开销 minimizing the number of interpreterto-trace and trace-to-interpreter transitions is essential for performance.
- At a side exit, the exiting trace only needs to write live register-carried values back to its trace activation record.
- 活动记录重用优化 In our implementation, identical type maps yield identical activation record layouts, so the trace activation record can be reused immediately by the branch trace.
- 小的trace增加指令数可能有开销 Trace stitching has a noticeable cost. Although writing to memory and then soon reading back would be expected to have a high L1 cache hit rate, for small traces the increased instruction count has a noticeable cost.
- 对跟踪树的重编译 The alternate solution is to recompile an entire trace tree, thus achieving inter-trace register allocation。 The disadvantage is that tree recompilation takes time quadratic in the number of traces. That problem might be mitigated by recompiling only at certain points, or only for very hot, stable trees

### Summary

1. PHP8 的JIT实现基本就是按照paper的过程来的
2. 关注流程图中的性能开销处，JIT对这些地方的性能要求很高，需要重点优化
3. 多个Trace Tree 之间的关联和对不同类型的处理是非常重要的