# CS111 Exam C Cheatsheet

This is tuned to the exam problems you provided, not to general C++ programming.

Use this as a C/systems refresher for:

- `fork`, `execvp`, `waitpid`
- `pipe`, `dup2`, `close`
- file descriptors
- `argv`
- pointers, arrays, strings, structs
- thread interleavings
- monitor-style synchronization problems

Some provided solutions use library types such as `std::mutex`, `std::condition_variable`, `std::vector`, and `std::map`. Treat those as course-provided tools for exam answers. The coding style you need is still mostly C-style reasoning: state, pointers, arrays, resources, blocking conditions, and ownership.

---

# 1. The C Mental Model

Every variable lives somewhere:

- Global variables live in the process's global/static region.
- Local variables live on the current thread's stack.
- Dynamically allocated objects live on the heap.
- File descriptors live in the process's file descriptor table.

Every pointer is just an address:

```c
int x = 5;
int *p = &x;
*p = 10;       // changes x
```

Key operators:

- `&x`: address of `x`
- `*p`: object at address `p`
- `p->field`: shorthand for `(*p).field`

---

# 2. Arrays and Pointers

Arrays often behave like pointers to their first element.

```c
int a[3] = {10, 20, 30};
int *p = a;        // same as &a[0]

a[1] == *(a + 1)
p[1] == *(p + 1)
```

Function parameters lose array length:

```c
void f(int a[]) {
    // a is really int *
}
```

So pass lengths separately:

```c
void f(int *a, size_t n) {
    for (size_t i = 0; i < n; i++) {
        printf("%d\n", a[i]);
    }
}
```

---

# 3. C Strings and `argv`

A C string is a `char` array ending with `'\0'`.

```c
char s[] = "sort";
```

Command-line arguments:

```c
int main(int argc, char *argv[]) {
    // argv[0] = program name
    // argv[1] = first command-line argument
}
```

As a parameter, this:

```c
char *argv[]
```

is basically:

```c
char **argv
```

For `execvp`, the argument vector must end with `NULL`:

```c
char *args[] = {"sort", "-n", NULL};
execvp(args[0], args);
```

If `execvp` succeeds, it does not return.

---

# 4. Structs

```c
typedef struct Point {
    int x;
    int y;
} Point;

Point p;
p.x = 10;
p.y = 20;
```

Pointer to struct:

```c
Point *ptr = &p;
ptr->x = 30;       // same as (*ptr).x = 30
```

Exam-style shared state is usually just a struct/class with fields protected by a lock.

---

# 5. Stack vs Heap

Stack local:

```c
void f(void) {
    int x = 5;
}
```

`x` disappears when `f` returns.

Bad:

```c
int *bad(void) {
    int x = 5;
    return &x;     // wrong: x dies when function returns
}
```

Heap:

```c
int *p = malloc(sizeof(int));
*p = 5;
free(p);
```

Common bugs:

- leak: never freeing
- use after free
- double free
- writing past array bounds
- returning pointer to stack local

---

# 6. Process vs Thread Memory

This appears directly in the past midterms.

## Threads Share Process Memory

```c
int x = 100;

void thread_main(void) {
    x++;
}
```

Threads in the same process share global variables and heap memory.

So if multiple threads access `x`, you need synchronization.

## Processes Do Not Share Normal Memory After `fork`

```c
int x = 100;

pid_t pid = fork();
if (pid == 0) {
    x += 100;              // child's x
} else {
    waitpid(pid, NULL, 0);
    printf("%d\n", x);     // parent's x, still 100
}
```

After `fork`, parent and child have separate address spaces.

They do share inherited file descriptors, which matters for pipes.

---

# 7. `fork`

Pattern:

```c
pid_t pid = fork();

if (pid == 0) {
    // child
} else {
    // parent; pid is child's process id
}
```

After `fork`:

- Parent and child both continue after the `fork`.
- Child gets return value `0`.
- Parent gets child's pid.
- Memory is separate.
- File descriptors are inherited.

Common child pattern:

```c
if (pid == 0) {
    execvp(argv[1], argv + 1);
    perror("execvp");
    exit(1);
}
```

The `perror`/`exit` part runs only if `execvp` fails.

---

# 8. `waitpid`

```c
waitpid(pid, NULL, 0);
```

Meaning:

- Parent waits for the specific child `pid`.
- Prevents zombie processes.
- Lets parent know child is done.

If there are two children:

```c
waitpid(pids[0], NULL, 0);
waitpid(pids[1], NULL, 0);
```

---

# 9. File Descriptors

File descriptors are integers:

```c
0   stdin
1   stdout
2   stderr
```

Named constants:

```c
STDIN_FILENO
STDOUT_FILENO
STDERR_FILENO
```

Useful calls:

```c
close(fd);
dup2(oldfd, newfd);
read(fd, buf, count);
write(fd, buf, count);
```

After `fork`, the child inherits the parent's open file descriptors.

---

# 10. `pipe`

```c
int fds[2];
pipe(fds);
```

Meaning:

- `fds[0]`: read end
- `fds[1]`: write end

Data written to `fds[1]` can be read from `fds[0]`.

Pipe EOF rule:

- A reader sees EOF only after all write ends of the pipe are closed.

This is a major exam trap.

If some process accidentally keeps a write end open, the reader may block forever.

---

# 11. `dup2`

```c
dup2(oldfd, newfd);
```

Makes `newfd` refer to the same open file/pipe endpoint as `oldfd`.

Redirect stdout to a pipe:

```c
dup2(fds[1], STDOUT_FILENO);
```

Redirect stdin from a pipe:

```c
dup2(fds[0], STDIN_FILENO);
```

After `dup2`, close the original descriptor if you no longer need it:

```c
dup2(fds[1], STDOUT_FILENO);
close(fds[1]);
```

---

# 12. Two-Command Pipeline Pattern

For:

```text
cmd1 | cmd2
```

Use:

```c
int fds[2];
pipe(fds);

pid_t p1 = fork();
if (p1 == 0) {
    close(fds[0]);                    // child 1 does not read
    dup2(fds[1], STDOUT_FILENO);      // stdout -> pipe write
    close(fds[1]);
    execvp(cmd1[0], cmd1);
    perror("execvp");
    exit(1);
}

pid_t p2 = fork();
if (p2 == 0) {
    close(fds[1]);                    // child 2 does not write
    dup2(fds[0], STDIN_FILENO);       // stdin <- pipe read
    close(fds[0]);
    execvp(cmd2[0], cmd2);
    perror("execvp");
    exit(1);
}

close(fds[0]);
close(fds[1]);
waitpid(p1, NULL, 0);
waitpid(p2, NULL, 0);
```

Checklist:

- First child closes read end.
- First child redirects stdout to write end.
- Second child closes write end.
- Second child redirects stdin to read end.
- Parent closes both ends.
- Parent waits for both children.

---

# 13. Practice Midterm `duet` Pattern

The practice midterm has a function like:

```c
static void duet(int incoming, char *one[], char *two[], int outgoing)
```

Meaning:

- `incoming`: fd for first command's stdin
- `one`: argv for first command
- `two`: argv for second command
- `outgoing`: fd for second command's stdout

Shape:

```c
int fds[2];
pipe(fds);

// child 1: incoming -> one -> pipe
// child 2: pipe -> two -> outgoing
```

Important close logic:

- Child 1 should not keep pipe read end.
- Child 2 should not keep pipe write end.
- Parent should close descriptors it no longer needs.
- Otherwise the downstream process may wait forever for EOF.

---

# 14. Timing a Child Process

Practice midterm `thyme` pattern:

```c
struct timespec start;
clock_gettime(CLOCK_REALTIME, &start);

pid_t pid = fork();
if (pid == 0) {
    execvp(argv[1], argv + 1);
    perror("execvp");
    exit(1);
}

waitpid(pid, NULL, 0);

struct timespec finish;
clock_gettime(CLOCK_REALTIME, &finish);
print_elapsed_time(&start, &finish);
```

Reasoning:

- Parent measures time.
- Child runs the command.
- Parent waits until child finishes.
- Parent computes elapsed time.

---

# 15. Thread Interleavings

If two threads modify shared state without synchronization, multiple outputs may be possible.

Example:

```c
int x = 10;

// Thread 1:
x += 10;
x *= 2;

// Thread 2:
x += 10;
x *= 2;
```

Different schedules can produce different results.

To solve these problems:

1. Break each line into read/compute/write if needed.
2. Consider one thread running to completion.
3. Consider the other running to completion.
4. Consider interleavings between individual operations.

The key idea: without locks, the scheduler can expose many valid orderings.

---

# 16. Threads

Threads are separate streams of execution inside the same process.

For this class, the key difference is:

```text
processes have separate memory
threads in the same process share memory
```

That one sentence explains why threads are powerful and why they cause synchronization bugs.

## Process vs Thread

Process:

- Has its own address space.
- Created with `fork`.
- Parent and child memory become separate.
- Communicates through files, pipes, sockets, shared memory, etc.

Thread:

- Runs inside a process.
- Shares global variables and heap memory with other threads in the same process.
- Has its own stack and registers.
- Communicates easily through shared variables.

Example:

```text
One process:

global variables: shared by all threads
heap:             shared by all threads

Thread A stack:   private to thread A
Thread B stack:   private to thread B
```

## Why Use Threads?

Threads are useful for:

- doing multiple things concurrently
- using multiple cores
- keeping a program responsive
- sharing data without setting up pipes
- parallelizing work

But shared data means races unless you use locks.

## Basic `std::thread` Syntax

Even if you think of the course as C-style systems programming, the thread examples use `std::thread`.

```cpp
#include <thread>

void worker() {
    printf("hello from thread\n");
}

int main() {
    std::thread t(worker);
    t.join();
    return 0;
}
```

Meaning:

```cpp
std::thread t(worker);
```

starts a new thread running `worker`.

```cpp
t.join();
```

waits for that thread to finish.

## `join`

`join` for threads is conceptually similar to `waitpid` for processes.

Process:

```c
waitpid(pid, NULL, 0);
```

Thread:

```cpp
t.join();
```

Meaning:

```text
do not continue past this line until that child/thread is done
```

Difference:

- `waitpid` waits for a child process.
- `join` waits for a thread.

## Thread Function With Arguments

```cpp
void worker(int id) {
    printf("thread %d\n", id);
}

int main() {
    std::thread t(worker, 5);
    t.join();
}
```

This starts a thread running:

```cpp
worker(5);
```

## Multiple Threads

```cpp
void worker(int id) {
    printf("thread %d\n", id);
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
}
```

Possible output:

```text
thread 1
thread 2
```

or:

```text
thread 2
thread 1
```

The scheduler decides which runs first.

## Vector of Threads

Common pattern:

```cpp
std::vector<std::thread> threads;

for (int i = 0; i < 5; i++) {
    threads.push_back(std::thread(worker, i));
}

for (std::thread &t : threads) {
    t.join();
}
```

Meaning:

1. Create five threads.
2. Store them in a vector.
3. Join all of them.

The join loop is important. If you create multiple threads, you usually need to wait for all of them.

## Lambdas as Thread Bodies

The section notes use lambdas. A lambda is an inline function.

Basic shape:

```cpp
[] {
    printf("hello\n");
}
```

Thread with lambda:

```cpp
std::thread t([] {
    printf("hello from lambda thread\n");
});

t.join();
```

## Lambda Captures

If a lambda uses a variable from outside, it must capture it.

Capture by value:

```cpp
int x = 5;

std::thread t([x] {
    printf("%d\n", x);
});
```

The lambda gets a copy of `x`.

Capture by reference:

```cpp
int x = 5;

std::thread t([&x] {
    x = 10;
});

t.join();
printf("%d\n", x);   // 10
```

The lambda uses the original `x`.

Exam intuition:

- `[x]` means copy.
- `[&x]` means shared/original variable.
- Shared mutable variables require synchronization.

## `std::ref`

Sometimes thread arguments are copied by default. To pass a reference, code may use `std::ref`.

Example from the practice-style expression evaluation pattern:

```cpp
std::thread t(evaluate, std::ref(expressions[i]), std::ref(info));
```

Meaning:

- Pass `expressions[i]` by reference.
- Pass `info` by reference.

Without `std::ref`, C++ may try to copy the argument into the new thread.

For exam reading, when you see:

```cpp
std::ref(info)
```

think:

```text
the thread shares the same info object
```

## Shared Object Passed to Threads

Practice midterm expression-evaluation idea:

```cpp
typedef struct ThreadInfo {
    vector<int> v;
    mutex m;
} ThreadInfo;
```

`ThreadInfo` contains:

- `v`: shared vector of results
- `m`: mutex protecting the vector

Thread function:

```cpp
static void evaluate(Expression& exp, ThreadInfo& info) {
    int result = exp.evaluate();
    info.m.lock();
    info.v.push_back(result);
    info.m.unlock();
}
```

Why lock?

Multiple threads may finish at the same time and call:

```cpp
info.v.push_back(result);
```

The vector is shared state. Concurrent `push_back` without a lock can corrupt it.

## Threaded Expression Evaluation Pattern

```cpp
static bool concurrentAnd(const vector<Expression>& expressions) {
    ThreadInfo info;
    vector<thread> threads;

    for (size_t i = 0; i < expressions.size(); i++) {
        threads.push_back(thread(evaluate,
                                 ref(expressions[i]),
                                 ref(info)));
    }

    for (thread& t : threads) {
        t.join();
    }

    printResults(info.v);
}
```

Line-by-line:

```cpp
ThreadInfo info;
```

One shared object for all threads.

```cpp
vector<thread> threads;
```

Store thread handles so you can join later.

```cpp
threads.push_back(thread(evaluate, ref(expressions[i]), ref(info)));
```

Start one thread per expression.

```cpp
for (thread& t : threads) {
    t.join();
}
```

Wait until all expression-evaluation threads finish.

```cpp
printResults(info.v);
```

Only print results after every thread is done.

## Threads Share Globals

```cpp
int x = 100;

void f() {
    x += 100;
}
```

If multiple threads call `f`, they all use the same global `x`.

This is different from `fork`, where the child gets a separate copy of memory.

Past midterm pattern:

```cpp
int x = 100;

void func2(...) {
    x--;
    std::thread t([](){ x = 300; });
    t.join();
    std::cout << "x = " << x << std::endl;
}
```

The lambda thread and the original thread refer to the same global `x`.

## Each Thread Has Its Own Stack

Local variables inside a function call are on that thread's stack.

```cpp
void worker(int id) {
    int local = id;
}
```

If two threads run `worker`, each has its own `local`.

But if both threads access a global or heap object, that object is shared.

## Output From Threads Can Interleave

If two threads print at the same time, output order can vary.

```cpp
void worker(int id) {
    for (int i = 0; i < 3; i++) {
        printf("thread %d line %d\n", id, i);
    }
}
```

Possible output may alternate between threads. Do not assume one thread finishes all prints before another starts unless you join or synchronize accordingly.

## Thread Lifecycle

Simple lifecycle:

```text
created -> ready -> running -> blocked/ready -> finished
```

Created:

- Thread object is created.

Ready:

- Thread can run but is waiting for a CPU/core.

Running:

- Thread is executing.

Blocked:

- Thread is waiting for something, such as a lock, condition variable, I/O, or join.

Finished:

- Thread function returned.

## Thread vs Process Exam Comparison

| Question | Process | Thread |
|---|---|---|
| Creation call | `fork()` | `std::thread(...)` |
| Wait call | `waitpid(pid, ...)` | `t.join()` |
| Memory | separate after `fork` | shared within process |
| Globals | copied/separate after `fork` | shared |
| File descriptors | inherited across `fork` | shared process-level resources |
| Communication | pipes/files/etc. | shared variables, locks |

## Thread Mistakes to Avoid

- Assuming threads have separate global variables.
- Forgetting to `join`.
- Passing shared objects without synchronization.
- Updating a shared vector/map without a mutex.
- Thinking `join` protects shared state by itself.
- Capturing a variable by reference when the thread may outlive that variable.

`join` only waits for completion. It does not make unsynchronized shared updates safe.

## Review Problem: News Aggregation

The midterm review problem asks you to take sequential article fetching and run one thread per article.

Original idea:

```cpp
for (int i = 0; i < articles.size(); i++) {
    fetchArticle(articles[i], nm, database);
}
```

Threaded idea:

```cpp
std::vector<std::thread> threads;

for (int i = 0; i < articles.size(); i++) {
    threads.push_back(std::thread(fetchArticle,
                                  std::ref(articles[i]),
                                  std::ref(nm),
                                  std::ref(database),
                                  std::ref(databaseLock)));
}

for (std::thread& t : threads) {
    t.join();
}
```

What each part means:

```cpp
std::vector<std::thread> threads;
```

Store the thread handles so you can join them later.

```cpp
std::thread(fetchArticle, ...)
```

Start a new thread running `fetchArticle`.

```cpp
std::ref(database)
```

Pass the actual shared database, not a copy.

```cpp
std::ref(databaseLock)
```

Pass the actual shared mutex, not a copy.

```cpp
t.join();
```

Wait for that article-fetching thread to finish.

The shared-state issue is inside `fetchArticle`:

```cpp
void fetchArticle(const Article& article,
                  const NewsManager& nm,
                  map<Article, vector<string>>& database,
                  mutex& databaseLock) {
    vector<string> articleContents = nm.fetchArticleContents(article);

    databaseLock.lock();
    database[article] = articleContents;
    databaseLock.unlock();
}
```

Important:

- Fetching article contents is slow but independent, so it happens outside the lock.
- Updating `database` is shared, so it happens inside the lock.

Better style with `unique_lock`:

```cpp
void fetchArticle(const Article& article,
                  const NewsManager& nm,
                  map<Article, vector<string>>& database,
                  mutex& databaseLock) {
    vector<string> articleContents = nm.fetchArticleContents(article);

    std::unique_lock<std::mutex> lock(databaseLock);
    database[article] = articleContents;
}
```

Exam lesson:

```text
parallelize independent slow work
lock only around the shared data update
join before using final results
```

---

# 17. Mutexes, Locks, and Synchronization

This is one of the most important exam topics. The big coding problems are usually synchronization problems disguised as real-world systems: trains, bridges, boarding, inventory, merging lanes.

## Shared State

Shared state is data that more than one thread can access.

Example:

```cpp
int counter = 0;
```

If two threads both do:

```cpp
counter++;
```

there is a race. Even though `counter++` looks like one operation, it is really more like:

```text
load counter
add 1
store counter
```

Bad interleaving:

```text
counter starts at 0

Thread A loads counter: 0
Thread B loads counter: 0
Thread A adds 1, stores 1
Thread B adds 1, stores 1

final counter is 1, but it should be 2
```

That is why shared state needs synchronization.

## Race Condition

A race condition means correctness depends on timing.

If your code works only when threads happen to run in a lucky order, it has a race.

Typical race ingredients:

- shared state
- multiple threads
- at least one thread writes
- no correct synchronization

## Mutex

A mutex is a lock.

`mutex` stands for mutual exclusion:

```text
only one thread can hold the mutex at a time
```

In CS111 exam code:

```cpp
std::mutex mutex;
```

creates the lock.

Then:

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

locks it.

While one thread holds the mutex, another thread trying to lock the same mutex must wait.

## Critical Section

A critical section is the code that must be protected by a lock.

Example:

```cpp
std::mutex mutex;
int counter = 0;

void increment() {
    std::unique_lock<std::mutex> lock(mutex);
    counter++;
}
```

The critical section is:

```cpp
counter++;
```

because it touches shared state.

## What `std::unique_lock` Does

This line:

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

means:

```text
lock the mutex now
keep it locked while this local variable exists
unlock it automatically when the function/block exits
```

Example:

```cpp
void add(int stock_id, int count) {
    std::unique_lock<std::mutex> lock(mutex);
    inventory[stock_id] += count;
}
```

When `add` returns, `lock` is destroyed, and the mutex unlocks automatically.

This is why you usually do not see:

```cpp
mutex.unlock();
```

in the exam solutions. `unique_lock` handles it.

## The Same Lock Must Protect the Same State

A mutex works only if every thread follows the same rule.

Good:

```cpp
std::mutex mutex;
int counter = 0;

void increment() {
    std::unique_lock<std::mutex> lock(mutex);
    counter++;
}

int get_counter() {
    std::unique_lock<std::mutex> lock(mutex);
    return counter;
}
```

Both methods use the same mutex before touching `counter`.

Bad:

```cpp
void increment() {
    std::unique_lock<std::mutex> lock(mutex);
    counter++;
}

int get_counter() {
    return counter;      // unprotected read
}
```

This is still a race because one method bypasses the lock.

## Locks Protect Invariants

An invariant is a fact that should always be true when no method is halfway through changing state.

Inventory invariant:

```text
inventory counts should not go negative
```

Bridge invariant:

```text
cars on the bridge should all travel in the same direction
```

Merge invariant:

```text
only one car can be merging at a time
```

The mutex lets a method temporarily update several variables without another thread seeing a half-updated state.

Example:

```cpp
waiting[lane]--;
totalWaiting--;
safe_to_merge[lane].notify_one();
```

The counts are updated while holding the lock, so other threads do not see inconsistent values.

## Mutexes Do Not Solve Waiting by Themselves

A mutex answers:

```text
Can I safely touch shared state?
```

It does not answer:

```text
Should I proceed now?
```

Example: inventory.

The mutex lets you safely check:

```cpp
inventory[id] < needed
```

But if there is not enough inventory, the thread should sleep until more inventory arrives. That needs a condition variable.

## Condition Variable

A condition variable lets a thread sleep until some shared-state condition might have changed.

Example:

```cpp
std::condition_variable new_inventory;
```

The condition variable does not store the condition. The condition is the expression you check.

For inventory:

```text
all requested stock items have enough inventory
```

For a passenger:

```text
a train is present and a seat is available
```

For a bridge car:

```text
no cars are crossing in the opposite direction
```

## `wait(lock)`

This line:

```cpp
cv.wait(lock);
```

does three things:

1. Unlocks the mutex.
2. Puts the current thread to sleep.
3. Re-locks the mutex before returning.

That unlock step is essential.

If a thread slept while still holding the lock, no other thread could acquire the lock to change the state and wake it up.

## Wait Predicate

The wait predicate is the condition that must be true before the thread can proceed.

Example:

```cpp
while (inventory[stock_id] < count) {
    new_inventory.wait(lock);
}
```

Predicate:

```text
inventory[stock_id] >= count
```

For multi-item inventory:

```cpp
while (!entire_order_available()) {
    new_inventory.wait(lock);
}
```

Predicate:

```text
every requested item has enough inventory
```

## Always Use `while`, Not `if`

Wrong:

```cpp
if (!ready) {
    cv.wait(lock);
}
```

Right:

```cpp
while (!ready) {
    cv.wait(lock);
}
```

Why:

- Another thread may run first and consume the resource.
- `notify_all` may wake threads whose conditions are still false.
- Some systems allow spurious wakeups.
- Waking means "check again", not "you are definitely allowed to proceed."

## `notify_one`

```cpp
cv.notify_one();
```

Wakes one waiting thread.

Use it when exactly one thread should proceed or when you know which group to wake.

Examples:

- One car from a selected merge lane.
- One passenger with the lowest boarding position.

## `notify_all`

```cpp
cv.notify_all();
```

Wakes all waiting threads on that condition variable.

Use it when different waiters may have different predicates and you do not know which one can proceed.

Example:

```cpp
void Inventory::add(int stock_id, int count) {
    std::unique_lock<std::mutex> lock(mutex);
    inventory[stock_id] += count;
    new_inventory.notify_all();
}
```

Why all?

- One order might need item 3.
- Another order might need items 3 and 7.
- Another might need item 9.
- After adding inventory, every waiting order should re-check its own condition.

## Minimal Lock + Condition Variable Example

```cpp
class Box {
public:
    void put(int value);
    int get();

private:
    std::mutex mutex;
    std::condition_variable not_empty;
    bool has_value = false;
    int stored_value;
};

void Box::put(int value) {
    std::unique_lock<std::mutex> lock(mutex);
    stored_value = value;
    has_value = true;
    not_empty.notify_one();
}

int Box::get() {
    std::unique_lock<std::mutex> lock(mutex);
    while (!has_value) {
        not_empty.wait(lock);
    }
    has_value = false;
    return stored_value;
}
```

Important:

- `mutex` protects `has_value` and `stored_value`.
- `get` waits while the box is empty.
- `put` changes the state, then notifies.
- `get` uses `while`, not `if`.

## How This Maps to Exam Problems

Inventory:

```text
mutex protects inventory vector
condition variable wakes orders when inventory changes
```

Flight:

```text
mutex protects map of waiting passengers
condition variables wake selected passengers
```

Merge:

```text
mutex protects lane counts and merging flag
condition variables wake cars in selected lanes
```

Bridge:

```text
mutex protects crossing/waiting counters
condition variables wake cars when direction becomes safe
```

## One-Sentence Summary

```text
mutex = protects shared state
condition variable = lets threads sleep until shared state might allow progress
```

---

# 18. Monitor-Style Synchronization, Exam Version

The past midterms' big coding problems use monitor-style synchronization.

Even if the syntax uses `std::mutex` / condition variables, the reasoning is C-style:

- One shared state object.
- One lock protecting that state.
- Wait while a predicate is false.
- Change state while holding the lock.
- Notify waiting thread(s).

Generic shape:

```c
lock(m);
while (!condition_over_shared_state) {
    wait(cv, m);       // releases m while sleeping, reacquires before returning
}

// update shared state

notify(cv);
unlock(m);
```

If using the exam's C++-library syntax, this becomes:

```cpp
std::unique_lock<std::mutex> lock(mutex);
while (!condition) {
    cv.wait(lock);
}
```

Same idea.

---

# 19. Condition Variable Rules

Always remember:

- The condition variable is not the condition.
- The condition is a predicate over shared state.
- Wait in a loop.
- Change shared state before notifying.
- Do not busy-wait.

Wrong:

```c
if (!ready) {
    wait(cv, m);
}
```

Right:

```c
while (!ready) {
    wait(cv, m);
}
```

Reason:

- Wakeup does not prove the condition is true.
- Another thread may have consumed the resource.
- Spurious wakeups may happen.

---

# 20. Big Coding Problem Design Checklist

Before writing code, identify:

- What shared state exists?
- What invariant must be protected?
- Which threads wait?
- What exact condition lets each waiter proceed?
- Which method changes that condition?
- Should you wake one thread or all threads?
- Is fairness/order required?
- Is partial reservation allowed?

For the provided exams:

## 2022 Merge

State:

- waiting count per lane
- total waiting
- current/previous lane
- whether a car is merging

Condition:

- a car waits until selected to merge

Fairness:

- rotate among lanes with waiting cars

## 2023 Flight

State:

- waiting passengers by boarding position

Condition:

- passenger waits until gate agent selects their position

Useful structure:

- ordered map from position to waiter

Wake:

- wake lowest waiting position

## 2024 Inventory

State:

- inventory count per item

Condition:

- every requested item has enough inventory

Important:

- do not partially reserve while waiting
- once all counts are available, subtract all counts while holding lock

Wake:

- `notify_all` is reasonable because many different orders may need to recheck

---

# 21. Minimal Library Tools from Exam Solutions

You do not need to become fluent in all of C++. For these exams, know the operations that appear.

## Vector-Like Dynamic Array

```cpp
std::vector<int> v(n, 0);
v[i]++;
v.size();
v.push_back(x);
```

Think: resizable array.

## Map-Like Ordered Dictionary

```cpp
std::map<size_t, condition_variable *> waiters;
waiters.emplace(position, &cv);
waiters.begin();       // smallest key
waiters.erase(it);
```

Think: ordered dictionary.

Useful for "lowest position goes first."

## Mutex / Lock

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

Think:

- lock acquired here
- automatically released when function returns / scope ends

## Condition Variable

```cpp
cv.wait(lock);
cv.notify_one();
cv.notify_all();
```

Think:

- `wait`: sleep and temporarily release lock
- `notify_one`: wake one waiter
- `notify_all`: wake all waiters

---

# 22. Exam Mistakes to Avoid

Process/file descriptor mistakes:

- Forgetting that both parent and child continue after `fork`.
- Forgetting that `execvp` does not return on success.
- Forgetting to `exit` after failed `execvp` in the child.
- Forgetting to close unused pipe ends.
- Keeping a pipe write end open and causing the reader to block forever.
- Waiting for only one child when you created two.

Thread/synchronization mistakes:

- Using `if` instead of `while` around waits.
- Accessing shared state without the lock.
- Not updating state before notifying.
- Busy-waiting.
- Partially reserving resources when the spec forbids it.
- Using `notify_one` when you do not know which waiter can proceed.

C/pointer mistakes:

- Confusing pointer and pointee.
- Returning address of a stack local.
- Forgetting `argv` is `NULL`-terminated for `execvp`.
- Assuming arrays know their own length in functions.
- Mixing parent/child memory after `fork`.

---

# 23. Annotated Hard Exam Questions

This section walks through the hardest code-shaped questions from the provided past/practice exams. The goal is not to memorize the exact code. The goal is to understand the moving parts well enough to reproduce the pattern.

## A. Practice Midterm Problem 2: `duet`

The problem shape:

```c
static void duet(int incoming, char *one[], char *two[], int outgoing)
```

You are building this pipeline:

```text
incoming -> program one -> pipe -> program two -> outgoing
```

Meaning:

- `incoming` is a file descriptor used as stdin for the first program.
- `one` is the `argv` array for the first program.
- `two` is the `argv` array for the second program.
- `outgoing` is a file descriptor used as stdout for the second program.

The middle connection must be a pipe:

```text
program one stdout -> pipe write end
program two stdin  <- pipe read end
```

Annotated solution:

```c
static void duet(int incoming, char *one[], char *two[], int outgoing) {
    pid_t pids[2];
    int fds[2];

    pipe(fds);
```

`pipe(fds)` creates two file descriptors:

```text
fds[0] = read end
fds[1] = write end
```

Think:

```text
write to fds[1] ---> pipe ---> read from fds[0]
```

Now create the first child:

```c
    pids[0] = fork();
    if (pids[0] == 0) {
```

Inside this `if`, we are in child 1. Child 1 should run `one`.

Child 1 does not read from the pipe:

```c
        close(fds[0]);
```

Child 1 also does not use `outgoing`; that is for child 2:

```c
        close(outgoing);
```

Make child 1's stdin come from `incoming`:

```c
        dup2(incoming, STDIN_FILENO);
        close(incoming);
```

After this:

```text
fd 0 / STDIN_FILENO -> incoming
```

So when program `one` reads from stdin, it reads from `incoming`.

Make child 1's stdout go to the pipe write end:

```c
        dup2(fds[1], STDOUT_FILENO);
        close(fds[1]);
```

After this:

```text
fd 1 / STDOUT_FILENO -> pipe write end
```

So when program `one` prints, its output goes into the pipe.

Now replace child 1 with the actual program:

```c
        execvp(one[0], one);
    }
```

If `execvp` succeeds, child 1 is no longer running this `duet` code. It has become program `one`.

Back in the parent, after creating child 1:

```c
    close(incoming);
    close(fds[1]);
```

The parent no longer needs:

- `incoming`, because only child 1 uses it.
- `fds[1]`, because only child 1 writes into the pipe.

This close is not optional. If the parent keeps `fds[1]` open, child 2 may never see EOF from the pipe.

Now create the second child:

```c
    pids[1] = fork();
    if (pids[1] == 0) {
```

Inside this `if`, we are in child 2. Child 2 should run `two`.

Make child 2's stdin come from the pipe read end:

```c
        dup2(fds[0], STDIN_FILENO);
        close(fds[0]);
```

After this:

```text
fd 0 / STDIN_FILENO -> pipe read end
```

So program `two` reads what program `one` wrote.

Make child 2's stdout go to `outgoing`:

```c
        dup2(outgoing, STDOUT_FILENO);
        close(outgoing);
```

After this:

```text
fd 1 / STDOUT_FILENO -> outgoing
```

Now replace child 2 with the actual program:

```c
        execvp(two[0], two);
    }
```

Back in the parent:

```c
    close(outgoing);
    close(fds[0]);
```

The parent no longer needs:

- `outgoing`, because child 2 uses it.
- `fds[0]`, because child 2 reads from the pipe.

Finally:

```c
    waitpid(pids[0], NULL, 0);
    waitpid(pids[1], NULL, 0);
}
```

The parent waits for both children to finish.

Big exam lessons:

- `pipe` gives you two fds: read end and write end.
- `dup2` rewires stdin/stdout.
- `fork` copies the fd table, so every process must close descriptors it does not need.
- `execvp` preserves open descriptors, so redirection set up before `execvp` remains active in the new program.
- Pipe EOF only happens when all write ends are closed.

Common wrong answer:

```c
// parent forgets this
close(fds[1]);
```

Why wrong:

- Parent still has the pipe write end open.
- Program `two` reads from the pipe.
- It may wait forever because the OS thinks someone could still write more data.

## B. Spring 2024 Problem 5: Inventory Monitor

The problem shape:

```cpp
void add(int stock_id, int count);
void reserve(std::vector<int> &stock_ids, std::vector<int> &counts);
```

Meaning:

- `add` increases inventory for one item.
- `reserve` waits until an entire order can be filled.
- `reserve` must not partially reserve items while waiting.

This is the central rule:

```text
Do not subtract anything until every requested item is available.
```

State:

```cpp
class Inventory {
private:
    std::mutex mutex;
    std::condition_variable new_inventory;
    std::vector<int> inventory;
};
```

Interpretation:

- `mutex` protects all shared state.
- `new_inventory` is used to wake orders when anything is added.
- `inventory[id]` is how many units of item `id` are available.

Constructor:

```cpp
Inventory::Inventory(int max_stock_id)
    : mutex()
    , new_inventory()
    , inventory(max_stock_id + 1, 0)
{}
```

`inventory(max_stock_id + 1, 0)` creates a vector with one slot for each stock id, all starting at `0`.

`add`:

```cpp
void Inventory::add(int stock_id, int count) {
    std::unique_lock<std::mutex> lock(mutex);
    inventory[stock_id] += count;
    new_inventory.notify_all();
}
```

Line-by-line:

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

Acquire the monitor lock.

```cpp
inventory[stock_id] += count;
```

Change shared state while holding the lock.

```cpp
new_inventory.notify_all();
```

Wake waiting orders so they can re-check whether they are now satisfiable.

Why `notify_all`, not `notify_one`?

- Different orders are waiting for different combinations of items.
- Adding item 7 might satisfy order A but not order B.
- The `add` method does not track exactly which waiter can proceed.
- Waking everyone is simple and correct; each thread re-checks its own predicate.

Now `reserve`:

```cpp
void Inventory::reserve(std::vector<int> &stock_ids,
                        std::vector<int> &counts) {
    bool order_ok = false;
    std::unique_lock<std::mutex> lock(mutex);
```

Acquire the lock before checking shared inventory.

Main wait loop:

```cpp
    while (!order_ok) {
        order_ok = true;
        for (size_t i = 0; i < stock_ids.size(); i++) {
            if (inventory[stock_ids[i]] < counts[i]) {
                new_inventory.wait(lock);
                order_ok = false;
                break;
            }
        }
    }
```

This loop means:

1. Assume the order is satisfiable.
2. Check every requested item.
3. If any item is short, sleep.
4. When woken, re-check from the beginning.

Important detail:

```cpp
new_inventory.wait(lock);
```

While sleeping, the thread releases the mutex. That lets `add()` acquire the mutex and add inventory.

After waking, `wait` reacquires the mutex before returning.

Only after the entire order is known to be available:

```cpp
    for (size_t i = 0; i < stock_ids.size(); i++) {
        inventory[stock_ids[i]] -= counts[i];
    }
}
```

This is the actual reservation. It happens atomically with respect to other orders because the lock is still held.

Common wrong solution:

```cpp
for each requested item:
    while inventory[item] < count:
        wait
    inventory[item] -= count
```

Why wrong:

- It partially reserves earlier items.
- Then it may sleep waiting for a later item.
- Other orders cannot use the partially reserved inventory.
- The problem explicitly says not to reserve anything until the whole order can complete.

Exam pattern name:

```text
multi-resource all-or-nothing wait
```

Whenever you see that, first check all resources, then consume all resources.

## C. Spring 2023 Problem 5: Flight Boarding Monitor

The problem shape:

```cpp
void wait_to_board(size_t position);
bool board_next();
```

Meaning:

- Passenger calls `wait_to_board(position)` and blocks.
- Gate agent calls `board_next()`.
- The lowest waiting boarding position should be woken.
- If nobody is waiting, `board_next()` returns `false`.

State:

```cpp
class Flight {
private:
    std::mutex mutex;
    std::map<size_t, std::condition_variable*> ready_passengers;
};
```

Why `std::map`?

- It keeps keys sorted.
- Boarding position is the key.
- The first entry is the lowest waiting position.

Passenger method:

```cpp
void Flight::wait_to_board(size_t position) {
    std::unique_lock<std::mutex> lock(mutex);
    std::condition_variable my_turn;

    ready_passengers.emplace(position, &my_turn);
    my_turn.wait(lock);
}
```

Line-by-line:

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

Acquire the monitor lock.

```cpp
std::condition_variable my_turn;
```

This passenger creates their own condition variable. That lets the gate agent wake exactly this passenger.

```cpp
ready_passengers.emplace(position, &my_turn);
```

Insert this passenger into the map. The key is the boarding position. The value is a pointer to this passenger's condition variable.

```cpp
my_turn.wait(lock);
```

Passenger sleeps until the gate agent wakes them.

Why is `my_turn` local? Is that safe?

- It exists while `wait_to_board` is running.
- The passenger is blocked inside `wait_to_board`.
- The map entry is erased before/when the passenger is woken.
- After `wait_to_board` returns, the condition variable is no longer needed.

Gate agent method:

```cpp
bool Flight::board_next() {
    std::unique_lock<std::mutex> lock(mutex);
    if (ready_passengers.empty()) {
        return false;
    }
    ready_passengers.begin()->second->notify_one();
    ready_passengers.erase(ready_passengers.begin());
    return true;
}
```

Line-by-line:

```cpp
if (ready_passengers.empty()) {
    return false;
}
```

If no passengers are currently waiting, no one can be boarded.

```cpp
ready_passengers.begin()
```

This points to the map entry with the lowest boarding position.

```cpp
ready_passengers.begin()->second
```

This is the condition variable pointer for that passenger.

```cpp
ready_passengers.begin()->second->notify_one();
```

Wake that passenger.

```cpp
ready_passengers.erase(ready_passengers.begin());
```

Remove them from the waiting map so they are not boarded twice.

Exam pattern name:

```text
ordered waiters
```

When the problem says "lowest id first", "smallest position first", or "next by priority", think about using an ordered structure.

Common wrong solution:

```cpp
std::vector<std::condition_variable> waiters;
```

Why it may be wrong:

- If position 3 arrives late after position 5 is waiting, position 3 must still board before position 5.
- A simple arrival-order queue does not preserve boarding-position order.

## D. Spring 2022 Problem 5: Merge Monitor

The problem shape:

```cpp
void wait(int lane);
void merged();
```

Meaning:

- A car calls `wait(lane)` before merging.
- `wait` returns only when this car may enter the single merge lane.
- After finishing, the car calls `merged()`.
- Only one car can merge at a time.
- If many lanes have cars waiting, the solution should rotate among lanes.

State:

```cpp
std::mutex mutex;
std::vector<int> waiting;
int totalWaiting;
std::vector<std::condition_variable> safe_to_merge;
int prev_lane;
bool merging;
```

Interpretation:

- `waiting[lane]`: number of cars waiting in that lane.
- `totalWaiting`: total waiting cars across all lanes.
- `safe_to_merge[lane]`: condition variable for cars in that lane.
- `prev_lane`: last lane selected.
- `merging`: whether one car is currently merging.

Constructor initializes counts and condition variables:

```cpp
Merge::Merge(int num_lanes)
    : mutex(), waiting(), totalWaiting(0),
      safe_to_merge(), prev_lane(0), merging(false)
{
    for (int i = 0; i < num_lanes; i++) {
        waiting.emplace_back(0);
        safe_to_merge.emplace_back();
    }
}
```

`wait`:

```cpp
void Merge::wait(int lane) {
    std::unique_lock<std::mutex> lock(mutex);
    if (!merging) {
        merging = true;
        return;
    }
    waiting[lane]++;
    totalWaiting++;
    safe_to_merge[lane].wait(lock);
}
```

Line-by-line:

```cpp
if (!merging) {
    merging = true;
    return;
}
```

If nobody is currently merging, this car can go immediately. It marks `merging = true` so no other car can also go.

Otherwise:

```cpp
waiting[lane]++;
totalWaiting++;
```

Record this car as waiting before sleeping.

```cpp
safe_to_merge[lane].wait(lock);
```

Sleep on the condition variable for this lane.

`merged`:

```cpp
void Merge::merged() {
    std::unique_lock<std::mutex> lock(mutex);
    if (totalWaiting == 0) {
        merging = false;
        return;
    }
```

If nobody is waiting, mark the merge lane free.

If there are waiting cars, rotate to find the next lane with waiters:

```cpp
    while (true) {
        prev_lane = prev_lane + 1;
        if (prev_lane >= waiting.size()) {
            prev_lane = 0;
        }
        if (waiting[prev_lane] > 0) {
            waiting[prev_lane]--;
            totalWaiting--;
            safe_to_merge[prev_lane].notify_one();
            break;
        }
    }
}
```

Important:

- `merging` stays true because the next car is being allowed to merge.
- The code decrements waiting counts before notifying.
- It wakes exactly one car from the selected lane.

Exam pattern name:

```text
category waiters with fairness rotation
```

Use this when:

- Waiters belong to groups.
- One group should not starve the others.
- You need to rotate among groups with waiters.

Common wrong solution:

```cpp
notify_all();
```

Why risky:

- Multiple cars may wake up.
- Unless each re-checks a strong predicate, more than one may merge.
- The problem wants one car at a time.

Common wrong solution:

```cpp
always check lane 0 first
```

Why wrong:

- Higher-numbered lanes can starve if lane 0 keeps receiving cars.

## E. How to Read Any Monitor Solution

When given monitor code on an exam, annotate it in this order:

1. What is the lock?

Find the mutex. Every shared-state access should happen while this lock is held.

2. What is the shared state?

Circle the fields:

- counters
- booleans
- vectors
- maps
- queues

3. What does each condition variable mean?

Do not write "this condition variable means ready." Write the real predicate.

Bad:

```text
cv means passenger can go
```

Better:

```text
cv wakes passengers so they can re-check available_seats > 0
```

4. Where can a thread sleep?

Look for:

```cpp
wait(lock)
```

Ask:

- What state was updated before sleeping?
- Who can change the condition?
- Who notifies?

5. Where does a thread wake others?

Look for:

```cpp
notify_one()
notify_all()
```

Ask:

- What state just changed?
- Which waiters might now proceed?
- Is one wakeup enough, or do many predicates need re-checking?

6. Does the method return while the object is in a consistent state?

Before every `return`, check:

- counts updated?
- lock released automatically?
- no resource partially reserved?
- relevant waiter notified?

## F. The Three Hardest Coding Patterns in One Table

| Pattern | Example | State Shape | Wake Strategy |
|---|---|---|---|
| Pipeline/process wiring | Practice `duet` | file descriptors and child pids | close unused fds, wait for children |
| Ordered waiters | 2023 Flight | map from key to waiter | notify one lowest key |
| Multi-resource all-or-nothing | 2024 Inventory | vector of resource counts | notify all, each waiter re-checks |
| Fair category rotation | 2022 Merge | counts per category/lane | notify one selected category |

If you can recognize which row a problem belongs to, the code becomes much easier to design.

## G. Midterm Review Problem: Dorm Selection

The dorm problem:

```cpp
std::string select_dorm(std::string &student_name, int chosen_dorm_id);
```

Rules:

- Student chooses a dorm id.
- They block until another student chooses the same dorm.
- Each returns the other student's name.
- Pairing is FIFO per dorm.

This is a same-category pair matching problem.

State:

```cpp
class DormSelection {
private:
    struct Student {
        std::string my_name;
        std::condition_variable cv;
        std::string match_name;
        bool matched = false;
    };

    std::mutex m;
    std::unordered_map<int, std::queue<Student*>> waiting;
};
```

Read this carefully:

```cpp
std::unordered_map<int, std::queue<Student*>> waiting;
```

Meaning:

```text
dorm id -> queue of students waiting for that dorm
```

Why a queue?

- The problem requires FIFO per dorm.
- The first waiting student for dorm 7 should be matched before later waiting students for dorm 7.

Annotated implementation:

```cpp
std::string DormSelection::select_dorm(std::string &student_name,
                                       int chosen_dorm_id) {
    std::unique_lock<std::mutex> ul(m);
```

Acquire the monitor lock.

```cpp
    std::queue<Student*> &q = waiting[chosen_dorm_id];
```

Get the queue for this dorm.

Important:

```cpp
waiting[chosen_dorm_id]
```

auto-creates an empty queue if this dorm id is not already in the map.

Case 1: someone is already waiting for this dorm.

```cpp
    if (!q.empty()) {
        Student *other = q.front();
        q.pop();
```

Take the oldest waiting student.

```cpp
        std::string other_name = other->my_name;
```

Save their name so this arriving student can return it.

```cpp
        other->match_name = student_name;
        other->matched = true;
        other->cv.notify_one();
```

Fill in the waiting student's result, mark them matched, and wake them.

```cpp
        return other_name;
    }
```

The arriving student returns the waiting student's name immediately.

Case 2: nobody is waiting for this dorm.

```cpp
    Student self;
    self.my_name = student_name;
    q.push(&self);
```

Create a local waiting record and put its address in the dorm queue.

Then wait:

```cpp
    while (!self.matched) {
        self.cv.wait(ul);
    }
```

Sleep until some future student chooses the same dorm and fills in this record.

Finally:

```cpp
    return self.match_name;
}
```

Return the future matching student's name.

Why is `Student self` local but still okay?

- The function is blocked inside `select_dorm`.
- Its stack frame still exists while it waits.
- A future matching student pops `&self` from the queue before notifying.
- Only after `self.matched` becomes true does this function return and destroy `self`.

What would be unsafe?

- Returning while `&self` is still stored in the queue.
- Not popping the queue entry before notifying.

Exam pattern:

```text
if compatible waiter exists:
    complete their result
    wake them
    return your result
else:
    enqueue yourself
    wait until someone completes your result
```

This pattern appears in Party-style matching too.

---

# 24. OOP Syntax for CS111 Exam Problems

The exam monitor problems often ask you to implement a class. You do not need deep OOP theory, but you do need to be comfortable with the syntax.

## Basic Class Shape

```cpp
class Inventory {
public:
    Inventory(int max_stock_id);
    ~Inventory();
    void add(int stock_id, int count);
    void reserve(std::vector<int> &stock_ids, std::vector<int> &counts);

private:
    std::mutex mutex;
    std::condition_variable new_inventory;
    std::vector<int> inventory;
};
```

Read it like this:

- `class Inventory`: defines a new type called `Inventory`.
- `public`: methods outside code is allowed to call.
- `private`: internal state/helper data only methods of the class should use.
- `Inventory(...)`: constructor, called when an object is created.
- `~Inventory()`: destructor, called when an object is destroyed.
- `mutex`, `new_inventory`, and `inventory` are member variables.

Do not forget the semicolon after the class:

```cpp
};
```

That final semicolon is mandatory.

## Declaring Methods Inside, Defining Methods Outside

Inside the class, you usually declare methods:

```cpp
class Flight {
public:
    Flight(size_t max_passengers);
    void wait_to_board(size_t position);
    bool board_next();
};
```

Outside the class, you define them using `ClassName::methodName`:

```cpp
Flight::Flight(size_t max_passengers) {
}

void Flight::wait_to_board(size_t position) {
}

bool Flight::board_next() {
    return false;
}
```

The `Flight::` part means:

```text
this function belongs to the Flight class
```

Without it:

```cpp
void wait_to_board(size_t position) {
}
```

that would be a standalone function, not a `Flight` method.

## Constructor Syntax

A constructor has the same name as the class and no return type.

```cpp
Inventory::Inventory(int max_stock_id) {
}
```

Wrong:

```cpp
void Inventory::Inventory(int max_stock_id) {
}
```

Constructors do not say `void`, `int`, or any return type.

## Initializer Lists

Exam solutions often use initializer lists:

```cpp
Inventory::Inventory(int max_stock_id)
    : mutex()
    , new_inventory()
    , inventory(max_stock_id + 1, 0)
{
}
```

This initializes member variables before the constructor body runs.

Meaning:

```cpp
mutex()
```

constructs the mutex.

```cpp
new_inventory()
```

constructs the condition variable.

```cpp
inventory(max_stock_id + 1, 0)
```

constructs a vector with `max_stock_id + 1` entries, each initialized to `0`.

Equivalent mental model:

```text
inventory has indexes 0 through max_stock_id
all counts start at 0
```

## Member Variables

Member variables are variables stored inside each object.

```cpp
class Counter {
private:
    int count;
};
```

If you create two counters:

```cpp
Counter a;
Counter b;
```

then `a` has its own `count`, and `b` has its own `count`.

For monitor problems, this matters because each object should synchronize independently.

Example:

```cpp
Inventory warehouse1(100);
Inventory warehouse2(100);
```

Each has its own:

- mutex
- condition variable
- inventory vector

## Accessing Member Variables Inside Methods

Inside a method, you can use member variables directly:

```cpp
void Inventory::add(int stock_id, int count) {
    std::unique_lock<std::mutex> lock(mutex);
    inventory[stock_id] += count;
    new_inventory.notify_all();
}
```

Here:

```cpp
mutex
inventory
new_inventory
```

refer to the member variables of the current `Inventory` object.

The hidden idea is `this`:

```cpp
this->inventory[stock_id] += count;
```

You usually do not need to write `this->`, but it is what the compiler is conceptually doing.

## Local Variables vs Member Variables

Local variable:

```cpp
void Inventory::reserve(...) {
    bool order_ok = false;
}
```

`order_ok` exists only during this method call.

Member variable:

```cpp
class Inventory {
private:
    std::vector<int> inventory;
};
```

`inventory` exists as long as the object exists.

Exam rule of thumb:

- State that must persist across method calls should be a member variable.
- Temporary calculation state should be a local variable.

Example:

```cpp
bool order_ok = false;
```

should be local because each `reserve` call has its own order.

```cpp
std::vector<int> inventory;
```

must be a member variable because all calls share the warehouse inventory.

## `public` vs `private`

```cpp
class Merge {
public:
    Merge(int num_lanes);
    void wait(int lane);
    void merged();

private:
    std::mutex mutex;
    std::vector<int> waiting;
    int totalWaiting;
    bool merging;
};
```

External code should call:

```cpp
merge.wait(lane);
merge.merged();
```

External code should not directly modify:

```cpp
merge.totalWaiting = 100;   // not allowed if private
```

For exam answers, put methods in `public` and synchronization state in `private`.

## Object Method Call Syntax

If you have an object:

```cpp
Merge m(4);
```

Call methods with dot:

```cpp
m.wait(2);
m.merged();
```

If you have a pointer to an object:

```cpp
Merge *mp = &m;
```

Call methods with arrow:

```cpp
mp->wait(2);
mp->merged();
```

Same rule as structs:

```cpp
mp->wait(2)
```

means:

```cpp
(*mp).wait(2)
```

## Destructor Syntax

Destructor:

```cpp
Flight::~Flight() {
}
```

It has:

- Same name as class.
- A `~` before the name.
- No return type.
- No parameters.

Many exam solutions have an empty destructor:

```cpp
Flight::~Flight() {}
```

If the problem asks for a destructor but there is nothing special to clean up, an empty destructor is often fine.

## Vectors as Member Variables

Declaration:

```cpp
std::vector<int> inventory;
```

Initialize in constructor:

```cpp
Inventory::Inventory(int max_stock_id)
    : inventory(max_stock_id + 1, 0)
{
}
```

Use in method:

```cpp
inventory[stock_id] += count;
```

Common vector operations:

```cpp
inventory.size()
inventory[i]
inventory.push_back(0)
```

## Condition Variables as Member Variables

One condition variable:

```cpp
std::condition_variable new_inventory;
```

Vector of condition variables:

```cpp
std::vector<std::condition_variable> safe_to_merge;
```

Map from key to condition variable pointer:

```cpp
std::map<size_t, std::condition_variable*> ready_passengers;
```

Why the flight solution uses pointers:

```cpp
std::condition_variable my_turn;
ready_passengers.emplace(position, &my_turn);
```

Each passenger has its own local condition variable. The map stores a pointer to it so the gate agent can wake that exact passenger.

## `std::unique_lock` Syntax Inside a Method

Typical monitor method:

```cpp
void Inventory::add(int stock_id, int count) {
    std::unique_lock<std::mutex> lock(mutex);
    inventory[stock_id] += count;
    new_inventory.notify_all();
}
```

Breakdown:

```cpp
std::unique_lock<std::mutex>
```

The type.

```cpp
lock
```

The local variable name.

```cpp
(mutex)
```

The mutex to lock.

So:

```cpp
std::unique_lock<std::mutex> lock(mutex);
```

means:

```text
create a local lock object that locks this object's mutex now
and unlocks it automatically when the method returns
```

## Method Return Types

Examples:

```cpp
void Merge::wait(int lane)
```

returns nothing.

```cpp
bool Flight::board_next()
```

returns `true` or `false`.

```cpp
std::string Party::meet(...)
```

returns a string.

Constructor:

```cpp
Flight::Flight(size_t max_passengers)
```

has no return type.

Destructor:

```cpp
Flight::~Flight()
```

has no return type.

## Common Syntax Mistakes

Forgetting class semicolon:

```cpp
class Foo {
}
```

Wrong. Need:

```cpp
class Foo {
};
```

Putting return type on constructor:

```cpp
void Foo::Foo() {}
```

Wrong. Need:

```cpp
Foo::Foo() {}
```

Forgetting `ClassName::`:

```cpp
void add(int stock_id, int count) {}
```

Wrong if you meant the method. Need:

```cpp
void Inventory::add(int stock_id, int count) {}
```

Accessing private state from outside:

```cpp
inventory.inventory[3] = 10;
```

Wrong design. Use methods:

```cpp
inventory.add(3, 10);
```

Confusing local variable with member variable:

```cpp
void Inventory::add(int stock_id, int count) {
    std::vector<int> inventory;     // wrong: creates new local vector
    inventory[stock_id] += count;
}
```

This shadows the member variable. Do not redeclare member state inside methods.

Correct:

```cpp
void Inventory::add(int stock_id, int count) {
    inventory[stock_id] += count;
}
```

## Mini Template To Memorize

```cpp
class Name {
public:
    Name(int arg);
    ~Name();
    void method1(int x);
    bool method2();

private:
    std::mutex mutex;
    std::condition_variable cv;
    int shared_count;
};

Name::Name(int arg)
    : mutex()
    , cv()
    , shared_count(arg)
{
}

Name::~Name() {
}

void Name::method1(int x) {
    std::unique_lock<std::mutex> lock(mutex);
    shared_count += x;
    cv.notify_all();
}

bool Name::method2() {
    std::unique_lock<std::mutex> lock(mutex);
    while (shared_count == 0) {
        cv.wait(lock);
    }
    shared_count--;
    return true;
}
```

This is the syntax shape behind most CS111 monitor answers.
