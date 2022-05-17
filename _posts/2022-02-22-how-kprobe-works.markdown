---
layout: post
title: "How Kprobe works"
date: 2022-02-22
categories: jekyll blogging
tags: [tracing, kprobe]
---

In this article, we will look at how kprobe is implemented in Linux kernel, in particular on X86 architecture.

## Kprobe data structure

First, let's look at the important attributes in [struct kprobe](https://github.com/torvalds/linux/blob/master/include/linux/kprobes.h#L59).

```c
struct kprobe {
    ...

    /* location of the probe point */
    kprobe_opcode_t *addr;

    /* EYONGGU: User can specify symbol+offset, which will be translated to addr */
    const char *symbol_name;
    unsigned int offset; /* offset into the symbol */

    /* called by addr is executed */
    kprobe_pre_handler_t pre_handler;
    /* called after addr is executed, optional */
    kprobe_post_handler_t post_handler;

    /* Saved opcode (first bype, which has been replaced with breakpoint) */
    kprobe_opcode_t opcode;

    /* copy of the original instruction + other inst attributes (e.g. size, type) */
    struct arch_specific_insn ainsn;

    /* EYONGGU:
        KPROBE_FLAG_DISABLED : temporarily disabled
        KPROBE_FLAG_OPTIMIZED: optimized
        KPROBE_FLAG_FTRACE:    using ftrace
    */
    u32 flags;
    ...
}
```

All the registered kprobes are stored in the global hash table.

```c
static struct hlist_head kprobe_table[KPROBE_TABLE_SIZE];
```


## Register Kprobe

The generic [register_kprobe()](https://github.com/torvalds/linux/blob/master/kernel/kprobes.c#L1632) is used to register a kprobe.

The function basically does:
1. check if addr can be probed
2. copy probed instruction
3. insert break point

```c
int register_kprobe(struct kprobe *p)
{
    ...

    /* Adjust probe address from symbol */
    p->addr = kprobe_addr(p);

    check_kprobe_address_safe();

    /* EYONGGU: add kprobe into kprobe_table */
    hlist_add_head_rcu(&p->hlist,
            &kprobe_table[hash_ptr(p->addr, KPROBE_HASH_BITS)]);

    /* EYONGGU: call arch_preare_kprobe_ftrace(p) if using ftrace,
       otherwise call arch_prepare_kprobe(p)
    */
    preare_kprobe(p);

    /* EYONGGU: call arch specific arch_arm_kprobe(p) below.
       note, if ftrace kprobe, it call arm_kprobe_ftrace(kp) instead.
    */
    arm_kprobe(p)

    /* Try to optimize normal (non-ftrace-based) kprobe */
    try_to_optimize_kprobe(p);
}
```
The implementation of these steps are arch specific under the generic hood, you can find the [X86 arch specific Kprobe impelemenation](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/kprobes/core.c#L722) here.

**arch_prepare_kprobe()** copies the original instruction and saves it in some instruction cache page, to single step the instruction for post_handle, another int3 is added after the saved original instruction.

**arch_arm_kprobe()** then replaces the orignal instuction with int3 (0xCC) at the address.

```c
int arch_prepare_kprobe(struct kprobe *p)
{
    /* EYONGGU: Check if addr can be probed,
     e.g.
     - if addr in SMP-alternatives range
     - if addr is at an instruction boundary
     */

    /* EYONGGU: allocate memory for original instruction copy */
    p->ainsn.insn = get_insn_slot();

    /* EYONGGU: following function does:
       - __copy_instruction(): copy the original instruction
       - prepare_singlestep(): add an int3 right after the copied instruction for single-step
       - text_poke(): write back the instruction into ROX insn buffer
     */
    arch_copy_kprobe(p);
}

void arch_arm_kprobe(struct kprobe *p)
{
    u8 int3 = INT3_INSN_OPCODE;  /* = 0xCC */

    /*EYONGGU: update the instructions on a live kernel */
    text_poke(addr, opcode, len);

    /*EYONGGU: sync to all CPUs icache */
    text_poke_sync();

    ...
}
```

After Kprobe_register() call, the program would looks like:

![Kprobed code](/assets/kprobe_diag1.png)


## When Kprobe is hit

Since now the int3 instruction is placed at the kprobed address, when the program runs to the address, the kernel is trapped into int3 handler, so let's start to look at the trap exception handling.

NOTE! The Kprobe can be reentrant, which is not coverred here.

```c
DEFINE_IDTENTRY_RAW(exc_int3)
{
    ...

    do_int3(regs);

    ...
}


static bool do_int3(struct pt_regs *regs)
{
    ...
#ifdef CONFIG_KPROBES
    kprobe_int3_handler(regs);
#endif

}
```

kprobe_int3_handler() would be entered twice for one Kprobe, i.e. before and after the single-steped instruction.

```c
int kprobe_int3_handler(struct pt_regs *regs)
{
    ...

    /* EYONGGU: get kprobe at addr (regs->ip - 8) from kprobe_table */
    p = get_kprobe(addr);

    if (p) {
        /* EYONGGU: first hit for the probe. */
        p->pre_handler(p, regs);

        /* EYONGGU:
           ... do something...

           kcb->kprobe_status = KPROBE_HIT_SS;

           but at the end, set ip to the saved instruction.
            regs->ip = (unsigned long)p->ainsn.insn;

           Because of another int3 after the save instruction, it is a single step.
         */
        setup_singlestep(p, regs, kcb, 0);

    } else {
        /* EYONGGU: after single-step */

        /* EYONGGU: restore the regs->ip to next instruction */
        resume_singlestep(p, regs, kcb);

        /* EYONGGU:  run post handler
           p->post_handler(cur, regs, 0);
         */
        kprobe_post_process(p, regs, kcb);
    }
}
```
## Kprobe optimization

As seen in the register_kprobe() above, the last step is to call a function try_to_optimize_kprobe(p), which tries to optimize the kprobe by replacing a breakpoint with a jump instruction. Jump optimization reduces probing overhead drastically.

Note, the optimization is only applicable to normal kprobe, i.e. excluding ftrace kprobe.

```c
static void try_to_optimize_kprobe(struct kprobe *p)
{
    struct kprobe *ap;
    struct optimized_kprobe *op;

    /* EYONGGU: not applicable for ftrace kprobe */
    if (kprobe_ftrace(p))
        return;

    /* EYONGGU: allocate an op, and prepare it, see below */
    ap = alloc_aggr_kprobe(p);

    /* EYONGGU: if not suitable for optimization, fallback to kprobe */
    if (!arch_prepared_optinsn(&op->optinsn)) {
        ...

        goto out;
    }

    init_aggr_kprobe(ap, p);

    /* EYONGGU: enqueues the kprobe to optimizing list
       and kicks kprobe-optimizer workqueue to optimize it
     */
    optimize_kprobe(ap);

    ...
}

int arch_prepare_optimized_kprobe(struct optimized_kprobe *op,
        struct kprobe *__unused)
{
    /* EYONGGU: safety check */
    if (!can_optimize((unsigned long)op->kp.addr))
        return -EILSEQ;

    /* EYONGGU: Preparing detour code */
    buf = kzalloc(MAX_OPTINSN_SIZE, GFP_KERNEL);

    op->optinsn.insn = slot = get_optinsn_slot();

    /* copy arch-dep-instance from template */
    memcpy(buf, optprobe_template_entry, TMPL_END_IDX);

    /* Copy instructions into the out-of-line buffer */
    ret = copy_optimized_instructions(buf + TMPL_END_IDX, op->kp.addr, slot + TMPL_END_IDX);

    ...

    perf_event_text_poke(slot, NULL, 0, buf, len);
    text_poke(slot, buf, len);

    ...
}
```

## How Does a Return Probe (kretprobe) Work

When register_kretprobe() is called, Kprobes establishes a kprobe at the entry to the function.

This kprobe pre_handler is registered with every kretprobe. When probe hits it will set up the return probe.
- saves a copy of the return address
- replaces the return address with the address of a "trampoline."

The trampoline is an arbitrary piece of code -- typically just a nop instruction.  At boot time, Kprobes registers a kprobe at the trampoline

```c
int register_kretprobe(struct kretprobe *rp)
{
    kprobe_on_func_entry(rp->kp.addr, rp->kp.symbol_name, rp->kp.offset);

    rp->kp.pre_handler = pre_handler_kretprobe;
    rp->kp.post_handler = NULL;

    ret = register_kprobe(&rp->kp);
}
```

When the probed function executes its return instruction, control passes to the trampoline and that probe is hit, Kprobes' trampoline handler calls the user-specified return handler associated with the kretprobe, then sets the saved instruction pointer to the saved return address, and that's where execution resumes upon return from the trap.

```c
static int pre_handler_kretprobe(struct kprobe *p, struct pt_regs *regs)
{
    ...

    /* EYONGGU: Within it, the return addr is replaced with trampoline addr */
    arch_prepare_kretprobe(ri, regs);

    ...
}
```
The trampoline function on X86 is defined in [arch/x86/kernel/kprobes/core.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/kprobes/core.c#L1020), which will saves registers and calls trampoline_handler().

## Links
- [Kernel Probes (Kprobes)](https://www.kernel.org/doc/Documentation/kprobes.txt)
- [kprobes: Kprobes jump optimization support](https://lwn.net/Articles/340319/)
