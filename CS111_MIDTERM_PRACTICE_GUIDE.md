# CS111 Past Midterm Practice Guide

This guide distills the three past midterm solution PDFs in this folder:

- `Spr2022MidtermSolution.pdf`
- `Spr2023MidtermSolution.pdf`
- `Spr2024MidtermSolution.pdf`
- `CS111-Practice-Midterm-Solutions.pdf`
- `Midterm review problems.pdf`

Use this after reading the comprehensive notes. The midterms show what the course repeatedly asks you to do under time pressure.

Important: the current-term practice midterm includes file systems, crash recovery/logging, inodes, pipes, `dup2`, and shell-style multiprocessing. Those topics are not covered by the 12 lecture PDFs currently in the `Lectures` folder, so you should treat them as missing material to learn separately.

The additional midterm review problems emphasize two code-heavy patterns: thread-per-task parallelism with a shared map protected by a mutex, and FIFO pair matching with condition variables.

## What the Midterms Emphasize

Across 2022, 2023, and 2024, the repeated themes are:

- Short true/false conceptual explanations.
- Process vs thread memory sharing.
- Scheduling policy and preemption.
- Locks, condition variables, and monitor-style synchronization.
- Dynamic storage management.
- Linkers and memory layout.
- Virtual memory/page table arithmetic.
- Trust models.
- File-system layout, inodes, directories, logging, and crash recovery.
- Unix multiprocessing with pipes and file descriptor management.
- Thread creation using `std::thread`, `std::ref`, vectors of threads, and `join`.
- FIFO pair-matching monitor design.

The biggest point value every year is a monitor-style synchronization implementation in C++ using `std::mutex` and `std::condition_variable`.

## High-Level Exam Pattern

Each exam is about 90 minutes and has roughly this shape:

- Problem 1: true/false with brief explanations.
- Problem 2-4: short conceptual or calculation problems.
- Problem 5: large synchronization coding problem, worth about half the exam.

That means your prep should heavily prioritize:

1. Recognizing conceptual traps quickly.
2. Doing page-table/scheduling arithmetic cleanly.
3. Writing a correct monitor under pressure.
4. Explaining file-system crash-recovery tradeoffs.
5. Tracing file descriptors across `fork`, `pipe`, `dup2`, and `close`.
6. Spawning one thread per independent task and protecting shared outputs.
7. Implementing match-making monitors with per-category queues.

---

# Year-by-Year Summary

## Current-Term Practice Midterm

Main topics:

- File systems and inodes.
- Directory entries and rename.
- Logging and idempotence.
- Crash recovery tradeoffs.
- Multithreaded race outcomes.
- Unix process pipelines.
- `pipe`, `fork`, `dup2`, `close`, `execvp`, and `waitpid`.
- Basic thread synchronization around a shared vector.

Problem highlights:

- Scattering inodes can reduce seeks between inode access and payload access, but it makes inode lookup/allocation more complex.
- Log replay requires idempotent operations. If replaying an operation twice changes the result, crash recovery can corrupt state.
- Doubly indirect indexing supports larger files, but can make small-file payload access require more disk reads.
- Rename works by finding the existing inode, removing the old directory entry, locating or creating the new parent path, then adding a new directory entry pointing to the same inode.
- Crash recovery often trades durability/consistency against performance. More aggressive writes or more complete logging improves recovery but costs I/O.
- Unsynchronized multithreaded code can produce multiple outputs depending on interleaving.
- Pipes require careful closing of unused file descriptors. A reader sees EOF only when all write ends of the pipe are closed.
- A shell pipeline is built with `pipe`, two `fork`s, `dup2` to connect stdin/stdout, `close` to remove unused descriptors, `execvp` in children, and `waitpid` in the parent.

Most important practical implication:

- Your current exam may care more about file systems and Unix process/file-descriptor mechanics than the three older midterms suggest. Do not study only locks and scheduling.

## Midterm Review Problems

Main topics:

- Parallelizing independent work with one thread per item.
- Passing arguments to threads using `std::ref`.
- Protecting a shared `map` with a mutex.
- Joining all spawned threads before using final shared results.
- FIFO pair matching by category.
- Condition variables stored inside waiting records.
- Queue per category using `unordered_map<int, queue<Student*>>`.

Problem 1: News Aggregation

The original program fetches articles sequentially. The review problem asks you to spawn one thread per article:

- Create a `vector<thread>`.
- For each article, push a new `thread`.
- Pass the article, `NewsManager`, shared database map, and database mutex by reference.
- In `fetchArticle`, do the slow fetch outside the lock.
- Lock only around the shared map update.
- Join every thread before calling `handleUserQueries`.

The key race:

- Multiple threads update the same `database` map.
- `database[article] = articleContents` must be protected by `databaseLock`.

The key performance point:

- Do not hold the lock while downloading the article. Fetching is slow and independent; only the map insertion is shared.

Problem 2: Stanford Dorm Selection

The dorm problem is a FIFO pair-matching monitor:

- Each student chooses one dorm id.
- A student blocks until another student chooses the same dorm.
- Both students return the other student's name.
- Pairing must be FIFO per dorm.

Useful state:

```cpp
struct Student {
    std::string my_name;
    std::condition_variable cv;
    std::string match_name;
    bool matched = false;
};

std::mutex m;
std::unordered_map<int, std::queue<Student*>> waiting;
```

Pattern:

- If the queue for this dorm is nonempty, pop the oldest waiting student, fill in their match fields, notify them, and return their name.
- Otherwise, create a local `Student self`, push `&self` into the dorm queue, and wait in a loop until `self.matched`.

The local `Student self` is safe because the waiting call does not return until the match has happened and the queue entry has been popped.

This is the same family as Party/Flight:

- Party: match complementary categories.
- Flight: wake lowest ordered waiter.
- Dorm: match FIFO within the same category.

## Spring 2022

Main topics:

- Critical sections.
- Scheduling independence and synchronization bugs.
- BSD-style priority scheduling.
- Linker symbol tables vs unresolved references.
- Process vs thread design for shells.
- Multilevel page tables.
- Monitor-style synchronization problem: lane merging.

Problem highlights:

- A critical section cannot have multiple threads inside it simultaneously, even on different cores.
- A scheduler change should not change correct program behavior; if an app starts crashing, it likely had a latent synchronization bug.
- The 4.4 BSD scheduler is preemptive.
- A file that uses a symbol has an unresolved reference; the file that defines the symbol has the symbol table definition.
- Running shell commands as threads in the shell process would be bad because of shared memory, file descriptor conflicts, security, and loading issues.
- Page-table arithmetic requires careful accounting of offset bits, index bits, page-map entry size, and number of levels.
- The big coding problem asks you to coordinate cars merging from multiple lanes without starvation.

Big synchronization idea:

- Track how many cars are waiting in each lane.
- Track whether a car is currently merging.
- When the current car finishes, choose the next lane in round-robin order among lanes with waiters.
- Use one monitor lock.
- Use condition variables to block cars until selected.

## Spring 2023

Main topics:

- Locks and thread ownership.
- Threads from one process running on different cores.
- Busy waiting and spin locks.
- Base-and-bound relocation.
- Trust by substitution.
- Dynamic storage management.
- Variable sharing across threads, processes, scopes, and stacks.
- Multilevel page tables.
- Monitor-style synchronization problem: airplane boarding.

Problem highlights:

- A lock can be owned by at most one thread at a time.
- Threads within one process can run on different cores.
- Busy waiting is usually bad, but can be appropriate for very short waits or low-level lock implementation.
- Base-and-bound relocation requires the entire process address space to be contiguous in physical memory.
- Backing up before an OS update is trust by substitution because it limits consequences if the update fails.
- Slab allocation can strand memory in underused slabs.
- Reference counting struggles with cycles.
- Garbage collection can be expensive in time and space.
- A global variable is shared by threads in the same process.
- After `fork`, parent and child have separate copies of memory.
- Local variables in different functions or scopes are distinct variables.

Big synchronization idea:

- Passengers call `wait_to_board(position)` and block.
- Gate agent calls `board_next()`.
- Board the lowest waiting boarding position.
- A `std::map<size_t, condition_variable*>` naturally keeps waiting passengers ordered by position.
- `board_next` notifies only the lowest-position waiter.

## Spring 2024

Main topics:

- STCF/STCF scheduling optimality.
- Dynamic linking.
- Linker section layout.
- Reference counting and dangling pointers.
- Virtual vs physical address sizes.
- Scheduling preemption.
- Trust by substitution through page-table protection.
- Multicore ready queues.
- Page-table architecture design.
- Monitor-style synchronization problem: inventory reservation.

Problem highlights:

- STCF/STCF gives optimal average response time when the scheduler knows completion times.
- Dynamic linking can reduce memory usage because shared library code can be shared.
- Linkers group like sections together; they do not necessarily keep all sections from one object file adjacent.
- Reference counting prevents dangling pointers if counts accurately represent all live pointers, but it may fail to reclaim cycles.
- Virtual and physical addresses need not be the same size.
- First-come-first-served is non-preemptive.
- Round robin preempts at the end of a time slice.
- STCF preempts when a newly ready thread has a shorter completion time than the currently running one.
- Separate per-core ready queues reduce lock contention but can cause load imbalance.

Big synchronization idea:

- Orders request multiple stock items.
- `reserve` must wait until the entire order can be filled.
- It must not partially reserve inventory while waiting.
- Use one monitor lock protecting the inventory vector.
- Use a condition variable for new inventory.
- On `add`, update inventory and `notify_all`, because many different waiting orders may need to recheck their predicates.
- In `reserve`, loop until every requested item has enough inventory, then subtract all counts atomically while holding the lock.

---

# Recurring Conceptual Traps

## Critical Section Trap

Wrong intuition:

- "Two threads can be in a critical section if they are on different cores."

Correct:

- A critical section is defined by mutual exclusion. If two threads can execute it simultaneously, it is not properly protected.

## Scheduler Trap

Wrong intuition:

- "Changing the scheduler should not affect whether a program crashes."

Correct:

- Correctly synchronized programs should behave the same except for timing. Incorrectly synchronized programs may expose hidden races under a new schedule.

## Thread vs Process Trap

Threads in the same process:

- Share globals.
- Share heap.
- Share open files.
- May run on different cores.

Processes after `fork`:

- Have separate address spaces.
- Initially appear copied from the parent.
- Later changes to globals in the child do not change the parent's globals.

## Linker Trap

Definition vs reference:

- The object file that defines a function has the symbol definition.
- The object file that calls the function has an unresolved reference.

Section layout:

- Linkers typically group like sections together, such as all code sections, all data sections, etc.
- They do not necessarily place all material from a single object file together.

## Condition Variable Trap

Wrong:

```cpp
if (!ready) {
    cv.wait(lock);
}
```

Correct:

```cpp
while (!ready) {
    cv.wait(lock);
}
```

Reason:

- Wakeups do not guarantee the predicate is true.
- Another thread may consume the resource first.
- Spurious wakeups are allowed.

## Partial Reservation Trap

For problems like inventory reservation:

- Do not grab some resources, then wait for the rest, unless the problem explicitly allows it.
- If the specification says the operation must not reserve anything until it can complete, first check that all resources are available, then reserve them in one atomic monitor operation.

---

# Virtual Memory and Page Table Crash Course

The past midterms include page-table questions. If your current lecture PDF set does not include the detailed virtual memory lecture, you should still learn this material because it appears repeatedly.

## Basic Page Translation

A virtual address is split into:

- Virtual page number bits.
- Page offset bits.

The page offset selects a byte within a page.

If page size is `2^k` bytes, then:

- Offset size is `k` bits.

Example:

- 4 KB pages = `2^12` bytes, so offset = 12 bits.
- 16 KB pages = `2^14` bytes, so offset = 14 bits.

## Physical Page Number

A physical address is also split into:

- Physical page number.
- Page offset.

If physical addresses are `P` bits and page offset is `k` bits:

- Physical page number size = `P - k` bits.

Example from 2024:

- 42-bit physical addresses.
- 16 KB pages = 14 offset bits.
- Physical page number = `42 - 14 = 28` bits.

## Page Map Entry Contents

A page map entry must contain enough information to translate and protect a page.

At minimum, common fields include:

- Physical page number.
- Present bit.
- Read-only or permission bits.

Other real systems may include:

- Dirty bit.
- Referenced/accessed bit.
- Kernel-only bit.
- Execute-disable bit.

## Multilevel Page Tables

Multilevel page tables break the virtual page number into chunks. Each chunk indexes one page-map level.

Useful facts:

- If a page map is one page large and each entry is `E` bytes, then number of entries per page map is `page_size / E`.
- If there are `2^n` entries, each level consumes `n` bits of virtual page number.

Example:

- Page size = 4 KB = `2^12`.
- Entry size = 8 bytes = `2^3`.
- Entries per page map = `2^12 / 2^3 = 2^9`.
- Each level indexes 9 bits.

## x86-64-Style Example from 2022/2023

Given:

- 48-bit virtual addresses.
- 4 KB pages.
- 4 levels of page maps.
- Each page map has 512 entries.

Then:

- Offset = 12 bits.
- Each level = 9 bits.
- Four levels = `4 * 9 = 36` virtual page number bits.
- Total translated virtual address bits = `36 + 12 = 48`.

If physical addresses are 52 bits:

- Physical page number = `52 - 12 = 40` bits.

## Increasing Page Size

If page size increases, two things happen:

- Offset bits increase.
- Each page map can hold more entries, if entry size stays the same.

Example from 2022:

- Original page size: 4 KB = 12 offset bits.
- Page size increased by factor of 4 = 16 KB = 14 offset bits.
- Page map entries still 8 bytes.
- Page map now holds `2^14 / 2^3 = 2^11` entries.
- Each page-map level covers 11 virtual address bits.

With 3 levels:

- Page-map index bits = `3 * 11 = 33`.
- Offset bits = 14.
- Total virtual address bits translated = `33 + 14 = 47`.

So 3 levels would not cover a 48-bit virtual address space.

## Designing a Page Table from Scratch

Example from 2024:

- Page size: 16 KB = `2^14`.
- Physical addresses: 42 bits.
- Virtual addresses: only lower 50 bits used.
- Page map entry size: 4 bytes.

Calculations:

- Offset bits = 14.
- Virtual page number bits = `50 - 14 = 36`.
- Physical page number bits = `42 - 14 = 28`.
- Entries per page map = `2^14 / 2^2 = 2^12`.
- Each page-map level maps 12 bits.
- Levels needed = `36 / 12 = 3`.

Page map entry must contain:

- 28-bit physical page number.
- Present bit.
- Read-only bit.
- Possibly other permission/status bits.

---

# File Systems and Crash Recovery Crash Course

This section is based on the current-term practice midterm. If you do not have the matching lectures, learn these concepts before taking the practice exam seriously.

## Inodes

An inode is a file-system data structure that stores metadata about a file and pointers/indexes to the file's payload blocks.

An inode commonly stores:

- File size.
- File type.
- Permissions.
- Owner/group.
- Timestamps.
- Link count.
- Block pointers or indexes to payload blocks.

An inode usually does not store the file name. File names live in directories.

## Directory Entries

A directory is a mapping from names to inode numbers.

Directory entry:

- Name.
- Inode number.

This is why renaming a file usually changes directory entries, not the file payload itself.

## Pathname Lookup

Pathname lookup means resolving a path like:

```text
/usr/class/cs111/index.html
```

Conceptually:

1. Start at the root directory inode.
2. Look up `usr` in the root directory payload.
3. Use that inode to look up `class`.
4. Use that inode to look up `cs111`.
5. Continue until the final name is resolved.

This repeated lookup is what the practice solution means by "drilling toward" a pathname.

## Rename

Rename generally works by manipulating directory entries.

High-level steps:

1. Find the inode number of the file being moved.
2. Remove the old directory entry from the old parent directory.
3. Resolve the destination parent directory.
4. Create the destination directory entry with the new name and the existing inode number.

The file's payload blocks do not need to move just because its name changes.

## Inode Placement

Scattering inodes means placing inodes near their payload data instead of keeping all inodes in one central region.

Benefit:

- Fewer disk seeks between reading the inode and reading the file payload.

Drawback:

- More complexity in finding and managing inodes.

This is a locality tradeoff.

## Direct, Indirect, and Doubly Indirect Blocks

A file system needs a way to map a file offset to a disk block.

Direct block pointer:

- Inode points directly to a payload block.
- Good for small files.

Indirect block pointer:

- Inode points to an index block.
- The index block points to many payload blocks.
- Supports larger files.

Doubly indirect block pointer:

- Inode points to an index block.
- That index block points to other index blocks.
- Those index blocks point to payload blocks.
- Supports even larger files.

Tradeoff:

- More levels support larger files.
- More levels can require more disk accesses, especially for small files if the design forces indirection.

## Crash Recovery

File systems must handle crashes that happen midway through updates.

Example danger:

- Directory entry is created but inode is not initialized.
- Inode points to a block that was not written.
- Free block bitmap says a block is free even though a file uses it.

The problem is consistency: related disk updates must appear all done or all not done.

## Logging/Journaling

A log records intended file-system changes before applying them to their final locations.

Recovery idea:

1. After a crash, inspect the log.
2. Determine which operations were committed.
3. Replay committed operations if needed.

The log helps the system recover to a consistent state.

## Idempotence

An operation is idempotent if doing it multiple times has the same effect as doing it once.

Examples:

- "Set block 100's bitmap bit to allocated" can be idempotent.
- "Append this directory entry to the end of this directory" is not naturally idempotent, because replaying it twice may append duplicate entries.

Why it matters:

- During crash recovery, the system may not know whether an operation had already reached disk.
- If replaying an operation twice changes the result, recovery can corrupt the file system.

## Durability vs Performance

More durable design:

- Write data more immediately.
- Log more information.
- Flush more often.
- Lose less after crash.

Cost:

- More disk I/O.
- Lower performance.

More performance-oriented design:

- Buffer writes.
- Delay flushing.
- Log less data.

Cost:

- More recent data may be lost.
- Recovery may be less complete.

Classic example:

- A block cache may delay writes for performance, but a crash can lose recent changes.

---

# Unix Processes, Pipes, and File Descriptors Crash Course

The practice midterm includes shell-style multiprocessing. This is separate from the monitor-style synchronization problems in the older exams.

## File Descriptors

A file descriptor is a small integer handle inside a process.

Standard descriptors:

- `0`: standard input.
- `1`: standard output.
- `2`: standard error.

After `fork`, the child inherits copies of the parent's file descriptors. These descriptors refer to the same underlying open file descriptions or pipe endpoints.

## `pipe`

`pipe(fds)` creates a unidirectional communication channel.

After:

```cpp
int fds[2];
pipe(fds);
```

- `fds[0]` is the read end.
- `fds[1]` is the write end.

Data written to `fds[1]` can be read from `fds[0]`.

## `dup2`

`dup2(oldfd, newfd)` makes `newfd` refer to the same open file/pipe endpoint as `oldfd`.

Common use:

```cpp
dup2(fds[1], STDOUT_FILENO);
```

This redirects standard output into the pipe's write end.

Another common use:

```cpp
dup2(fds[0], STDIN_FILENO);
```

This redirects standard input from the pipe's read end.

## Why Close Matters

Closing unused pipe ends is essential.

A process reading from a pipe sees end-of-file only when all write ends of that pipe are closed.

If the parent or another child accidentally keeps a write end open:

- The reader may block forever.
- It believes more data might still arrive.

This is exactly the issue in the practice midterm's multiprocessing question.

## Two-Command Pipeline Pattern

For a pipeline like:

```text
cmd1 | cmd2
```

The shell-like structure is:

```cpp
int fds[2];
pipe(fds);

pid_t p1 = fork();
if (p1 == 0) {
    // child 1: stdout -> pipe write end
    close(fds[0]);
    dup2(fds[1], STDOUT_FILENO);
    close(fds[1]);
    execvp(cmd1[0], cmd1);
}

pid_t p2 = fork();
if (p2 == 0) {
    // child 2: stdin <- pipe read end
    close(fds[1]);
    dup2(fds[0], STDIN_FILENO);
    close(fds[0]);
    execvp(cmd2[0], cmd2);
}

// parent
close(fds[0]);
close(fds[1]);
waitpid(p1, NULL, 0);
waitpid(p2, NULL, 0);
```

The exact practice solution also handles incoming and outgoing descriptors, but the idea is the same.

## Timing a Child Process

The practice `thyme` problem follows this pattern:

1. Record start time.
2. `fork`.
3. Child calls `execvp`.
4. Parent waits for child.
5. Record finish time.
6. Print elapsed time.

Important:

- The child should `exec` the target program.
- The parent should measure until the child completes.

---

# Monitor-Style Coding Template

Most large coding problems can be started from this template:

```cpp
class Thing {
public:
    Thing(...);
    void method1(...);
    void method2(...);

private:
    std::mutex mutex;
    std::condition_variable cv;

    // Shared state protected by mutex.
};
```

Method pattern:

```cpp
void Thing::method(...) {
    std::unique_lock<std::mutex> lock(mutex);

    while (!predicate_over_shared_state()) {
        cv.wait(lock);
    }

    // Update shared state while holding lock.

    cv.notify_one();   // or notify_all, depending on who may now proceed
}
```

## How to Design the Shared State

For each coding problem, identify:

- What condition must a thread wait for?
- What shared state determines that condition?
- Which method changes that state?
- Who needs to be woken after the state changes?
- Is fairness required?
- Is FIFO required?
- Is partial progress allowed?

## When to Use `notify_one`

Use `notify_one` when:

- Exactly one waiting thread can proceed.
- You know which condition variable corresponds to that thread/group.
- The problem restricts you to one wakeup.

Examples:

- Boarding exactly one passenger.
- Waking one car from a chosen lane.

## When to Use `notify_all`

Use `notify_all` when:

- Many waiters may have different predicates.
- You cannot easily know which waiter can proceed.
- The state change may enable any of several waiting threads.

Example:

- Inventory arrives for one item, but many orders may need to recheck whether all requested items are now available.

`notify_all` is less efficient but often simpler and correct.

## Common Big-Problem State Patterns

### Ordered Waiting

Use an ordered structure:

```cpp
std::map<size_t, std::condition_variable*> waiters;
```

Useful when the smallest/largest key should proceed next.

### Count Per Category

Use a vector:

```cpp
std::vector<int> waiting_per_lane;
```

Useful when fairness rotates among groups.

### Resource Availability

Use a vector of counts:

```cpp
std::vector<int> available;
```

Useful when operations need one or more resources.

---

# Recommended Practice Order

## Pass 1: Read Solutions for Pattern Recognition

Do this quickly:

1. Read 2024 Problems 1-4.
2. Read 2023 Problems 1-4.
3. Read 2022 Problems 1-4.

Goal:

- Identify repeated traps.
- Make sure vocabulary feels familiar.

## Pass 2: Drill Page Tables

Redo:

- 2022 Problem 4.
- 2023 Problem 4.
- 2024 Problem 4.

For each one, write:

- Page size.
- Offset bits.
- Virtual page number bits.
- Physical page number bits.
- Entries per page map.
- Bits per page-map level.
- Number of levels.

## Pass 3: Drill Scheduling

Redo:

- 2022 Problem 3.
- 2024 Problem 2.
- 2024 Problem 3(b)-(c).

Be able to explain:

- FIFO/non-preemptive.
- Round robin/time-slice preemption.
- STCF/STCF preemption.
- Priority queues.
- Per-core ready queues.
- Starvation risk.

## Pass 4: Drill Memory and Linking

Redo:

- 2022 Problem 1(d).
- 2023 Problem 2.
- 2023 Problem 3.
- 2024 Problem 1(b)-(e).

Be able to explain:

- Object file symbol definitions vs references.
- Dynamic linking memory savings.
- Globals shared by threads.
- Memory copied across `fork`.
- Stack locals and scope.
- Slab allocation disadvantages.
- Reference counting cycles.
- Garbage collection overhead.

## Pass 5: Code the Big Problems Without Looking

Do the big 40-point problems last:

1. 2024 Inventory.
2. 2023 Flight boarding.
3. 2022 Merge lanes.

For each:

1. Read the spec.
2. Do not look at the solution.
3. Write shared state first.
4. Write waiting predicates.
5. Write methods.
6. Compare to solution.

Expected time target:

- First attempt: 45-60 minutes each.
- After practice: 25-35 minutes each.

---

# Exam-Day Checklist for the Coding Problem

Before writing code:

- Underline what each method must block for.
- Underline whether fairness/FIFO/order is required.
- Underline whether `notify_all` is allowed.
- Identify the shared state.
- Identify the invariant protected by the mutex.

While writing code:

- Use one monitor lock unless there is a clear reason not to.
- Every shared-state access happens while holding the lock.
- Every wait is inside a `while` loop unless the problem-specific design makes a single-use condition variable safe and obvious.
- Change state before notifying.
- Avoid busy waiting.
- Avoid sleeping while holding partial resources unless allowed.

After writing code:

- Check the no-waiter case.
- Check the first-arrival case.
- Check multiple waiters.
- Check whether the selected thread is actually notified.
- Check whether state is updated before the next thread can enter.
- Check starvation if the spec forbids it.

---

# Two-Sided Notes Sheet Suggestions

Since the exam historically allows two double-sided pages, your notes sheet should include:

- Process vs thread table.
- `fork` / `exec` / `wait` summary.
- Thread states diagram.
- Condition variable wait-loop template.
- Deadlock four conditions.
- Scheduling comparison table.
- Page-table arithmetic formulas.
- Memory layout diagram.
- Linker pipeline.
- Dynamic allocation pros/cons.
- Trust definitions: assumption, inference, substitution.
- Monitor coding checklist.

Do not waste much notes-sheet space on long prose. Use templates and formulas.
