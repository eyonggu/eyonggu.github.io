---
layout: post
title: "How Ftrace works"
date: 2022-03-08
categories: jekyll blogging
tags: [tracing, ftrace]
published: false
---

Ftrace is a tracing utility built in kernel to see what is happening inside the kernel.

Ftrace is not equal to function tracing, instead, it is really a framework of several tracing utilities.

There are many kinds of [tracers](https://www.kernel.org/doc/html/latest/trace/ftrace.html#the-tracers) supported by Ftrace framework.

In this article, only function/function_graph tracer will be looked at.

## Profiling Implementation Basics

Before start to dig into FTrace implementation, we first take a look at the basics of profiling implemenation, which is also used in FTrace.

Profiling works by changing how every function is compiled, so that when it is called, it will stash away some information about where it was called from. From this, the profiler can figure out what function called it, and can count how many times it was called.

The change is made by the compiler when the code is compiled with the ``-pg`` option, which causes every function to call mcount (or _mcount, or __mcount, depending on the OS and compiler) as one of the first operation.

The mcount routine, included in the profiling library, is responsible for recording in an in-memory call graph table both its parent routine (the child) and its parent's parent. This is typically done by examining the stack frame to find both the address of the child, and the return address in the original parent. Since this is a very machine-dependant operation, mcount itself is typically a short assembly-language stub routine that extracts the required information, and then calls __mcount_internal (a normal C function) with two arguments - frompc and selfpc. __mcount_internal is responsible for maintaining the in-memory call graph, which records frompc, selfpc, and the number of times each of these call arcs was transversed.

```sh
# -o- : output to stdout
# - : input from stdin
echo 'main(){}' | gcc -x c -S -pg -o- -  | grep mcount
```

Enough now, let's move to FTrace implementation.

## Code compiling with FTrace

First, in the top Makefile, if ``CONFIG_FUNCTION_TRACER`` is defined, code will be compiled with ``-pg`` option, which means mcount routine is added to each function call.

```c
ifdef CONFIG_FUNCTION_TRACER
    CC_FLAGS_FTRACE := -pg
endif
```
Note! On X86, starting with gcc verson 4.6, the ``-mfentry`` has been added, which calls "__fentry__" instead of "mcount", which is called before the creation of the stack frame.

A section called "__mcount_loc" is created that holds references to all the mcount/fentry call sites in the .text section.

## Dynamic Ftrace

As you can image, mcount will add overhead to the function call during runtime. To overcome it, another kernel option ``CONFIG_DYNAMIC_FTRACE`` is always recommended to be enabled togeth with ``CONFIG_FUNCTION_TRACER``. With Dynamic Ftrace, it is possible to enable individual function trace, functions not enabled will not impact on system performance.

Now let's look at what will happen with dynamic Ftrace enabled during runtime.

ftrace_init() in [kernel/trace/ftrace.c](https://github.com/torvalds/linux/blob/master/kernel/trace/trace_kprobe.c) is called by start_kernel() during boot up.

```c
void __init ftrace_init(void)
{
    ...

    ftrace_dyn_arch_init();

    ftrace_process_locs(NULL, __start_mcount_loc, __stop_mcount_loc);

    set_ftrace_early_filters();
}
```

As you can see now, ftrace_process_locs() tries to look at all the mcounts, and update the code, i.e. change mcount jump to NOP instruction. With this update, the system will run with virtually no overhead when function tracing is disabled.

```c
static int ftrace_process_locs(struct module *mod,
        unsigned long *start,
        unsigned long *end)
{
    ...

    ftrace_allocate_pages(count);

    ...

    ftrace_update_code(mod, start_pg);

    ...
}

static int ftrace_update_code(struct module *mod, struct ftrace_page *new_pgs)
{
}
```
## Enable dynamic tracing of functions

Two sysfs files are created to enable/disable tracing of specified functions.
- set_ftrace_filter
- set_ftrace_notrace

A list of available functions that you can add to these two files is listed in
- available_filter_functions

When tracing is enabled, the call can jump to either ftrac_caller() or ftrace_regs_caller().


## Links
- [ftrace - Function Tracer](https://www.kernel.org/doc/html/latest/trace/ftrace.html#)
- [Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
- [Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
- [Secrets of the Ftrace function tracer](https://lwn.net/Articles/370423/)
- [function tracer guts](https://www.kernel.org/doc/Documentation/trace/ftrace-design.txt)
