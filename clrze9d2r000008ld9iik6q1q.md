---
title: "Interrupts: The Clockwork Orchestra of the Linux Kernel"
datePublished: Mon Jan 29 2024 20:40:36 GMT+0000 (Coordinated Universal Time)
cuid: clrze9d2r000008ld9iik6q1q
slug: linux-interrupts
tags: linux

---

Imagine your computer as a bustling marketplace. Printers churn out reports, keyboards clatter with ideas, and network cables pulse with information. To orchestrate this constant activity, a hidden conductor keeps everything in rhythm: the **Interrupt Handlers**. Just like a clockwork orchestra, interrupts signal hardware events – a keystroke, a network packet, a disk click – that demand the processor's attention.

In this article, we'll lift the curtain on this fascinating mechanism, unveiling the hidden language that keeps your computer humming. We'll explore how interrupts dance between hardware and software, prioritise competing requests, and ensure a smooth data flow, even in the face of constant demands. Get ready to dive into the clockwork orchestra of the Linux kernel and discover the secret sauce of a responsive computing experience.

# Introduction

You may already have started to think about how the processor can work with the hardware without impacting the overall performance. One way of achieving this is by **polling** the hardware periodically for its status and then acting accordingly. However, this comes with a cost, because this process must happen repeatedly, even if the hardware is inactive. A better approach that can be thought of, is to have a system in place for the hardware to notify the kernel whenever an action is needed. This is the **interrupt** mechanism.

The hardware sends out an electrical signal to the processor to notify the kernel to take respective action. The signals are what are called **interrupts**.

*Note: Hardware devices generate interrupts asynchronously (i.e., not in sync with the processor clock); hence, hardware interrupts are asynchronous. However, unlike interrupts, exceptions occur in sync with the processor clock during the execution of instructions, hence termed synchronous interrupts. Most architectures handle hardware interrupts and exceptions with the same kernel infrastructure.*

It is the role of an **Interrupt Controller** IC to capture the signals issued by the hardware devices, and multiplex them into a single line, directed to the processor. The interrupt controller IC has several registers as well to store information about the various interrupts; some of them are:

1. **Interrupt Request Register (IRR)**: Stores all the interrupts requesting for interrupt services.
    
2. **Interrupt Service Register (ISR**): Stores all the interrupts currently being handled.
    
3. **Interrupt Mark Register (INTM)**: Enables or Masks interrupts from being triggered.
    

The various interrupts are differentiated using numeric values, often called **Interrupt Request (IRQ) Lines**. This value is typically hard-coded for legacy devices like the RTC (Real-Time Clock) or keyboard. For most of the other recent devices, it is dynamically determined. This allows the kernel to recognise the type of interrupt raised and handle it accordingly.

The kernel handles the received interrupt by executing a function called **Interrupt Handler**, or **Interrupt Service Routine (ISR)**. Each device that generates an interrupt requires an interrupt handler to be registered in the kernel, and it is the responsibility of the *device driver* to do so.

# Top Halves and Bottom Halves

Interrupt handlers are also codes written in the C language as the rest of the kernel APIs, but are executed in the interrupt context of the kernel (also called the atomic context), and code here is non-blockable. And because interrupt handlers can start their execution at any time (in response to the received interrupt), care should be taken such that the handler finishes execution quickly and the kernel returns to the process context to resume the execution of the blocked instructions. Hence, the handler has to service the interrupt with minimal delay, but at the same time, take minimum time to execute (contradicting requirements, obviously).

To address this issue, the processing of the interrupts is split into two halves: the **top half** and the **bottom half**. The top half only contains code that is time-critical, such as acknowledgements and hardware resets. The rest of the processing is kept in the bottom half, and its execution is deferred to a convenient time.

To understand this better, let us consider the example of a keyboard interrupt. As soon as a key is pressed on the keyboard, an interrupt is raised, and the kernel executes the top half, hence sending an acknowledgement receipt back to the keyboard, and copying the corresponding character into the input buffer present in the main memory. The character (from the buffer) is then dealt with later, in the bottom half.

# Registering and Freeing a Handler

As discussed earlier, registering interrupt handlers is the responsibility of the device driver. The Linux Kernel provides an API `request_irq()` declared in the header file `<linux/interrupt.h>` for registering an interrupt:

```c
/**
 * request_irq - Add a handler for an interrupt line
 * @irq:	The interrupt line to allocate
 * @handler:	Function to be called when the IRQ occurs.
 *		Primary handler for threaded interrupts
 *		If NULL, the default primary handler is installed
 * @flags:	Handling flags
 * @name:	Name of the device generating this interrupt
 * @dev:	A cookie passed to the handler function
 */
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
```

The return type `irq_handler_t` is defined in the header file `<linux/interrupt.h>` as follows:

```c
typedef irqreturn_t (*irq_handler_t)(int, void *);
```

In the above type definition, the return type `irqreturn_t` is simply an integer that denotes the handler's status. It can take on two special values `IRQ_NONE` and `IRQ_HANDLED`. `IRQ_NONE` denotes that the handler detected an interrupt for which its corresponding device was not the originator. `IRQ_HANDLED` denotes that the handler was appropriately invoked, and the corresponding device was the originator.

The parameter `name` is generally used by *procfs* to display interrupt details in userspace.

The `dev` parameter is very useful when it comes to shared interrupt lines. (*yes, multiple devices can share interrupt lines, except for legacy devices that do not have status registers on the hardware for the handler to check*). It acts like a "cookie", an identification, to help the kernel know which handler needs to be executed or removed from a given interrupt line. `NULL` can be passed into this parameter if the device does not share the interrupt line with other devices. It is a common practice to pass in a pointer to the device structure for the `dev` parameter, as this pointer is unique for every device and may prove useful in the interrupt handler.

The `request_irq()` function returns 0 if the handler registration was successful; else, it returns a non-zero value indicating an error during the registration process. Some of the common errors are `-EBUSY` (indicating that the handler cannot be registered to the specified IRQ line) and `-EIO` (indicating I/O errors).

*Note: The* `request_irq()` *function can be preempted, and, hence, must never be called from within the interrupt context, to prevent the kernel from getting stuck in the interrupt context forever and eventually crashing (the* `request_irq()` *API indirectly depends upon the* `kmalloc()` *API, which can sleep, and hence must never be called when it is unsafe to sleep).*

The Linux kernel also provides an API to unregister the handler when the driver is unloaded from the kernel, and also disable the IRQ line if the unregistered handler is the only one for the IRQ line. This API is the `free_irq()` method defined in `<linux/interrupt.h>` as a prototype:

```c
/*
* @param-1: The IRQ line number.
* @param-2: The pointer to the device structure (the dev param).
*/
extern const void *free_irq(unsigned int, void *);
```

*Note: Calls to the* `free_irq()` *API must be made from the process context!*

# Interrupt Handler Flags

The Linux Kernel provides us with various flags to have fine-grained control over the behaviour of the handlers. Some of the most used flags are:

1. `IRQF_DISABLED`: If this flag is set, then the kernel disables all other interrupts during the execution of this handler. This flag must be used cautiously, only in performance-sensitive interrupts, which are handled quickly.  
    *Note: Disabling interrupts also disables kernel preemption!*
    
2. `IRQF_SAMPLE_RANDOM`: When set, the interrupts generated by this device contribute to the kernel entropy pool (a pool of totally random numbers). The Linux Kernel generates entropy from keyboard timings, mouse movements and Integrated Drive Electronics (IDE) timings, and makes it available for various userspace APIs through `/dev/random` and `/dev/urandom` files. It is better not to set this flag if the timings are predictable or can be influenced by external sources.
    
3. `IRQF_TIMER`: This flag specifies that this handler processes interrupts for the system timer.
    
4. `IRQF_SHARED`: Set for devices that will share the IRQ line with other devices. When the kernel receives an interrupt on this line, it executes each registered handler in order, and quickly exits from the handler if it is not the one.
    

*Note: A particular IRQ line can be shared only when the following conditions are met:*

* *The* `IRQF_SHARED` *flag must be set for each and every handler registered on the line.*
    
* *The* `dev` *parameter of the* `request_irq()` *API must be specified, and unique for each of the handlers; it cannot be set to* `NULL`*.*
    
* *The registered handler must be capable of recognizing whether its device raised the interrupt or not. This requires hardware support (as discussed earlier) and associated logic in the handler itself for verification.*
    

# Interrupt Context

All the interrupt and exception handlers are executed in this special context of the Linux Kernel, the **interrupt context**. This context is not tied to any process running on the kernel. The execution cannot be preempted here and cannot be rescheduled as well! Hence, the APIs that can be invoked from this context are very restricted.

The interrupt context is also time-critical, as it interrupts other instructions. Hence, the code that will be executed here must be as quick and as simple as possible.

Before the release of Linux Kernel v2.6, the interrupt handlers shared the register stack of the process that was interrupted for the execution of the handler. The kernel stack is *two pages* in size, so the handler code must be extremely cautious with the data they store on the stack.

With the release of kernel v2.6, interrupt handlers were given their own stack, *one stack per processor, one page in size*, named the **interrupt stack**. Although the total size of the stack halves, each interrupt gets its own stack, allowing the kernel to reduce the stack size from two pages to one.

# Implementation of Handlers

As soon as the kernel is notified by the processor about an interrupt, it immediately saves the current register values of the processor onto a stack (such that it can be retrieved later), jumps into the interrupt context, looks up the interrupt vector table (a vector table containing addresses of entry points to interrupt handlers), locates the required handler with the help of the IRQ line number, and starts executing the handler.

The kernel invokes the `do_irq()` API, which is defined in `<asm/asm-prototypes.h>` as follows:

```c
/*
* @regs: Contains initial register values previously saved in the 
* assembly entry routine.
*/
asmlinkage void do_irq(struct pt_regs *regs);
```

The above API calculates the IRQ line number, sends out an acknowledgement receipt to the device that raised the interrupt (handled by the `mask_and_ack_8259A()` API), disables interrupt delivery on the line (or also totally disables interrupt delivery on the entire system if `IRQF_DISABLED` is set), and checks if a valid handler is registered on the specified IRQ line.

If all the checks are passed, the `do_irq()` function calls the `handle_irq_event()` function to execute the corresponding handler:

```c
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
	irqreturn_t ret;

	desc->istate &= ~IRQS_PENDING;
	irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
	raw_spin_unlock(&desc->lock);

	ret = handle_irq_event_percpu(desc);

	raw_spin_lock(&desc->lock);
	irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);
	return ret;
}

irqreturn_t handle_irq_event_percpu(struct irq_desc *desc)
{
	irqreturn_t retval;

	retval = __handle_irq_event_percpu(desc);

	add_interrupt_randomness(desc->irq_data.irq);

	if (!irq_settings_no_debug(desc))
		note_interrupt(desc, retval);
	return retval;
}

irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc)
{
	irqreturn_t retval = IRQ_NONE;
	unsigned int irq = desc->irq_data.irq;
	struct irqaction *action;

	record_irq_time(desc);

	for_each_action_of_desc(desc, action) {
		irqreturn_t res;

		/*
		 * If this IRQ would be threaded under force_irqthreads, mark it so.
		 */
		if (irq_settings_can_thread(desc) &&
		    !(action->flags & (IRQF_NO_THREAD | IRQF_PERCPU | IRQF_ONESHOT)))
			lockdep_hardirq_threaded();

		trace_irq_handler_entry(irq, action);
		res = action->handler(irq, action->dev_id);
		trace_irq_handler_exit(irq, action, res);

		if (WARN_ONCE(!irqs_disabled(),"irq %u handler %pS enabled interrupts\n",
			      irq, action->handler))
			local_irq_disable();

		switch (res) {
		case IRQ_WAKE_THREAD:
			/*
			 * Catch drivers which return WAKE_THREAD but
			 * did not set up a thread function
			 */
			if (unlikely(!action->thread_fn)) {
				warn_no_thread(irq, action);
				break;
			}

			__irq_wake_thread(desc, action);
			break;

		default:
			break;
		}

		retval |= res;
	}

	return retval;
}
```

First, in the function `handle_irq_event()`, spin locks are set on the `irq_data` to ensure that it is not accessed concurrently by other processors in a Symmetric Multi-Processor (SMP) setup. Then, the `handle_irq_event_percpu()` function is called, which in turn calls the `__handle_irq_event_percpu()` API and also invokes the `add_interrupt_randomness()` function if the `IRQF_SAMPLE_RANDOM` flag was set for the handler during registration. In the function `__handle_irq_event_percpu()`, each handler registered on the specified IRQ line is executed in order.

The disabled interrupts are turned on (except when the `IRQF_DISABLED` flag is set), the handler is executed, and the interrupts are turned back off before the function returns, as the `do_irq()` function expects interrupts to be disabled when it executes the clean-up code and returns to the process context.

*Note: When the kernel exits the interrupt context, it executes a function that checks if reschedules are pending (basically checks if the* `need_resched` *flag is set). If yes, and if the kernel is returning to userspace,* `schedule()` *API is invoked. If the kernel is returning to kernel space, then the* `preempt_count` *counter is checked to see if there are no more kernel threads to be scheduled, and then the* `schedule()` *API is invoked.*

# Controlling Interrupts

The Linux Kernel provides us with a large family of architecture-dependent APIs to control the state of the interrupts on the machine. The most commonly used Interrupt Control methods are described below:

* `local_irq_disable()` - Disables **local** interrupt delivery (*local* meaning only the *current* processor).
    
* `local_irq_enable()` - Enables **local** interrupt delivery.
    
* `local_irq_save()` - Saves and disables the current state of local interrupt delivery.
    
* `local_irq_restore()` - Restores local interrupt delivery to the previously saved state.
    
* `disable_irq()` - Disables the specified interrupt line **across all the processors** and ensures no handler on the line is still executing.
    
* `disable_irq_nosync()` - Disables the specified interrupt line **across all the processors**.
    
* `enable_irq()` - Enables the specified interrupt line **across all the processors**.
    
* `irqs_disabled()` - Returns a non-zero value if local interrupt delivery is disabled; else, returns zero.
    
* `in_interrupt()` - Returns a non-zero value if the kernel is in interrupt context; else, returns zero (basically returns a non-zero value if either the top half or bottom half is executing).
    
* `in_irq()` - Returns a non-zero value if currently executing a handler; else, returns zero (basically returns a non-zero value if top half is executing; else, returns zero).
    

# Conclusion

So, at the end of this write-up, one question remains: who holds the real power in this clockwork orchestra? Is it the conductor, flawlessly wielding the baton of interrupts? Or are the users, demanding ever-increasing bandwidth, processing power, and instant gratification? Perhaps, like every great performance, the power lies not in a single entity but in the harmonious interplay between all the elements. So, the next time your keystroke registers instantly, your network streams seamlessly, and your coffee brews precisely on time, remember the silent heroes behind the scenes – the Linux Kernel's interrupts, keeping the show running smoother than a maestro's tuxedo.