# OS Project 1

**Register No: 24BCE2743**

**Name: Ahmed Hadi**

**Course name: Operating Systems**

**Course code: BCSE303L**

**Question:**

1. Mini Project: Modifying a Small OS Kernel. Choose a small operating system kernel and justify the choice.
2. Build the unmodified kernel source and boot it inside a virtual machine, with evidence of a successful baseline boot before any modification.
3. Make a substantive, precisely defined modification to real kernel code.
4. Show the change working, with code listings, GDB session logs and screenshots.
5. Explain how the modification was implemented, the obstacles encountered and how they were resolved, and what was learned.

**Solution:**

The kernel modified is Linux 0.11. The scheduler now measures every process it runs, and two new system calls let a user program change the scheduler's time slice and read back the measurements. The work was done on Windows with WSL2 running Ubuntu 24.04, using gcc, gdb and qemu-system-i386 from the Ubuntu repositories.

## 1) Choice of OS and scope of the change

**Why Linux 0.11.** Linux 0.11 is the kernel Linus Torvalds released in December 1991. The complete source is about 12,500 lines of C and assembly, and the scheduler is a single file, `kernel/sched.c`, of 412 lines. That is small enough to read in full within the project timeframe, which is not true of a current Linux kernel with several million lines and a build measured in hours. It is also a real ancestor of today's Linux, so the structures it uses (the task structure, the system call table, `int 0x80`, the timer interrupt driving the scheduler) are the same ideas found in production kernels. It boots in QEMU in a few seconds, and GDB can attach to it through QEMU's debug stub, which the submission requires.

The source was taken from `github.com/yuan-xy/Linux-0.11`, a copy of the original kernel patched only so that a modern gcc can compile it. The project is pinned at commit `49075e7`. Later commits in that repository contain another student's coursework (two extra system calls, `sys_iam` and `sys_whoami`, and a changed boot message), so they are not a clean baseline. At `49075e7` the kernel has the original 72 system calls.

**The modification, stated precisely.**

- Three fields are added to `struct task_struct`: `wait_ticks`, `first_run` and `nswitch`. The scheduler and the timer interrupt fill them in.
- The refill rule in `schedule()` uses a global time slice when one has been set, instead of each task's `priority`.
- System call 72, `settimeslice(n)`, sets that time slice.
- System call 73, `schedtrace(on)`, switches tracing on and off. While tracing is on, `do_exit()` prints one line per process with its turnaround, CPU time, waiting time, response time and number of times scheduled; switching it off prints the averages.

Seven kernel files are changed, with 107 lines added.

## 2) Environment setup and baseline build

The tools were installed with `sudo apt install build-essential gcc-multilib gdb qemu-system-x86 git python3`. `gcc-multilib` is needed because the kernel is compiled as 32-bit code (`-m32`) on a 64-bit host. The source was cloned, checked out at `49075e7`, and the original disk image `hdc-0.11.img` was backed up, since QEMU writes to it on every boot.

One change was made to the Makefile before anything else. Its `start` and `debug` targets ran `qemu-system-x86_64`; these were changed to `qemu-system-i386` (the reason is in section 7). This is the only commit on top of `49075e7`, and it does not touch kernel code. The unmodified kernel was then built and booted.

```bash
$ git log --oneline -2
$ make
$ ls -l Image
$ make start
```

![Figure 1. Baseline source at commit 49075e7 and the unmodified kernel image produced by make.](img/fig1.png)

*Figure 1. Baseline source at commit 49075e7 and the unmodified kernel image produced by make.*

![Figure 2. The unmodified kernel booting in QEMU: root filesystem mounted and shell prompt reached.](img/fig2.png)

*Figure 2. The unmodified kernel booting in QEMU: root filesystem mounted and shell prompt reached.*

![Figure 3. Baseline check: a C program compiled and run with the gcc inside Linux 0.11.](img/fig3.png)

*Figure 3. Baseline check: a C program compiled and run with the gcc inside Linux 0.11.*

The boot sequence in Figure 2 is the kernel loading from the floppy image (`Image`), the setup code switching to protected mode, and `init` mounting the Minix filesystem from `hdc-0.11.img` as root (the free block and inode counts come from its superblock) before starting a shell. This was captured before any kernel source was edited.

## 3) How the original scheduler works

Every task has a `counter`, the number of 10 ms ticks it may still run, and a `priority`, which is 15 by default. `do_timer()` runs on every timer interrupt (HZ is 100) and decrements the running task's counter; when it reaches zero and the task was in user mode, `schedule()` is called. `schedule()` picks the runnable task with the largest counter. If every runnable task has a counter of zero, it refills all of them and tries again.

```c
while (1) {
    c = -1; next = 0; i = NR_TASKS; p = &task[NR_TASKS];
    while (--i) {
        if (!*--p) continue;
        if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
            c = (*p)->counter, next = i;
    }
    if (c) break;
    for (p = &LAST_TASK ; p > &FIRST_TASK ; --p)
        if (*p)
            (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
}
switch_to(next);
```

The refill loop visits every task, including sleeping ones. A task that was blocked on I/O still has part of its counter left, so after the refill it holds half of that plus its priority, more than a CPU-bound task that used everything. When it wakes it is chosen first. This is how 0.11 favours interactive processes without any explicit classification. With all priorities at 15, the scheduler behaves as round robin with a 150 ms slice.

## 4) The modification

Every changed line is marked `[MOD]` in the source.

![Figure 4. Files changed relative to the baseline commit.](img/fig4.png)

*Figure 4. Files changed relative to the baseline commit.*

include/linux/sched.h

```c
    struct tss_struct tss;
/* [MOD] scheduling accounting - added at the END so the hardcoded
 * offsets above (used by system_call.s) do not move */
    long wait_ticks;    /* ticks spent RUNNABLE but not on the CPU */
    long first_run;     /* jiffies when first given the CPU, -1 = never */
    long nswitch;       /* how many times the scheduler picked this task */
};
```

The fields are appended at the end of the structure deliberately. `system_call.s` reads `state`, `counter`, `priority` and `signal` at the fixed byte offsets 0, 4, 8 and 12, and the header carries the comment "these are hardcoded - don't touch". Inserting fields anywhere above them would shift those offsets and break signal delivery and rescheduling on return from every system call. The static initialiser `INIT_TASK` for task 0 gets three matching zeros, and the new globals are declared `extern` here so that `exit.c` can see them.

kernel/sched.c: schedule()

```c
        if (c) break;
        for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
            if (*p)
                (*p)->counter = ((*p)->counter >> 1) +
                    (sched_quantum ? sched_quantum : (*p)->priority);
    }
    /* [MOD] accounting: count the switch and record first dispatch */
    if (task[next] != current) {
        task[next]->nswitch++;
        if (task[next]->first_run < 0)
            task[next]->first_run = jiffies;
    }
    switch_to(next);
```

When `sched_quantum` is zero the refill is exactly the original. When it is set, every task receives the same fixed number of ticks in place of its priority, so the time slice can be varied on the same workload. The accounting runs immediately before `switch_to`, the one point where the next task is known, and only when the CPU actually changes hands.

kernel/sched.c: do\_timer()

```c
    /* [MOD] every tick, charge one tick of waiting to each task that
     * is ready to run but is not the one on the CPU */
    {
        struct task_struct ** t;
        for (t = &LAST_TASK ; t > &FIRST_TASK ; --t)
            if (*t && *t != current && (*t)->state == TASK_RUNNING)
                (*t)->wait_ticks++;
    }
```

In 0.11, `TASK_RUNNING` means runnable, not necessarily on the CPU. A task in that state other than `current` is in the ready queue, so this measures ready-queue waiting time at one-tick resolution. Time spent sleeping on I/O is not counted as waiting.

kernel/sched.c: the two system calls

```c
int sys_settimeslice(long n)
{
    long old = sched_quantum;

    if (n < 0 || n > 100)
        return -EINVAL;
    sched_quantum = n;
    printk("[sched] time slice %d -> %d ticks%s\n\r", old, n,
        n ? "" : " (stock: use priority)");
    return old;
}

int sys_schedtrace(long on)
{
    long n = trace_n;

    if (on) {
        trace_n = trace_turn = trace_wait = trace_resp = trace_sw = 0;
        sched_trace = 1;
        printk("[trace] on (quantum=%d)\n\r", sched_quantum);
        return 0;
    }
    sched_trace = 0;
    printk("[trace] off: %d processes\n\r", n);
    if (n) {
        print_avg("turnaround", trace_turn, n, "ticks");
        print_avg("waiting   ", trace_wait, n, "ticks");
        print_avg("response  ", trace_resp, n, "ticks");
        print_avg("switches  ", trace_sw, n, "");
    }
    return n;
}
```

`settimeslice()` accepts 0 to 100 ticks and returns the previous value, so a program can restore it. `print_avg()` prints one decimal place using integer arithmetic (sum multiplied by 10, divided by the count), because the kernel has no floating point support.

kernel/fork.c: copy\_process()

```c
    p->start_time = jiffies;
    /* [MOD] the child was copied from the parent, so reset its counters */
    p->wait_ticks = 0;
    p->first_run = -1;
    p->nswitch = 0;
```

`copy_process()` begins with `*p = *current`, copying the parent's entire task structure. Without the reset, a child would start with the parent's waiting time and switch count.

kernel/exit.c: do\_exit()

```c
    if (sched_trace) {
        long turn = jiffies - current->start_time;
        long resp = current->first_run < 0 ? 0 :
                current->first_run - current->start_time;
        printk("[exit] pid=%d turn=%d run=%d wait=%d resp=%d sw=%d\n\r",
            current->pid, turn, current->utime + current->stime,
            current->wait_ticks, resp, current->nswitch);
        trace_n++;
        trace_turn += turn;
        trace_wait += current->wait_ticks;
        trace_resp += resp;
        trace_sw += current->nswitch;
    }
```

The report is made in `do_exit()` because it runs in the context of the exiting process, so `current` is the process being measured and its turnaround is final. It is placed before the page tables and files are released.

include/linux/sys.h, kernel/system\_call.s, include/unistd.h

```c
/* sys.h */
sys_setreuid,sys_setregid, sys_settimeslice, sys_schedtrace };

/* system_call.s */
nr_system_calls = 74    # [MOD] was 72: added settimeslice, schedtrace

/* unistd.h */
#define __NR_settimeslice   72  /* [MOD] */
#define __NR_schedtrace     73  /* [MOD] */
```

A user program places the call number in `eax` and the argument in `ebx`, then executes `int 0x80`. `system_call` compares `eax` against `nr_system_calls - 1`, jumping to `bad_sys_call` (which returns -1) if it is too large, and otherwise calls through `sys_call_table[eax]`. Appending the two functions at positions 72 and 73 and raising the limit to 74 makes them reachable.

The kernel was rebuilt with `make clean && make`, and `nm` confirms that both new functions are in the linked kernel.

![Figure 5. Both new system calls present in the compiled kernel.](img/fig5.png)

*Figure 5. Both new system calls present in the compiled kernel.*

## 5) The test program

The test program runs inside Linux 0.11. It sets the time slice from its first argument, switches tracing on, forks five children, waits for all of them, then switches tracing off and restores the original scheduler behaviour. Three children are CPU-bound with 2, 4 and 8 units of work, so there is a short, a medium and a long job. Two are I/O-bound: each writes a 512-byte file and calls `sync()` six times, with one unit of computation between writes.

schedtest.c

```c
#define __LIBRARY__       /* exposes the _syscallN macros in unistd.h */
#define __NR_settimeslice 72
#define __NR_schedtrace   73
#include <unistd.h>
#include <stdio.h>
#include <fcntl.h>

_syscall1(int, settimeslice, int, n)
_syscall1(int, schedtrace, int, on)

long unit = 20000000L;    /* busy-loop iterations per unit of CPU work */

void cpu_job(int units)
{
    volatile long i, x = 0;
    for (i = 0; i < units * unit; i++)
        x += i;
}

void io_job(int rounds, int id)
{
    char buf[512];
    char name[20];
    int fd, r;

    for (r = 0; r < 512; r++)
        buf[r] = 'a' + id;
    sprintf(name, "/tmp/io%d", id);
    for (r = 0; r < rounds; r++) {
        fd = open(name, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        write(fd, buf, 512);
        close(fd);
        sync();             /* blocks until the disk write finishes */
        cpu_job(1);         /* a little CPU between I/O bursts */
    }
    unlink(name);
}

int main(int argc, char **argv)
{
    int q, k, status, old;
    int pid;
    static int cpu_units[3] = { 2, 4, 8 };

    q = (argc > 1) ? atoi(argv[1]) : 15;
    if (argc > 2)
        unit = atoi(argv[2]) * 1000000L;
    old = settimeslice(q);
    if (old < 0) {
        printf("settimeslice(%d) failed: use 0..100\n", q);
        return 1;
    }
    printf("schedtest: quantum=%d (was %d)\n", q, old);
    schedtrace(1);

    for (k = 0; k < 3; k++) {
        pid = fork();
        if (pid == 0) {
            cpu_job(cpu_units[k]);
            _exit(0);
        }
        printf("  cpu child %d: %d units\n", pid, cpu_units[k]);
    }
    for (k = 0; k < 2; k++) {
        pid = fork();
        if (pid == 0) {
            io_job(6, k);
            _exit(0);
        }
        printf("  io  child %d\n", pid);
    }

    while (wait(&status) > 0)
        ;
    k = schedtrace(0);
    settimeslice(0);
    printf("schedtest: done, %d processes traced\n", k);
    return 0;
}
```

Linux 0.11 has no library wrappers for new system calls; the `_syscall1` macro in `unistd.h` generates a function that loads the registers and executes `int 0x80`. Those macros are only defined when `__LIBRARY__` is defined before the include, which is why that line comes first.

## Output:

The program was copied into the root filesystem image on the host (section 7 explains why), then compiled and run inside Linux 0.11 with time slices of 15, 1 and 50 ticks. A slice of 15 is the same length as the default priority, so that run is the baseline behaviour.

```bash
$ python3 ../minixput.py hdc-0.11.img put ../schedtest.c /usr/root/schedtest.c
$ make start
[/usr/root]# gcc -o schedtest schedtest.c
[/usr/root]# ./schedtest 15
[/usr/root]# ./schedtest 1
[/usr/root]# ./schedtest 50
```

![Figure 6. Time slice 15 ticks (same as the default priority).](img/fig6.png)

*Figure 6. Time slice 15 ticks (same as the default priority).*

![Figure 7. Time slice 1 tick.](img/fig7.png)

*Figure 7. Time slice 1 tick.*

![Figure 8. Time slice 50 ticks.](img/fig8.png)

*Figure 8. Time slice 50 ticks.*

All times are in ticks of 10 ms. *turn* is turnaround (exit time minus fork time), *run* is CPU time (`utime + stime`), *wait* is ready-queue time, *resp* is the time from fork to first dispatch and *sw* is the number of times the task was given the CPU.

## 6) GDB session

QEMU was started paused with its GDB stub on port 1234 (`make debug`), and GDB was attached to the kernel's symbol file `tools/system` in a second terminal, running a script of breakpoints. `./schedtest 15 2` (a smaller workload) was then run in the guest.

```bash
$ make debug                                        # terminal 1
$ gdb -q -batch -x ../gdb-demo.gdb tools/system 2>&1 | tee ../gdb-session.log   # terminal 2
```

gdb-demo.gdb

```gdb
set pagination off
target remote :1234
break sys_settimeslice
continue
bt
print n
print sched_quantum
finish
print sched_quantum
print current->pid
delete
break do_exit if sched_trace == 1
continue
print current->pid
print current->utime + current->stime
print current->wait_ticks
print current->first_run - current->start_time
print current->nswitch
print jiffies - current->start_time
delete
break sched.c:156 if task[next] != current
continue
print current->pid
print next
print task[next]->pid
print task[next]->counter
print task[next]->nswitch
delete
detach
```

![Figure 9. GDB session against the running modified kernel.](img/fig9.png)

*Figure 9. GDB session against the running modified kernel.*

| Value | Expression | Result | Meaning |
| --- | --- | --- | --- |
| $1 | n | 15 | argument passed from user space arrived in the kernel |
| $2 | sched\_quantum (before) | 0 | stock behaviour before the call |
| $3 | return value | 0 | old value returned to the program |
| $4 | sched\_quantum (after) | 15 | the call took effect |
| $5 | current->pid | 8 | schedtest itself made the call |
| $6 | current->pid | 12 | first traced process to exit |
| $7 | utime + stime | 7 | CPU ticks used |
| $8 | wait\_ticks | 1 | ticks spent ready but not running |
| $9 | first\_run - start\_time | 1 | response time |
| $10 | nswitch | 1 | dispatched once |
| $11 | jiffies - start\_time | 8 | turnaround, equal to 7 + 1 |
| $12 | current->pid | 12 | task giving up the CPU |
| $13, $14 | next, task[next]->pid | 9, 13 | slot 9, pid 13, is chosen next |
| $15 | task[next]->counter | 15 | its slice after refill equals the set quantum |
| $16 | task[next]->nswitch | 2 | second time it is dispatched (incremented just before switch\_to) |

The backtrace at the first breakpoint shows `system_call` directly beneath `sys_settimeslice`, the `int 0x80` path described in section 4. The values at the second breakpoint are read directly from the new `task_struct` fields of a live process.

## Observations

| Time slice (ticks) | Avg turnaround | Avg waiting | Avg response | Avg switches |
| --- | --- | --- | --- | --- |
| 1 | 230.6 | 172.0 | 30.0 | 41.8 |
| 15 (stock length) | 240.8 | 181.6 | 15.0 | 5.6 |
| 50 | 269.8 | 210.8 | 30.0 | 2.6 |

The number of context switches follows the slice length directly. Moving from 15 ticks to 1 tick multiplied the average switch count by about 7.5 (5.6 to 41.8); moving to 50 ticks more than halved it (2.6).

Average turnaround and waiting time were lowest at 1 tick and highest at 50 ticks. The effect is concentrated in the short job. The longest job's turnaround hardly moves:

| Job | Slice 1 | Slice 15 | Slice 50 |
| --- | --- | --- | --- |
| 2 units (short) | 114 | 142 | 262 |
| 4 units (medium) | 201 | 247 | 254 |
| 8 units (long) | 293 | 296 | 296 |

With a 50-tick slice the short job finished after the medium one (Figure 8: pid 28, 4 units, exits before pid 27, 2 units), and it did not receive the CPU until 60 ticks after it was forked.

For every process in all three runs, run plus wait equals turnaround exactly; for example 22 + 120 = 142 for pid 13 and 92 + 204 = 296 for pid 15 in Figure 6. That holds for the I/O-bound children as well.

## Explanation

A shorter slice approaches ideal round robin. The short job gets a turn every few ticks instead of waiting behind a whole 50-tick slice of each other job, so it finishes much earlier, while the long job, which needs the CPU for the whole run anyway, finishes at almost the same time in every case. A 50-tick slice moves the scheduler towards first-come first-served: whichever job is dispatched first can hold the CPU for half a second, and the order of completion stops reflecting job length. That is why the short job finished second at 50 ticks.

The usual cost of a small slice is the overhead of switching, and the switch count did rise more than sevenfold. Here that overhead did not show up in turnaround, because one switch in 0.11 (a `schedule()` pass over 64 slots and a hardware task switch through the TSS) takes microseconds, which is negligible against a 10 ms tick. The benefit to the short job outweighed it. On a workload with many more processes, or with a slice far smaller than a tick, the balance would be different.

Response times were 15 to 30 ticks on average and did not follow the slice length. The individual values (60, 45, 30, 15 and 0 ticks) depend mostly on fork order: the first child forked waits while the parent keeps running to fork the others, and the last children are dispatched almost immediately once the parent blocks in `wait()`. At this scale response time measures the test's structure more than the scheduler.

Run plus wait equalling turnaround means no process spent measurable time blocked. For the CPU-bound children that is expected. For the I/O-bound children it shows that QEMU's emulated IDE disk completes each `sync()` within the same tick, so they never slept long enough to be counted. A real disk, or a slower emulated one, would be needed to show 0.11's preference for I/O-bound processes.

## 7) Obstacles encountered and how they were resolved

1. **Background updates held the package lock.** The first `apt install` failed with *Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 1119 (unattended-upgr)*. Ubuntu's automatic updater was running on the fresh WSL install. The fix was to wait for the lock to be released and re-run the setup, rather than deleting the lock file, which can corrupt the package database.
2. **The newest commit was not a clean baseline.** The repository's latest commit already contained another student's system calls and a changed boot message. Resolved by checking out `49075e7`, the last commit with the original kernel.
3. **GDB could not attach.** GDB failed with *Remote 'g' packet reply is too long (expected 312 bytes, got 608 bytes)*. The Makefile started `qemu-system-x86_64`, whose stub reports 64-bit registers, while the kernel and its symbols are 32-bit. Changing the Makefile to `qemu-system-i386` fixed it.
4. **`make` ignored header changes.** After editing `sched.h`, a plain `make` rebuilt nothing and produced an identical old kernel image, because the 0.11 Makefiles do not track header dependencies. Every rebuild is done with `make clean && make`, and the result is checked with `nm`.
5. **No way to copy files into the guest.** The usual method is a floppy image read with `mcopy` inside 0.11, but the floppy driver in this build panics under QEMU (*general protection*, then *Kernel panic: free\_page\_tables called with wrong alignment*). Mounting the Minix filesystem on the host needs a `minix` kernel module that WSL does not have. A short Python tool, `minixput.py`, was written to add files to the image directly: it reads the partition table and superblock, allocates an inode and data zones from the bitmaps, writes the data and adds a directory entry. The guest's `fsck` reported the same warnings before and after using it, which are already present in the original image.
6. **The test program did not compile at first.** gcc inside 0.11 reported *parse error before 'settimeslice'*. The `_syscallN` macros are hidden unless `__LIBRARY__` is defined before `unistd.h` is included; adding that line fixed it.
7. **The first workload was too small to measure.** Each child finished in 1 to 4 ticks, so all three slice settings gave the same result. The work unit was raised to 20 million loop iterations, giving runs of a few seconds.
8. **The disk image blocked restoring the changes.** While switching between the baseline and the modified kernel with `git stash`, `git stash pop` failed with *Your local changes to the following files would be overwritten by merge: hdc-0.11.img*. Booting the kernel writes to the disk image, which git tracks. Restoring the image first with `git checkout -- hdc-0.11.img` allowed the changes to be reapplied. The baseline screenshots were then taken from a separate clone of the repository so the modified tree did not need to be switched back.

## 8) What was learned

- The full path of a system call, from the `_syscall1` macro in user space through `int 0x80`, the bounds check in `system_call.s` and `sys_call_table`, to the C function, and how the return value travels back in `eax`. Adding a call needs four coordinated edits, and missing any one of them fails in a different way.
- Kernel data structures can be constrained by assembly code that knows their layout. The position of a field in `task_struct` mattered as much as its type.
- Where measurement code is placed decides what it measures: the timer interrupt for time-based counts, the switch point for dispatch counts, process exit for final totals, and the fork path to reset state that is otherwise inherited.
- The time slice trade-off from scheduling theory appears in real numbers: small slices help short jobs and cost switches, large slices approach FCFS. It also showed that the cost of a switch has to be compared against the tick length before assuming overhead dominates.
- Debugging a kernel with GDB through QEMU: conditional breakpoints, reading structures of a live process and confirming that a value set from user space arrived in kernel memory.

## 9) Video demonstration

Link: <https://drive.google.com/drive/folders/1Wxbd4Wt7L1eO0kORf6_uCOBU1XiHFK7i?usp=sharing>

| Team member | Timestamp | Part explained |
| --- | --- | --- |
| Ahmed Hadi (24BCE2743) | Entire video (sole team member) | All parts: OS choice and baseline boot, data structures and system calls, scheduler change, live test runs, GDB session |
