---
title: "CFS Scheduler in the Linux Kernel"
description: "Unveiling the Secret Sauce of Linux Performance!"
date: 2024-01-08
tags: ["linux", "linux-kernel", "scheduling-algorithms"]
author: "Vinit Puranik"
toc: true
---

The Linux kernel is the backbone of countless devices, servers, and systems worldwide. But have you ever wondered why Linux systems feel so snappy, even with multiple applications running? The answer lies in a hidden gem called the **Completely Fair Scheduler**. This unsung hero works tirelessly behind the scenes, juggling tasks with surgical precision, ensuring your system feels smooth and responsive.

In this article, we'll lift the hood and examine CFS' inner workings, uncovering its techniques for maximizing CPU utilization, prioritizing interactive tasks, and preventing resource hogs from ruining the party.

_Note: The CFS Scheduler is the default scheduling class for processes spun up in the system. Linux supports multiple scheduling classes, each having priorities, and any process can be assigned to any scheduler._

# History

Since the first version of the Linux Kernel, up until the 2.4 series, the scheduler implemented was a simple scheduler (like the ones in other UNIX systems). It had an easy-to-understand design language, but performed very poorly at scale.

In the 2.5 series, the kernel received a new scheduler algorithm, namely the **O(1) scheduler** (named on the fact that it took constant time to calculate timeslices and per-processor runqueues), which aimed at addressing the issues with the previous scheduler.

The O(1) scheduler was performing up to the mark and scaled very efficiently, but it was sub-par when it came to desktop systems, which involved interactive processes. Later, many new schedulers were tested to overcome this issue, the best amongst them being the **Rotating Staircase Deadline** scheduler, which introduced the concept of **_Fair Scheduling._** This concept eventually made its way into the kernel as a replacement for the O(1) scheduler in v2.6.23, and was named the **Completely Fair Scheduler (CFS)**.

# The concept of Fair Scheduling

The concept of Fair Scheduling, in general, is very simple: Processes should be scheduled as if the system had a **perfectly multitasking** processor. All the competing processes must get a fair share of the CPU time. CFS implements this using the concept of `vruntime` (virtual runtime).

Fairness can be quantified by a **Fairness Metric (F),** which is simply the time taken by the first process to complete divided by the time taken by the second process to complete. In the case of a perfectly multitasking processor, the value of F becomes 1, indicating that both the processes got an equal share of the CPU time, hence being "fairly scheduled".

Each process accumulates `vruntime` at the same rate in proportion to the real-time (the proportion constant is usually the weight of the process). When a scheduling decision occurs, CFS schedules the process with the least `vruntime` next.

But, you may ask, how does the scheduler decide when to stop the current running process and schedule the next one in the runqueue? CFS utilizes a periodic timer interrupt, giving CFS a chance to wake up and determine if the current process has reached its run (which also means that CFS can make scheduling decisions only at fixed time intervals, but that shouldn't be an issue as the interrupt goes on frequently).

(Note that even if a process has a timeslice that is not a perfect multiple of the timer interrupt interval, CFS tracks the `vruntime` of each process precisely, and in the long term, it ends up approximately giving ideal CPU shares to all processes.)

Also, if CFS switches between processes too frequently, even though the fairness is increased, the performance is compromised (due to context switch overheads), and if CFS switches less often, even though the performance is increased, the near-term fairness is lost.

To strike a balance between these two extremes, CFS provides various control parameters, like `sched_latency`, which is the target latency used to determine how long the process must run before scheduling a switch.

But if there are too many processes in the `TASK_RUNNING` state in the runqueue, you might think that the scheduler ends up providing minuscule timeslices to each of the processes (which will result in too many context switches), but that's not the case. The control parameter `min_granularity` imposes a floor on the timeslice that can be assigned to each process, ensuring that there is a ceiling on the switch costs that the processor has to incur.

This may seem contradictory to the idea of CFS being completely "fair", as the parameter floors the CPU proportions. But in the common case of having only a few processes in the runqueue, CFS ends up being completely fair!

CFS also allows processes to have their priorities, allowing them to allot certain processes a higher CPU time share than the rest. This is implemented using a classic UNIX mechanism known as the **nice** level of a process. The nice values are nothing but process weights exported into the system's user space. Nice value controls the percentage of CPU usage allowed for processes as proportions. In Linux kernel v2.6.34, this was defined in an array `prio_to_weight` as follows:

```c
static const int prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

As we can observe in the above code block, nice values range from -20 to +19; lower nice values imply a higher weight for the process, and vice-versa. (Higher nice valued processes tend to be background and processor-intensive.) Using this mapping, we can now calculate dynamically the timeslice of each process with the the following mathematical expression:

<p align="center">
  <img src="/0xcode/images/timeslice.jpg" />
</p>

This formula and the given mapping show that the proportion of processor time allotted to each process solely depends upon the relative difference in niceness between it and the other processes in the runqueue.

And for `vruntime`, the following formula is used for the ith iteration:

<p align="center">
  <img src="/0xcode/images/vruntime.jpg" />
</p>

Where, `weight0` is the default weight of the process.

# Advantages over other Schedulers

The CFS scheduler has multiple advantages over other UNIX schedulers, such as:

1. There will be inefficient switching behaviours if nice values of processes are directly mapped onto timeslices, which would then require absolute values of timeslices for each nice value. This would result in having different processor shares for different nice values having the same relative difference, meaning "nicing down a process by one" has widely different results based on the starting nice value.
2. UNIX schedulers try to reduce the latency of interactive processes by providing freshly woken-up processes a priority boost and allowing them to run immediately, even if their timeslice has expired. Processes may exploit this advantage and try to obtain an unfair share of the CPU time. CFS tackles this issue by assigning time to processes in terms of CPU usage proportions.

# Implementation in the Linux Kernel

Every process can have a scheduler class to handle its scheduling; the corresponding scheduler entity structure `sched_entity` is pointed to in the process descriptor structure `task_struct`, as a member-variable `se`.

The accounting of the virtual runtime `vruntime` consumed by every process is handled in the `update_curr()` function defined in `/kernel/sched/fair.c`.

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
    // Currently running entity in the CFS runqueue
	struct sched_entity *curr = cfs_rq->curr;
    // Clock of the CPU runqueue
	u64 now = rq_clock_task(rq_of(cfs_rq));

	u64 delta_exec;

	if (unlikely(!curr))
		return;

    // calculating the difference in the start-of-process and current
    // timestamps, giving the total time consumed by the process.
	delta_exec = now - curr->exec_start;
	if (unlikely((s64)delta_exec <= 0))
		return;

	curr->exec_start = now;

    // Updating the kernel statistics
	if (schedstat_enabled()) {
		struct sched_statistics *stats;

		stats = __schedstats_from_se(curr);
		__schedstat_set(stats->exec_max,
				max(delta_exec, stats->exec_max));
	}

	curr->sum_exec_runtime += delta_exec;
	schedstat_add(cfs_rq->exec_clock, delta_exec);

    // Updating the vruntime of the current process
	curr->vruntime += calc_delta_fair(delta_exec, curr);
	update_deadline(cfs_rq, curr);
	update_min_vruntime(cfs_rq);

    // If the current entity is a task, then update the time stats.
	if (entity_is_task(curr)) {
		struct task_struct *curtask = task_of(curr);

		trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
		cgroup_account_cputime(curtask, delta_exec);
		account_group_exec_runtime(curtask, delta_exec);
	}

	account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

As discussed, CFS schedules the process that has the least `vruntime`. CFS uses a **Red-Black** tree (which is a self-balancing binary search tree) to manage the list of processes in the runqueue and efficiently find the process with the least `vruntime`. So, the task for the CFS scheduler is to basically pick the leftmost node in the Red-Black tree of processes (the key for the nodes being their respective `vruntime`s). This is implemented in the `__pick_eevdf()` function defined in `/kernel/sched/fair.c`.

```c
/*
 * Earliest Eligible Virtual Deadline First
 *
 * In order to provide latency guarantees for different request sizes
 * EEVDF selects the best runnable task from two criteria:
 *
 *  1) the task must be eligible (must be owed service)
 *
 *  2) from those tasks that meet 1), we select the one
 *     with the earliest virtual deadline.
 *
 * We can do this in O(log n) time due to an augmented RB-tree. The
 * tree keeps the entries sorted on service, but also functions as a
 * heap based on the deadline by keeping:
 *
 *  se->min_deadline = min(se->deadline, se->{left,right}->min_deadline)
 *
 * Which allows an EDF like search on (sub)trees.
 */
static struct sched_entity *__pick_eevdf(struct cfs_rq *cfs_rq)
{
	struct rb_node *node = cfs_rq->tasks_timeline.rb_root.rb_node;
	struct sched_entity *curr = cfs_rq->curr;
	struct sched_entity *best = NULL;
	struct sched_entity *best_left = NULL;

	if (curr && (!curr->on_rq || !entity_eligible(cfs_rq, curr)))
		curr = NULL;
	best = curr;

	/*
	 * Once selected, run a task until it either becomes non-eligible or
	 * until it gets a new slice. See the HACK in set_next_entity().
	 */
	if (sched_feat(RUN_TO_PARITY) && curr && curr->vlag == curr->deadline)
		return curr;

	while (node) {
		struct sched_entity *se = __node_2_se(node);

		/*
		 * If this entity is not eligible, try the left subtree.
		 */
		if (!entity_eligible(cfs_rq, se)) {
			node = node->rb_left;
			continue;
		}

		/*
		 * Now we heap search eligible trees for the best (min_)deadline
		 */
		if (!best || deadline_gt(deadline, best, se))
			best = se;

		/*
		 * Every se in a left branch is eligible, keep track of the
		 * branch with the best min_deadline
		 */
		if (node->rb_left) {
			struct sched_entity *left = __node_2_se(node->rb_left);

			if (!best_left || deadline_gt(min_deadline, best_left, left))
				best_left = left;

			/*
			 * min_deadline is in the left branch. rb_left and all
			 * descendants are eligible, so immediately switch to the second
			 * loop.
			 */
			if (left->min_deadline == se->min_deadline)
				break;
		}

		/* min_deadline is at this node, no need to look right */
		if (se->deadline == se->min_deadline)
			break;

		/* else min_deadline is in the right branch. */
		node = node->rb_right;
	}

	/*
	 * We ran into an eligible node which is itself the best.
	 * (Or nr_running == 0 and both are NULL)
	 */
	if (!best_left || (s64)(best_left->min_deadline - best->deadline) > 0)
		return best;

	/*
	 * Now best_left and all of its children are eligible, and we are just
	 * looking for deadline == min_deadline
	 */
	node = &best_left->run_node;
	while (node) {
		struct sched_entity *se = __node_2_se(node);

		/* min_deadline is the current node */
		if (se->deadline == se->min_deadline)
			return se;

		/* min_deadline is in the left branch */
		if (node->rb_left &&
		    __node_2_se(node->rb_left)->min_deadline == se->min_deadline) {
			node = node->rb_left;
			continue;
		}

		/* else min_deadline is in the right branch */
		node = node->rb_right;
	}
	return NULL;
}
```

The CFS scheduler adds processes to the RB tree and caches the process in the leftmost node in the `__enqueue_entity()` function defined in `/kernel/sched/fair.c`.

```c
// The enqueue_entity() function updates the runtime and other stats,
// and then invokes the following function.

/*
 * Enqueue an entity into the rb-tree:
 */
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	avg_vruntime_add(cfs_rq, se);
	se->min_deadline = se->deadline;
	rb_add_augmented_cached(&se->run_node, &cfs_rq->tasks_timeline,
				__entity_less, &min_deadline_cb);
}
```

Last but not least of code blocks for the article, here is the function where CFS removes a process from the RB-tree, also present in `/kernel/sched/fair.c`:

```c
// The dequeue_entity() function updates the runtime and other stats,
// and then invokes the following function.

static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	rb_erase_augmented_cached(&se->run_node, &cfs_rq->tasks_timeline,
				  &min_deadline_cb);
	avg_vruntime_sub(cfs_rq, se);
}
```

# Conclusion

The CFS scheduler is a testament to the power of elegant engineering. It took a complex problem - sharing a single processor among countless competing processes - and crafted a fair and efficient solution. CFS has revolutionized the Linux experience and extended its influence to other operating systems and even real-time scheduling domains. As computing demands evolve, CFS remains a bedrock of fairness and performance, a living reminder that a well-designed algorithm can truly conquer chaos.

_PS: This is my first-ever technical blog... So please do leave comments on how I can improve my content and writing, and keep supporting as a community!!_
