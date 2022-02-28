---
layout: post
title: "Kernel start"
date: 2022-02-25
categories: jekyll blogging
---

Linux kernel start is also a very interesting area for anyone whole wants to understand everything from bootom.

## main() function of kernel

Main function is arch specific, for X86, the main function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L134)

```c
void main(void)
{
    /* First, copy the boot header into the "zeropage" */
    copy_boot_params();

    /* Initialize the early-boot console */
    console_init();

    init_heap();

    /* Tell the BIOS what CPU mode we intend to run in. */
    set_bios_mode();

    /* Detect memory layout */
    detect_memory();

    keyboard_init();

    go_to_protected_mode();

}
```

## Kernel startup
[start_kernel()](https://github.com/torvalds/linux/blob/master/init/main.c#L927) is a generic function that is very big and does lot of initialization. It's not possible to cover all, instead, only some important/interesting init will be detailed here.

```c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    ...

    local_irq_disable();

    boot_cpu_init();
    page_address_init();

    setup_arch(&command_line);
    setup_boot_config(command_line);
    setup_command_line(command_line);

    setup_nr_cpu_ids();
    setup_per_cpu_areas();
    smp_prepare_boot_cpu();
    boot_cpu_hotplug_init();

    build_all_zonelists(NULL);
    page_alloc_init();

    /* EYONGGU: print out the complete command line */
    pr_notice("Kernel command line: %s\n", saved_command_line);
    parse_early_param();
    parse_args(...);

    sort_main_extable();
    trap_init();
    mm_init();

    ftrace_init();
    early_trace_init();

    sched_init();

    radix_tree_init();
    housekeeping_init();

    workqueue_init_early();

    rcu_init();

    early_irq_init();
    init_IRQ();
    tick_init();
    rcu_init_nohz();
    init_timers();
    hrtimers_init();
    softirq_init();
    timekeeping_init();

    time_init();

    local_irq_init();

    console_init();

    lockdep_init();

    cred_init();
    fork_init();

    cpuset_init();
    cgroup_init();

    /* Do the rest non-__init'ed, we're now alive */
    arch_call_rest_init();

}

/* X86 version */
void __init setup_arch(char **cmdline_p)
{
    ...

    printk(KERN_INFO "Command line: %s\n", boot_command_line);

    early_cpu_init();

    e820__memory_setup();
    parse_setup_data();

    /* EYONGGU: Merge CONFIG_CMDLINE to boot command line or override */

    /* EYONGGU: call do_early_param() to walk through all early_param() */
    parse_early_param();

    ...
}
```





