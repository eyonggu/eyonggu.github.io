---
layout: post
title: "Workqueue introduction"
date: 2022-05-04
categories: jekyll blogging
published: false
---

## Global work queue data

```c
/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], cpu_worker_pools);

/* PL: hash of all unbound pools keyed by pool->attrs */
static DEFINE_HASHTABLE(unbound_pool_hash, UNBOUND_POOL_HASH_ORDER);

```

## Kworker thread creation

```c
void __init workqueue_init(void)
{
    for_each_possible_cpu(cpu) {
        for_each_cpu_worker_pool(pool, cpu) {
            pool->node = cpu_to_node(cpu);
        }
    }

    list_for_each_entry(wq, &workqueues, list) {
        wq_update_unbound_numa(wq, smp_processor_id(), true);
        WARN(init_rescuer(wq),
                "workqueue: failed to create early rescuer for %s",
                wq->name);
    }

    /* create the initial workers

       EYONGGU: Each CPU has two per-CPU pool, so create two kworker:
         - kworker/<cpu>:0
         - kworker/<cpu>:0H
    */
    for_each_online_cpu(cpu) {
        for_each_cpu_worker_pool(pool, cpu) {
            pool->flags &= ~POOL_DISASSOCIATED;
            BUG_ON(!create_worker(pool));
        }
    }

    /* EYONGGU: create kworker for unbound workqueue */
    hash_for_each(unbound_pool_hash, bkt, pool, hash_node)
        BUG_ON(!create_worker(pool));

    wq_online = true;
    wq_watchdog_init();
}
```

## Work queue API

Work queue API definition can be find in ./include/linux/workqueue.h, and implementation in ./kernel/workqueue.c.

The work is defined by the **work_struct** structure. macro:s are provided to initialize the work.

```c
INIT_WORK( work, func );
INIT_DELAYED_WORK( work, func );
INIT_DELAYED_WORK_DEFERRABLE( work, func );
```

With the work structure intialized, the next step is to enqueuing the work into a work queue. There are several APIs to do it in different ways.

```c
/* queue the work to the CPU on which it was submitted,
   but it can also processed by another CPU */
int queue_work( struct workqueue_struct *wq, struct work_struct *work );

/* queue work on specific cpu */
int queue_work_on( int cpu, struct workqueue_struct *wq, struct work_struct *work );

int queue_delayed_work( struct workqueue_struct *wq,
            struct delayed_work *dwork, unsigned long delay );

int queue_delayed_work_on( int cpu, struct workqueue_struct *wq,
            struct delayed_work *dwork, unsigned long delay );
```

There are also APIs to use global work queue.
```c
int schedule_work( struct work_struct *work );
int schedule_work_on( int cpu, struct work_struct *work );

int scheduled_delayed_work( struct delayed_work *dwork, unsigned long delay );
int scheduled_delayed_work_on(
        int cpu, struct delayed_work *dwork, unsigned long delay );

```

There are also a number of helper functions that you can use to flush or cancel work on work queues.

## Links


