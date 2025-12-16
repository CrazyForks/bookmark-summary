Title: How a Kernel Bug Froze My Machine: Debugging an Async-profiler Deadlock | QuestDB

URL Source: https://questdb.com/blog/async-profiler-kernel-bug/

Markdown Content:
QuestDB is the open-source time-series database for demanding workloads—from trading floors to mission control. It delivers ultra-low latency, high ingestion throughput, and a multi-tier storage engine. Native support for Parquet and SQL keeps your data portable, AI-ready—no vendor lock-in.

* * *

I've been a Linux user since the late 90s, starting with [Slackware](http://www.slackware.com/) on an underpowered AMD K6. Over the years I've hit plenty of bugs, but the last decade has been remarkably stable - until a kernel bug started freezing my machine whenever I used [async-profiler](https://github.com/async-profiler/async-profiler/).

I'm not a kernel developer, but I found myself poking around kernel source code to understand the problem better and figure out what was going on under the hood.

[The problem](https://questdb.com/blog/async-profiler-kernel-bug/#the-problem)
------------------------------------------------------------------------------

I was about to start an investigation of latency spikes in QuestDB reported by a user. To do that, I wanted to use the async-profiler to capture [CPU heatmaps](https://github.com/async-profiler/async-profiler/blob/3a493bedc4c6463ba19970018c50e0c9d7dcbfda/docs/Heatmap.md).

![Image 1: Screenshot of async-profiler heatmap](https://questdb.com/images/blog/2025-12-11/heatmap.webp)

Async-profiler heatmap example

However, when I tried to attach the profiler, my machine froze completely. It did not respond to any keys, it was impossible to switch to a terminal, it did not respond to SSH. The only way to recover was to hard reboot it. I tried to start QuestDB with the profiler already configured to start at launch - the same result, a frozen machine almost immediately after the launch.

I thought that was weird, this had not happened to me in years. It was already late in the evening, I felt tired anyway so I decided to call it a day. There was a tiny chance I was hallucinating and the problem would go away by itself overnight. A drowning man will clutch at a straw after all.

The next day, I tried to attach the profiler again - same result, frozen machine. Async-profiler integration in QuestDB is a relatively new feature, so I thought there might be a bug in the integration code, perhaps a regression in the recent QuestDB release. So I built an older QuestDB version: The same result, frozen machine. This was puzzling - I positively knew this worked before. How do I know? Because I worked on the [integration code](https://github.com/questdb/questdb/pull/6150) not too long ago, and I tested the hell out of it.

This was a strong hint that the problem was not in QuestDB, but rather in the environment. I've gotten lazy since my Slackware days and I have been using Ubuntu for years now and I realized that I had recently updated Ubuntu to the latest version: 25.10. Could it be that the problem is in the new Ubuntu version?

At this point I started Googling around and I found a [report](https://github.com/async-profiler/async-profiler/issues/1578) created by a fellow performance aficionado, [Francesco Nigro](https://x.com/forked_franz), describing exactly the same problem: machine freeze when using async-profiler. This was the final confirmation I was not hallucinating! Except Francesco is using Fedora, not Ubuntu. However, his Fedora uses the same kernel version as my Ubuntu: 6.17. I booted a machine with an older Ubuntu, started QuestDB and attached the profiler and it worked like a charm. This was yet another indication that the problem was in the system, possibly even in the kernel. This allowed me to narrow down my Google keywords and find this [kernel patch](https://patchew.org/linux/176216460800.2601451.733142302683512228.tip-bot2@tip-bot2/) which talks about the very same problem!

I found it quite interesting: A kernel bug triggered by async-profiler causing machine freezes on recent mainstream distributions. After some poking I found a workaround: Start the profiler with `-e ctimer` option to avoid using the problematic kernel feature. I tried the workaround and indeed, with this option, the profiler worked fine and my machine did not freeze.

Normally I'd move on, but I was curious. What exactly is going on under the hood? Why is it freezing? What is this `ctimer` thing? What exactly is the bug and how does the patch work? So I decided to dig deeper.

Async-profiler is a sampling profiler. It periodically interrupts threads in the profiled application and collects their stack traces. The collected stack traces are then aggregated and visualized in various ways (flame graphs are one of the most popular visualizations). It has multiple ways to interrupt the profiled application, the most common one is using [`perf_events`](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) kernel feature. This is how it works by default on Linux assuming [kernel paranoia settings](https://www.kernel.org/doc/html/latest/admin-guide/perf-security.html) allow it.

### [perf_events Under the Hood](https://questdb.com/blog/async-profiler-kernel-bug/#perf_events-under-the-hood)

The `perf_events` subsystem is a powerful Linux kernel feature for performance monitoring. For CPU profiling, async-profiler uses a software event called `cpu-clock`, which is driven by high-resolution timers ([hrtimers](https://docs.kernel.org/timers/hrtimers.html)) in the kernel.

Here's the sequence of events during profiling:

1.   **Setup**: For each thread in the profiled application, async-profiler opens a [`perf_event`](https://github.com/async-profiler/async-profiler/blob/c6c2fc1497ab06ff357146ed91fd27c0ce5640ba/src/perfEvents_linux.cpp#L606) file descriptor configured to generate a signal after a specified interval of CPU time (e.g., 10ms).
2.   **Arming the event**: The profiler calls [`ioctl(fd, PERF_EVENT_IOC_REFRESH, 1)`](https://github.com/async-profiler/async-profiler/blob/c6c2fc1497ab06ff357146ed91fd27c0ce5640ba/src/perfEvents_linux.cpp#L645) to arm the event for exactly one sample. This is a one-shot mechanism, combined with the `RESET` at the end of the handler. The goal is to measure application CPU time only and exclude the signal's handler own overhead.
3.   **Timer fires**: When the configured CPU time elapses, the kernel's [hrtimer fires](https://github.com/torvalds/linux/blob/2424e146bee00ddb4d4f79d3224f54634ca8d2bc/kernel/time/hrtimer.c#L1761) and delivers a signal to the target thread.
4.   **Signal handler**: Async-profiler's signal handler captures the stack trace and [records the sample](https://github.com/async-profiler/async-profiler/blob/c6c2fc1497ab06ff357146ed91fd27c0ce5640ba/src/perfEvents_linux.cpp#L704). At the end of the handler, it resets the counter and [re-arms the event](https://github.com/async-profiler/async-profiler/blob/c6c2fc1497ab06ff357146ed91fd27c0ce5640ba/src/perfEvents_linux.cpp#L709-L710) for the next sample:

ioctl(fd, PERF_EVENT_IOC_RESET, 0);

ioctl(fd, PERF_EVENT_IOC_REFRESH, 1);

This cycle repeats for the duration of the profiling session, creating a stream of stack trace samples that are later aggregated into flame graphs or heatmaps.

[The Kernel Bug](https://questdb.com/blog/async-profiler-kernel-bug/#the-kernel-bug)
------------------------------------------------------------------------------------

The kernel bug that caused my machine to freeze was introduced by commit [18dbcbfabfff ("perf: Fix the POLL_HUP delivery breakage")](https://github.com/torvalds/linux/commit/18dbcbfabfff). Ironically, this commit was fixing a different bug, but it introduced a deadlock in the cpu-clock event handling.

Here's what happens in the buggy kernel when the `PERF_EVENT_IOC_REFRESH(1)` counter reaches zero:

1.   [hrtimer](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11824) fires for cpu-clock event - [`perf_swevent_hrtimer()`](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11749) is called (inside hrtimer interrupt context)
2.   [`perf_swevent_hrtimer()` calls](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11769)[`__perf_event_overflow()`](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L10300) - this processes the counter overflow
3.   `__perf_event_overflow()`[decides to stop](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L10333) the event (counter reached 0 after `PERF_EVENT_IOC_REFRESH(1)`) - calls [`cpu_clock_event_stop()`](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11916)
4.   [`cpu_clock_event_stop()`](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11861) calls [`perf_swevent_cancel_hrtimer()`](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11800) - this calls [`hrtimer_cancel()`](https://github.com/torvalds/linux/blob/18dbcbfabfffc4a5d3ea10290c5ad27f22b0d240/kernel/events/core.c#L11813) to cancel the timer
5.   **DEADLOCK**: [`hrtimer_cancel()`](https://github.com/torvalds/linux/blob/2424e146bee00ddb4d4f79d3224f54634ca8d2bc/kernel/time/hrtimer.c#L1483-L1494) waits for the hrtimer callback to complete - but we ARE inside the hrtimer callback! The system hangs forever waiting for itself

The function `hrtimer_cancel()` is a blocking call - it spins waiting for any active callback to finish.

int hrtimer_cancel(struct hrtimer *timer)

{

int ret;

do {

ret = hrtimer_try_to_cancel(timer);

if (ret < 0)

hrtimer_cancel_wait_running(timer);

} while (ret < 0);

return ret;

}

When called from inside that same callback, it waits forever. Since this happens in interrupt context with interrupts disabled on the CPU, that CPU becomes completely unresponsive. When this happens on multiple CPUs (which it does, since each thread has its own `perf_event`), the entire system freezes.

Click to see the deadlock visualized

[The Fix](https://questdb.com/blog/async-profiler-kernel-bug/#the-fix)
----------------------------------------------------------------------

The [kernel patch](https://github.com/torvalds/linux/commit/eb3182ef0405ff2f6668fd3e5ff9883f60ce8801) fixes this deadlock with two changes:

1.   Replace `hrtimer_cancel()` with `hrtimer_try_to_cancel()`

- hrtimer_cancel(&hwc->hrtimer);

+ hrtimer_try_to_cancel(&hwc->hrtimer);

[`hrtimer_try_to_cancel()`](https://github.com/torvalds/linux/blob/2424e146bee00ddb4d4f79d3224f54634ca8d2bc/kernel/time/hrtimer.c#L1347) is non-blocking - it returns immediately with:

*   `0` if the timer was not active
*   `1` if the timer was successfully cancelled
*   `-1` if the timer callback is currently running

Unlike `hrtimer_cancel()`, it doesn't spin waiting for the callback to finish. So when called from within the callback itself, it simply returns `-1` and continues.

1.   Use `PERF_HES_STOPPED` flag as a deferred stop signal

The stop function now sets a flag:

static void cpu_clock_event_stop(struct perf_event *event, int flags)

{

+ event->hw.state = PERF_HES_STOPPED;

perf_swevent_cancel_hrtimer(event);

...

}

And the hrtimer callback checks this flag:

static enum hrtimer_restart perf_swevent_hrtimer(struct hrtimer *hrtimer)

{

- if (event->state != PERF_EVENT_STATE_ACTIVE)

+ if (event->state != PERF_EVENT_STATE_ACTIVE ||

+ event->hw.state & PERF_HES_STOPPED)

return HRTIMER_NORESTART;

### [How It Works Together](https://questdb.com/blog/async-profiler-kernel-bug/#how-it-works-together)

When `cpu_clock_event_stop()` is called from within the hrtimer callback:

1.   `PERF_HES_STOPPED` flag is set
2.   `hrtimer_try_to_cancel()` returns `-1` (callback running) - but doesn't block
3.   Execution returns up the call stack back to `perf_swevent_hrtimer()`
4.   `perf_swevent_hrtimer()` completes and returns `HRTIMER_NORESTART` (because `__perf_event_overflow()` returned `1`, indicating the event should stop)
5.   The hrtimer subsystem sees `HRTIMER_NORESTART` and [doesn't reschedule the timer](https://github.com/torvalds/linux/blob/2424e146bee00ddb4d4f79d3224f54634ca8d2bc/kernel/time/hrtimer.c#L1776-L1778)

When `cpu_clock_event_stop()` is called from outside the callback (normal case):

1.   `PERF_HES_STOPPED` flag is set
2.   `hrtimer_try_to_cancel()` returns `0` or `1` - timer is cancelled immediately
3.   If by chance the callback fires before cancellation completes, it sees `PERF_HES_STOPPED` and returns `HRTIMER_NORESTART`

The `PERF_HES_STOPPED` flag acts as a safety net to make sure the timer stops regardless of the race between setting the flag and the timer firing.

### [Debugging a kernel](https://questdb.com/blog/async-profiler-kernel-bug/#debugging-a-kernel)

The explanation above is my understanding of the kernel bug and the fix based on reading the kernel source code. I am a hacker, I like to tinker. A theoretical understanding is one thing, but I wanted to see it in action. But how do you even debug a kernel? I'm not a kernel developer, but I decided to try. Here is how I did it.

My intuition was to use [QEMU](https://www.qemu.org/) since it allows one to emulate or virtualize a full machine. QEMU also has a built-in [GDB](https://www.sourceware.org/gdb/) server that allows you to [connect GDB to the emulated machine](https://qemu-project.gitlab.io/qemu/system/gdb.html).

### [Setting up QEMU with Ubuntu](https://questdb.com/blog/async-profiler-kernel-bug/#setting-up-qemu-with-ubuntu)

I downloaded an Ubuntu 25.10 ISO image and created a new empty VM disk image:

$ qemu-img create -f qcow2 ubuntu-25.10.qcow2 20G

Then I launched QEMU to install Ubuntu:

$ qemu-system-x86_64 \

-enable-kvm \

-m 4096 \

-smp 4 \

-drive file=ubuntu-25.10.qcow2,if=virtio \

-cdrom ubuntu-25.10-desktop-amd64.iso \

-boot d \

-vga qxl

The second command boots the VM from the ISO image and allows me to install Ubuntu on the VM disk image. I went through the installation process as usual. I probably could have used a server edition or a prebuilt image, but at this point I was already in unknown territory, so I wanted to make other things as simple as possible.

![Image 2: QEMU screen showing Ubuntu installation](https://questdb.com/images/blog/2025-12-11/ubuntu.webp)

Ubuntu installation in QEMU

Once the installation was complete, I rebooted the VM:

$ qemu-system-x86_64 \

-enable-kvm \

-m 4096 \

-smp 4 \

-drive file=ubuntu-25.10.qcow2,if=virtio \

-netdev user,id=net0,hostfwd=tcp::9000-:9000 \

-device virtio-net-pci,netdev=net0 \

-monitor tcp:127.0.0.1:55555,server,nowait \

-s

and downloaded, unpacked and started QuestDB:

$ curl -L https://github.com/questdb/questdb/releases/download/9.2.2/questdb-9.2.2-rt-linux-x86-64.tar.gz -o questdb.tar.gz

$ tar -xzvf questdb.tar.gz

$ cd questdb-9.2.2-rt-linux-x86-64

$ ./bin/questdb start

This was meant to validate that QuestDB works in the VM at all. Firefox was already installed in the Ubuntu desktop edition, so I just opened `http://localhost:9000` in Firefox and verified QuestDB web console was up and running.

![Image 3: QuestDB web console running in Ubuntu in QEMU](https://questdb.com/images/blog/2025-12-11/console.webp)

QuestDB web console in QEMU

The next step was to stop QuestDB and start it with a profiler attached:

$ ./bin/questdb stop

$ ./bin/questdb start -p

At this point, I expected the virtual machine to freeze. However, it didn't. It was responsive as if nothing bad had happened. That was a bummer. I wanted to see the deadlock in action! I thought that perhaps QEMU is in a way shielding the virtual machine from the bug. But then I realized that the default Ubuntu uses paranoia settings that prevent `perf_events` from working properly and async-profiler [falls back to using `ctimer`](https://github.com/async-profiler/async-profiler/blob/c6c2fc1497ab06ff357146ed91fd27c0ce5640ba/src/profiler.cpp#L995-L998) when `perf_events` are restricted. The kernel bug specifically lives in the `perf_events` hrtimer code path, so we must force async-profiler to use that path to trigger the bug.

To fix this, I changed the paranoia settings:

$ echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid

After this, I restarted QuestDB with the profiler again:

$ ./bin/questdb stop

$ ./bin/questdb start -p

And this time, the virtual machine froze as expected! Success! I was able to reproduce the problem in QEMU!

### [Attaching GDB to QEMU](https://questdb.com/blog/async-profiler-kernel-bug/#attaching-gdb-to-qemu)

Now that I was able to reproduce the problem in QEMU, I wanted to attach GDB to the emulated machine to see the deadlock in action.

Let's start `GDB` on the host machine and connect it to QEMU's built-in GDB server:

$ gdb

GNU gdb (Ubuntu 16.3-1ubuntu2) 16.3

[...]

(gdb) target remote :1234

Remote debugging using :1234

warning: No executable has been specified and target does not support

determining executable automatically. Try using the "file" command.

0xffffffff82739398 in ?? ()

(gdb) info threads

Id Target Id Frame

* 1 Thread 1.1 (CPU#0 [running]) 0xffffffff82739398 in ?? ()

2 Thread 1.2 (CPU#1 [running]) 0xffffffff82739398 in ?? ()

3 Thread 1.3 (CPU#2 [running]) 0xffffffff827614d3 in ?? ()

4 Thread 1.4 (CPU#3 [running]) 0xffffffff82739398 in ?? ()

(gdb) thread apply all bt

> Side note: We just casually attached a debugger to a live kernel! How cool is that?

We can see 4 threads corresponding to the 4 CPUs in the VM. The `bt` command shows the stack traces of all threads, but there is not much useful information since we don't have the kernel symbols loaded in GDB. Let's fix this. I am lazy again and take advantage of running exactly the same kernel version as the host machine so I can use the host's kernel image and symbol files.

On the host machine, we need to add repositories with debug symbols and install the debug symbols for the running kernel:

echo "deb http://ddebs.ubuntu.com questing main restricted universe multiverse" | sudo tee /etc/apt/sources.list.d/ddebs.list

echo "deb http://ddebs.ubuntu.com questing-updates main restricted universe multiverse" | sudo tee -a /etc/apt/sources.list.d/ddebs.list

echo "deb http://ddebs.ubuntu.com questing-proposed main restricted universe multiverse" | sudo tee -a /etc/apt/sources.list.d/ddebs.list

sudo apt install ubuntu-dbgsym-keyring

sudo apt update

sudo apt install linux-image-$(uname -r)-dbgsym

With the debug symbols installed, I started GDB again and loaded the kernel image and symbols:

$ gdb /usr/lib/debug/boot/vmlinux-$(uname -r)

GNU gdb (Ubuntu 16.3-1ubuntu2) 16.3

[...]

gdb) target remote :1234

Remote debugging using :1234

0xffffffff9e9614d3 in ?? ()

[...]

(gdb) info threads

Id Target Id Frame

* 1 Thread 1.1 (CPU#0 [running]) 0xffffffff9e9614d3 in ?? ()

2 Thread 1.2 (CPU#1 [running]) 0xffffffff9e939398 in ?? ()

3 Thread 1.3 (CPU#2 [running]) 0xffffffff9e9614d3 in ?? ()

4 Thread 1.4 (CPU#3 [running]) 0xffffffff9e9614d3 in ?? ()

(gdb) quit

and symbols were still NOT resolved! I had to capitulate and ask a LLM for help. After a bit of brainstorming, we realized that the kernel is compiled with [KASLR](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html) enabled, so the kernel is loaded at a random address at each boot. The simplest way to fix this is to disable KASLR, I could not care less about security in my test VM. To disable KASLR, I edited the GRUB configuration, added the `nokaslr` parameter, updated GRUB and rebooted the VM:

$ vim /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nokaslr"

$ sudo update-grub

$ sudo reboot

Then I set the paranoia settings again, started QuestDB with the profiler and attached GDB again. This time, the symbols were resolved correctly!

$ gdb /usr/lib/debug/boot/vmlinux-$(uname -r)

GNU gdb (Ubuntu 16.3-1ubuntu2) 16.3

[...]

(gdb) target remote :1234

[...]

(gdb) info threads

Id Target Id Frame

* 1 Thread 1.1 (CPU#0 [running]) csd_lock_wait (csd=0xffff88813bd3a460) at /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c:351

2 Thread 1.2 (CPU#1 [running]) csd_lock_wait (csd=0xffff88813bd3b520) at /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c:351

3 Thread 1.3 (CPU#2 [running]) hrtimer_try_to_cancel (timer=0xffff88802343d028) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

4 Thread 1.4 (CPU#3 [running]) hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

This looks much better! We can see that the first 2 threads are stuck in `csd_lock_wait()` function, presumably waiting for locks held by the other CPUs and threads 3 and 4 are in `hrtimer_try_to_cancel()`.

The threads 3 and 4 are the interesting ones since they execute a function related to the kernel bug we are investigating. Let's switch to thread 4 and see its stack trace:

(gdb) thread 4

[Switching to thread 4 (Thread 1.4)]

#0 hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

1359 in /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c

(gdb) bt

#0 hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

#1 hrtimer_cancel (timer=timer@entry=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1488

#2 0xffffffff81700605 in perf_swevent_cancel_hrtimer (event=<optimized out>) at /build/linux-8YMEfB/linux-6.17.0/kernel/events/core.c:11818

#3 perf_swevent_cancel_hrtimer (event=0xffff888023439f80) at /build/linux-8YMEfB/linux-6.17.0/kernel/events/core.c:11805

#4 cpu_clock_event_stop (event=0xffff888023439f80, flags=0) at /build/linux-8YMEfB/linux-6.17.0/kernel/events/core.c:11868

#5 0xffffffff81715488 in __perf_event_overflow (event=event@entry=0xffff888023439f80, throttle=throttle@entry=1, data=data@entry=0xffffc90002cd7cc0, regs=0xffffc90002cd7f48) at /build/linux-8YMEfB/linux-6.17.0/kernel/events/core.c:10338

#6 0xffffffff81716eaf in perf_swevent_hrtimer (hrtimer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/events/core.c:11774

#7 0xffffffff81538a03 in __run_hrtimer (cpu_base=<optimized out>, base=<optimized out>, timer=0xffff88802343a0e8, now=0xffffc90002cd7e58, flags=<optimized out>) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1761

#8 __hrtimer_run_queues (cpu_base=cpu_base@entry=0xffff88813bda1400, now=now@entry=48514890563, flags=flags@entry=2, active_mask=active_mask@entry=15) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1825

#9 0xffffffff8153995d in hrtimer_interrupt (dev=<optimized out>) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1887

#10 0xffffffff813c4ac8 in local_apic_timer_interrupt () at /build/linux-8YMEfB/linux-6.17.0/arch/x86/kernel/apic/apic.c:1039

#11 __sysvec_apic_timer_interrupt (regs=regs@entry=0xffffc90002cd7f48) at /build/linux-8YMEfB/linux-6.17.0/arch/x86/kernel/apic/apic.c:1056

#12 0xffffffff82621724 in instr_sysvec_apic_timer_interrupt (regs=0xffffc90002cd7f48) at /build/linux-8YMEfB/linux-6.17.0/arch/x86/kernel/apic/apic.c:1050

#13 sysvec_apic_timer_interrupt (regs=0xffffc90002cd7f48) at /build/linux-8YMEfB/linux-6.17.0/arch/x86/kernel/apic/apic.c:1050

#14 0xffffffff81000f0b in asm_sysvec_apic_timer_interrupt () at /build/linux-8YMEfB/linux-6.17.0/arch/x86/include/asm/idtentry.h:574

#15 0x00007b478171db80 in ?? ()

#16 0x0000000000000001 in ?? ()

#17 0x0000000000000000 in ?? ()

We can see the exact sequence of function calls leading to the deadlock: `hrtimer_try_to_cancel()` called from `cpu_clock_event_stop()`, called from `__perf_event_overflow()`, called from `perf_swevent_hrtimer()`. This matches our understanding of the bug perfectly! This is the infinite loop in `hrtimer_cancel()` that causes the deadlock.

[Forensics and Playing God](https://questdb.com/blog/async-profiler-kernel-bug/#forensics-and-playing-god)
----------------------------------------------------------------------------------------------------------

Okay, I have to admit that seeing a kernel stack trace is already somewhat satisfying, but we have a live (well, half-dead) kernel under a debugger. Let's have some fun. I want to touch the deadlock and understand why it took down the whole machine, and see if we can perform a miracle and bring it back to life.

#### Confirming the suspect

We know `hrtimer_cancel` is waiting for a callback to finish. But which callback? The stack trace says `perf_swevent_cancel_hrtimer`, but let's verify the hrtimer struct in memory actually points to the function we blame.

I switched to the stuck thread (Thread 4 in my case) and looked at frame #0:

(gdb) thread 4

[Switching to thread 4 (Thread 1.4)]

#0 hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

1359 in /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c

(gdb) frame 0

#0 hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

1359 in /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c

(gdb) print *timer

$1 = {node = {node = {__rb_parent_color = 18446612682661667048, rb_right = 0x0, rb_left = 0x0}, expires = 48514879474}, _softexpires = 48514879474, function = 0xffffffff81716dd0 <perf_swevent_hrtimer>, base = 0xffff88813bda1440, state = 0 '\000', is_rel = 0 '\000', is_soft = 0 '\000', is_hard = 1 '\001'}

Let me explain these GDB commands: `frame 0` selects the innermost stack frame - the function currently executing. In a backtrace, frame 0 is the current function, frame 1 is its caller, frame 2 is the caller's caller, and so on. By selecting frame 0, I can inspect local variables and parameters in `hrtimer_try_to_cancel()`.

The `print *timer` command dereferences the `timer` pointer and displays the contents of the `struct hrtimer`:

struct hrtimer {

struct timerqueue_node node;

ktime_t _softexpires;

enum hrtimer_restart (*function)(struct hrtimer *);

struct hrtimer_clock_base *base;

u8 state;

u8 is_rel;

u8 is_soft;

u8 is_hard;

};

The key field here is `function` - a pointer to a callback function that takes a `struct hrtimer *` and returns `enum hrtimer_restart`. This callback is invoked when the timer fires. GDB shows it points to `0xffffffff81716dd0` and helpfully resolves this address to `perf_swevent_hrtimer`. Since we're currently _inside_`perf_swevent_hrtimer` (look at frame #6 in our backtrace above), this confirms the self-deadlock: the timer is trying to cancel itself while its own callback is still running!

#### The Mystery of the "Other" CPUs

One question remained: If CPUs 3 and 4 are deadlocked in a loop, why did the entire machine freeze? Why couldn't I just SSH in and kill the process? The answer lies in those other threads we saw earlier, stuck in `csd_lock_wait`:

(gdb) thread 1

[Switching to thread 1 (Thread 1.1)]

#0 csd_lock_wait (csd=0xffff88813bd3a460) at /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c:351

warning: 351 /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c: No such file or directory

`CSD` stands for Call Function Single Data. In Linux, when one CPU wants another CPU to do something (like flush a TLB or stop a perf_event), it sends an IPI (Inter-Processor Interrupt). If the target CPU is busy with interrupts disabled (which is exactly the case for our deadlocked CPUs 3 and 4), it never responds.

The sender (CPU 0) sits there [spinning](https://github.com/torvalds/linux/blob/fe90f3967bdb3e13f133e5f44025e15f943a99c5/include/asm-generic/barrier.h#L260-L274), waiting for the other [CPU to say "Done!"](https://github.com/torvalds/linux/blob/83e6384374bac8a9da3411fae7f24376a7dbd2a3/kernel/smp.c#L547). Eventually, all CPUs end up waiting for the stuck CPUs and the entire system grinds to a halt.

#### Performing a Kernel Resurrection

This is the part where the real black magic starts. We know the kernel is stuck in this loop in `hrtimer_cancel`:

do {

ret = hrtimer_try_to_cancel(timer);

} while (ret < 0);

As long as `hrtimer_try_to_cancel` returns `-1` (which it does, because the callback is running), the loop continues forever.

But we have GDB. We can change reality.

If we force the function to return `0` (meaning "timer not active"), the loop should break, `cpu_clock_event_stop` should finish, and the kernel should unfreeze. It might crash 1 millisecond later because we left the timer in an inconsistent state, but perhaps it's worth trying.

First, let's double-check we are in the innermost frame, inside `hrtimer_try_to_cancel`:

(gdb) thread 4

[Switching to thread 4 (Thread 1.4)]

#0 hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

warning: 1359 /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c: No such file or directory

(gdb) frame 0

#0 hrtimer_try_to_cancel (timer=0xffff88802343a0e8) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

1359 in /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c

Use the GDB `finish` command to let the function run to completion and pause right when it returns to the caller:

(gdb) finish

We are now sitting at line 1490, right at the check `if (ret < 0)`.

int hrtimer_cancel(struct hrtimer *timer)

{

int ret;

do {

ret = hrtimer_try_to_cancel(timer);

if (ret < 0)

hrtimer_cancel_wait_running(timer);

} while (ret < 0);

return ret;

}

On x86_64, integer return values are passed in the `%rax` register. Since `hrtimer_try_to_cancel` returns an `int` (32-bit), we can use `$eax` (the lower 32 bits of `%rax`):

(gdb) print $eax

$2 = -1

Exactly as expected. `-1` means the timer callback is running, so the loop will continue. But since the CPU is paused, we can overwrite this value. We can lie to the kernel and tell it the timer was successfully cancelled (return code 1) or inactive (return code 0). I chose 0.

(gdb) set $eax = 0

(gdb) print $eax

$3 = 0

I crossed my fingers and unpaused the VM:

(gdb) continue

Continuing.

And it did nothing. The VM was still frozen. Let's see what is going on:

(gdb) info threads

Id Target Id Frame

1 Thread 1.1 (CPU#0 [running]) csd_lock_wait (csd=0xffff88813bd3a460) at /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c:351

2 Thread 1.2 (CPU#1 [running]) csd_lock_wait (csd=0xffff88813bd3b520) at /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c:351

3 Thread 1.3 (CPU#2 [running]) hrtimer_try_to_cancel (timer=0xffff88802343d028) at /build/linux-8YMEfB/linux-6.17.0/kernel/time/hrtimer.c:1359

* 4 Thread 1.4 (CPU#3 [running]) csd_lock_wait (csd=0xffff88813bd3b560) at /build/linux-8YMEfB/linux-6.17.0/kernel/smp.c:351

Now, thread 4 is also stuck in `csd_lock_wait`, just like threads 1 and 2. We managed to escape from the infinite loop in thread 4, but thread 3 is still stuck in `hrtimer_try_to_cancel`.

We could try the same trick on thread 3, but would this be enough to unfreeze the entire system? For starters, we tricked the kernel into thinking the timer was inactive, but in reality it is still active. This is very thin ice to skate on - we might have just created more problems for ourselves. And more importantly, even if the kernel could escape the deadlock, the profiler would immediately try to re-arm the timer again, leading us back into the same deadlock.

So I decided to give up on the resurrection attempt. The kernel was stuck, but at least I understood the problem now and I was pretty happy with my newly acquired kernel debugging skills.

[Conclusion](https://questdb.com/blog/async-profiler-kernel-bug/#conclusion)
----------------------------------------------------------------------------

While I couldn't perform a miracle and resurrect the frozen kernel, I walked away with a much deeper understanding of the machinery behind Linux `perf_events` and hrtimers. I learned how to set up QEMU for kernel debugging, how to attach GDB to a live kernel, and how to inspect kernel data structures in memory.

For QuestDB users, the takeaway is simple: if you are on a kernel version 6.17, use the `-e ctimer` flag when profiling. It bypasses the buggy `perf_events` hrtimer path entirely. Or just wait for either the kernel fix to land in your distro or the next QuestDB release, which will [include](https://github.com/questdb/questdb/pull/6473) an async-profiler version that [works around this issue](https://github.com/async-profiler/async-profiler/pull/1599).

As for me, I’m going back to my code. The next time my machine freezes, I might ju