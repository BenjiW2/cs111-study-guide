# CS111 Catch-Up Guide

This guide is meant to replace "read every slide linearly" with a tighter path through the material. Use the PDFs as the source of truth, but study them through the questions and checkpoints below.

## How to Use This

For each lecture:

1. Read the "big idea" first.
2. Open the lecture PDF and skim the relevant slides.
3. Write answers to the self-check questions without looking.
4. If you cannot answer a question, go back only to the relevant slides.

Do not try to memorize every slide. CS111 is mostly about mechanisms, invariants, and failure cases.

## Unit 1: Why Operating Systems Exist

### Lecture 1: Introduction

File: `Lecture1.pdf`

Big idea: Operating systems exist because raw hardware is hard to use safely and efficiently. The OS gives programs abstractions while managing shared hardware resources.

Learn:

- Why early computers had minimal OS support.
- How batch systems, multiprogramming, timesharing, networking, and modern systems changed OS design.
- What the kernel is.
- The three big course areas: CPU, memory, storage/filesystems.

First-pass skip:

- Course logistics unless you need grading or assignment details.
- Historical hardware names beyond the rough timeline.

Self-check:

- What problem did batch monitors solve?
- Why did timesharing matter?
- What is the kernel responsible for?
- Why is an OS both an abstraction provider and a resource manager?

## Unit 2: Execution: Processes, Threads, and Dispatching

### Lecture 2: Threads, Processes, and System Calls

File: `Lecture2.pdf`

Big idea: The OS decomposes running programs into processes and threads. System calls are the controlled boundary between user code and the kernel.

Learn:

- Thread: the smallest unit of execution.
- Process: one or more threads plus the execution state/resources around them.
- Execution state: registers, stack, program counter, memory mappings, open files, etc.
- System calls: how programs ask the OS to do privileged work.
- `fork`, `exec`, and `wait`.
- Why Unix creates a process by copying and then replacing the child image with `exec`.

Self-check:

- What is the difference between a process and a thread?
- What happens after `fork` in the parent and child?
- Why does `exec` not return on success?
- Why are system calls needed instead of letting programs directly manipulate hardware?

### Lecture 3: Dispatching and Context Switching

File: `Lecture3.pdf`

Big idea: Threads do not magically run. The OS dispatcher chooses a runnable thread, restores its state, and lets it execute on a CPU core.

Learn:

- Cores execute threads.
- Thread states: running, ready, blocked.
- Process/thread control blocks store execution state.
- Dispatcher loop.
- Context switch: save current thread state, choose another thread, restore its state.
- What causes the dispatcher to run: blocking, interrupts, timer preemption, thread creation, exit.

Self-check:

- What information must be saved during a context switch?
- What is the difference between ready and blocked?
- Why does a blocked thread not consume CPU?
- What event lets the OS regain control from a running thread?

## Unit 3: Concurrency and Synchronization

This is the core section. Spend extra time here.

### Lecture 4: Concurrency Basics

File: `Lecture4.pdf`

Big idea: Concurrent code is difficult because shared state makes behavior depend on execution order.

Learn:

- Independent threads vs cooperating threads.
- Nondeterminism.
- Atomic operations.
- Race conditions.
- Critical sections.
- The "too much milk" example as a model for coordination bugs.

Self-check:

- Why are independent threads easier to reason about?
- What makes a sequence of operations non-atomic?
- What is a race condition?
- What is the critical section in the milk example?

### Lecture 5: Locks and Condition Variables

File: `Lecture5.pdf`

Big idea: Locks provide mutual exclusion. Condition variables let threads sleep until shared state changes.

Learn:

- Mutex/lock basics: `lock`, critical section, `unlock`.
- Why locks fix mutual exclusion but not all coordination.
- Producer/consumer buffer.
- Why waiting by spinning or repeatedly checking is bad.
- Condition variables.
- The standard pattern:

```cpp
std::unique_lock<std::mutex> lock(m);
while (!condition_is_true) {
    cv.wait(lock);
}
// use the protected state
```

- Why the condition must be checked in a loop.
- Why monitor-style locking means one lock protects one shared object/invariant.

Self-check:

- What invariant does the producer/consumer buffer maintain?
- Why does `wait` need to release the lock while sleeping?
- Why is `if` usually wrong around a condition-variable wait?
- When should you use `notify_one` vs `notify_all`?

### Lecture 6: Lock Implementation

File: `Lecture6.pdf`

Big idea: Locks themselves need lower-level atomicity from the hardware or from interrupt control.

Learn:

- On one core, disabling interrupts can create a kernel critical section.
- On multiple cores, disabling interrupts on one core is not enough.
- Why multiprocessor locks need hardware atomic instructions.
- Busy waiting/spin locks.
- Why blocking a thread while holding low-level scheduler state must be done carefully.

Self-check:

- Why can disabling interrupts work on a uniprocessor?
- Why does that fail on a multiprocessor?
- What is the difference between spinning and blocking?
- Why must lock implementation avoid races inside the lock itself?

### Lecture 7: Deadlock

File: `Lecture7.pdf`

Big idea: Multiple locks can cause a system to get stuck forever if threads wait in a cycle.

Learn:

- Four deadlock conditions:
  - Mutual exclusion.
  - Hold and wait.
  - No preemption.
  - Circular wait.
- Circular wait as the most practical condition to prevent.
- Lock ordering.
- Deadlock detection vs deadlock prevention.

Self-check:

- Give a two-lock, two-thread deadlock example.
- Which deadlock condition does global lock ordering break?
- Why are multiple locks useful even though they create deadlock risk?
- How can deadlock happen with resources other than locks?

## Unit 4: Scheduling

### Lecture 8: CPU Scheduling

File: `Lecture8.pdf`

Big idea: Dispatching is the mechanism; scheduling is the policy for choosing which thread runs.

Learn:

- FIFO/non-preemptive scheduling.
- Preemption and time slices.
- Round robin.
- Shortest Remaining Processing Time, or SRPT.
- Fairness vs response time.
- Priority scheduling.
- Ready queues and priority queues.
- Multicore scheduling basics.
- Work-conserving schedulers.

Self-check:

- Why can FIFO give terrible response time?
- What does round robin improve?
- Why is SRPT good in theory but hard in practice?
- What is the difference between fairness and minimizing average response time?
- What does work-conserving mean?

## Unit 5: Memory and Linking

### Lecture 9: Linkers and Process Memory Layout

File: `Lecture9.pdf`

Big idea: A running process has a structured memory layout, and the linker/loader pipeline creates that layout from source code and object files.

Learn:

- Process memory regions: code/text, global/static data, heap, stack.
- Where local variables, globals, heap objects, and pointers live.
- Compiler, assembler, linker, loader.
- Object files.
- Symbol resolution.
- Static vs dynamic linking.
- Jump tables/dynamic loader at a high level.

Self-check:

- Where does a local variable live?
- Where does a `new`/`malloc` allocation live?
- What is the linker responsible for?
- What does the loader do?
- Why does dynamic linking exist?

### Lecture 10: Dynamic Storage Management

File: `Lecture10.pdf`

Big idea: Memory allocators manage unpredictable allocation and free patterns while trying to reduce wasted space and overhead.

Learn:

- `allocate(size)` and `free(ptr)`.
- Stack allocation vs heap allocation.
- Free lists.
- Fragmentation.
- First fit vs best fit.
- Slab allocation.
- Bitmaps.
- Storage reclamation.
- Reference counting.
- Mark-and-sweep garbage collection.

Note: `Lecture11.pdf` is identical to `Lecture10.pdf`, so you can skip it unless your instructor later replaces it.

Self-check:

- Why is stack allocation easy to manage?
- Why is heap allocation harder?
- What is fragmentation?
- What tradeoff does a slab allocator make?
- Why can reference counting fail on cycles?
- What are the two phases of mark-and-sweep GC?

## Unit 6: Trust and Operating Systems

### Lecture 12: Trust and Operating Systems

File: `Lecture12.pdf`

Big idea: Users and software depend on operating systems as roots of trust, even though the full system is too complex to personally verify.

Learn:

- Trust as willingness to be vulnerable.
- Trust by assumption, inference, and substitution.
- Why software trust is difficult.
- The OS kernel as a root of trust.
- Why users and developers trust Linux.
- Supply-chain risk, including the xz/ssh attack example.
- AI-generated code policy as a trust question.

Self-check:

- Why is the OS kernel a root of trust?
- What is over-trust?
- How is trust established in open-source software?
- Why was the xz attack dangerous?

## Recommended Catch-Up Schedule

### Day 1: Execution Basics

Do Lectures 1, 2, and 3.

Deliverable: Draw a diagram showing a process with one or more threads, then show how the dispatcher moves threads between ready, running, and blocked.

### Day 2: Concurrency Foundations

Do Lectures 4 and 5.

Deliverable: Explain producer/consumer with a mutex and condition variable. Write the wait loop pattern from memory.

### Day 3: Locks, Deadlock, Scheduling

Do Lectures 6, 7, and 8.

Deliverable: Compare FIFO, round robin, and SRPT on a small example with three jobs. Also write one deadlock example and fix it with lock ordering.

### Day 4: Memory

Do Lectures 9 and 10. Skip Lecture 11 unless it changes.

Deliverable: Draw a process memory layout and label code, globals, heap, and stack. Then explain how a free-list allocator works.

### Day 5: Trust and Review

Do Lecture 12 and revisit weak self-checks.

Deliverable: Write a short answer: "Why do operating systems require trust, and how do real systems try to earn it?"

## What to Prioritize If You Are Short on Time

Highest priority:

- Process vs thread.
- `fork`, `exec`, `wait`.
- Thread states and context switching.
- Race conditions and atomicity.
- Locks.
- Condition variables.
- Deadlock conditions and lock ordering.
- FIFO, round robin, SRPT.
- Process memory layout.
- Heap allocation and fragmentation.

Lower priority on first pass:

- Detailed historical dates.
- Exact C++ syntax beyond synchronization patterns.
- Specific dynamic-linking jump-table mechanics.
- Philosophical trust definitions beyond the core OS trust idea.

## One-Page Mental Model

An OS manages three main things:

- CPU: Which threads run, when they stop, and how they switch.
- Memory: What address space each process sees and how storage inside memory is allocated.
- Trust/protection: Why programs cannot directly control everything, and why the kernel must mediate access.

Most bugs in this unit come from one of these:

- Shared state changed without synchronization.
- A thread went to sleep without a correct wakeup condition.
- Locks were acquired in inconsistent orders.
- The scheduler policy optimized one goal while hurting another.
- Memory was allocated/freed under assumptions that later became false.

