---
layout: post
title: "How Kprobe-based trace event works"
date: 2022-02-23
categories: jekyll blogging
tags: [kprobe, tracing]
---

In previous article, we looked at how normal kprobe works. Based on kprobe, it is possible to create dynamic trace event. In this one, let's look at ohow the kprobe-based trace event works.

Compared to other kind trace events (TracePoint/function tracer), kprobe-based trace event has the advantages:
- can probe anywhere (symbol + offset) and function return (kretprobe).
- can add/remove probe points on the fly.
- can fetch various registers/memory/data symbols, dereferencing(resolving pointer) is also supported.

There are two kernel options to enable this functionality.
- CONFIG_KPROBE_EVENT=y
- CONFIG_DYNAMIC_FTRACE=y

The kprobe-based events are be added/removed via sysfs virtual file system. For example:

```sh
# Add a new probe event */
echo 'p:myprobe do_sys_open dfd=%ax filename=%dx flags=%cx mode=+4($stack)' > /sys/kernel/debug/tracing/kprobe_events

# remove a probe by name */
echo -:myprobe >> kprobe_events

# show profiles of each event
cat /sys/kernel/debug/tracing/kprobe_profile
```

Synopsis of kprobe_events:

```
 p[:[GRP/]EVENT] [MOD:]SYM[+offs]|MEMADDR [FETCHARGS]  : Set a probe
 r[MAXACTIVE][:[GRP/]EVENT] [MOD:]SYM[+0] [FETCHARGS]  : Set a return probe
 p:[GRP/]EVENT] [MOD:]SYM[+0]%return [FETCHARGS]       : Set a return probe
 -:[GRP/]EVENT                                         : Clear a probe
```

An event is defined by ``[GRP/][EVENT]``, Kprobe is defined by ``[MOD:]SYM[+offs]|MEMADDR``.

It is possible to add multiple kprobes to the same event, for example:

```sh
echo p:testevent _do_fork > kprobe_events
echo p:testevent fork_idle >> kprobe_events
```

## Overview of Key structs.

![Overview of structs](/assets/kprobe_trace.png)


## Create a new kprobe event

The kprobe-based event trace is implemented in [kernel/trace/trace_kprobe.c](https://github.com/torvalds/linux/blob/master/kernel/trace/trace_kprobe.c).

When a new event is written to sysfs *kprobe_events* file, function *probes_write*(kprobe_events_ops.write) is called to create a new event.

```c
static const struct file_operations kprobe_events_ops = {
    .write          = probes_write;
};

static ssize_t probes_write(struct file *file, const char __user *buffer,
        size_t count, loff_t *ppos)
{
    return trace_parse_run_command(file, buffer, count, ppos,
            create_or_delete_trace_kprobe);
}
```

The event creation is done in *__trace_kprobe_create()* function.
```c
static int __trace_kprobe_create(int argc, const char *argv[])
{
    /* EYONGGU: parse the kprobe_event string */
    ...

    /* EYONGGU: alloc a new trace_kprobe */
    tk = alloc_trace_kprobe(group, event, addr, symbol, offset, maxactive,
                               argc - 2, is_return);

    traceprobe_set_print_fmt(&tk->tp, ptype);

    /* EYONGGU: register new event and k*probe */
    register_trace_kprobe(tk);

    ...
}
```

First, it allocates an instance of struct trace_kprobe and initializes it.
The embeded kprobe struct is initialized with symbol/addr and pre_handler.
```c
static struct trace_kprobe *alloc_trace_kprobe(const char *group,
                                            const char *event,
                                            void *addr,
                                            const char *symbol,
                                            unsigned long offs,
                                            int maxactive,
                                            int nargs, bool is_return)
{
    /* EYONGGU: allocate a new instance of struct trace_kprobe */
    tk = kzalloc(SIZEOF_TRACE_KPROBE(nargs), GFP_KERNEL);

    tk->nhit = alloc_percpu(unsigned long);

    /* EYONGGU: assign tp->rp.kp.symbol_name/offset or addr from the function arguments */


    /* EYONGGU: Assign common handler for the kprobe */
    if (is_return)
        tk->rp.handler = kretprobe_dispatcher;
    else
        tk->rp.kp.pre_handler = kprobe_dispatcher;

    /* EYONGGU: initialize the trace probe / event struct */
    trace_probe_init(&tk->tp, event, group, false);

    dyn_event_init(&tk->devent, &trace_kprobe_ops);

    ...
}
```

It supports multiple probes per event, if the event (group/name) has been registered, the new trace probe will be appended to the event, otherwise, it registers a new event. In either case, new kprobe always needs to registerred.

```c
static int register_trace_kprobe(struct trace_kprobe *tk)
{
    /* EYONGGU: Check if the event (group & name) is already registerred, append to it */
    old_tk = find_trace_kprobe(trace_probe_name(&tk->tp),
                                trace_probe_group_name(&tk->tp));
    if (old_tk)
        append_trace_kprobe(tk, old_tk);

    /* Register new event */
    ret = register_kprobe_event(tk);

    /* Register k*probe */
    ret = __register_trace_kprobe(tk);

    dyn_event_add(&tk->devent, trace_probe_event_call(&tk->tp));

    ...
}
```

The new event is registerred to ftrace framework.

```c
static int register_kprobe_event(struct trace_kprobe *tk)
{
    /* EYONGGU: set struct trace_event funcs and filds_array,
        for kprobe:    funcs.trace = print_kprobe_event()
        for kretprobe: funcs.trace = print_kretprobe_event()
     */
    init_trace_event_call(tk);

    return trace_probe_register_event_call(&tk->tp);
}

int trace_probe_register_event_call(struct trace_probe *tp)
{
    ...

    /* EYONGGU: register to the global ftrace_event_list:
       static LIST_HEAD(ftrace_event_list);
     */
    register_trace_event(&call->event);

    /* EYONGGU: call generic functions in kernel/trace/trace_events.c
       __register_eventadd(call, NULL):   add to ftrace_events,
      __add_event_to_tracers(call): add the dynamic event to the tracing/events/
     */
    trace_add_event_call(call);

    ...
}
```

Finally, *register_kprobe()* is called to register the kprobe.

Note, the kprobe is in disabled state at this point, i.e. not armed.

```c
static int __register_trace_kprobe(struct trace_kprobe *tk)
{
    ...

    if (trace_kprobe_is_return(tk))
        register_kretprobe(&tk->rp);
    else
        register_kprobe(&tk->rp.kp);

    ...
}
```
## Enable kprobe trace event

The event can be enabled by echoing 1 to the "enable" file of either the group or the event, then the kprobe is armed. Likewise, kprobe is disarmed when the event is disabed.

```c
static int kprobe_register(struct trace_event_call *event,
        enum trace_reg type, void *data)
{
    struct trace_event_file *file = data;

    switch (type) {
    case TRACE_REG_REGISTER:
        return enable_trace_kprobe(event, file);
    case TRACE_REG_UNREGISTER:
        return disable_trace_kprobe(event, file);
    ...
    }
    return 0;
}

static int enable_trace_kprobe(struct trace_event_call *call,
        struct trace_event_file *file)
{
    ...

    list_for_each_entry(tk, trace_probe_probe_list(tp), tp.list) {
        /* EYONGGU: it call enable_kprobe(&tk->rp.kp) to arm the kprobe */
        __enable_trace_kprobe(tk);
    }

    ...
}
```

## When the kprobe is hit

As seen above, kprobe_dispatcher() is registerred as the pre_handler for each kprobe.

```c
static int kprobe_dispatcher(struct kprobe *kp, struct pt_regs *regs)
{
    struct trace_kprobe *tk = container_of(kp, struct trace_kprobe, rp.kp);

    raw_cpu_inc(*tk->nhit);

    if (trace_probe_test_flag(&tk->tp, TP_FLAG_TRACE))
        kprobe_trace_func(tk, regs);

    ...
}

static void
kprobe_trace_func(struct trace_kprobe *tk, struct pt_regs *regs)
{
    struct event_file_link *link;

    trace_probe_for_each_link_rcu(link, &tk->tp)
        __kprobe_trace_func(tk, regs, link->file);
}

static nokprobe_inline void
__kprobe_trace_func(struct trace_kprobe *tk, struct pt_regs *regs,
        struct trace_event_file *trace_file)
{
    ...

    struct trace_event_buffer fbuffer;

    fbuffer.regs = regs;
    fbuffer.entry = ring_buffer_event_data(fbuffer.event);
    fbuffer.entry.ip = (unsigned long)tk->rp.kp.addr;
    store_trace_args(...)

    trace_event_buffer_commit(&fbuffer);
}
```

## Links
- [Kprobe-based Event Tracing](https://www.kernel.org/doc/html/latest/trace/kprobetrace.html)
- [Dynamic probes with ftrace](https://lwn.net/Articles/343766/)
- [An introduction to KProbes](https://lwn.net/Articles/132196/)

