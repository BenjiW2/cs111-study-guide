# C++ / C Systems Programming Cheatsheet for CS111

This is a compact refresher for the programming dialect used in CS111.

The course is not using plain C only. It uses C++ source files and C++ library types, but many of the operating-system APIs are old C/Unix APIs. In practice, you need:

- C++ classes, constructors, destructors, `std::vector`, `std::map`, `std::deque`, `std::thread`, `std::mutex`, `std::unique_lock`, and condition variables.
- C-style pointers, arrays, strings, `char **argv`, and memory/lifetime reasoning.
- Unix C APIs such as `fork`, `execvp`, `pipe`, `dup2`, `close`, and `waitpid`.

It is not a full C++ tutorial; it focuses on the stuff you are likely to need under exam pressure.

## Mental Model

C is close to the machine:

- Variables have concrete storage.
- Pointers are addresses.
- Arrays often decay into pointers.
- Strings are `char` arrays ending with `'\0'`.
- Memory lifetime matters.
- System calls usually report errors with return values.

C++ adds:

- Classes and methods.
- Constructors/destructors.
- Standard library containers.
- RAII objects that clean up automatically.
- Threads, mutexes, and condition variables.

When stuck, ask:

- Where does this object live?
- Who owns it?
- How long is it valid?
- Is this value the thing itself, or an address of the thing?

---

# Course Dialect: What You Are Actually Writing

CS111 code often looks like this:

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <vector>
#include <map>

class Flight {
public:
    Flight(size_t max_passengers);
    void wait_to_board(size_t position);
    bool board_next();

private:
    std::mutex mutex;
    std::map<size_t, std::condition_variable *> waiters;
};
```

That is C++ code, but it may call C/Unix functions:

```cpp
pid_t pid = fork();
if (pid == 0) {
    execvp(argv[1], argv + 1);
}
waitpid(pid, NULL, 0);
```

So the right mental model is:

- Use C++ for structure and data types.
- Use C/Unix APIs for OS operations.

---

# Basic Types

Common scalar types:

```c
char c = 'A';
int x = 42;
long y = 1000000L;
size_t n = 10;      // unsigned size/count type
bool ok = true;     // C++ built-in; in C use #include <stdbool.h>
```

Integer traps:

- `size_t` is unsigned.
- Comparing signed and unsigned values can surprise you.
- Use `size_t` for sizes and indexes when matching library APIs.

---

# Pointers

## Pointer Basics

```c
int x = 5;
int *p = &x;    // p stores address of x
*p = 7;         // writes to x through p
```

Read declarations right-to-left:

```c
int *p;         // p is a pointer to int
char **argv;    // argv is a pointer to pointer to char
```

Core operators:

- `&x`: address of `x`.
- `*p`: object pointed to by `p`.

## Pointer vs Pointee

```c
int x = 10;
int *p = &x;
```

- `p` is a variable that stores an address.
- `*p` is the `int` at that address.
- `&p` is the address of the pointer variable itself.

## Null Pointers

```c
int *p = NULL;
if (p != NULL) {
    printf("%d\n", *p);
}
```

Never dereference `NULL`.

---

# Arrays and Strings

## Arrays

```c
int a[3] = {10, 20, 30};
printf("%d\n", a[1]);   // 20
```

In many expressions, `a` decays to a pointer to its first element.

```c
int *p = a;      // same as &a[0]
*(p + 1) == a[1]
```

## Array Parameter Trap

These are effectively the same as function parameters:

```c
void f(int a[]);
void f(int *a);
```

The function does not know the array length unless you pass it separately.

```c
void print_all(int *a, size_t len) {
    for (size_t i = 0; i < len; i++) {
        printf("%d\n", a[i]);
    }
}
```

## C Strings

A C string is a `char` array ending with `'\0'`.

```c
char s[] = "hello";  // {'h','e','l','l','o','\0'}
char *t = "hello";  // pointer to string literal
```

Common functions:

```c
strlen(s);          // length not counting '\0'
strcmp(a, b);       // 0 if equal
strcpy(dst, src);   // copies including '\0'; unsafe if dst too small
```

String literal trap:

```c
char *s = "hello";
s[0] = 'H';     // bad: string literals should not be modified
```

Use:

```c
char s[] = "hello";
s[0] = 'H';     // ok
```

---

# Structs

```c
typedef struct Point {
    int x;
    int y;
} Point;

Point p = {1, 2};
p.x = 10;
```

Pointer to struct:

```c
Point *ptr = &p;
ptr->x = 20;        // same as (*ptr).x = 20
```

CS111-style example:

```c
typedef struct ThreadInfo {
    vector<int> v;
    mutex m;
} ThreadInfo;
```

That example mixes C-style `struct` syntax with C++ types.

---

# C++ Classes

Basic class:

```cpp
class Counter {
public:
    Counter();
    void increment();
    int value();

private:
    int count_;
};

Counter::Counter()
    : count_(0)
{
}

void Counter::increment() {
    count_++;
}

int Counter::value() {
    return count_;
}
```

Key syntax:

- `public`: methods other code can call.
- `private`: internal state/helper methods.
- `Counter::increment`: method definition outside the class.
- `count_`: common naming style for member variables.

## Constructors and Initializer Lists

Constructor:

```cpp
class Printer {
public:
    Printer(std::string name);

private:
    std::string my_name_;
    int dummy_;
};

Printer::Printer(std::string name)
    : my_name_(name), dummy_(42)
{
}
```

The initializer list initializes member variables before the constructor body runs.

This matters for:

- `std::mutex`.
- `std::condition_variable`.
- `std::vector`.
- `std::string`.
- Members without default constructors.

## Destructors

```cpp
class Printer {
public:
    ~Printer();
};

Printer::~Printer() {
    // cleanup
}
```

Destructors run:

- For stack objects, when the object goes out of scope.
- For heap objects, when `delete` is called.

```cpp
void f() {
    Printer p1("stack");           // destructor at end of scope
    Printer *p2 = new Printer("heap");
    delete p2;                     // destructor here
}
```

## Static Members

Instance variable:

- One copy per object.

Static variable:

- One copy shared by the class.

```cpp
class Demo {
public:
    Demo();
    ~Demo();
    static int num_live();

private:
    static int live_objects;
};

int Demo::live_objects = 0;

Demo::Demo() {
    live_objects++;
}

Demo::~Demo() {
    live_objects--;
}

int Demo::num_live() {
    return live_objects;
}
```

Static methods:

- Called as `Demo::num_live()`.
- Do not have a `this` pointer.
- Cannot access instance variables unless given an object.

---

# C++ Standard Library Types You Need

## `std::string`

```cpp
std::string s = "hello";
s += " world";
std::cout << s << std::endl;
```

Use `std::string` for C++ string manipulation. Use `char *` / `char **` when calling C APIs like `execvp`.

## `std::vector`

Expandable array:

```cpp
std::vector<int> nums;
nums.push_back(10);
nums.push_back(20);

for (size_t i = 0; i < nums.size(); i++) {
    std::cout << nums[i] << std::endl;
}
```

Useful methods:

- `push_back(x)`: append.
- `size()`: number of elements.
- `empty()`: true if no elements.
- `operator[]`: indexed access.

## `std::map`

Ordered key-value map:

```cpp
std::map<size_t, std::condition_variable *> waiters;
waiters[position] = &cv;

if (waiters.find(position) != waiters.end()) {
    // key exists
}
```

Important:

- Keys are ordered.
- `begin()` gives the smallest key.
- `map[key]` auto-inserts if the key is missing.

This is useful for problems like "wake the lowest boarding position."

```cpp
auto it = waiters.begin();
it->second->notify_one();
waiters.erase(it);
```

## `std::unordered_map`

Hash map:

- Fast key lookup.
- No ordering.

Use `std::map` when order matters. Use `std::unordered_map` when order does not matter.

## `std::deque`

Double-ended queue:

```cpp
std::deque<int> q;
q.push_back(1);
q.push_front(2);

int x = q.front();
q.pop_front();
```

Important:

- `pop_front()` removes but does not return the value.
- Call `front()` first if you need the value.

---

# Lambdas

A lambda is an inline function object.

General shape:

```cpp
[captures](arguments) -> return_type {
    body
}
```

Example:

```cpp
int target = 50;
std::sort(&values[0], &values[num_values],
    [target](int a, int b) {
        return abs(a - target) < abs(b - target);
    });
```

Capture examples:

```cpp
[x]     // capture x by value, makes a copy
[&x]    // capture x by reference
[&]     // capture needed variables by reference
[=]     // capture needed variables by value
```

For exam reasoning, the main issue is whether the lambda sees a copy or the original variable.

---

---

# Stack vs Heap

## Stack Allocation

```c
void f(void) {
    int x = 5;      // stack local
}
```

Stack locals disappear when the function returns.

Bad:

```c
int *bad(void) {
    int x = 5;
    return &x;      // returns pointer to dead stack variable
}
```

## Heap Allocation

```c
int *p = malloc(sizeof(int));
*p = 5;
free(p);
```

For arrays:

```c
int *a = malloc(n * sizeof(int));
for (size_t i = 0; i < n; i++) {
    a[i] = 0;
}
free(a);
```

Always check allocation in production code:

```c
if (a == NULL) {
    perror("malloc");
    exit(1);
}
```

Common memory bugs:

- Leak: forgetting `free`.
- Use after free: using `p` after `free(p)`.
- Double free: calling `free(p)` twice.
- Buffer overflow: writing past allocation.
- Returning address of stack local.

---

# `sizeof`

```c
sizeof(int)
sizeof(char *)
sizeof arr
```

Trap:

```c
void f(int *a) {
    sizeof(a);      // size of pointer, not array
}
```

Use this allocation pattern:

```c
int *a = malloc(n * sizeof(*a));
```

This stays correct if the type of `a` changes.

---

# Function Pointers and `argv`

## `main`

```c
int main(int argc, char *argv[]) {
    // argv[0] is program name
    // argv[1] is first argument, if argc > 1
}
```

`char *argv[]` means array of C strings. As a parameter, it behaves like:

```c
char **argv
```

## `execvp`

```c
execvp(argv[0], argv);
```

`execvp` expects:

- Program name/path.
- Argument vector terminated by `NULL`.

Example:

```c
char *args[] = {"ls", "-l", NULL};
execvp(args[0], args);
```

If `execvp` succeeds, it does not return.

---

# Error Handling

Many Unix calls return:

- `-1` on error.
- `0` or positive values on success.

Pattern:

```c
if (pipe(fds) < 0) {
    perror("pipe");
    exit(1);
}
```

`perror("pipe")` prints a message based on `errno`.

For exam code, exact error handling may not be required, but knowing the return convention helps.

---

# File Descriptors

File descriptors are small integers local to a process.

Standard descriptors:

```c
STDIN_FILENO   // 0
STDOUT_FILENO  // 1
STDERR_FILENO  // 2
```

Include:

```c
#include <unistd.h>
```

Useful calls:

```c
read(fd, buf, count);
write(fd, buf, count);
close(fd);
dup2(oldfd, newfd);
```

---

# `fork`

```c
pid_t pid = fork();

if (pid == 0) {
    // child
} else {
    // parent; pid is child's pid
}
```

After `fork`:

- Parent and child both continue after the `fork`.
- They have separate address spaces.
- File descriptors are inherited.
- Changes to normal memory in child do not change parent memory.

Common pattern:

```c
pid_t pid = fork();
if (pid == 0) {
    execvp(argv[1], argv + 1);
    perror("execvp");
    exit(1);
}
waitpid(pid, NULL, 0);
```

---

# `pipe`, `dup2`, and `close`

## Pipe Basics

```c
int fds[2];
pipe(fds);
```

- `fds[0]`: read end.
- `fds[1]`: write end.

## Redirect stdout to Pipe

```c
dup2(fds[1], STDOUT_FILENO);
close(fds[1]);
```

Now writes to stdout go into the pipe.

## Redirect stdin from Pipe

```c
dup2(fds[0], STDIN_FILENO);
close(fds[0]);
```

Now reads from stdin come from the pipe.

## Pipeline Pattern

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

Critical rule:

- Close every pipe end a process does not need.

EOF rule:

- A pipe reader sees EOF only when all write ends are closed.

If some process accidentally keeps the write end open, the reader may block forever.

---

# `waitpid`

```c
int status;
waitpid(pid, &status, 0);
```

Use:

- Parent waits for a specific child.
- Prevents zombie processes.
- Lets parent observe completion.

For simple exam code:

```c
waitpid(pid, NULL, 0);
```

---

# Timing a Program

Pattern from practice midterm:

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

Meaning:

- Parent records start time.
- Child runs target command.
- Parent waits.
- Parent records finish time.

---

# C APIs vs C++ APIs in This Class

CS111 examples may mix styles:

C-style:

- `fork`
- `execvp`
- `pipe`
- `dup2`
- `close`
- `waitpid`
- `char *argv[]`
- `struct timespec`

C++-style:

- `std::thread`
- `std::mutex`
- `std::condition_variable`
- `std::vector`
- `std::map`
- `std::unique_lock`

Do not let the mixed syntax throw you. The OS calls are C APIs; the synchronization problems are usually C++ monitor-style code.

The course handout specifically lists these useful C++ types:

- `std::string`
- `std::vector`
- `std::map`
- `std::unordered_map`
- `std::queue`
- `std::deque`
- `std::thread`
- `std::mutex`
- `std::unique_lock`
- `std::condition_variable_any`
- `std::function`

Past midterms and examples may also use `std::condition_variable`. The concept is the same for your purposes: wait on a condition while releasing the lock, then reacquire the lock when woken.

---

# C++ Monitor Syntax You Should Remember

```cpp
#include <mutex>
#include <condition_variable>

class Thing {
public:
    void wait_until_ready();
    void make_ready();

private:
    std::mutex mutex;
    std::condition_variable cv;
    bool ready = false;
};

void Thing::wait_until_ready() {
    std::unique_lock<std::mutex> lock(mutex);
    while (!ready) {
        cv.wait(lock);
    }
}

void Thing::make_ready() {
    std::unique_lock<std::mutex> lock(mutex);
    ready = true;
    cv.notify_one();
}
```

Important:

- `std::unique_lock<std::mutex>` is needed for condition variable waits.
- `cv.wait(lock)` releases the lock while sleeping and reacquires it before returning.
- Use `while`, not `if`.

## `std::condition_variable_any`

Some CS111 handouts mention `std::condition_variable_any`. It is similar in spirit to `std::condition_variable`, but it can work with more lock-like types.

Typical pattern is still:

```cpp
std::unique_lock<std::mutex> lock(mutex);
while (!predicate) {
    cv.wait(lock);
}
```

For the course's custom lock assignment, the API may be simplified:

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

The semantics are the same:

- `wait(m)` unlocks `m`.
- The thread blocks.
- After wakeup, it locks `m` again before returning.

---

# Common Exam Traces

## Global with Threads

```cpp
int x = 100;

void f() {
    x++;
}
```

Threads in the same process share `x`.

## Global with `fork`

```c
int x = 100;

pid_t pid = fork();
if (pid == 0) {
    x += 100;
} else {
    waitpid(pid, NULL, 0);
    printf("%d\n", x);
}
```

Parent still prints `100`, because parent and child have separate address spaces after `fork`.

## Local Shadowing

```c
int x = 1;

void f(void) {
    int x = 2;
    printf("%d\n", x);    // 2
}
```

Inner/local `x` shadows outer/global `x`.

---

# Quick Checklist Before Writing C Code

- Did every `fork` branch handle parent vs child?
- Does every child that should run another program call `execvp`?
- If `execvp` returns, do you call `exit`?
- Did the parent `waitpid` for children?
- For pipes, did each process close unused ends?
- Did you use `dup2` before closing the descriptor you need?
- Did you avoid returning pointers to stack locals?
- Did you pass array lengths separately?
- Did you remember C strings need `NULL` or `'\0'` termination depending on context?
