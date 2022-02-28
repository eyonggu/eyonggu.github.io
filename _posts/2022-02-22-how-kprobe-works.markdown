---
layout: post
title: "How Kprobe works"
date: 2022-02-22
categories: jekyll blogging
tags: [kprobe]
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

The implementation of these steps are arch specific under the generic hood, below we will only look at X86 implementation.

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

    /* EYONGGU: call arch specific arch_arm_kprobe(p) below */
    arm_kprobe(p)

    /* Try to optimize kprobe */
    try_to_optimize_kprobe(p);
}
```
Now, let's look at the [X86 arch specific impelemenation](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/kprobes/core.c#L722)

On X86, Kprobe saves the original instruction at the address, replaces it with int3, ands add another int3 after the original instruction for single-step.

Some articles say that int1 debug exception after the original insturction is used to execute post_handle, but I didn't find it for X86 implementation, instead I see another int3 is inserted after the instruction.
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

After the Kprobe is registered, the program would looks like:
![Kprobed code](/assets/kprobe_diag1.png)


## When Kprobe is hit

On X86, when the kprobed addr is hit, the kernel is trapped into int3 handler, so we start with the trap exception handling.

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



