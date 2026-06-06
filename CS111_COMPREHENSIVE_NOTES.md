# CS111 Comprehensive Catch-Up Notes

These notes are a standalone study document for the lectures currently in the `Lectures` folder. They are organized chronologically, but the explanations are expanded beyond the slide headings so you can study without constantly jumping back to the PDFs.

## Site Navigation

- [Slide Text Reference](CS111_SLIDE_TEXT_REFERENCE.html): slide-by-slide extracted lecture text coverage backstop.
- [Quick Final Study Guide](CS111_FINAL_STUDY_GUIDE.html): final-review version with traps, drills, and problem patterns.
- [Exam C/C++ Cheatsheet](CS111_EXAM_C_CHEATSHEET.html): C/C++ syntax and exam-code reference.
- [Midterm Practice Guide](CS111_MIDTERM_PRACTICE_GUIDE.html): midterm practice guide, still useful for concurrency and memory foundations.
- [Compact C Refresher](C_FOR_CS111_CHEATSHEET.html): compact C refresher.

Lecture 11 is byte-for-byte identical to Lecture 10 in this folder, so it is treated as a duplicate.

Lecture 14 is byte-for-byte identical to Lecture 13 in this folder, so it is treated as a duplicate.

Lecture 17 is byte-for-byte identical to Lecture 16 in this folder, so it is treated as a duplicate.

Lecture 21 is byte-for-byte identical to Lecture 20 in this folder, so it is treated as a duplicate.

Lecture 24 repeats the crash-recovery material from Lecture 23, so it is treated as a duplicate.

Lecture 25 `(1)` is a duplicate copy of Lecture 25.

## Table of Contents

1. Lecture 1: Introduction to Operating Systems
2. Lecture 2: Threads, Processes, and System Calls
3. Lecture 3: Dispatching and Context Switching
4. Lecture 4: Concurrency
5. Lecture 5: Locks and Condition Variables
6. Lecture 6: Lock Implementation
7. Lecture 7: Deadlock
8. Lecture 8: Scheduling
9. Lecture 9: Linkers and Process Memory Layout
10. Lecture 10: Dynamic Storage Management
11. Lecture 12: Trust and Operating Systems
12. Lecture 13: Virtual Memory, Base and Bound, and Segmentation
13. Lecture 15: Paging and Page Translation
14. Lecture 16: Demand Paging and Page Replacement
15. Lecture 18: Magnetic Disks and I/O Devices
16. Lecture 19: File Systems
17. Lecture 20: BSD Inodes, Block Cache, Free Space, and Disk Scheduling
18. Lecture 22: Directories and Links
19. Lecture 23: File System Crash Recovery
20. Lecture 25: Truth, Trust, and Technology
21. Lecture 26: Flash Memory
22. Lecture 27: Virtual Machines
23. Lecture 28: Course Review
24. Final Review Map

---

# Lecture 1: Introduction to Operating Systems

## What Is an Operating System?

An operating system is the software layer that sits between applications and hardware. It is hard to define with one sentence because operating systems evolved historically as solutions to many different practical problems.

The core idea is:

- Hardware is powerful but awkward, unsafe, and shared.
- Applications want convenient abstractions.
- Users want efficient and reliable use of the machine.
- The operating system mediates between these needs.

The most privileged part of the operating system is the kernel. The kernel controls hardware, manages resources, enforces protection, and provides system calls to applications.

## Why Study OS History?

Operating systems make more sense if you see why each feature appeared. Major OS ideas were not invented all at once. They emerged because computer systems changed:

- Early computers were expensive and humans were cheap.
- Later, hardware became cheaper and human time became expensive.
- Then computers became networked, personal, mobile, and embedded everywhere.

Each phase changed what the OS needed to optimize.

## 1940s and Early Systems

Early computers had little or no operating system. One user worked directly at the console. Programs were often loaded manually. The first OS-like pieces were shared libraries or routines for common operations like input/output.

Why even that minimal support mattered:

- Convenience: programmers did not want to rewrite device I/O each time.
- Efficiency: shared routines reduced duplicated effort.
- Standardization: common interfaces made programs easier to write.

## Batch Systems

As machines became valuable, the goal became keeping the hardware busy. A human manually loading each job wasted machine time.

A batch monitor allowed many jobs to be submitted together. The system would run one job, then automatically move to the next.

Batch systems improved utilization, but they created new issues:

- Users had little interactivity.
- Debugging was slow.
- A bad job could waste machine time.
- The machine still needed mechanisms to isolate jobs from one another.

## Multiprogramming and the Kernel

Multiprogramming means keeping multiple programs in memory and switching among them. If one program waits for I/O, another can use the CPU.

This requires the OS to manage:

- CPU scheduling.
- Memory allocation.
- I/O devices.
- Protection between programs.
- Program state when switching.

This is where the kernel becomes central. The kernel is trusted code that can run privileged instructions and control hardware resources.

## Timesharing

Timesharing made computers interactive. Instead of running one batch job at a time, the computer rapidly switched among users or programs, giving each the illusion of having its own machine.

Timesharing required:

- Preemption: the OS can interrupt a running program.
- Scheduling: the OS decides what runs next.
- Protection: one user cannot corrupt another user's work.
- Accounting/fairness: users share the system.

This idea is still visible in modern systems: your laptop runs many programs by rapidly switching CPU time among them.

## Networking and Modern Systems

Networking expanded the OS role. Machines became connected, so operating systems needed to handle:

- Communication.
- Remote access.
- Security.
- Distributed resources.
- Failure across machine boundaries.

Modern systems include laptops, phones, servers, embedded devices, virtual machines, containers, and cloud systems. The OS still provides the same core services, but the scale and threat model are larger.

## What Operating Systems Do Today

An OS commonly provides:

- Process and thread abstractions.
- CPU scheduling.
- Virtual memory and address spaces.
- File systems.
- Device drivers.
- Networking.
- Security and protection.
- Interprocess communication.
- Resource accounting.

## The OS as Abstraction Provider

Applications usually do not talk directly to raw hardware. Instead, they use OS abstractions:

- A process abstracts a running program.
- A thread abstracts a stream of execution.
- A file abstracts persistent storage.
- A socket abstracts network communication.
- Virtual memory abstracts physical memory.

Abstractions make programming easier, but they also hide complex resource management.

## The OS as Resource Manager

The OS must divide finite hardware resources among competing programs:

- CPU time.
- Memory.
- Disk space.
- Network bandwidth.
- Device access.

Good resource management requires policy decisions. For example, CPU scheduling is not just "run a program"; it asks which program should run, for how long, and according to what goal.

## Core Takeaways

- OS features emerged to solve concrete historical problems.
- The kernel is the trusted privileged core.
- The OS provides abstractions and manages resources.
- The course is broadly organized around CPU, memory, storage/filesystems, and trust/protection.

## Exam Questions To Do

Do these after Lecture 1 if you want a quick orientation check:

- `Spr2023MidtermSolution.pdf` Problem 1(e): trust by substitution from OS backup.
- `Spr2024MidtermSolution.pdf` Problem 3(a): OS/page tables as trust by substitution.

There are not many pure "what is an OS?" midterm questions. Lecture 1 mostly gives context for later process, memory, file-system, and trust questions.

---

# Lecture 2: Threads, Processes, and System Calls

## Problem Decomposition

A fundamental idea in computer science is decomposing a hard problem into smaller pieces. Operating systems use this idea heavily.

For execution, the OS decomposes activity into:

- Processes.
- Threads.
- System calls.
- Kernel-managed resources.

This lets the OS reason about what is running, what resources it owns, and how to switch between activities.

## Concurrency Motivation

Modern systems do many things at once:

- Multiple applications are open.
- A browser has many tabs.
- A server handles many clients.
- A program may perform computation while waiting for disk or network I/O.

Concurrency is the ability to have multiple activities in progress at the same time. It does not always mean true simultaneous execution. On one core, the OS can interleave execution. On multiple cores, activities can literally run at the same time.

## Thread Definition

A thread is the smallest unit of execution managed by the OS or runtime.

A thread has:

- A program counter: where it is executing.
- Registers: current CPU state.
- A stack: function calls and local variables.
- A scheduling state: running, ready, or blocked.

Threads execute instructions. If a process has multiple threads, those threads share much of the process state, especially the address space.

## Process Definition

A process is a running program plus its execution environment.

A process usually includes:

- One or more threads.
- An address space.
- Open file descriptors.
- Memory mappings.
- Credentials/security identity.
- Other OS-managed resources.

The process is the resource ownership boundary. Threads are execution streams inside that boundary.

## Thread vs Process

Think of the distinction this way:

- A process owns resources.
- A thread runs code.

Threads in the same process can share global variables, heap memory, and open files. Processes are more isolated from each other.

This difference matters:

- Creating a thread is often cheaper than creating a process.
- Threads communicate easily through shared memory.
- Threads can also corrupt each other's shared state if synchronization is wrong.
- Processes provide stronger isolation.

## Hardware Executes Instructions Concurrently

Modern hardware supports concurrency at multiple levels:

- A single CPU core executes one instruction stream at a time, but can be interrupted.
- Multiple cores can execute multiple threads simultaneously.
- Devices such as disks and network cards operate concurrently with the CPU.

The OS coordinates all of this.

## Execution State

Execution state is the information needed to stop a thread and later resume it correctly.

Important pieces include:

- Program counter.
- Stack pointer.
- General-purpose registers.
- Floating-point/vector registers if needed.
- Memory mappings through the process.
- Kernel bookkeeping state.

Without saving execution state, the OS could not switch between threads.

## System Calls

A system call is a controlled entry into the kernel.

Applications cannot directly perform privileged operations such as:

- Creating processes.
- Reading raw disk blocks.
- Configuring page tables.
- Sending data through hardware devices.
- Killing other processes.

Instead, they request services from the kernel using system calls.

Common examples:

- `fork`
- `exec`
- `wait`
- `open`
- `read`
- `write`
- `close`
- `mmap`

System calls are important because they are the boundary between untrusted application code and trusted kernel code.

## Linux Process Creation: `fork`

In Unix-like systems, `fork` creates a new process by duplicating the calling process.

After `fork`:

- There are two processes: parent and child.
- Both continue executing after the `fork` call.
- The parent receives the child's process ID.
- The child receives `0`.
- If `fork` fails, the parent receives an error value.

Typical pattern:

```cpp
int child_pid_or_zero = fork();
if (child_pid_or_zero == 0) {
    // Child process
    execvp("ls", argv);
    // Not reached if exec succeeds
} else {
    // Parent process
    waitpid(child_pid_or_zero, &status, options);
}
```

## `exec`

`exec` replaces the current process image with a new program.

Important points:

- It does not create a new process.
- It keeps the same process identity.
- It replaces the code, globals, heap, and stack with the new program image.
- If `exec` succeeds, it does not return to the old program.

This is why Unix process creation often looks like:

1. `fork` to create a child.
2. In the child, `exec` to run a different program.
3. In the parent, `wait` to observe child completion.

## `wait`

`wait` or `waitpid` lets a parent process wait for a child process to finish.

This matters because:

- The parent may need the child's exit status.
- The OS must clean up child process bookkeeping.
- Without waiting, terminated children can remain as zombies until collected.

## Why Copy Then Overwrite?

Unix's `fork` followed by `exec` may seem odd: why copy a process only to replace it?

Reasons this model is useful:

- The child can adjust its environment before `exec`.
- The child can redirect file descriptors.
- The parent can continue independently.
- It gives a simple composition model for shells.

Modern systems optimize `fork` using copy-on-write so the full address space does not need to be physically copied immediately.

## Windows Process Creation

Windows uses a different model, commonly `CreateProcess`, where process creation and program loading are more directly combined. The course contrast is mainly to show that Unix's `fork`/`exec` split is a design choice, not the only possible interface.

## Thread Creation

A process usually starts with one thread. Additional threads can be created with OS or language runtime APIs.

Examples:

- Linux has low-level `clone`.
- C++ has `std::thread`.
- Other languages provide their own thread APIs or async runtimes.

Multiple threads in one process share the process address space, which makes communication convenient but synchronization necessary.

## Core Takeaways

- A process is a resource container.
- A thread is an execution stream.
- System calls are the safe entry point into the kernel.
- Unix process creation commonly uses `fork`, `exec`, and `wait`.
- Threads are powerful because they share state, but shared state creates concurrency bugs.

## Exam Questions To Do

Do these after Lecture 2:

- `Spr2022MidtermSolution.pdf` Problem 2: why a shell should create processes, not threads, for commands.
- `Spr2023MidtermSolution.pdf` Problem 1(b): threads in one process can run on different cores.
- `Spr2023MidtermSolution.pdf` Problem 3(a)-(b): global variables with threads vs after `fork`.
- `CS111-Practice-Midterm-Solutions.pdf` Problem 2: `duet`, using `fork`, `execvp`, pipes, and file descriptors.
- `CS111-Practice-Midterm-Solutions.pdf` Problem 4(b): `thyme`, timing a child process with `fork`, `execvp`, and `waitpid`.

---

# Lecture 3: Dispatching and Context Switching

## Review of Execution Abstractions

From Lecture 2:

- A thread is the smallest unit of execution.
- A process contains one or more threads and their shared execution environment.
- Processes can be created with `fork`.
- A child can replace its program with `exec`.
- A parent can wait for child completion with `wait`.

Lecture 3 asks: once threads exist, how does the OS actually run them?

## Cores

A CPU core executes instructions. A machine with multiple cores can truly execute multiple threads at the same time.

However:

- Usually there are more runnable threads than cores.
- Some threads block waiting for I/O or synchronization.
- The OS must decide which threads get cores.

This requires dispatching and scheduling.

## Dispatching vs Scheduling

These terms are related but different.

Scheduling is the policy decision:

- Which thread should run?
- How long should it run?
- What goal are we optimizing?

Dispatching is the mechanism:

- Save the old thread's state.
- Load the new thread's state.
- Transfer control to the new thread.

Lecture 3 focuses mostly on dispatching. Lecture 8 focuses on scheduling policy.

## Thread States

A simple OS model has three important thread states:

- Running: currently executing on a core.
- Ready: able to run, but waiting for CPU time.
- Blocked: unable to run until some event happens.

Examples:

- A thread doing computation is running.
- A thread waiting in the ready queue is ready.
- A thread waiting for disk I/O, a lock, or a condition variable is blocked.

The difference between ready and blocked is critical:

- Ready threads could run immediately if given a core.
- Blocked threads cannot make progress even if a core is free.

## Process Control Block and Thread Control Block

The OS stores bookkeeping information for processes and threads.

A process control block may include:

- Process ID.
- Address-space information.
- Open files.
- Credentials.
- Child/parent process relationships.

A thread control block may include:

- Register state.
- Stack pointer.
- Program counter.
- Scheduling state.
- Links for ready or blocked queues.

These structures are how the OS remembers what each process and thread is doing.

## Dispatcher

The dispatcher is the mechanism that switches the CPU from one thread to another.

Conceptually:

1. Save the current thread's CPU state.
2. Mark the current thread ready or blocked, depending on why it stopped.
3. Pick another ready thread.
4. Restore that thread's saved CPU state.
5. Return to user mode or kernel mode at the point where that thread should resume.

The thread experiences this as if it simply paused and later continued.

## Context Switch

A context switch is the act of switching from one thread's execution context to another's.

The OS must save and restore enough state that each thread behaves as if it had its own CPU.

State to save may include:

- Program counter.
- Stack pointer.
- General registers.
- Processor status bits.
- Address-space pointer/page table base if switching processes.

Context switches are not free. They cost CPU time and can disrupt cache locality.

## What Causes the Dispatcher to Run?

The dispatcher runs when the OS regains control and decides another thread may need to run.

Common triggers:

- A running thread blocks on I/O.
- A running thread waits for a lock or condition variable.
- A thread exits.
- A timer interrupt fires.
- A higher-priority thread becomes ready.
- A new thread is created.

The timer interrupt is especially important for preemptive multitasking. It lets the OS regain control even if a program does not voluntarily yield.

## Blocking

When a thread blocks, it cannot continue until an event occurs.

Examples:

- Waiting for data from disk.
- Waiting for network input.
- Waiting for a lock.
- Waiting for a condition variable.
- Waiting for a child process.

Blocking is useful because the OS can run other threads instead of wasting CPU time.

## Ready Queues

Ready threads are commonly stored in one or more ready queues. The scheduler chooses from these queues.

For now, think of a ready queue as a list of threads that could run. Later scheduling policies change how this list is organized.

## Core Takeaways

- Threads move among running, ready, and blocked states.
- Dispatching is the mechanism for switching threads.
- Scheduling is the policy for choosing threads.
- Context switches save and restore execution state.
- Timer interrupts let the OS preempt running code.

## Exam Questions To Do

Do these after Lecture 3:

- `Spr2024MidtermSolution.pdf` Problem 2: identify which scheduling algorithms are preemptive.
- `Spr2024MidtermSolution.pdf` Problem 3(c): per-core ready queues vs one shared ready queue.
- `Spr2022MidtermSolution.pdf` Problem 1(c): BSD scheduler preemption.
- `CS111-Practice-Midterm-Solutions.pdf` Problem 4(a): why open write ends keep a pipe reader from seeing EOF.

---

# Lecture 4: Concurrency

## Independent Threads

Independent threads do not share mutable state.

Properties:

- One thread cannot affect another.
- One thread cannot be affected by another.
- Execution is deterministic given the same input.
- Scheduling order does not matter.

Independent threads are easier to reason about because interleavings do not change correctness.

Unfortunately, many useful programs need threads to cooperate through shared state.

## Cooperating Threads

Cooperating threads share state or coordinate with each other.

Examples:

- Producer and consumer sharing a buffer.
- Web server threads sharing a cache.
- Bank transactions updating account balances.
- A GUI thread and worker thread sharing task state.

Cooperation is useful but dangerous.

Problems:

- Behavior may be nondeterministic.
- Bugs may be hard to reproduce.
- Correctness can depend on precise timing.
- The scheduler can expose unexpected interleavings.

## Why Permit Thread Cooperation?

Cooperating threads are useful because they enable:

- Sharing results.
- Parallel computation.
- Efficient communication.
- Responsiveness.
- Better resource utilization.

The goal is not to avoid cooperation entirely. The goal is to coordinate cooperation safely.

## When Does Order Matter?

If two operations access the same shared state and at least one writes it, order can matter.

Example:

```cpp
x = x + 1;
```

This looks like one operation, but it may compile into:

1. Load `x`.
2. Add `1`.
3. Store back to `x`.

If two threads do this concurrently, they can both load the same old value and one increment can be lost.

## Atomic Operation

An atomic operation appears indivisible. Other threads cannot observe it halfway done.

Atomicity matters because many high-level statements are not atomic at the machine level.

Examples that are usually not atomic as full operations:

- Incrementing a shared integer.
- Checking a condition and then updating a variable.
- Appending to a shared data structure.
- Checking whether a buffer is empty and then removing an item.

## Race Conditions

A race condition occurs when program correctness depends on the timing or ordering of concurrent execution.

A data race is a specific kind of race involving unsynchronized conflicting accesses to shared memory.

Symptoms:

- Program usually works but sometimes fails.
- Adding print statements changes the bug.
- Bug appears only under load.
- Bug depends on core count or timing.

## Critical Sections

A critical section is code that accesses shared state and must not be executed concurrently by multiple threads unless carefully controlled.

Example:

```cpp
// Critical section
balance = balance - amount;
```

If two withdrawals happen simultaneously, the updates must be coordinated.

## The "Too Much Milk" Problem

The milk example models a simple coordination problem:

- Two people/threads want to ensure there is milk.
- If there is no milk, one should buy it.
- If both check at the same time and both see none, both may buy milk.

The core issue is check-then-act:

1. Check shared state.
2. Decide what to do.
3. Update shared state.

Without synchronization, another thread can interfere between these steps.

## Bad Attempts and Lessons

Naive solution:

```cpp
if (noMilk) {
    buyMilk();
}
```

Problem: two threads can both observe `noMilk`.

Attempts using notes/flags show that ad hoc synchronization is complicated and error-prone. You need clear mechanisms that provide:

- Mutual exclusion.
- Atomic check/update sequences.
- Blocking or waiting when progress is not possible.

## Key Definitions

Mutual exclusion:

- At most one thread can execute a critical section at a time.

Synchronization:

- Coordinating execution order between threads.

Atomicity:

- An operation behaves as if it happens all at once.

Race condition:

- Correctness depends on timing.

Critical section:

- Code that must be protected because it accesses shared state.

## Core Takeaways

- Shared mutable state makes concurrency hard.
- Many source-level operations are not atomic.
- Correctness can depend on interleavings.
- Critical sections need synchronization.
- Ad hoc synchronization is usually fragile.

## Exam Questions To Do

Do these after Lecture 4:

- `Spr2022MidtermSolution.pdf` Problem 1(a): critical section mutual exclusion.
- `Spr2022MidtermSolution.pdf` Problem 1(b): scheduler change exposing a synchronization bug.
- `CS111-Practice-Midterm-Solutions.pdf` Problem 1(f): possible outputs from unsynchronized multithreading.
- `CS111-Practice-Midterm-Solutions.pdf` Problem 3: expression evaluation with multiple threads writing shared results.

---

# Lecture 5: Locks and Condition Variables

## Why Locks?

The "too much milk" example shows that hand-built coordination is too complicated. We want higher-level synchronization mechanisms.

A lock provides mutual exclusion. It lets code create a critical section where only one thread at a time may execute.

Basic pattern:

```cpp
mutex.lock();
// critical section
mutex.unlock();
```

In C++, prefer RAII-style locking:

```cpp
std::lock_guard<std::mutex> lock(m);
// critical section
```

or:

```cpp
std::unique_lock<std::mutex> lock(m);
// critical section
```

RAII helps ensure the lock is released even if the function returns early or throws an exception.

## Mutual Exclusion

Locks solve this problem:

> Only one thread may execute the protected code at a time.

Example:

```cpp
std::mutex m;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(m);
    counter++;
}
```

The lock protects the invariant of `counter`.

## What Locks Do Not Automatically Solve

Locks do not automatically tell a thread when it should proceed.

For example, in a bounded buffer:

- A producer cannot insert if the buffer is full.
- A consumer cannot remove if the buffer is empty.

A lock protects the buffer data structure, but threads also need a way to wait for the buffer's state to change.

That is where condition variables come in.

## Producer/Consumer Problem

Producer/consumer is a classic concurrency problem.

- Producers add items to a shared buffer.
- Consumers remove items from the shared buffer.
- The buffer has limited capacity.

Required invariants:

- Consumers must not remove from an empty buffer.
- Producers must not insert into a full buffer.
- The buffer's internal state must remain consistent.

Shared state might include:

- The buffer contents.
- The number of items.
- Head/tail indexes.
- Capacity.

## Broken Version: No Lock

If producers and consumers update the buffer concurrently without a lock, they can corrupt shared state.

Problems:

- Two producers may write to the same slot.
- Two consumers may read the same item.
- Count/head/tail values can become inconsistent.

## Broken Version: Lock but No Waiting

A lock prevents simultaneous access, but what should a consumer do if the buffer is empty?

Bad options:

- Return an error even though an item may arrive soon.
- Spin repeatedly checking.
- Sleep for arbitrary time and retry.

These are inefficient or incorrect.

## Condition Variables

A condition variable lets a thread sleep until some condition involving shared state may have changed.

Important: the condition variable does not itself store the condition. The condition is a predicate over shared state.

Example predicates:

- `buffer is not empty`
- `buffer is not full`
- `work queue has tasks`
- `shutdown flag is true`

The shared state must be protected by a mutex.

## Correct Wait Pattern

The standard condition-variable pattern is:

```cpp
std::unique_lock<std::mutex> lock(m);
while (!condition_is_true) {
    cv.wait(lock);
}
// condition is true and lock is held
// safely use shared state
```

Why `while`, not `if`?

- Multiple threads may be woken.
- Another thread may consume the resource first.
- Spurious wakeups are allowed.
- The condition may no longer be true by the time the thread reacquires the lock.

The loop rechecks the real predicate.

## What `wait` Does

`cv.wait(lock)` must do something subtle:

1. The thread currently holds the mutex.
2. `wait` atomically releases the mutex and puts the thread to sleep.
3. Another thread can now acquire the mutex and change shared state.
4. When notified, the waiting thread wakes up.
5. Before returning from `wait`, it reacquires the mutex.

The atomic release-and-sleep step prevents lost wakeups.

## Notification

After changing shared state, a thread may notify waiters:

```cpp
cv.notify_one();
```

or:

```cpp
cv.notify_all();
```

Use `notify_one` when one waiting thread can make progress. Use `notify_all` when multiple waiters may need to recheck different conditions, or when it is simpler and correct to wake everyone.

Because waiters recheck predicates in a loop, extra wakeups are usually safe, though they can be less efficient.

## Producer/Consumer with Condition Variables

Bounded buffer sketch:

```cpp
std::mutex m;
std::condition_variable not_empty;
std::condition_variable not_full;
std::queue<Item> q;
const size_t capacity = 10;

void put(Item item) {
    std::unique_lock<std::mutex> lock(m);
    while (q.size() == capacity) {
        not_full.wait(lock);
    }
    q.push(item);
    not_empty.notify_one();
}

Item get() {
    std::unique_lock<std::mutex> lock(m);
    while (q.empty()) {
        not_empty.wait(lock);
    }
    Item item = q.front();
    q.pop();
    not_full.notify_one();
    return item;
}
```

Key points:

- The mutex protects `q`.
- Producers wait for "not full."
- Consumers wait for "not empty."
- Both use `while`.
- State is changed while holding the lock.
- Notification happens after the state change.

## Monitor-Style Locking

Monitor-style locking means a lock protects a specific shared object or invariant, and operations on that object acquire the lock before touching the state.

Good style:

- One lock per shared object or coherent group of state.
- Clear invariant protected by that lock.
- No access to protected state without holding the lock.

Bad style:

- Many unrelated locks with unclear ownership.
- Accessing state sometimes with and sometimes without the lock.
- Holding locks across slow or unknown operations unnecessarily.

## How Many Locks?

One coarse-grained lock:

- Simpler.
- Less risk of deadlock.
- More contention.
- Less parallelism.

Many fine-grained locks:

- More parallelism.
- Less contention.
- More complexity.
- More deadlock risk.

A practical approach is to start with clear correctness and only add finer-grained locking when contention matters.

## Core Takeaways

- Locks provide mutual exclusion.
- Condition variables provide blocking until shared state changes.
- The condition lives in shared state, not in the condition variable.
- Always wait in a loop.
- `wait` releases the lock while sleeping and reacquires it before returning.
- Protect invariants consistently.

## Exam Questions To Do

Do these after Lecture 5:

- `Spr2023MidtermSolution.pdf` Problem 1(a): lock ownership.
- `Spr2022MidtermSolution.pdf` Problem 5: multi-lane merge monitor.
- `Spr2023MidtermSolution.pdf` Problem 5: airplane boarding monitor.
- `Spr2024MidtermSolution.pdf` Problem 5: inventory reservation monitor.

These are the highest-value practice problems for the synchronization unit.

---

# Lecture 6: Lock Implementation

## The Question

Lecture 5 uses locks and condition variables as tools. Lecture 6 asks:

> How are locks themselves implemented inside the OS?

This is tricky because implementing a lock requires some lower-level way to create atomicity.

## Uniprocessor Lock Implementation

On a single-core machine, only one thread can execute at a time. A thread can lose control because of:

- A trap/system call.
- An interrupt.
- An explicit block/yield.

If the kernel disables interrupts on a uniprocessor, the current kernel code can run without being interrupted by the timer or devices.

This can create a short critical section inside the kernel.

## Example Lock State

A simple lock might contain:

```cpp
class Lock {
    bool locked = false;
    ThreadQueue q;
};
```

The lock needs:

- A flag saying whether it is held.
- A queue of threads waiting for it.

## Lock Acquire on One Core

Conceptually:

```cpp
void Lock::lock() {
    intrDisable();
    if (!locked) {
        locked = true;
    } else {
        q.add(currentThread);
        blockThread();
    }
    intrEnable();
}
```

The important idea is that checking `locked`, adding to the queue, and blocking must be atomic with respect to other lock operations.

## Lock Release on One Core

Conceptually:

```cpp
void Lock::unlock() {
    intrDisable();
    if (q.empty()) {
        locked = false;
    } else {
        unblockThread(q.remove());
    }
    intrEnable();
}
```

If no one is waiting, the lock becomes free. If someone is waiting, a thread is unblocked and will eventually acquire or continue with the lock according to the implementation's semantics.

## Why Disable Interrupts?

Suppose `lock()` checks that the lock is unavailable and is about to add the thread to the wait queue. If an interrupt occurs at the wrong time and another thread runs, the lock state and wait queue can become inconsistent.

Disabling interrupts prevents the kernel from being interrupted in the middle of manipulating lock internals.

## Why Must Blocking Be Careful?

Blocking a thread changes scheduler state. The OS must avoid a lost wakeup where:

1. A thread decides it needs to sleep.
2. Another thread releases the lock and tries to wake a waiter.
3. The first thread is not yet properly recorded as waiting.
4. The wakeup is lost.
5. The first thread sleeps forever.

This is the same broad issue that condition-variable `wait` solves by atomically releasing a lock and going to sleep.

## Multiprocessor Locks

Disabling interrupts on one core does not stop another core from executing.

On multicore systems:

- Core 1 can disable its interrupts.
- Core 2 can still run and access the same memory.

Therefore, uniprocessor interrupt disabling is insufficient.

## Hardware Atomic Instructions

Multiprocessor locks require hardware support for atomic memory operations.

Common examples:

- Test-and-set.
- Compare-and-swap.
- Exchange.
- Fetch-and-add.

These instructions let one core perform a read-modify-write sequence atomically with respect to other cores.

## Spin Locks

A spin lock repeatedly checks until it can acquire the lock.

Example idea:

```cpp
while (test_and_set(&locked)) {
    // spin
}
```

Spinning wastes CPU cycles, but it can be acceptable for very short waits, especially inside the kernel where sleeping may be too expensive or impossible.

## Blocking Locks vs Spin Locks

Spin lock:

- Thread keeps running and repeatedly checks.
- Good only for short critical sections.
- Bad if wait may be long.

Blocking lock:

- Thread sleeps and another thread runs.
- Better for longer waits.
- Requires scheduler involvement.

Operating systems often use both depending on context.

## Why Busy Waiting Can Be Acceptable

Busy waiting can be acceptable when:

- The critical section is extremely short.
- The lock holder is running on another core.
- Blocking and waking would cost more than spinning.
- The code is inside low-level kernel paths where sleeping is not safe.

It is not acceptable for long waits or application-level waiting in most cases.

## Races Inside Lock Implementation

The lock implementation itself is shared state. It has its own races:

- Two threads may try to acquire at once.
- A releaser may wake a waiter.
- A waiter may go to sleep.
- State must remain consistent across cores.

This is why lock implementation depends on hardware atomicity and careful interaction with the dispatcher.

## Core Takeaways

- Locks require lower-level atomicity.
- On one core, disabling interrupts can protect kernel critical sections.
- On multiple cores, hardware atomic instructions are needed.
- Spin locks busy-wait; blocking locks sleep.
- Lock implementation must avoid lost wakeups and internal races.

## Exam Questions To Do

Do these after Lecture 6:

- `Spr2023MidtermSolution.pdf` Problem 1(c): when busy-waiting can be a good idea.
- `Spr2022MidtermSolution.pdf` Problem 5: revisit the monitor solution and identify where blocking happens instead of spinning.
- `Spr2024MidtermSolution.pdf` Problem 5: identify why `reserve` waits on a condition variable instead of busy-waiting.

The past midterms do not have a large lock-implementation coding question, but they do test the busy-waiting vs blocking distinction.

---

# Lecture 7: Deadlock

## Why Multiple Locks Exist

Programs often use multiple locks because:

- One global lock creates too much contention.
- Different data structures need independent protection.
- Modularity suggests each component manages its own lock.
- Some operations need to touch multiple shared objects at once.

Multiple locks improve structure and parallelism, but they introduce deadlock risk.

## Simple Deadlock Example

Two locks:

```cpp
std::mutex m1;
std::mutex m2;
```

Thread A:

```cpp
m1.lock();
m2.lock();
// work
m2.unlock();
m1.unlock();
```

Thread B:

```cpp
m2.lock();
m1.lock();
// work
m1.unlock();
m2.unlock();
```

Possible interleaving:

1. Thread A locks `m1`.
2. Thread B locks `m2`.
3. Thread A waits for `m2`.
4. Thread B waits for `m1`.

Neither can proceed.

## Informal Definition

Deadlock occurs when a group of threads is permanently stuck because each is waiting for an event that only another stuck thread can cause.

In lock deadlock, each thread waits for a lock held by another thread in the cycle.

## Four Conditions for Deadlock

Deadlock requires all four conditions:

1. Mutual exclusion.
2. Hold and wait.
3. No preemption.
4. Circular wait.

If you prevent any one of these conditions, deadlock cannot occur.

## Mutual Exclusion

At least one resource cannot be shared simultaneously.

Locks are mutually exclusive by design: only one thread can hold a mutex at a time.

## Hold and Wait

A thread holds one resource while waiting for another.

Example:

- Thread A holds `m1`.
- Thread A waits for `m2`.

## No Preemption

Resources cannot be forcibly taken away.

If a thread holds a lock, the OS or runtime usually does not simply steal it, because that could leave protected state inconsistent.

## Circular Wait

There is a cycle of waiting:

- A waits for B.
- B waits for C.
- C waits for A.

With two threads:

- A waits for a lock held by B.
- B waits for a lock held by A.

Circular wait is often the most practical condition to prevent.

## Deadlock Is Broader Than Locks

Deadlock can involve many resource types:

- Locks.
- Semaphores.
- Disk space.
- Network connections.
- Threads in a thread pool.
- Files.
- Processes waiting for each other.

The same four-condition reasoning applies.

## Solution 1: Deadlock Detection

Deadlock detection allows deadlocks to happen, then tries to find and recover from them.

Potential recovery strategies:

- Kill a thread or process.
- Roll back work.
- Release resources.

Problems:

- Recovery may be hard or unsafe.
- Killing a thread can corrupt state.
- Detection itself has overhead.

Deadlock detection is useful in some systems, such as databases, where transactions can be aborted and retried.

## Solution 2: Deadlock Prevention

Deadlock prevention designs the system so deadlock cannot happen.

Common approach: prevent circular wait with a global lock ordering.

Rule:

- Assign every lock an order.
- Threads must acquire locks only in increasing order.
- Threads release locks in reverse order.

If all code follows the order, cycles cannot form.

## Practical Lock Ordering

Example:

- Always acquire `accountA` before `accountB` by account ID.
- Always acquire filesystem parent directory before child directory.
- Always acquire global manager lock before per-object lock.

The hard part is making the order clear and consistently enforced.

## Case Study: Two Processes Doing `mv`

Moving files or directories may require locking multiple filesystem objects. If two moves acquire locks in different orders, they can deadlock.

The design insight is to impose an ordering on resources so operations that need multiple locks acquire them consistently.

## Core Takeaways

- Deadlock requires four conditions.
- Multiple locks are useful but dangerous.
- Circular wait is commonly prevented with lock ordering.
- Deadlock can happen with resources other than locks.
- Systems must choose between prevention, detection, and accepting risk.

## Exam Questions To Do

Do these after Lecture 7:

- `Spr2024MidtermSolution.pdf` Problem 5: inventory reservation, focusing on why it must not partially reserve resources while waiting.
- `Spr2022MidtermSolution.pdf` Problem 5: merge monitor, focusing on avoiding starvation.
- `Spr2023MidtermSolution.pdf` Problem 5: boarding monitor, focusing on choosing exactly one waiter.

There is not a direct deadlock short-answer question in the provided exams, but the big synchronization problems require the same discipline: do not hold inconsistent partial state while waiting unless the spec permits it.

---

# Lecture 8: Scheduling

## What Is CPU Scheduling?

Given:

- A dispatcher that can switch between threads.
- A set of ready threads.
- One or more CPU cores.

CPU scheduling decides:

- Which thread runs?
- On which core?
- For how long?

Scheduling is policy. Dispatching is mechanism.

## Single-Core First

It is easier to reason about scheduling on one core:

- Only one thread can run at a time.
- Ready threads wait in some queue.
- The scheduler chooses the next thread.

Then the ideas can be extended to multiple cores.

## FIFO Scheduling

FIFO means first-in, first-out. It is also called non-preemptive scheduling in the lecture.

Approach:

- Keep ready threads in a queue.
- New ready threads go to the back.
- Run the front thread until it exits or blocks.

Advantages:

- Simple.
- Low overhead.
- Predictable ordering.

Problems:

- A long-running CPU-bound job can delay all jobs behind it.
- Interactive response time can be terrible.
- It does not handle mixed workloads well.

## Preemption and Time Slices

Preemption means the OS can stop a running thread and choose another, even if the thread did not block or exit.

A time slice is the amount of CPU time a thread gets before the timer interrupt gives the OS a chance to reschedule.

Preemption improves responsiveness, but context switches have overhead.

## Round Robin Scheduling

Round robin is FIFO with preemption.

Approach:

- Each ready thread gets a time slice.
- If it does not finish or block, it goes to the back of the queue.
- The next thread runs.

Advantages:

- Better response time than FIFO.
- Fairer sharing among CPU-bound threads.
- Simple.

Problems:

- Too small a time slice causes too many context switches.
- Too large a time slice behaves like FIFO.
- It does not prioritize short jobs automatically.

## Time Slice Tradeoff

Short time slice:

- Better interactivity.
- More frequent context switches.
- More overhead.
- Worse cache locality.

Long time slice:

- Less overhead.
- Better throughput for CPU-bound jobs.
- Worse response time.

Scheduling involves tradeoffs, not one universal best answer.

## Scheduling Goals

Possible goals include:

- Fairness.
- Low response time.
- High throughput.
- Low turnaround time.
- Meeting deadlines.
- Avoiding starvation.
- Prioritizing interactive tasks.
- Efficient core utilization.

Different systems optimize different goals.

## Response Time vs Turnaround Time

Response time:

- How long until a job first responds or begins producing output.

Turnaround time:

- How long from job arrival to completion.

Interactive systems care heavily about response time. Batch systems may care more about throughput or turnaround.

## SRPT: Shortest Remaining Processing Time

SRPT runs the job with the shortest remaining processing time.

In theory, SRPT is excellent for minimizing average response/turnaround time.

Advantages:

- Short jobs complete quickly.
- Reduces average waiting time.

Disadvantages:

- Requires knowing remaining run time, which is usually impossible.
- Long jobs can starve if short jobs keep arriving.
- It may be unfair.

## Can We Approximate SRPT?

Real systems cannot usually know exact remaining time, but they can infer behavior:

- Interactive tasks often block frequently.
- CPU-bound tasks use entire time slices.
- Recent behavior can predict near-future behavior.

Schedulers often use heuristics to favor tasks that appear interactive.

## Priority-Based Scheduling

In priority scheduling, each thread has a priority. The scheduler prefers higher-priority threads.

Implementation often uses multiple ready queues:

- One queue per priority level.
- Scheduler chooses from the highest non-empty priority queue.

Problems:

- Low-priority threads may starve.
- Priority inversion can occur.
- Choosing priorities is a policy question.

## Unix Nice Level

Unix-like systems expose a user-controlled priority hint through "nice" levels.

Conceptually:

- A nicer process is more willing to yield CPU to others.
- Lower priority means less CPU preference.

This is user input into scheduling policy, not absolute control.

## BSD Scheduler

The lecture references the 4.4 BSD scheduler as an example of a real Unix scheduler. The important conceptual point is that real schedulers combine:

- Priorities.
- Recent CPU usage.
- User hints.
- Interactivity heuristics.

They are more complex than textbook FIFO or round robin.

## Multicore Scheduling

With multiple cores, the scheduler must decide:

- Which thread runs?
- On which core?
- How to balance load?
- Whether to keep a thread on the same core for cache locality?

Simple approach:

- Use one global ready queue.
- Whenever a core needs work, take the next ready thread.

Problems:

- Contention on the global queue.
- Cache locality may suffer.
- Work may not be evenly balanced.

## Work-Conserving Scheduling

A scheduler is work-conserving if it does not leave a core idle when there is ready work that could run.

This sounds obviously good, but real systems may sometimes make more nuanced decisions for power, locality, or priority reasons.

## Core Takeaways

- Scheduling is policy; dispatching is mechanism.
- FIFO is simple but can have poor response time.
- Round robin improves responsiveness through preemption.
- SRPT is theoretically strong but hard to implement fairly.
- Priority scheduling supports policy but can starve low-priority work.
- Multicore scheduling adds load balancing and locality concerns.

## Exam Questions To Do

Do these after Lecture 8:

- `Spr2022MidtermSolution.pdf` Problem 1(b): scheduler changes exposing synchronization bugs.
- `Spr2022MidtermSolution.pdf` Problem 1(c): BSD scheduler preemption.
- `Spr2022MidtermSolution.pdf` Problem 3: priority queues, CPU-bound vs I/O-bound threads, and time-slice lengths.
- `Spr2024MidtermSolution.pdf` Problem 1(a): STCF optimality.
- `Spr2024MidtermSolution.pdf` Problem 2: preemptive vs non-preemptive scheduling.
- `Spr2024MidtermSolution.pdf` Problem 3(b)-(c): round robin vs FIFO and per-core ready queues.

---

# Lecture 9: Linkers and Process Memory Layout

## Transition to Memory

The first part of the course focused on CPU issues:

- Threads.
- Processes.
- Synchronization.
- Scheduling.

Lecture 9 begins the memory section:

- Process memory layout.
- Linking.
- Loading.
- Dynamic linking.

## Main Memory

Main memory is DRAM.

Properties:

- Volatile: contents disappear when power is lost.
- Byte-addressable: programs refer to individual memory addresses.
- Accessed by hardware in larger chunks such as cache lines.
- Much slower than CPU registers/cache but much faster than disk.

The OS must manage memory among processes and protect processes from each other.

## Application Memory of a Process

Consider:

```cpp
int global = 7;
int* gptr = &global;

void func(int x) {
    int local = x;
    int* lptr = &local;
    int* heap = new int(42);
    lptr = heap;
    delete heap;
}
```

Where things live:

- `global`: global/static data segment.
- `gptr`: global/static data segment, containing an address.
- `x`: stack parameter/local state.
- `local`: stack.
- `lptr`: stack variable containing an address.
- `new int(42)`: heap allocation.
- `heap`: stack variable containing the heap object's address.

Important distinction:

- A pointer variable lives somewhere.
- The object it points to may live somewhere else.

## Process Memory Layout

A typical process address space contains:

- Text/code segment: executable instructions.
- Read-only data: constants.
- Global/static data: global variables.
- Heap: dynamic allocations.
- Stack: function calls, local variables.
- Memory-mapped regions: libraries, mapped files, shared memory.

The exact layout varies by OS and architecture, but the conceptual regions are important.

## Source Code to Running Process

The pipeline:

1. Source code.
2. Compiler produces assembly.
3. Assembler produces object files.
4. Linker combines object files and resolves symbols.
5. Loader loads executable into memory.
6. Program runs as a process.

Each step transforms the program representation.

## Compiler

The compiler turns high-level language code into assembly or lower-level intermediate output.

It handles:

- Parsing.
- Type checking.
- Optimization.
- Code generation.

The compiler usually does not know final absolute addresses for all symbols across the whole program.

## Assembler

The assembler turns assembly code into machine-code object files.

Object files may contain:

- Machine instructions.
- Data.
- Symbol definitions.
- Unresolved symbol references.
- Relocation information.

An object file is not always directly runnable because it may refer to symbols in other object files or libraries.

## Linker

The linker combines object files into an executable or library.

Responsibilities:

- Merge code and data sections.
- Resolve symbol references.
- Assign addresses or relative offsets.
- Apply relocations.
- Include needed library code or references.

Example:

- `main.o` calls `printf`.
- `printf` is not defined in `main.o`.
- The linker finds `printf` in a library or records a dynamic reference.

## Symbol Resolution

Symbols are names for functions or data objects.

Definitions:

- "Here is the code/data for symbol X."

References:

- "This instruction/data needs the address of symbol X."

The linker matches references to definitions.

## Loader

The OS loader loads the executable into a process address space.

It sets up:

- Code segment.
- Data segment.
- Stack.
- Heap start.
- Initial register state.
- Dynamic libraries if needed.

Then execution begins at the program's entry point.

## Static Linking

Static linking includes library code directly in the executable.

Advantages:

- Fewer runtime dependencies.
- Potentially simpler deployment.

Disadvantages:

- Larger executables.
- Library updates require relinking/redeploying.
- Memory may be wasted if many processes include their own copy.

## Dynamic Linking

Dynamic linking resolves some library references at load time or runtime.

Advantages:

- Shared libraries can be shared across processes.
- Executables are smaller.
- Libraries can be updated independently.

Disadvantages:

- More runtime complexity.
- Versioning issues.
- Startup overhead.
- Security/trust implications.

## Jump Tables and Dynamic Loader

One dynamic linking approach uses indirection through tables. Calls go through entries that can be filled in by the dynamic loader.

High-level idea:

- The program contains a placeholder/indirect call path.
- The dynamic loader locates the actual library function.
- The table entry is updated to point to the resolved function.

You do not need to memorize a specific table layout unless the course later asks for it. Understand why indirection supports late binding.

## Core Takeaways

- Process memory has structured regions.
- Pointers and pointees may live in different regions.
- The compiler/assembler/linker/loader pipeline turns source into a running process.
- Linkers resolve symbols across object files.
- Dynamic linking trades simplicity for sharing and flexibility.

## Exam Questions To Do

Do these after Lecture 9:

- `Spr2022MidtermSolution.pdf` Problem 1(d): symbol table definition vs unresolved reference.
- `Spr2023MidtermSolution.pdf` Problem 1(d): base-and-bound relocation requires a contiguous physical region.
- `Spr2023MidtermSolution.pdf` Problem 3(a)-(d): globals, forked memory, local variables, and stack scope.
- `Spr2024MidtermSolution.pdf` Problem 1(b): dynamic linking and memory usage.
- `Spr2024MidtermSolution.pdf` Problem 1(c): linker section layout.
- `Spr2024MidtermSolution.pdf` Problem 1(e): virtual vs physical address sizes.
- `Spr2022MidtermSolution.pdf` Problem 4, `Spr2023MidtermSolution.pdf` Problem 4, and `Spr2024MidtermSolution.pdf` Problem 4: page-table arithmetic.

The page-table questions go beyond the linker material in this chapter, but they are memory-unit questions and show up repeatedly.

---

# Lecture 10: Dynamic Storage Management

## The Problem

Dynamic storage management asks:

> How do we manage a region of memory or storage to satisfy unpredictable allocation and free requests?

This applies to:

- Application heap memory.
- Kernel memory.
- Disk block allocation.
- Other resource pools.

Operations:

```cpp
ptr = allocate(size);
free(ptr);
```

The hard part is unpredictability:

- You do not know future allocation sizes.
- You do not know when blocks will be freed.
- You do not know what pattern of use will occur.

## Stack Allocation

Stack allocation follows a strict last-in, first-out pattern.

Example:

```cpp
void f() {
    int x;
    g();
}
```

When `f` is called, its stack frame is pushed. When `f` returns, its frame is popped.

Advantages:

- Very fast.
- Simple pointer adjustment.
- No fragmentation problem in the same way as heap allocation.
- Freeing is automatic.

Limitation:

- Only works for lifetimes that follow call/return nesting.

If an object must outlive the function that created it, stack allocation may not work.

## Heap Allocation

Heap allocation supports arbitrary allocation and free order.

Example:

```cpp
int* p = new int(42);
// p can be used beyond the current function if shared
delete p;
```

Advantages:

- Flexible lifetimes.
- Supports dynamic data structures.

Disadvantages:

- Harder to manage.
- Can fragment.
- Can leak.
- Can suffer use-after-free or double-free bugs.

## Free Space Management

The allocator must track which parts of memory are:

- In use.
- Free.

It must find a free block large enough for each allocation request and return freed blocks to the available pool.

## Fragmentation

Fragmentation means free memory exists but is not usable in the desired way.

External fragmentation:

- Free space is split into many small holes.
- Total free space may be enough, but no single hole is large enough.

Internal fragmentation:

- Allocated blocks are larger than requested.
- Wasted space exists inside allocated blocks.

Allocators often trade one kind of overhead for another.

## Free List

A free-list allocator keeps a linked list of free blocks.

Each free block may store:

- Size.
- Pointer to next free block.

Allocation:

1. Search free list for a large enough block.
2. Possibly split the block.
3. Return part of it to the caller.

Free:

1. Add the block back to the free list.
2. Possibly coalesce with neighboring free blocks.

## First Fit

First fit chooses the first free block large enough to satisfy the request.

Advantages:

- Simple.
- Often fast.

Disadvantages:

- Can leave small holes near the front of the list.
- Fragmentation depends on workload.

## Best Fit

Best fit chooses the smallest free block that is large enough.

Advantages:

- Tries to reduce leftover space per allocation.

Disadvantages:

- May require searching more of the list.
- Can create many tiny unusable holes.
- Not always better in practice.

## Splitting and Coalescing

Splitting:

- If a free block is larger than requested, allocator may split it into an allocated block and a smaller free block.

Coalescing:

- When a block is freed, allocator may merge it with adjacent free blocks.

Coalescing reduces external fragmentation but requires knowing whether neighboring blocks are free.

## Slab Allocator

A slab allocator manages objects of fixed or similar sizes.

Idea:

- Allocate large chunks called slabs.
- Divide each slab into equal-size objects.
- Maintain free objects within the slab.

Advantages:

- Fast allocation/free.
- Low per-object overhead.
- Good for frequently allocated fixed-size kernel objects.
- Reduces fragmentation for known object sizes.

Disadvantages:

- Less flexible for arbitrary sizes.
- Can waste memory if slabs are underused.

## Bitmaps

A bitmap can track allocation status.

Example:

- One bit per block.
- `0` means free.
- `1` means allocated.

Advantages:

- Compact representation.
- Simple for fixed-size blocks.

Disadvantages:

- Finding long contiguous runs can be costly.
- Less natural for variable-size allocations.

## Storage Reclamation

Storage reclamation means recovering memory that is no longer needed.

Manual reclamation:

- Programmer calls `free`/`delete`.

Automatic reclamation:

- Runtime detects unreachable objects and frees them.

Both approaches have failure modes.

## Manual Memory Bugs

Common bugs:

- Memory leak: allocated memory is never freed.
- Double free: same block is freed twice.
- Use after free: program uses memory after freeing it.
- Invalid free: freeing a pointer that was not allocated.

These bugs can cause crashes, corruption, or security vulnerabilities.

## Reference Counting

Reference counting tracks how many references point to an object.

Idea:

- Increment count when a reference is created.
- Decrement count when a reference is destroyed.
- Free object when count reaches zero.

Advantages:

- Simple conceptually.
- Reclaims objects promptly when count reaches zero.

Disadvantages:

- Overhead on reference updates.
- Cannot reclaim cycles by itself.

Cycle example:

- Object A points to B.
- Object B points to A.
- No outside references exist.
- Counts are nonzero, so neither is freed.

## Garbage Collection

Garbage collection automatically finds unreachable objects.

An object is live if it can be reached from roots such as:

- Stack variables.
- Registers.
- Global variables.
- Runtime references.

Objects not reachable from roots can be reclaimed.

## Mark and Sweep

Mark-and-sweep GC has two conceptual phases:

1. Mark:
   - Start from roots.
   - Traverse references.
   - Mark all reachable objects.

2. Sweep:
   - Scan heap.
   - Free unmarked objects.
   - Clear marks for next cycle.

Advantages:

- Can collect cycles.

Disadvantages:

- Can be expensive.
- May pause program execution.
- Requires runtime support.

## Core Takeaways

- Stack allocation is simple because lifetimes are nested.
- Heap allocation is hard because lifetimes are arbitrary.
- Free lists, slabs, and bitmaps are allocation strategies.
- Fragmentation is a central allocator problem.
- Manual memory management risks leaks and invalid memory use.
- Reference counting is prompt but struggles with cycles.
- Garbage collection can reclaim cycles but has runtime cost.

## Exam Questions To Do

Do these after Lecture 10:

- `Spr2023MidtermSolution.pdf` Problem 2: disadvantages of slab allocation, reference counting, and garbage collection.
- `Spr2024MidtermSolution.pdf` Problem 1(d): reference counting and dangling pointers.
- `Spr2023MidtermSolution.pdf` Problem 3(c)-(d): local variable lifetime and stack allocation.
- `Spr2024MidtermSolution.pdf` Problem 5: inventory reservation, as a resource-allocation analogy.

---

# Lecture 12: Trust and Operating Systems

## What Is Trust?

Trust involves being willing to be vulnerable to another party because you expect that party to behave acceptably, even when you cannot fully monitor or control them.

In operating systems, users and applications are constantly trusting software they cannot fully inspect.

## Philosophy Version

Trust can be understood as an unquestioning attitude. When you trust something, you stop actively verifying every action and assume it will work.

This is necessary in computing because systems are too complex to fully check at every moment.

## Why Trust Matters

Trust extends agency:

- You can do more by relying on others or on systems.
- You do not personally implement every piece of software.
- You rely on compilers, kernels, libraries, cloud services, package managers, and maintainers.

Trust improves efficiency:

- Constant verification would be too expensive.
- Users need to get work done.
- Developers need reusable components.

Trust also preserves sanity:

- Modern systems are too complex for one person to audit end-to-end.

## Risks of Trust

Trust creates vulnerability.

Risks include:

- Bugs.
- Malicious code.
- Supply-chain attacks.
- Misplaced assumptions.
- Over-trust.

Over-trust means trusting more than is warranted by evidence or controls.

## How Trust Is Established

The lecture identifies several ways trust may be established.

Trust by assumption:

- You assume something is trustworthy because you must proceed.

Trust by inference:

- You infer trustworthiness from evidence, reputation, past behavior, transparency, testing, or review.

Trust by substitution:

- You trust one party because another trusted party vouches for it.

Example:

- You trust a package because it is distributed by a trusted repository.
- You trust a binary because it is signed by a trusted key.

## Trust and Software

Software is different from many physical products:

- It can be copied perfectly.
- It can be modified invisibly.
- It can have hidden behavior.
- It can depend on many layers.
- It can update automatically.
- Its failure modes can be non-obvious.

This makes trust difficult.

## Establishing Trust in Software

Possible mechanisms:

- Source-code review.
- Testing.
- Reproducible builds.
- Digital signatures.
- Sandboxing.
- Least privilege.
- Open development processes.
- Reputation.
- Formal verification for critical components.

No mechanism is perfect. Trust is usually layered.

## Confirmation Bias

Confirmation bias matters because people may trust software after seeing evidence that supports what they already believe while ignoring warning signs.

In security and reliability, this is dangerous. Trust should be calibrated to evidence.

## Operating System as Root of Trust

The OS kernel is a root of trust because it controls:

- Memory protection.
- Process isolation.
- File access.
- Device access.
- System calls.
- Privilege boundaries.

If the kernel is compromised, applications cannot reliably protect themselves from it.

The kernel can observe or modify almost everything a process does.

## Why Users Trust Linux

Users may trust Linux because:

- It is widely used.
- It is open source.
- Many people review and test it.
- It has a long track record.
- Distributions package and maintain it.
- Security fixes are regularly released.

This does not mean Linux is perfect. It means trust is based on process, evidence, community, and adoption.

## Why Developers Trust Linux

Developers may trust Linux because:

- Development is public.
- Maintainers review patches.
- There is version control history.
- Subsystems have expert maintainers.
- Bad changes can be reverted.
- The ecosystem has strong incentives to catch problems.

Again, this is not absolute trust. It is calibrated trust.

## Trust Within Linux Developers

Large open-source projects depend on internal trust:

- Maintainers trust contributors to some degree.
- Contributors earn trust over time.
- Review processes reduce risk.
- Signed commits and maintainership structures help establish accountability.

The system is social and technical.

## Trojan Horse Example: xz / ssh Attack

The xz incident is an example of a supply-chain attack.

High-level lesson:

- An attacker can target widely used dependencies.
- Trust can be built socially over time.
- A compromise in a low-level library can affect critical software.
- Even open source can be vulnerable if review and trust processes fail.

The important OS connection is that trusted system components often depend on other trusted components. Trust is transitive, and transitive trust is risky.

## AI-Generated Code and Trust

The lecture raises the question of whether Linux should accept AI-generated code.

Trust questions include:

- Who is accountable for the code?
- Can reviewers understand it?
- Does it introduce subtle bugs?
- Was it copied from incompatible sources?
- Can maintainers verify its correctness?

The issue is not simply whether AI can produce useful code. It is whether the development process can establish enough trust in that code.

## Core Takeaways

- Trust is necessary because complete verification is impossible.
- Trust creates vulnerability.
- OS kernels are roots of trust.
- Open-source trust depends on process, review, reputation, and transparency.
- Supply-chain attacks exploit trust relationships.
- AI-generated code raises accountability and verification questions.

## Exam Questions To Do

Do these after Lecture 12:

- `Spr2023MidtermSolution.pdf` Problem 1(e): backup before OS update as trust by substitution.
- `Spr2024MidtermSolution.pdf` Problem 3(a): page-table present bits as trust by substitution.
- `CS111-Practice-Midterm-Solutions.pdf` Problem 1(e): crash recovery tradeoffs, as a trust/reliability design question.

The trust questions are usually short-answer. The key is to name the trust form and explain the backup or enforcement mechanism.

---

# Lecture 13: Virtual Memory, Base and Bound, and Segmentation

Lecture 14 is identical to Lecture 13 in the current `Lectures` folder, so this chapter covers both files.

## Why Virtualize Memory?

Earlier lectures focused on sharing one or more CPU cores among many threads. Virtual memory asks a similar question for memory:

> How can several processes share one physical memory safely and conveniently?

The OS wants memory sharing to satisfy four goals:

- Multitasking: multiple processes can be memory-resident at the same time.
- Transparency: each process should feel like it has its own memory.
- Isolation: one process should not corrupt another process or the OS.
- Efficiency: memory sharing should not make the system too slow or wasteful.

Early single-tasking systems were efficient but failed the other goals. One program occupied memory along with the OS. A bad program could overwrite the OS or another part of memory because there was little hardware enforcement.

## Single-Tasking Memory Layout

Early systems could look conceptually like:

```text
low addresses
+----------------+
| operating sys  |
+----------------+
| program code   |
+----------------+
| data           |
+----------------+
| stack          |
+----------------+
high addresses
```

This is simple, but it does not support true memory isolation. If the program uses a bad pointer, it can write almost anywhere.

## Load-Time Relocation

One early sharing approach is load-time relocation.

Idea:

- Programs are compiled as if they start at address 0.
- When the OS loads a program into physical memory, it adjusts addresses in the program so they refer to the chosen physical location.

Example:

- Program expects its data at virtual-ish address `1000`.
- OS loads the program starting at physical address `50000`.
- References must be relocated to point near `51000`.

This can support multiple programs in different physical regions.

## Load-Time Relocation Limitations

Problems:

- Every instruction/data reference that contains an address may need relocation.
- Moving a process later is difficult because addresses have already been patched.
- Protection is weak unless hardware also checks accesses.
- Sharing and growth are awkward.

The key limitation is that relocation happens once, before execution. Modern systems prefer translation on every memory access.

## Dynamic Address Translation

Dynamic address translation means the CPU issues virtual addresses, and hardware translates them to physical addresses while the program runs.

The hardware unit that performs translation is the MMU, or memory management unit.

Conceptually:

```text
program uses virtual address
        |
        v
      MMU
        |
        v
physical memory address
```

The OS configures the MMU. User programs benefit from the abstraction but cannot arbitrarily rewrite the translation rules.

## Address Spaces

An address space is the set of addresses a process can use.

With virtual memory:

- Each process can have its own virtual address space.
- Different processes can use the same virtual address for different physical memory.
- A process can be prevented from accessing addresses outside its valid regions.

Example:

```text
Process A virtual address 0x1000 -> physical page X
Process B virtual address 0x1000 -> physical page Y
```

Both processes think they are using address `0x1000`, but the MMU maps them differently.

## Base and Bound

Base-and-bound is an early dynamic translation mechanism.

Hardware stores two values for the current process:

- Base: the starting physical address of the process's memory region.
- Bound: the size or upper limit of the process's allowed region.

For each memory reference:

1. Check that the virtual address is less than the bound.
2. Add the base to get the physical address.
3. If the address is outside the bound, trap into the OS.

Formula:

```text
if virtual_address >= bound:
    trap
else:
    physical_address = base + virtual_address
```

## Base and Bound Example

Suppose:

```text
base  = 10000
bound = 4000
```

Then:

```text
virtual 0     -> physical 10000
virtual 100   -> physical 10100
virtual 3999  -> physical 13999
virtual 4000  -> trap
```

This gives each process the illusion that its memory starts at address 0, even though it actually lives somewhere in physical memory.

## Process to OS Transitions

When the OS switches from one process to another, it must update the hardware translation state.

For base and bound, that means loading the new process's:

- Base register.
- Bound register.

This is part of the process context. If the OS forgets to update it correctly, a process could access the wrong physical memory.

## Base and Bound Evaluation

Advantages:

- Simple hardware.
- Fast translation.
- Good basic isolation.
- Transparent relocation: a program can run at different physical locations without changing its code.

Disadvantages:

- A process must occupy one contiguous physical memory region.
- Stack, heap, code, and data all move together.
- Growing a process can be hard if adjacent physical memory is unavailable.
- Sharing part of an address space is awkward.
- It can cause external fragmentation.

External fragmentation occurs when free memory exists, but it is split into pieces that are not large enough for a requested contiguous allocation.

## Segmentation

Segmentation improves on base and bound by giving a process multiple regions instead of one.

Common segments:

- Code.
- Data.
- Heap.
- Stack.

Each segment can have its own:

- Base.
- Bound.
- Permissions.

This supports more natural program layout.

## Determining the Segment

For each memory reference, hardware must determine:

- Which segment is being accessed.
- Whether the offset is within that segment's bound.
- What physical address corresponds to the segment base plus offset.

Conceptually:

```text
virtual address = segment number + offset
physical address = segment_base[segment] + offset
```

If the offset exceeds the segment bound, the hardware traps.

## Segmentation Advantages

Segmentation supports:

- Separate protection for code/data/stack.
- Sharing a segment between processes, such as shared code.
- Independent growth of different regions.
- More natural mapping to program structure.

Example:

- Code segment can be read-only.
- Stack segment can grow separately from heap.

## Segmentation Disadvantages

Problems:

- Segments are variable-sized, so external fragmentation remains.
- Hardware and OS bookkeeping are more complex.
- Every memory reference needs segment translation and bounds checking.
- Managing growth and placement of segments is still hard.

Segmentation improves flexibility over base and bound, but it does not solve the variable-sized allocation problem. Paging addresses that by using fixed-size chunks.

## Core Takeaways

- Virtual memory gives each process its own address-space abstraction.
- The MMU translates virtual addresses to physical addresses.
- The OS controls translation state and uses it for isolation.
- Base and bound is simple but requires each process to be contiguous in physical memory.
- Segmentation gives multiple protected regions, but still suffers from variable-size fragmentation.

## Exam Questions To Do

Do these after Lecture 13:

- `Spr2023MidtermSolution.pdf` Problem 1(d): base-and-bound relocation and why code/data/stack move together.
- `Spr2024MidtermSolution.pdf` Problem 3(a): present bits/page tables as trust by substitution.
- `Spr2024MidtermSolution.pdf` Problem 1(e): virtual and physical addresses need not be the same size.
- `Spr2022MidtermSolution.pdf` Problem 4, `Spr2023MidtermSolution.pdf` Problem 4, and `Spr2024MidtermSolution.pdf` Problem 4: these use the virtual-memory concepts that lead into paging.

---

# Lecture 15: Paging and Page Translation

## Why Paging?

Base-and-bound and segmentation both run into placement problems because they manage variable-sized regions.

Paging changes the approach:

- Divide virtual address spaces into fixed-size pages.
- Divide physical memory into fixed-size page frames.
- Map virtual pages to physical page frames.

Because all chunks are the same size, memory management becomes simpler.

## Pages and Page Frames

Terminology:

- Virtual page: fixed-size chunk of a process's virtual address space.
- Physical page frame: fixed-size chunk of physical memory.
- Page size: number of bytes per page.

Common page sizes:

- 4 KB on x86-style systems.
- 16 KB on some modern systems.

If page size is `2^k`, then the lower `k` bits of an address are the page offset.

Example:

```text
4 KB page = 2^12 bytes -> 12 offset bits
16 KB page = 2^14 bytes -> 14 offset bits
```

## Page Map / Page Table

The page table maps virtual page numbers to physical page numbers.

Virtual address:

```text
+----------------------+-------------+
| virtual page number  | page offset |
+----------------------+-------------+
```

Physical address:

```text
+----------------------+-------------+
| physical page number | page offset |
+----------------------+-------------+
```

Translation:

1. Split virtual address into virtual page number and offset.
2. Look up virtual page number in the page table.
3. Get physical page number.
4. Combine physical page number with the same offset.

The offset does not change because the byte's position inside the page is the same before and after translation.

## Page Table Entry

A page table entry, or PTE, commonly stores:

- Physical page number.
- Present bit.
- Writable/read-only permission bit.
- Other permission/status bits depending on the architecture.

The present bit says whether the virtual page is currently valid/present. If a process accesses a page whose present bit is clear, the hardware traps into the OS.

## Fixed Size Makes Memory Management Easier

Paging helps because any virtual page can go into any free physical page frame.

Advantages:

- No need to find a large contiguous physical region for a whole process.
- Processes can grow one page at a time.
- Physical memory can be allocated flexibly.
- External fragmentation is greatly reduced.

Tradeoff:

- There can be internal fragmentation within the last page of a region if not all bytes are used.

## x86-64-Style Address Translation

The lecture uses an x86-64-style example:

- Virtual addresses are conceptually 64 bits, but a subset may be used.
- Page size is 4 KB.
- 4 KB means 12 offset bits.
- Page tables are organized in multiple levels.

Why multiple levels?

- A single flat page table for a huge virtual address space would be enormous.
- Multilevel page tables allocate lower-level tables only for regions of the address space that are actually used.

## Multilevel Page Tables

A multilevel page table treats pieces of the virtual page number as indexes through a tree-like structure.

For a 4 KB page map with 8-byte entries:

```text
page map size = 2^12 bytes
entry size    = 2^3 bytes
entries       = 2^9
```

So each level indexes 9 bits of the virtual address.

For a 48-bit virtual address with 4 KB pages:

```text
offset bits = 12
virtual page number bits = 48 - 12 = 36
levels = 36 / 9 = 4
```

That is why the past midterms use four levels for the x86-64-style mechanism.

## Translation Example

For a virtual address:

```text
virtual address = [level 4 index][level 3 index][level 2 index][level 1 index][offset]
```

Translation proceeds:

1. Use the top-level index to find the next page map.
2. Use the next index to find the next page map.
3. Continue through all levels.
4. The final entry gives the physical page number.
5. Append the original offset.

The structure resembles a tree/trie over address bits.

## Mapping Code, Data, and Stack

A process may need only a few pages:

- One page for code.
- One page for data.
- One or more pages for stack.

These virtual pages can be far apart in the virtual address space. For example:

- Code/data may be near low addresses.
- Stack may be near high addresses.

Multilevel page tables avoid allocating page-map entries for the unused gap.

This is why a small process with code/data at low addresses and stack at high addresses may still need multiple upper-level page maps.

## Sharing Memory Between Processes

Paging makes sharing natural:

- Two processes can have different virtual page numbers map to the same physical page frame.
- Shared code pages can be read-only.
- Shared memory regions can be explicitly mapped into multiple processes.

Example:

```text
Process A VPN 20 -> PPN 100
Process B VPN 44 -> PPN 100
```

Both processes access the same physical memory through their own virtual addresses.

## Translation Lookaside Buffer

Page-table translation can require several memory accesses for one program memory access. That is expensive.

A TLB, or translation lookaside buffer, is a hardware cache of recent virtual-to-physical translations.

Conceptually:

```text
virtual page number -> physical page number
```

If the translation is in the TLB:

- Hardware can translate quickly.

If not:

- Hardware or the OS must walk the page table.

## TLB OS Complications

The OS must handle cases where translations change.

Examples:

- Context switch to a different process.
- Page unmapped.
- Permissions changed.
- Page moved or swapped.

Potential issue:

- A stale TLB entry could let a process access memory it should no longer access.

The OS and hardware therefore need mechanisms to flush, invalidate, or tag TLB entries.

## OS Access to User Memory

The OS often needs to access user memory during system calls.

Example:

```c
read(fd, user_buffer, count);
```

The kernel must copy data into `user_buffer`, which is a user virtual address.

This raises questions:

- Is that user address valid?
- Is it writable?
- What if the user passes a bad pointer?
- How does the kernel safely access memory described by user page tables?

The OS must validate and handle faults rather than blindly trusting user pointers.

## OS and User in the Same Address Space

Some systems map the kernel into part of every process's virtual address space.

Benefit:

- System calls and traps can enter the kernel without switching to a completely different address space.

Risk/control:

- User code must not be allowed to access kernel-only pages.
- Page permissions enforce this.

This design makes the OS part of the virtual-memory protection story.

## Memory Aliasing

Aliasing means two virtual addresses refer to the same physical memory.

This can be useful:

- Shared memory.
- Mapping the same file in multiple places.
- Kernel/user mappings of the same physical page.

But it complicates reasoning:

- Updating through one virtual address changes what is seen through the other.
- Caches/TLBs must remain coherent.

## Fragmentation: Internal and External

External fragmentation:

- Free memory exists, but not in a large enough contiguous piece.
- Paging mostly avoids this for physical memory allocation because all frames are same-sized.

Internal fragmentation:

- Allocated space inside a fixed-size page is unused.
- Paging can create this when a region does not fill its final page.

Paging trades external fragmentation problems for some internal fragmentation and page-table overhead.

## Core Takeaways

- Paging maps fixed-size virtual pages to fixed-size physical page frames.
- The offset bits are unchanged by translation.
- Page tables store physical page numbers and permission/present bits.
- Multilevel page tables save space for sparse virtual address spaces.
- TLBs cache translations to avoid repeated expensive page-table walks.
- Page permissions are central to isolation and OS trust.

## Exam Questions To Do

Do these after Lecture 15:

- `Spr2022MidtermSolution.pdf` Problem 4: page-map entry size and whether larger pages reduce the level count enough.
- `Spr2023MidtermSolution.pdf` Problem 4: how many PML2 page maps are needed for code/data/stack and when stack growth needs another map.
- `Spr2024MidtermSolution.pdf` Problem 4: design a page-table architecture from page size, physical address size, virtual address size, and entry size.
- `Spr2024MidtermSolution.pdf` Problem 3(a): present bits as trust by substitution.
- `Spr2024MidtermSolution.pdf` Problem 1(e): virtual vs physical address sizes.

For any page-table problem, write down page size, offset bits, virtual page number bits, physical page number bits, entries per page map, bits per level, and number of levels before answering.

---

# Lecture 16: Demand Paging and Page Replacement

Lecture 17 is identical to Lecture 16 in the current `Lectures` folder, so this chapter covers both files.

## Why Demand Paging?

Earlier paging maps virtual pages to physical page frames, but it still sounds like every page of a process must be in memory. Demand paging removes that requirement.

Goal:

> Let a program run even when not all of its pages are currently in DRAM.

Demand paging keeps actively used pages in physical memory and stores idle pages on disk in a paging file, backing store, or swap space.

This works because of locality of reference:

- Programs usually use a small part of their code/data intensely for a while.
- They do not access every page uniformly all the time.
- If the OS keeps the working set in memory, performance can be good even if the full address space is much larger than RAM.

## True Virtual Memory

With demand paging, each virtual page can be:

- Present in physical memory.
- Not present in memory, but stored on disk.

The page table's present bit tells the hardware whether a page is currently in memory.

If the present bit is set:

```text
virtual page -> physical page frame
```

If the present bit is clear:

```text
virtual page -> not currently in memory
```

The OS may know where that page lives on disk.

## Page Faults

A page fault happens when a process accesses a virtual page whose page-table entry says the page is not present.

Mechanism:

1. CPU tries to access a virtual address.
2. MMU checks page table.
3. Present bit is off.
4. Hardware traps into the OS.
5. OS page fault handler runs.

The OS then decides whether this is:

- A valid page that just needs to be brought in from disk.
- An invalid access, such as dereferencing a bad pointer.

If valid:

1. Find a free physical page frame.
2. Read page contents from backing store.
3. Update the page table entry.
4. Set the present bit.
5. Resume the faulting thread.

From the program's point of view, the memory access eventually completes, just much more slowly.

## Hardware Support for Paging

Demand paging needs hardware support:

- Page-table present bits.
- Permission bits.
- Trap on invalid/not-present access.
- Ability for OS to update page tables.
- Often reference/use and dirty bits.

Reference/use bit:

- Indicates a page has been accessed recently.

Dirty bit:

- Indicates a page has been modified since being loaded.

Dirty pages must be written back to disk before eviction if the backing store needs the updated contents.

Clean pages may be dropped if they can be reloaded from their original source.

## Paging Policies

Demand paging requires policy decisions:

- Fetching policy: when should a page be brought into memory?
- Replacement policy: if memory is full, which page should be evicted?
- Scope: should replacement be global or per-process?

These policies affect performance dramatically.

## Demand Fetching

Demand fetching means:

> Bring in a page only when a process actually faults on it.

Advantages:

- Does not waste I/O on pages that may never be used.
- Simple.
- Good when future access is unpredictable.

Disadvantages:

- First access to a missing page pays full page-fault cost.
- Can cause many faults when a program starts or moves into a new phase.

## Prefetching

Prefetching means:

> Bring in pages before they are explicitly requested.

Example:

- If a process faults on page 100, the OS might also read pages 101 and 102.

Advantages:

- Can reduce future page faults.
- Can exploit sequential access patterns.

Disadvantages:

- May waste I/O and memory if guessed pages are not used.
- Bad predictions can evict useful pages.

## Page Replacement

When memory is full and a new page must be loaded, the OS must evict some page.

Question:

> Which page should be removed from physical memory?

An ideal policy would evict the page that will not be used for the longest time in the future. But the OS cannot know the future.

Practical policies approximate this by using past behavior.

## LRU

LRU means least recently used.

Idea:

- Evict the page that has not been used for the longest time.

Reasoning:

- Locality suggests pages used recently are likely to be used again soon.

Problem:

- Exact LRU is expensive to implement.
- It would require tracking precise ordering of page references.

## Clock Algorithm

Clock is an approximation of LRU.

It uses:

- A circular list of pages.
- A clock hand.
- A reference/use bit per page.

Algorithm:

1. Look at page under clock hand.
2. If reference bit is set, clear it and move on.
3. If reference bit is clear, choose that page for eviction.

Interpretation:

- A page gets a "second chance" if it was referenced recently.
- If it is not referenced again before the hand returns, it can be evicted.

## Clock Hand Speed

Clock hand speed is a useful signal.

Slow hand:

- Pages are not being evicted too aggressively.
- Memory pressure may be moderate.

Fast hand:

- OS is scanning many pages quickly looking for victims.
- May indicate high memory pressure.
- If many pages keep getting referenced, the OS may struggle to find good victims.

## Global vs Per-Process Replacement

Global replacement:

- Any process's page can be selected for eviction.

Advantages:

- Flexible.
- Memory can flow to processes that need it most.

Disadvantages:

- One memory-hungry process can hurt others.
- Performance isolation is weaker.

Per-process replacement:

- Each process has its own allocation of physical pages.
- Replacement happens within that allocation.

Advantages:

- More predictable per-process behavior.
- Better isolation.

Disadvantages:

- Can waste memory if one process has unused allocation while another needs more.

## Thrashing

Thrashing happens when the system spends most of its time paging rather than doing useful work.

Typical cause:

- The active working sets of running processes do not fit in physical memory.

Symptoms:

- Very high page fault rate.
- Disk or SSD heavily used for paging.
- CPU may be underutilized because processes are waiting on page I/O.
- System feels extremely slow.

Handling thrashing:

- Reduce degree of multiprogramming: run fewer processes at once.
- Add memory.
- Improve replacement policy.
- Identify memory-heavy processes.
- Use working-set ideas to keep each process's active pages resident.

## Core Takeaways

- Demand paging lets programs run without all pages in memory.
- Page faults are traps that let the OS load missing pages.
- Replacement policy matters when physical memory is full.
- LRU is a useful ideal; Clock approximates it cheaply.
- Thrashing means memory pressure is so high that paging dominates execution.

## Study Checks

- What happens step-by-step during a page fault?
- Why is the present bit central to demand paging?
- Why is exact LRU hard to implement?
- How does the Clock algorithm approximate LRU?
- What is thrashing, and how can the OS respond?

---

# Lecture 18: Magnetic Disks and I/O Devices

## Why Disks Matter

The memory unit focused on DRAM and virtual memory. File systems depend on persistent storage, historically magnetic disks.

Disks are important because:

- They store data durably.
- They are much slower than memory.
- Their performance depends heavily on physical layout and access patterns.

The OS must hide device complexity while still making good performance choices.

## Hard Disk Drive Structure

A magnetic hard disk has:

- Platters: circular disks coated with magnetic material.
- Spindle: rotates platters.
- Tracks: circular paths on a platter.
- Sectors: fixed-size chunks within tracks, often 4096 bytes today.
- Heads: read/write mechanisms.
- Actuator arm: moves heads to the right track.

Data is organized physically by track and sector.

## Reading and Writing Disks

A disk access has several phases:

1. Seek: move the head to the right track.
2. Select the correct head/platter surface.
3. Rotational latency: wait for the desired sector to rotate under the head.
4. Transfer: read or write sectors as they pass under the head.

Latency is dominated by mechanical movement:

- Seek time.
- Rotational latency.

Sequential access is much faster than random access because it avoids many seeks.

## Disk Performance Intuition

Important asymmetry:

- Random small reads/writes are expensive.
- Sequential reads/writes are much faster.

This shapes file-system design:

- Place related data near each other.
- Cache disk blocks.
- Avoid unnecessary seeks.
- Schedule disk requests intelligently.

## Disk API

At a high level, the OS treats a disk as a block device.

Operations look like:

```text
read block N
write block N
```

The file system builds higher-level abstractions, such as files and directories, on top of block reads/writes.

## Communicating With I/O Devices

The OS communicates with devices through device controllers.

Device controllers expose registers for:

- Commands.
- Status.
- Data transfer addresses.
- Error information.

The OS writes commands to device registers and later observes completion.

## Example Disk Read

Conceptually:

1. OS tells disk controller which block to read.
2. OS tells controller where in memory to place data.
3. Device performs the read.
4. Device signals completion.
5. OS wakes or resumes the waiting thread.

## Interrupts

Devices often signal completion with interrupts.

Without interrupts:

- OS might have to repeatedly poll the device.
- CPU time would be wasted checking status.

With interrupts:

- OS can run other work while the device operates.
- Device interrupts when done.
- Interrupt handler records completion and wakes blocked threads if needed.

## Data Transfer

Modern devices often transfer data using DMA, direct memory access.

DMA means:

- Device controller copies data directly between device and memory.
- CPU does not manually copy every byte.
- CPU configures the transfer and handles completion.

This improves performance and reduces CPU overhead.

## Modern Device Interfaces

Modern storage devices, especially SSDs, have very different physical behavior from magnetic disks. But OS abstractions still often present them as block devices.

The details differ:

- No mechanical seek for SSDs.
- Parallelism inside devices.
- Flash translation layers.

But file systems still care about persistence, block layout, caching, and crash recovery.

## Core Takeaways

- Disk access is slow mostly because of seek and rotational latency.
- Sequential access is much faster than random access on magnetic disks.
- OS uses device controllers, interrupts, and DMA to communicate with devices efficiently.
- File-system design is shaped by storage-device performance.

## Study Checks

- What are seek time, rotational latency, and transfer time?
- Why is sequential disk access faster than random access?
- What is an interrupt, and why is it useful for I/O?
- What does DMA avoid?

---

# Lecture 19: File Systems

## What Problems Do File Systems Solve?

A file system turns raw disk blocks into useful persistent storage.

Modern file systems address:

- Disk space management.
- Naming.
- Reliability.
- Protection.
- Efficient access.

A disk provides blocks. Users want named files and directories.

## What Is a File?

User view:

- A named collection of bytes.
- Durable across program exits and reboots.

Kernel/file-system view:

- A collection of disk blocks.
- Metadata describing the file.

Metadata can include:

- Size.
- Owner.
- Permissions.
- Timestamps.
- Block pointers.
- Link count.

## File Access Patterns

Common access patterns:

- Sequential access: read bytes in order from beginning to end.
- Random access: jump to offsets and read/write.
- Small file access.
- Large file streaming.
- Append-heavy workloads.

File-system design tries to make common patterns fast.

## Issues To Consider

File-system design must consider:

- How to map file offsets to disk blocks.
- How to allocate free blocks.
- How to avoid fragmentation.
- How to make small files efficient.
- How to make large files possible.
- How to recover after crashes.
- How to enforce permissions.

## Inodes

An inode is the file-system object that represents a file's metadata and block locations.

An inode usually does not store the file name. Names live in directories.

Inode contains information such as:

- File type.
- Size.
- Permissions.
- Owner.
- Link count.
- Pointers/indexes to data blocks.

The inode is the bridge between "file as a named byte stream" and "file as disk blocks."

## File Structure

The file system needs a way to map:

```text
file block number -> disk block number
```

Different designs make different tradeoffs.

## Contiguous Allocation

Contiguous allocation stores each file in one contiguous run of disk blocks.

Metadata needs:

- Starting block.
- Length.

Advantages:

- Excellent sequential performance.
- Simple block lookup.
- Few seeks.

Disadvantages:

- Hard to grow files.
- External fragmentation.
- Need to know file size or reserve extra space.

## Linked Files

Linked allocation stores each file as a linked list of blocks.

Each block points to the next block.

Advantages:

- Easy to grow files.
- No need for contiguous free space.

Disadvantages:

- Random access is poor because you must follow links.
- Pointer overhead.
- Reliability risk if a pointer is corrupted.
- Sequential access may still require many seeks if blocks are scattered.

## FAT

The FAT file system stores linked-list information in a table rather than inside each data block.

FAT means file allocation table.

The table maps:

```text
block number -> next block number
```

Advantages:

- Data blocks contain only data.
- Easier to traverse file block chains from the table.

Disadvantages:

- FAT can be large.
- Random access still requires following chains unless cached.
- Reliability and fragmentation issues remain.

## Core Takeaways

- A file is a named durable byte stream for users, but disk blocks plus metadata for the OS.
- Inodes store file metadata and block-location information.
- Contiguous allocation is fast but hard to grow.
- Linked allocation grows easily but has poor random access.
- FAT centralizes block-chain pointers in a table.

## Study Checks

- What information belongs in an inode?
- Why do file names usually live in directories rather than in inodes?
- Why is contiguous allocation fast?
- Why is linked allocation bad for random access?
- What problem does FAT move into a table?

---

# Lecture 20: BSD Inodes, Block Cache, Free Space, and Disk Scheduling

Lecture 21 is identical to Lecture 20 in the current `Lectures` folder, so this chapter covers both files.

## Multi-Level Indexes

BSD Unix uses inodes with multi-level block pointers.

The lecture example:

- Disk divided into 4 KB blocks.
- Files divided into 4 KB logical blocks.
- Inode contains 14 block pointers.
- First 12 are direct pointers.
- 13th points to an indirect block.
- 14th points to a doubly indirect block.

This makes small files efficient while still supporting large files.

## Direct Blocks

Direct block pointer:

```text
inode pointer -> data block
```

If a file uses blocks 0 through 11, the inode can point directly to each block.

Advantages:

- Fast.
- No extra index block.
- Good for small files.

## Indirect Blocks

An indirect block contains many block pointers.

If block size is 4 KB and each pointer is 4 bytes:

```text
4096 / 4 = 1024 pointers
```

The inode's 13th pointer points to an indirect block, which points to data blocks.

This supports file blocks beyond the first 12.

## Doubly Indirect Blocks

A doubly indirect block points to indirect blocks, which point to data blocks.

Structure:

```text
inode -> doubly indirect block -> indirect block -> data block
```

This supports much larger files without making every inode huge.

## Reading File Blocks

Example:

- File block 5 uses a direct pointer.
- File block 23 uses the indirect block.
- File block 1040 may require the doubly indirect structure, depending on the direct/indirect ranges.

Study method:

1. Check whether block number is less than 12.
2. If yes, use direct pointer.
3. Otherwise subtract 12 and see if it fits in the indirect range.
4. If not, use doubly indirect indexing.

## BSD Inode Evaluation

Advantages:

- Small files are efficient.
- Large files are supported.
- Index blocks are allocated only when needed.
- Random access is much better than linked allocation.

Disadvantages:

- Large-file lookup can require multiple disk reads.
- Maximum file size is fixed by pointer structure.
- More complex than contiguous allocation.

## Block Cache

A block cache stores recently used disk blocks in memory.

Why:

- Disk is slow.
- Many blocks are reused.
- Reads can be served from memory if cached.
- Writes can be delayed and combined.

The block cache improves performance but complicates crash recovery.

## Modified Blocks

When a cached block is modified:

- The in-memory copy differs from disk.
- The block is dirty.

The OS can:

- Write it immediately.
- Delay the write for performance.

Delayed writes improve performance but risk losing recent changes in a crash.

## Free Space Management

The file system must know which disk blocks are free.

Common method:

- Bitmap/free map.

Each bit represents whether a block is free or allocated.

Advantages:

- Compact.
- Easy to find free blocks with scanning.

Issues:

- Large disks mean large bitmaps.
- Finding contiguous free space can still be expensive.
- Bitmap itself must be kept consistent with inodes and directories.

## Block Sizes

Large block sizes:

- Better sequential throughput.
- Fewer block pointers.
- Less metadata overhead.

But:

- More internal fragmentation for small files.

Small block sizes:

- Less wasted space for small files.
- More metadata overhead.
- More block pointers and potentially more I/O.

BSD uses techniques such as multiple block sizes/fragments to balance these tradeoffs.

## File System Challenges

Challenges include:

- Keeping related blocks close together.
- Avoiding fragmentation.
- Supporting both small and large files.
- Efficient random and sequential access.
- Crash consistency.
- Free-space management.

## Disk Scheduling

Disk scheduling chooses the order of disk requests.

For magnetic disks, request order matters because seek time dominates.

FIFO:

- Serve requests in arrival order.
- Simple and fair.
- May cause excessive head movement.

SPTF:

- Shortest positioning time first.
- Choose request with smallest seek/rotation cost.
- Better performance.
- Can starve far-away requests.

CSCAN:

- Sweep in one direction and wrap around.
- More predictable waiting time.
- Balances fairness and seek efficiency.

## Core Takeaways

- BSD inodes combine direct, indirect, and doubly indirect pointers.
- Direct pointers make small files fast.
- Indirect pointers allow large files.
- Block caches improve performance but complicate persistence.
- Free-space bitmaps track available disk blocks.
- Disk scheduling matters because seek/rotational delays are expensive.

## Study Checks

- How many data blocks can one indirect block index if blocks are 4 KB and pointers are 4 bytes?
- Why are direct pointers good for small files?
- What makes a cached block dirty?
- What tradeoff does block size create?
- How do FIFO, SPTF, and CSCAN differ?

---

# Lecture 22: Directories and Links

## Naming Files

Previous file-system lectures explain how an inode finds a file's blocks. Directories explain how the OS finds an inode from a name.

User:

```text
/a/b/c
```

OS:

```text
resolve names through directories until it finds an inode number
```

## i-number

An i-number is an index into the inode array.

It uniquely identifies an inode within a file system.

The OS uses i-numbers internally to identify files.

## Storing Inodes on Disk

Inodes must be stored on disk so they survive reboots.

Approaches:

- Store inode array in fixed known locations.
- Divide inode array into blocks.
- In BSD-style systems, spread inode/storage structures to improve locality.

The OS can find an inode by using its i-number to locate the right inode block.

## Inodes and Open Files

When a process opens a file:

1. OS resolves the pathname to an inode.
2. OS creates open-file state.
3. Process receives a file descriptor.

The file descriptor is process-local, but it ultimately refers to open-file state connected to an inode.

This is why multiple names or descriptors can refer to the same underlying file.

## File Naming

A directory maps names to inode numbers.

Directory entry:

```text
name -> i-number
```

The directory itself is a file whose contents are directory entries.

## Hierarchical Directories

Hierarchical directories organize names into a tree or graph-like structure.

Path:

```text
/a/b/c
```

Resolution:

1. Start at root directory `/`.
2. Look up `a` to get its inode.
3. Read directory `a`.
4. Look up `b`.
5. Read directory `b`.
6. Look up `c`.

## `open("/a/b/c")`

When a process calls:

```c
open("/a/b/c")
```

the OS performs pathname lookup, then returns a file descriptor if successful.

After:

```c
fd = open("/a/b/c");
read(fd, buf, 32);
```

the `read` uses the open file state associated with the inode found during lookup.

The path is not re-resolved on every `read`; the descriptor already refers to the opened file.

## Working Directories

Relative paths are resolved starting from a process's current working directory.

Example:

```text
open("notes.txt")
```

If current directory is `/Users/benji/Documents`, this resolves like:

```text
/Users/benji/Documents/notes.txt
```

Absolute paths start at root. Relative paths start at the working directory.

## Hard Links

A hard link is another directory entry pointing to the same inode.

Example:

```text
name1 -> inode 42
name2 -> inode 42
```

Both names refer to the same underlying file.

The inode stores a link count: how many directory entries point to it.

Deleting a name removes a directory entry and decrements the link count. The file's blocks are freed only when the link count reaches zero and no open references remain.

## Hard Link Limitations

Common limitations:

- Usually cannot hard-link directories, to avoid cycles in the directory structure.
- Usually cannot hard-link across file systems, because inode numbers are local to a file system.

## Symbolic Links

A symbolic link is a special file that stores a path.

Example:

```text
shortcut -> "/some/other/path"
```

When opened, the OS follows the stored path.

Differences from hard links:

- Symlinks can cross file systems.
- Symlinks can point to directories.
- Symlinks can dangle if the target path no longer exists.

## Core Takeaways

- Directories map names to inode numbers.
- Pathname lookup repeatedly resolves directory entries.
- File descriptors refer to open files after lookup.
- Hard links are multiple names for the same inode.
- Symbolic links store paths and can dangle.

## Study Checks

- What is an i-number?
- Why is a directory a mapping from names to inode numbers?
- What happens during `open("/a/b/c")`?
- How is a hard link different from a symbolic link?
- Why can symlinks dangle?

---

# Lecture 23: File System Crash Recovery

## Why Crash Recovery Is Hard for File Systems

Most OS state disappears after a crash and can be rebuilt on reboot. File-system state is different because users expect disk data to survive.

Crash recovery must handle:

- Data loss: recent changes may not have reached disk.
- Inconsistency: a crash may interrupt a multi-block update.
- Reordered writes: the block cache may write blocks in a different order than the program issued updates.

The hard part:

> File-system updates often require multiple disk blocks to change consistently, but disks do not provide atomic multi-block writes.

## Example Inconsistencies

Adding a block to a file may require:

- Update free map.
- Write data block.
- Update inode to point to block.

If a crash happens between these updates, the disk may contain inconsistent metadata.

Creating a link may require:

- Add directory entry.
- Increment inode link count.

If only one update reaches disk, the file system is inconsistent.

## Approach 1: Repair After Crash

Classic Unix approach:

- Run `fsck`, file system check, after an unclean shutdown.

The file system can store a clean bit:

- Clean shutdown sets it.
- If not set on boot, run recovery scan.

`fsck` scans metadata and tries to repair inconsistencies.

## fsck Checks

Possible checks:

- Does every allocated block belong to a file?
- Does any block appear in multiple files?
- Do inode link counts match directory references?
- Are free-map bits consistent with inode pointers?
- Are directories well-formed?

fsck may repair by:

- Freeing orphaned blocks.
- Moving files to a recovery directory.
- Fixing counts.
- Rebuilding free maps.

## fsck Limitations

Problems:

- Full-disk scans are slow.
- It may not know the user's intended operation.
- Some inconsistencies are hard to repair correctly.
- Data loss can still happen.

Repair after crash is reactive: it fixes damage after the fact.

## Approach 2: Ordered Writes

Ordered writes try to arrange disk writes so the file system is always recoverable.

General idea:

- Write blocks in a safe order.
- Ensure metadata does not point to uninitialized or unallocated data.

Example:

- Write data block before writing inode pointer to it.
- Update allocation metadata in an order that avoids dangerous references.

## Problems With Ordered Writes

Problems:

- Requires careful reasoning for every operation.
- Can reduce performance because writes must wait for earlier writes.
- Complex operations may involve many dependencies.
- Disk/block cache reordering can violate assumptions unless controlled with flushes/barriers.

Ordered writes improve consistency but can be hard to get right.

## Approach 3: Write-Ahead Logging

Write-ahead logging, or journaling, records intended metadata changes in a log before applying them to their final locations.

Basic idea:

1. Write a description of changes to the log.
2. Ensure the log reaches disk.
3. Apply changes to home locations.
4. After crash, replay committed log records if needed.

The log lets the OS recover a consistent set of updates.

## Log Entries and Consistent Groups

A log entry describes a change, such as:

- Set this bitmap bit.
- Update this inode field.
- Add this directory entry.

Related updates are grouped so recovery sees them as one operation.

A consistent group should be:

- Fully applied.
- Or not applied.

This approximates atomic multi-block updates.

## Idempotence

Recovery may replay log operations. Therefore, log operations should be idempotent when possible.

Idempotent:

```text
set block 100 allocated
```

Doing it twice has the same result as doing it once.

Not naturally idempotent:

```text
append this entry to the directory
```

Doing it twice can create duplicate entries.

This is why logging design must be careful about how changes are described.

## Logging Advantages

Advantages:

- Faster recovery than full fsck scan.
- Stronger consistency model.
- Groups related metadata updates.
- Common in modern file systems.

## Logging Disadvantages

Disadvantages:

- Extra writes: changes may be written to log and then home location.
- Log space management.
- Complexity.
- Data journaling can be expensive if file contents are logged too.

Many systems log metadata but not all file data to balance performance and consistency.

## Delayed Log Writes

Delayed writes improve performance by batching updates, but increase the amount of recent work that can be lost.

Tradeoff:

- Write immediately: more durable, slower.
- Delay writes: faster, but more recent changes may disappear after crash.

This is the same durability/performance tradeoff that appeared in earlier practice problems.

## Remaining Problems

Even with logging:

- Hardware can fail.
- Drives can lie about when data is durable.
- Logs can fill.
- Data blocks may not be journaled.
- Application-level consistency may require `fsync` or careful protocols.

Crash recovery improves reliability, but it does not make storage magical.

## Core Takeaways

- File systems need crash recovery because disk data must persist.
- Crashes can leave metadata updates half-complete.
- fsck repairs after the fact but can be slow and imperfect.
- Ordered writes prevent some bad states but constrain performance.
- Write-ahead logging records changes before applying them and enables faster recovery.
- Durability and performance are in tension.

## Study Checks

- Why is file-system crash recovery harder than restarting most OS state?
- What is an example of a multi-block file-system update?
- What does fsck check?
- Why can ordered writes hurt performance?
- What does write-ahead logging guarantee?
- Why does idempotence matter for log replay?

---

# Lecture 25: Truth, Trust, and Technology

Lecture 25 returns to trust, but now in a broader social and technology setting. This material is exam-relevant because CS111 finals often include short-answer ethics/trust questions, not only systems mechanics.

## Trust Refresher

Trust means being willing to be vulnerable to another party because you expect that party to act in an important way, even though you cannot fully monitor or control them.

In CS111, trust can be established by:

- Assumption: "I trust this because I choose to." This is weak and risky.
- Inference: "I have evidence this deserves trust." This is usually strongest.
- Substitution: "I trust this because it is backed by something else I trust." For example, trusting software because it was signed by a vendor you trust.

Trust extends agency. You can do more because you rely on other people, systems, tools, and institutions. But it is dangerous when misplaced.

Two recurring failure modes:

- Over-trust: the trustor extends trust beyond what the evidence justifies.
- Untrustworthiness: the trustee fails to show reliability, integrity, or care.

## Societal Conflicts and Truth

The lecture argues that many societal conflicts depend on incompatible beliefs about basic facts. People must rely on sources because they cannot personally verify everything about elections, crime, the economy, climate, science, health, or technology.

That creates a trust problem:

- Different groups trust different sources.
- Some sources must be wrong or untrustworthy.
- People often use poor inference methods to decide whom to trust.

## Confirmation Bias

Confirmation bias means giving more trust to information that agrees with what you already believe.

Exam phrasing may ask you to identify why a person or organization over-trusted a source. A strong answer should mention:

- What the person wanted to believe.
- What evidence they ignored.
- What independent validation they failed to seek.
- Why the source did not deserve the amount of trust it received.

## False Trust in Numbers

False trust in numbers means treating popularity as truth:

- "Lots of people shared this, so it must be true."
- "The algorithm keeps showing me this, so it must be important."
- "Many outputs say the same thing, so they must be independent confirmation."

This is especially dangerous when one false claim is copied, reposted, or regenerated many times. Quantity can look like independent evidence even when it came from one weak source.

## Social Media Algorithms

Social platforms optimize for attention because attention drives revenue. Content that confirms beliefs, provokes anger, or triggers fear often gets more engagement.

Important takeaways:

- Likes, shares, comments, and views are not truth signals.
- Algorithmic recommendation is not the same as editorial validation.
- Users may see a distorted evidence stream.
- Different users may see different realities.
- Platforms can profit from confirmation bias.

## Generative AI

Generative AI can be useful, but it often sounds confident even when wrong. That creates over-trust because:

- The writing sounds authoritative.
- Explanations are detailed.
- The tool gives concrete-sounding facts.
- Errors may appear without warning.
- AI output can be embedded in other tools, obscuring where the claim came from.

The safe mental model is:

- Treat AI output as hypotheses.
- Verify factual claims independently.
- Use substitution: rely on primary sources, official records, reproducible evidence, or trusted institutions.

## Synthetic Media

Photos, video, and audio historically carried trust because convincing fabrication was hard. Deepfakes and synthetic media weaken that inference.

The danger is not only that fake media may be believed. It is also that real media may be dismissed as fake. This can lead to a collapse of epistemic trust: people stop believing evidence can reliably show what happened.

Potential mitigations:

- Better labeling of AI-generated content.
- Detection tools, with awareness that detection is an arms race.
- Stronger journalistic and institutional verification.
- Legal and technical protections against unauthorized voice/image cloning.
- Cross-checking media against independent sources.

## How To Answer Ethics/Trust Questions

Use this structure:

1. Identify the trustor and trustee.
2. Say what was trusted.
3. Say whether trust was established by assumption, inference, or substitution.
4. Identify over-trust or untrustworthiness.
5. Point to the missing validation.
6. Propose a concrete improvement.

Example:

> The team over-trusted the generated report because it sounded authoritative and agreed with their expectations. That is weak inference plus confirmation bias. They should have validated the claims against primary logs, reproducible tests, or an independently trusted source before acting on it.

## Exam Questions To Do

- `Final-Exam-Ethics-Practice.pdf`: all questions, then compare with `Final-Exam-Ethics-Practice-Solutions.pdf`.
- `Spr2024FinalSolution.pdf` Problem 2(b)-(c): inferring trust in closed-source and open-source operating systems.
- `Spr2023FinalSolution.pdf` Problem 10: how crash recovery substitutes for trust in file-system correctness.

## Study Checks

- What is the difference between trust by assumption, inference, and substitution?
- What is over-trust?
- Why are social-media engagement metrics not truth metrics?
- Why is generative AI especially easy to over-trust?
- Why do deepfakes threaten trust even when people know deepfakes exist?

---

# Lecture 26: Flash Memory

Flash memory is nonvolatile storage used inside SSDs, phones, laptops, and USB drives. It behaves differently from both magnetic disks and DRAM, so the OS and storage stack need extra machinery to use it well.

## Flash Compared With Disks and DRAM

Compared with magnetic disks:

- Flash has no moving parts.
- Random access latency is much lower.
- It is more shock-resistant.
- It costs more per bit.

Compared with DRAM:

- Flash is nonvolatile.
- It is cheaper per bit.
- It is much slower.
- It wears out after enough erase cycles.

## Flash Pages and Erase Units

Flash is read and written in pages, often 4 KB to 16 KB.

But erase happens at a larger granularity: the erase unit, often 1 MB to 8 MB.

This mismatch is the core weirdness:

- Reads are page-sized and relatively fast.
- Writes can change bits from `1` to `0`.
- To change bits back from `0` to `1`, the device must erase an entire erase unit.
- Erasing is slow and wears out the device.

## Write Asymmetry

Flash writes are asymmetric:

- `1 -> 0` is relatively fast.
- `0 -> 1` requires erasing the containing erase unit first.

You can think of writing as a bitwise AND operation: writing can clear more bits, but cannot freely set cleared bits back to 1.

## Why Existing File Systems Need an FTL

Traditional file systems assume a disk-like interface:

- Logical block 0, logical block 1, logical block 2, ...
- Read any block.
- Overwrite any block in place.

Flash does not naturally support cheap in-place overwrite. SSDs hide this using a Flash Translation Layer, or FTL.

The FTL exports a disk-like block interface:

- The OS writes logical block `N`.
- The FTL stores it in some physical flash page.
- The FTL maintains a map from logical block number to physical page.

## Direct-Mapped FTL

A naive direct-mapped FTL stores logical block `N` in physical page `N`.

Read:

- Read physical page `N`.

Write:

- Read the erase unit containing page `N`.
- Erase the erase unit.
- Rewrite the erase unit with the updated page.

Problems:

- Every overwrite becomes very slow.
- Repeated writes wear out the same erase unit.
- Crashes during erase/rewrite can lose old data.

## Better FTL: Out-of-Place Writes

Better FTLs separate logical identity from physical location.

On write:

1. Find a free erased page.
2. Write the new version there.
3. Update the logical-to-physical map.
4. Mark the old physical page as garbage.

This avoids immediate erase on every write, but it creates garbage pages that must eventually be cleaned up.

## Page Header Bits: A-W-G

One approach stores metadata in each flash page:

- `A`: allocated bit.
- `W`: written bit.
- `G`: garbage bit.

Because erased flash bits start as `1`, state transitions are designed to move from `1` to `0`.

Useful lifecycle:

```text
A W G
1 1 1   erased/free
0 1 1   allocated but not yet written
0 0 1   successfully written/live
0 0 0   garbage/dead version
```

The `0 1 1` state helps detect crashes that occur while a block is being written.

On startup, the FTL can scan page headers to rebuild the map.

## Garbage Collection

Out-of-place writes leave old versions behind. These pages are garbage: they cannot be reused until their erase unit is erased.

Garbage collection:

1. Pick an erase unit with many garbage pages.
2. Copy any live pages out to clean pages elsewhere.
3. Update the map.
4. Erase the old erase unit.

This creates extra writes, because live pages must be rewritten even though the OS did not ask to rewrite them.

## Write Amplification

Write amplification measures how much physical writing happens per logical write.

If an erase unit has utilization `U`, then only `1 - U` of it becomes usable for new writes after garbage collection. The simplified write amplification formula is:

```text
write amplification = 1 / (1 - U)
```

Examples:

```text
U = 0.50  -> 2x
U = 0.90  -> 10x
U = 0.99  -> 100x
```

High utilization makes garbage collection expensive. Low utilization improves write performance but wastes capacity.

## Hot and Cold Data

FTLs try to exploit locality:

- Hot data changes often.
- Cold data changes rarely.

Putting hot data together makes it more likely that an erase unit will become mostly garbage, which is good for garbage collection. Mixing hot and cold data is bad because cold live pages keep getting copied during garbage collection.

## Wear-Leveling

Erase units wear out after many erase cycles. If the same hot erase units are reused constantly, they fail early.

Wear-leveling spreads erase cycles across the device.

Sometimes the FTL deliberately garbage-collects cold erase units even if it does not recover much space, because this moves cold data away and frees a less-worn erase unit for future writes.

## TRIM

The FTL does not automatically know when the file system has deleted a file. From the FTL's perspective, a logical block may still look live until overwritten.

TRIM lets the OS tell the SSD that certain logical blocks no longer contain useful data. This reduces unnecessary copying during garbage collection.

## Exam Questions To Do

- `Spr2023FinalSolution.pdf` Problem 9(e): performance degradation in flash translation layers as free space falls.
- `Spr2024FinalSolution.pdf` Problem 11(e): replacing a thrashing paging disk with flash.
- Review `Spr2024FinalSolution.pdf` Problem 8 for the performance/consistency/durability tradeoff, then compare with flash's performance and durability tradeoffs.

## Study Checks

- Why can flash not cheaply overwrite pages in place?
- What does the FTL map?
- Why does garbage collection create write amplification?
- Why is `U = .99` terrible for writes?
- What is wear-leveling?
- Why does TRIM help SSDs?

---

# Lecture 27: Virtual Machines

A virtual machine makes a process-like execution environment look like a whole physical machine. Instead of running just an application, a VM can run a complete guest operating system and its applications.

## Process Abstraction vs Machine Abstraction

A normal process receives a restricted abstraction:

- User-mode CPU instructions.
- Virtual memory.
- System calls for files, processes, networking, and other services.
- No direct control over privileged hardware.

A virtual machine receives something that looks much more like hardware:

- CPU instructions and registers.
- Privileged operations, at least as virtualized behavior.
- Physical memory, from the guest's point of view.
- I/O devices.
- Traps and interrupts.
- An MMU interface.

## Hypervisor / Virtual Machine Monitor

The software that provides virtual machines is a hypervisor, also called a virtual machine monitor.

Terms:

- Host: the real machine and software layer controlling hardware.
- Guest OS: the operating system running inside the VM.
- VM: the guest's private machine abstraction.
- Hypervisor/VMM: the layer that multiplexes real hardware among VMs.

## Hosted VMM

A hosted VMM runs on top of a conventional host operating system. The host OS controls hardware, and the VMM is an application or service that creates virtual machines.

This is different from a bare-metal hypervisor, which directly controls the hardware.

## Simulation

One implementation approach is full simulation:

- Simulate CPU instructions.
- Simulate physical memory as a large array.
- Simulate I/O devices.
- Simulate kernel/user mode, interrupts, and trap behavior.

This is flexible but slow, especially for CPU and memory.

## Direct Execution and Trap-and-Simulate

A faster approach is direct execution:

- Run the guest OS mostly on the real CPU.
- Run it in a less-privileged mode than the hypervisor.
- Let ordinary instructions execute directly.
- Make privileged instructions trap into the hypervisor.

When a privileged instruction traps, the hypervisor:

1. Inspects the trapped instruction.
2. Updates virtual machine state as if the instruction executed.
3. Returns control to the guest.

This is called trap-and-simulate.

Example: `CLI`

- Real `CLI` disables interrupts, so user-mode guest code cannot execute it directly.
- The CPU traps to the hypervisor.
- The hypervisor records that interrupts are disabled for this virtual CPU.
- The guest resumes after the `CLI`.

## System Calls Inside a VM

From an application inside a VM:

1. The application executes a syscall instruction.
2. The hypervisor intercepts or mediates the trap.
3. The guest OS handles the syscall in its virtual kernel mode.
4. The guest OS eventually executes `sysret`.
5. The hypervisor simulates the return to guest user mode.

The point is that the guest OS thinks it controls trap handling, but the real machine's trap handling must remain under hypervisor control.

## Virtual I/O Devices

Guest OSes expect devices:

- Disks.
- Network cards.
- Timers.
- Displays.
- Interrupt controllers.

The hypervisor can expose virtual devices. Reads and writes to virtual device registers trap to the hypervisor, which simulates device behavior. On completion, the hypervisor simulates an interrupt to the virtual CPU.

Paravirtualization means modifying the guest OS or drivers to use hypervisor-aware interfaces, reducing expensive traps.

## Virtualizing Memory

There are three address spaces to keep straight:

- Guest virtual address: what an application inside the VM uses.
- Guest physical address: what the guest OS thinks is physical memory.
- Machine address: real physical memory on the host.

The guest OS manages guest virtual -> guest physical mappings.

The hypervisor manages guest physical -> machine mappings.

## Shadow Page Maps

Without hardware support, the hypervisor may maintain shadow page tables that directly map guest virtual addresses to machine addresses.

The real MMU uses the shadow page tables, while the guest OS thinks it is managing normal page tables.

Downside: shadow page tables can be expensive to maintain because guest OS page-table updates must be trapped and reflected into the shadow structures.

## Hardware Support for VM Memory

Modern x86-64 CPUs add another level of page-table translation:

- Guest page table: guest virtual -> guest physical.
- Hypervisor page table: guest physical -> machine physical.

This reduces the overhead of shadow page tables because the hardware can compose the translations.

## Why VMs Matter

Virtual machines encapsulate execution state. That makes them useful for:

- Running different operating systems on one machine.
- Testing software on multiple OS versions.
- Isolating applications in data centers.
- Consolidating underused servers.
- Cloud computing.
- Saving, copying, moving, and restoring whole machine states.

## Exam Questions To Do

- Review `Spr2024FinalSolution.pdf` Problem 1(a): the dispatcher is OS code, not a thread. This helps distinguish normal OS execution from VM/hypervisor execution.
- Use `Spr2023FinalSolution.pdf` Problem 11 and `Spr2024FinalSolution.pdf` Problem 9 as page-table practice before studying virtualized memory.
- If a final asks about VMs conceptually, be ready to explain trap-and-simulate, virtual I/O, and the three memory address spaces.

## Study Checks

- What does a VM provide that a normal process does not?
- What is the role of the hypervisor?
- Why is full simulation slow?
- What does trap-and-simulate mean?
- What are guest virtual, guest physical, and machine addresses?
- Why did hardware vendors add memory virtualization support?

---

# Lecture 28: Course Review

Lecture 28 is a map of the whole course. For the final, this is the cleanest way to see how topics connect.

## Concurrency Management

Final-relevant concurrency topics:

- Processes and threads.
- Forking, waiting, and exec.
- Dispatching and context switching.
- Thread states: ready, running, blocked.
- Race conditions.
- Locks/mutexes.
- Condition variables.
- Monitors.
- Lock implementation with atomic operations and interrupt control.
- Deadlock conditions, detection, and prevention.
- Scheduling: time slices, round robin, priorities, 4.4 BSD scheduling.

Finals often test concurrency in two ways:

- Conceptual T/F: "Does preemption release locks?" No.
- Design/coding: build monitor state and wait predicates for a synchronization problem.

## Memory Management

Final-relevant memory topics:

- Linking and loading.
- Static vs dynamic linking.
- Process memory layout: text, data, heap, stack.
- Dynamic allocation, fragmentation, leaks, dangling pointers.
- Static relocation.
- Dynamic relocation with base and bound.
- Segmentation.
- Paging.
- x86-64 multi-level page tables.
- TLBs.
- Demand paging.
- Page replacement: FIFO, random, MIN, LRU, clock.
- Local vs global replacement.
- Thrashing.

Finals often test memory by asking:

- Which fields must be in a TLB?
- How many page-table pages are needed?
- Whether a page-size change helps TLB behavior.
- Whether a replacement policy behaves more like LRU.
- Whether a workload is thrashing and what would actually help.

## Storage Management

Final-relevant storage topics:

- Disk mechanics and I/O.
- Programmed I/O, interrupts, DMA.
- Files as persistent named byte sequences.
- Sequential and random access.
- Inodes.
- Contiguous, linked, FAT, and multi-level indexed layouts.
- Unix V6 small and large files.
- Block-size tradeoffs.
- Free-space structures.
- Buffer cache and delayed writes.
- Disk scheduling.
- Directories.
- Hard links and symbolic links.
- Crash recovery: fsck, ordered writes, write-ahead logging.
- Flash memory and FTLs.

Finals often test storage by asking:

- What changes when block size changes?
- How many blocks must be touched?
- Where filenames live and why that matters.
- How hard links and symbolic links behave after deletion/recreation.
- How to write C code that walks inodes, indirect blocks, or directories.
- How caching, delayed writes, ordered writes, and WAL affect performance, consistency, and durability.

## Major Ideas

Virtualization:

- Threads virtualize CPU execution.
- Address spaces virtualize memory.
- Files virtualize storage.
- Virtual machines virtualize the whole hardware interface.

Atomicity:

- Locks make critical sections appear indivisible.
- File-system transactions make multi-block updates recoverable.
- Hardware atomic operations make locks possible.

Locality:

- Recent CPU usage predicts scheduling behavior.
- Recently used pages may be used again.
- TLBs exploit locality in address translation.
- File caches exploit locality in file access.
- Flash FTLs use hot/cold locality to reduce garbage collection cost.

Layering:

- Applications rely on OS abstractions.
- File systems rely on block devices.
- SSDs hide flash details behind FTLs.
- VMs hide real hardware behind virtual hardware.

## Final Exam Strategy

The final is broad. Do not study it as disconnected facts. Study by problem shape:

- If the problem has threads waiting: write shared state and predicates.
- If the problem has virtual addresses: split address into indexes and offset.
- If the problem has files: identify inode, directory entry, data block, indirect block, free map, and reference count changes.
- If the problem has a crash: list which writes made it to disk and which did not.
- If the problem asks performance/consistency/durability: treat those as three separate axes.
- If the problem asks trust: name the trustor, trustee, evidence, and failure mode.

## Exam Questions To Do

Do the final materials in this order:

1. `CS111 Practice Final.pdf`, then `CS111-Practice-Final-Solutions.pdf`.
2. `Spr2024Final.pdf`, then `Spr2024FinalSolution.pdf`.
3. `Spr2023Final.pdf`, then `Spr2023FinalSolution.pdf`.
4. `Spr2022Final.pdf`, then `Spr2022FinalSolution.pdf`.
5. `Final-Exam-Ethics-Practice.pdf`, then `Final-Exam-Ethics-Practice-Solutions.pdf`.

For each final, mark mistakes by topic, not by problem number. The topics that repeat across years are the ones to drill.

---

# Final Review Map

## The Course Story So Far

CS111 so far can be understood as a sequence:

1. Operating systems exist to manage hardware and provide abstractions.
2. Programs execute as processes and threads.
3. The OS dispatches threads onto CPU cores.
4. Shared state creates concurrency bugs.
5. Locks and condition variables coordinate threads.
6. Locks require lower-level atomicity to implement.
7. Multiple locks can deadlock.
8. Scheduling chooses which ready thread should run.
9. Processes have structured memory layouts created by compilers, linkers, and loaders.
10. Dynamic allocators manage unpredictable memory use.
11. Trust is necessary because the OS and its software supply chain cannot be fully monitored by each user.
12. Virtual memory gives each process its own address-space abstraction.
13. Paging translates virtual pages to physical page frames and enforces protection.
14. Demand paging lets the OS keep only active pages in memory and handle missing pages with page faults.
15. Disks provide persistent block storage but have high latency and strong sequential/random access differences.
16. File systems turn disk blocks into named durable files.
17. Inodes, directories, links, caches, and free-space structures organize persistent data.
18. Crash recovery keeps file-system metadata consistent across failures.
19. Technology changes what evidence is trustworthy, so systems people must reason about trust explicitly.
20. Flash memory forces storage systems to handle erase-before-write behavior, garbage collection, write amplification, and wear.
21. Virtual machines extend OS virtualization from one process to an entire hardware-like machine.

## Most Important Definitions

Process:

- A running program plus its resources and execution environment.

Thread:

- A stream of execution within a process.

System call:

- Controlled entry from user code into the kernel.

Kernel:

- Privileged OS core that manages hardware and protection.

Dispatcher:

- OS code that chooses/switches the currently running thread. It is not itself a special thread.

Context switch:

- Saving one thread's execution state and restoring another's.

Race condition:

- Correctness depends on timing or interleaving.

Critical section:

- Code accessing shared state that requires synchronization.

Lock/mutex:

- Mechanism for mutual exclusion.

Condition variable:

- Mechanism for sleeping until a shared-state predicate may be true.

Deadlock:

- Permanent waiting cycle among threads/resources.

Scheduler:

- OS component/policy that chooses which ready thread runs.

Linker:

- Tool that combines object files and resolves symbols.

Heap:

- Region for dynamically allocated objects with arbitrary lifetimes.

Fragmentation:

- Wasted memory due to allocation layout.

Root of trust:

- Component that must be trusted because other protections depend on it.

Virtual memory:

- Address-space abstraction where programs use virtual addresses that the MMU translates to physical addresses.

MMU:

- Hardware that translates virtual addresses and enforces memory permissions.

Base and bound:

- Dynamic translation scheme where physical address equals base plus virtual address if the virtual address is within bounds.

Segmentation:

- Translation scheme with multiple variable-sized regions, each with its own base, bound, and permissions.

Paging:

- Translation scheme that maps fixed-size virtual pages to fixed-size physical page frames.

Page table:

- Data structure that maps virtual page numbers to physical page numbers and permissions.

TLB:

- Hardware cache of recent virtual-to-physical address translations.

Page fault:

- Trap caused by accessing a virtual page that is not currently present or not legally accessible.

Demand paging:

- Virtual-memory strategy that loads pages into memory only when needed.

Backing store:

- Disk space used to hold pages that are not currently in physical memory.

Inode:

- File-system metadata object that records file attributes and pointers/indexes to data blocks.

i-number:

- Index identifying an inode within a file system.

Directory:

- File-system structure mapping names to inode numbers.

Hard link:

- Directory entry that maps another name to an existing inode.

Symbolic link:

- Special file containing a path to another file or directory.

Block cache:

- In-memory cache of disk blocks.

Write-ahead logging:

- Crash-recovery technique that records intended metadata changes in a log before applying them to home locations.

FTL:

- Flash Translation Layer; maps logical disk blocks to physical flash pages and manages garbage collection and wear-leveling.

Write amplification:

- Extra physical flash writes caused by garbage collection and relocation of live pages.

Hypervisor:

- Software layer that provides virtual machines and multiplexes real hardware among guest operating systems.

Guest physical address:

- Address that a guest OS thinks is physical memory; the hypervisor maps it to real machine memory.

Trust by inference:

- Trust based on evidence, tests, history, inspection, or other observations.

Trust by substitution:

- Trust based on something else already trusted, such as signatures, vendors, institutions, or recovery mechanisms.

## High-Value Exam/Quiz Patterns

### Process Creation

Be able to trace:

```cpp
pid_t pid = fork();
if (pid == 0) {
    execvp(...);
} else {
    waitpid(pid, ...);
}
```

Know what runs in parent vs child.

### Thread State Transitions

Be able to explain:

- Running to blocked: thread waits for I/O/lock/CV.
- Running to ready: preempted by timer.
- Ready to running: scheduler selects it.
- Blocked to ready: event occurs.

### Race Conditions

Look for:

- Shared state.
- Concurrent access.
- At least one write.
- Missing or inconsistent synchronization.

### Condition Variables

Always check:

- Is there a mutex?
- Is the predicate protected by that mutex?
- Is `wait` inside a `while` loop?
- Is shared state changed before notification?

### Deadlock

Check four conditions:

- Mutual exclusion.
- Hold and wait.
- No preemption.
- Circular wait.

To fix, usually break circular wait with lock ordering.

### Scheduling

For scheduling examples, compute:

- Arrival time.
- Run time.
- Completion time.
- Waiting time.
- Response time.
- Turnaround time.

Then compare FIFO, round robin, and SRPT.

### Memory Layout

Be able to place:

- Code in text segment.
- Globals/statics in data segment.
- Locals and parameters on stack.
- `new`/`malloc` objects on heap.
- Pointer variables wherever they are declared.

### Allocation

For allocator questions, ask:

- How is free space represented?
- How is a block chosen?
- Is the block split?
- Are adjacent free blocks coalesced?
- What fragmentation can occur?

### Virtual Memory and Paging

For address-translation questions, compute:

- Page size.
- Offset bits.
- Virtual page number bits.
- Physical page number bits.
- Page table entry size.
- Entries per page map.
- Bits translated per page-table level.
- Number of levels.

Then identify which bits index each level and which bits become the page offset.

### File Systems

For file-system questions, ask:

- What metadata changes?
- Which disk blocks are read or written?
- Is the operation updating an inode, directory, free map, data block, or log?
- What happens if the system crashes halfway through?
- Is the design optimizing small files, large files, sequential access, random access, reliability, or space efficiency?

## Minimal Study Checklist

You are in decent shape if you can answer these without notes:

- What is the difference between a process and a thread?
- What happens during `fork`, `exec`, and `wait`?
- What is saved during a context switch?
- Why does shared mutable state cause nondeterminism?
- What does a lock guarantee?
- Why do condition variables require a loop?
- Why does disabling interrupts only solve lock implementation on one core?
- What are the four deadlock conditions?
- What is the difference between deadlock and starvation?
- How do FIFO, round robin, and SRPT differ?
- Where do stack, heap, global, and code objects live?
- What does the linker do?
- Why is heap allocation harder than stack allocation?
- Why is the OS kernel a root of trust?
- What are the goals of virtual memory?
- How does base-and-bound translation work?
- What problem does segmentation solve, and what problem remains?
- How does paging translate a virtual address to a physical address?
- Why do multilevel page tables save space?
- What does a TLB cache?
- What causes a page fault?
- How does Clock approximate LRU?
- What does thrashing look like, and what actually helps it?
- Which fields belong in a TLB entry?
- Why are disks slow for random access?
- What is stored in an inode?
- How does pathname lookup use directories?
- What is the difference between hard and symbolic links?
- Why does file-system crash recovery need logging or repair?
- How do delayed writes, ordered writes, and WAL differ on performance, consistency, and durability?
- Why does flash need an FTL?
- What causes write amplification?
- What is wear-leveling?
- What does a hypervisor do?
- What is trap-and-simulate?
- How do guest virtual, guest physical, and machine addresses differ?
- How do you answer a trust/ethics question without hand-waving?

---

# Appendix: Locks and Synchronization From Sections

This appendix is the practical version of the synchronization unit. It is aimed at the section problems and the large exam coding problems, where the task is usually to design a small monitor that coordinates many threads.

## The Monitor Style

A monitor is an object whose shared state is protected by one lock.

For this class, the expected style is:

- Put all synchronization state inside the class/object.
- Use exactly one mutex for that object unless the problem explicitly suggests otherwise.
- Every method that reads or writes shared state acquires the mutex.
- Threads wait on condition variables when they cannot proceed.
- The object should work independently if several instances exist.

Skeleton:

```cpp
class Thing {
public:
    Thing();
    void method1();
    void method2();

private:
    std::mutex mutex_;
    std::condition_variable cv_;

    // Shared state protected by mutex_.
    int count_;
    bool active_;
};
```

Method pattern:

```cpp
void Thing::method1() {
    std::unique_lock<std::mutex> lock(mutex_);

    while (!can_proceed()) {
        cv_.wait(lock);
    }

    // Update shared state while lock is held.

    cv_.notify_one();   // or notify_all()
}
```

Do not think of the condition variable as storing the condition. The condition is the Boolean expression over the shared state. The condition variable is only a sleeping/wakeup mechanism.

## `std::unique_lock`

Sections emphasize `std::unique_lock` because it prevents a common bug: forgetting to unlock before returning.

Without a lock guard:

```cpp
void foo() {
    mutex_.lock();
    if (some_case) {
        mutex_.unlock();
        return;
    }
    mutex_.unlock();
}
```

With `std::unique_lock`:

```cpp
void foo() {
    std::unique_lock<std::mutex> lock(mutex_);
    if (some_case) {
        return;
    }
}
```

When `lock` goes out of scope, its destructor unlocks the mutex. This is why exam solutions often start methods with:

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

## Condition Variable Semantics

The custom section API for Assignment 4 is:

```cpp
class Mutex {
public:
    void lock();
    void unlock();
    bool mine();
};

class Condition {
public:
    void wait(Mutex &m);
    void notify_one();
    void notify_all();
};
```

It has the same conceptual behavior as the standard-library condition variables used in exam solutions.

`wait(m)` does three things:

1. Unlocks `m`.
2. Blocks the current thread.
3. Re-locks `m` before returning after wakeup.

This matters because a waiting thread must not keep the lock while asleep. If it did, no other thread could acquire the lock to change the shared state that would make the waiter proceed.

`notify_one()` wakes one blocked thread, if any.

`notify_all()` wakes all blocked threads on that condition variable.

## Why Waits Must Be in a Loop

Use:

```cpp
while (!condition) {
    cv.wait(lock);
}
```

Not:

```cpp
if (!condition) {
    cv.wait(lock);
}
```

Reasons:

- Waking up does not prove the condition is true.
- Another thread may run first and consume the resource.
- `notify_all` wakes threads whose conditions may still be false.
- Some systems allow spurious wakeups.

The loop means: "I will sleep while my predicate is false, and every time I wake up I will re-check reality."

## How To Design a Monitor

For each synchronization problem, answer these in order.

1. What is the shared state?

Examples:

- Number of waiting passengers.
- Number of open seats.
- Which direction cars are crossing.
- Inventory count per item.
- Whether a car is currently merging.

2. What is the invariant?

Examples:

- Cars on a one-lane bridge all travel in the same direction.
- Inventory cannot go negative.
- Only one car can merge at a time.
- A train does not leave while passengers are still boarding.

3. Who waits?

Examples:

- Passenger waits for a train with an available seat.
- Car waits until the bridge is safe for its direction.
- Order waits until all requested inventory is available.

4. What exact predicate lets the waiter proceed?

Write this as a Boolean expression over shared state.

Examples:

```cpp
available_seats_ > 0
```

```cpp
crossing_wb_ == 0 && !(waiting_wb_ > 0 && consecutive_eb_ >= 5)
```

```cpp
inventory[id] >= requested_count
```

For multi-resource requests, the predicate may require a loop over arrays.

5. Who changes that predicate?

The thread that changes the state should usually notify.

Examples:

- `add()` changes inventory, so it notifies waiting orders.
- `leave_eb()` decreases eastbound cars on bridge, so it may notify westbound waiters.
- `boarded()` decreases boarding passengers, so it may let the train leave.

## CalTrain Pattern

The CalTrain section problem has two kinds of threads:

- Train thread calls `load_train(available)`.
- Passenger thread calls `wait_for_train()`, then later `boarded()`.

The tricky parts:

- Do not overbook the train.
- Once a passenger starts boarding, that seat is no longer available.
- The train must wait for passengers who were allowed to board to actually call `boarded()`.
- Passengers should be able to board concurrently; do not unnecessarily serialize them one-by-one.

Useful shared state:

- `waiting_passengers`: number of passengers waiting at the station.
- `available_seats`: seats currently available on the train.
- `boarding_passengers`: passengers allowed to board but not yet safely on board.

Passenger wait predicate:

```cpp
available_seats > 0
```

When a passenger starts boarding:

```cpp
waiting_passengers--;
available_seats--;
boarding_passengers++;
```

When passenger finishes:

```cpp
boarding_passengers--;
```

Train can leave when:

```cpp
available_seats == 0 || waiting_passengers == 0
```

and:

```cpp
boarding_passengers == 0
```

The second condition is the easy one to forget. "No more seats" does not mean everyone who got a seat is safely on board.

## One-Lane Bridge Pattern

The bridge section problem:

- Eastbound cars call `arrive_eb()` and `leave_eb()`.
- Westbound cars call `arrive_wb()` and `leave_wb()`.
- Cars on the bridge must all travel in the same direction.
- Any number of cars may be on the bridge at once if they are going the same direction.
- Once five cars have passed in one direction, waiting cars in the other direction must get a chance.

Useful shared state:

```cpp
int waiting_eb_;
int waiting_wb_;
int crossing_eb_;
int crossing_wb_;
int consecutive_eb_;
int consecutive_wb_;
```

Eastbound can enter if:

- No westbound cars are currently crossing.
- It is not the case that westbound cars are waiting and eastbound has already had too many consecutive crossings.

Predicate:

```cpp
while ((crossing_wb_ > 0) ||
       ((waiting_wb_ > 0) && (consecutive_eb_ >= 5))) {
    done_wb_.wait(lock);
}
```

When eastbound enters:

```cpp
waiting_eb_--;
crossing_eb_++;
consecutive_eb_++;
consecutive_wb_ = 0;
```

When eastbound leaves:

```cpp
crossing_eb_--;
if (crossing_eb_ == 0) {
    done_eb_.notify_all();
}
```

Why `notify_all`? Because multiple cars in the opposite direction may now be eligible to re-check their condition.

The bridge problem teaches a general lesson: if multiple threads are allowed through together, your state must count both "waiting" and "currently inside."

## Party Matching Pattern

The Party section problem:

```cpp
std::string meet(std::string& my_name, int my_sign, int other_sign)
```

Each person has:

- Their own name.
- Their own sign.
- Desired sign of the person they want to meet.

A valid match is mutual:

- Bob has sign 3 and wants sign 6.
- Alice has sign 6 and wants sign 3.
- They can match.

The main design issue is avoiding a global bottleneck where one waiting person prevents unrelated matches.

Useful mental model:

- Store waiters by `(my_sign, other_sign)`.
- When a new person arrives, look for someone waiting in the opposite bucket `(other_sign, my_sign)`.
- If found, exchange names and wake that person.
- If not found, wait in this person's own bucket.

This is the same pattern as exam problems where threads wait in categories and the arriving thread may either match immediately or block.

## Exam Problem Pattern: 2022 Merge

The merge problem requires:

- Only one car merges at a time.
- If many cars are waiting, lanes take turns.
- Not every lane has a waiting car.
- No starvation.

Useful shared state:

```cpp
std::vector<int> waiting;
int totalWaiting;
std::vector<std::condition_variable> safe_to_merge;
int prev_lane;
bool merging;
```

Key idea:

- If nobody is merging, the arriving car can go immediately.
- Otherwise it increments its lane's waiting count and sleeps on that lane's condition variable.
- When a car calls `merged()`, choose the next lane with waiters by rotating from the previous lane.

This is a category-queue problem. Waiters are grouped by lane.

## Exam Problem Pattern: 2023 Flight

The flight problem requires:

- Passengers board in increasing boarding-position order.
- If a position is not present when reached, it is skipped.
- If that skipped passenger arrives later, they must board before higher positions that are still waiting.
- `board_next()` wakes one passenger at most.

Useful shared state:

```cpp
std::map<size_t, std::condition_variable*> ready_passengers;
```

Why a map?

- It keeps keys sorted.
- The lowest waiting position is `ready_passengers.begin()`.

Passenger:

- Creates a local condition variable.
- Inserts pointer into the map under their position.
- Waits.

Gate agent:

- If map is empty, return `false`.
- Otherwise notify the first entry and erase it.

This is an ordered-waiter problem.

## Exam Problem Pattern: 2024 Inventory

The inventory problem requires:

- `add(stock_id, count)` adds inventory.
- `reserve(stock_ids, counts)` blocks until the entire order can be satisfied.
- `reserve` must not reserve anything until the whole order can complete.
- If enough inventory exists for some waiting order, that order must be allowed to proceed.

Useful shared state:

```cpp
std::vector<int> inventory;
std::condition_variable new_inventory;
```

Reserve predicate:

```cpp
bool order_ok = true;
for (size_t i = 0; i < stock_ids.size(); i++) {
    if (inventory[stock_ids[i]] < counts[i]) {
        order_ok = false;
        break;
    }
}
```

Wait loop:

```cpp
while (!order_ok) {
    new_inventory.wait(lock);
    // re-check entire order
}
```

Only after all items are available:

```cpp
for (size_t i = 0; i < stock_ids.size(); i++) {
    inventory[stock_ids[i]] -= counts[i];
}
```

Why `notify_all` in `add()`?

- Different waiting orders may need different stock items.
- The adding thread may not know which waiting order is now satisfiable.
- Waking all waiters lets each re-check its own predicate.

This is a multi-resource predicate problem. The crucial rule is: do not partially reserve while waiting.

## `notify_one` vs `notify_all`

Use `notify_one` when:

- Exactly one waiter should proceed.
- You know the condition variable/category for that waiter.
- Waking more would be wasteful or forbidden.

Examples:

- Wake one selected lane in Merge.
- Wake the lowest boarding position in Flight.

Use `notify_all` when:

- Many waiters may have different predicates.
- A state change may enable more than one thread.
- You do not know which waiter can proceed.
- The problem allows it and simplicity matters.

Examples:

- Inventory `add()` wakes all orders to re-check.
- Bridge direction switch may wake all cars waiting in a direction.

## Common Wrong Solutions

Wrong: Busy-waiting.

```cpp
while (!condition) {
    // keep checking
}
```

This wastes CPU and usually violates the problem statement.

Wrong: Holding the lock while doing long external work.

For example, once a method returns and the simulated car/passenger does real-world work, the monitor lock should not still be held.

Wrong: Updating state after notification.

Usually change shared state first, then notify. Waiters wake up to inspect shared state.

Wrong: Waiting after partially consuming resources when the spec forbids it.

Inventory is the main example. Do not subtract item A, then sleep waiting for item B. That can block other orders and violate the "all or nothing" requirement.

Wrong: Forgetting "in progress" state.

CalTrain needs to count passengers who are boarding but have not called `boarded()`.

Merge needs to remember a car is currently merging.

Bridge needs to count cars currently crossing.

## How to Solve a New Synchronization Problem Under Time Pressure

Use this checklist:

1. List all methods and which threads call them.
2. Circle every phrase like "must block until", "must not return until", "only one", "at most", "in order", "no starvation", "must not use busy-waiting".
3. Define shared state in plain English before writing code.
4. Write each wait predicate as a Boolean condition.
5. Decide whether waiters are one group, categories, ordered keys, or multi-resource requests.
6. Write constructor initialization.
7. In each method, acquire lock first.
8. Update waiting counts before sleeping.
9. Use `while` around waits.
10. Change shared state while holding the lock.
11. Notify the right condition variable after changing state.
12. Check edge cases: no waiters, first arrival, last departure, multiple waiters, skipped categories, and fairness.

If the solution feels clever, simplify it. The section notes explicitly value simple and obvious monitor solutions.
