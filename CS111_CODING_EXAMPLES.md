---
title: "CS111 Coding Examples"
---

# CS111 Coding Examples

This page is for the coding section. It is deliberately code-heavy.

Assumption: Unix/process/file-descriptor examples are C-style code. Synchronization examples use the C++ standard library syntax that appears in CS111-style monitor questions: `std::mutex`, `std::unique_lock`, `std::condition_variable`, `std::deque`, and `std::thread`.

## How To Use This Page

For each practice coding problem, first classify it:

- Process code: `fork`, `execvp`, `waitpid`, `pipe`, `dup2`, file descriptors.
- Synchronization code: mutex, condition variable, monitor, exact waiter selection.
- Memory code: address translation, base/bound, page table indexes.
- Filesystem code: Unix V6 helpers, inodes, directories, indirect blocks.
- Crash code: ordered writes, logging, replay.

Then copy the closest skeleton and adapt the state variables.

## API Reference: Process, Exec, FD

```c
pid_t fork(void);
// child: returns 0
// parent: returns child's PID
// error: returns -1, no child created

int execvp(const char *file, char *const argv[]);
// replaces current process image
// searches PATH because of the "p"
// returns only on failure

pid_t waitpid(pid_t pid, int *status, int options);
// pid > 0: wait for that exact child
// pid == -1: wait for any child
// options == 0: blocking wait

int pipe(int fds[2]);
// fds[0] = read end
// fds[1] = write end

int dup2(int oldfd, int newfd);
// makes newfd refer to same open file description as oldfd
// direction matters: dup2(file_fd, STDOUT_FILENO)

int open(const char *path, int flags, mode_t mode);
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
int close(int fd);
```

Exam traps:

- `execvp` does not create a process. It overwrites the current one.
- If `execvp` succeeds, code after it does not run.
- `fork` copies memory but shares inherited open-file descriptions.
- Pipe EOF happens only after every write end is closed in every process.
- `dup2(oldfd, newfd)` changes `newfd`, not `oldfd`.

## Example 1: `argv` And `execvp`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    char *args[] = {"ls", "-l", "/tmp", NULL};

    execvp(args[0], args);

    // Only runs if execvp fails.
    perror("execvp");
    exit(1);
}
```

What is happening:

- `args[0]` is the program name: `"ls"`.
- `args[1]` and beyond are command-line arguments.
- The `args` array must end with `NULL`.
- Each string like `"ls"` is itself a `char[]` ending in `'\0'`.
- The argument vector ends with `NULL`; the individual strings end with `'\0'`.

## Example 2: Shell Pattern: `fork` Then `execvp`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
    char *args[] = {"wc", "-l", "input.txt", NULL};

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        exit(1);
    }

    if (pid == 0) {
        // Child becomes wc.
        execvp(args[0], args);
        perror("execvp");
        exit(1);
    }

    // Parent is still the original program.
    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status)) {
        printf("child exited with %d\n", WEXITSTATUS(status));
    }
}
```

Why this is the shell pattern:

- Parent keeps control and can wait.
- Child changes into the command.
- Same child PID exists before and after `execvp`.

## Example 3: Multiple `fork` Values

```c
pid_t a = fork();
pid_t b = fork();

printf("a=%d b=%d\n", a, b);
```

There are four processes:

```text
P0: a = PID(P1), b = PID(P2)
P1: a = 0,       b = PID(P3)
P2: a = PID(P1), b = 0
P3: a = 0,       b = 0
```

Rule:

- A process sees `0` only for a fork call where it is the newly created child.
- A child created by the second fork inherits the value of `a` from its parent.

## Example 4: Loop Fork Bug

Bad:

```c
for (int i = 0; i < 3; i++) {
    if (fork() == 0) {
        printf("child for i=%d\n", i);
        // BUG: child keeps looping and forks more children.
    }
}
```

Correct:

```c
#include <sys/wait.h>
#include <unistd.h>
#include <stdlib.h>

for (int i = 0; i < 3; i++) {
    pid_t pid = fork();

    if (pid == 0) {
        do_child_work(i);
        exit(0);       // critical
    }
}

while (waitpid(-1, NULL, 0) > 0) {
    // reap every child
}
```

Why `exit(0)` matters:

- The child should do its child job and stop.
- If it falls through, it continues the parent loop and creates more children than intended.

## Example 5: Redirect Child Stdout To A File

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
    int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open");
        exit(1);
    }

    pid_t pid = fork();

    if (pid == 0) {
        // Make stdout point at the file.
        dup2(fd, STDOUT_FILENO);

        // fd is no longer needed; stdout now refers to same open file.
        close(fd);

        char *args[] = {"echo", "hello", NULL};
        execvp(args[0], args);
        perror("execvp");
        exit(1);
    }

    close(fd);
    waitpid(pid, NULL, 0);
}
```

Key line:

```c
dup2(fd, STDOUT_FILENO);
```

This means future writes to fd `1` go to the same file as `fd`.

## Example 6: Pipeline `left | right`

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
    char *left[]  = {"ls", NULL};
    char *right[] = {"wc", "-l", NULL};

    int p[2];
    if (pipe(p) < 0) {
        perror("pipe");
        exit(1);
    }

    pid_t left_pid = fork();
    if (left_pid == 0) {
        // left stdout -> pipe write end
        dup2(p[1], STDOUT_FILENO);

        // close inherited pipe fds after dup2
        close(p[0]);
        close(p[1]);

        execvp(left[0], left);
        perror("exec left");
        exit(1);
    }

    pid_t right_pid = fork();
    if (right_pid == 0) {
        // right stdin <- pipe read end
        dup2(p[0], STDIN_FILENO);

        close(p[0]);
        close(p[1]);

        execvp(right[0], right);
        perror("exec right");
        exit(1);
    }

    // Parent must close both ends too.
    close(p[0]);
    close(p[1]);

    waitpid(left_pid, NULL, 0);
    waitpid(right_pid, NULL, 0);
}
```

What goes wrong if parent forgets `close(p[1])`:

- `wc -l` reads until EOF.
- EOF happens only when all write ends are closed.
- If parent keeps a write end open, `wc` may block forever.

## Example 7: Pipe Read/Write Without `exec`

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
    int p[2];
    pipe(p);

    pid_t pid = fork();

    if (pid == 0) {
        // child reads
        close(p[1]);

        char buf[100];
        int n = read(p[0], buf, sizeof(buf) - 1);
        buf[n] = '\0';

        printf("child got: %s\n", buf);
        close(p[0]);
        exit(0);
    }

    // parent writes
    close(p[0]);
    write(p[1], "hello", strlen("hello"));
    close(p[1]);       // sends EOF

    waitpid(pid, NULL, 0);
}
```

Use this to understand pipe direction before adding `dup2` and `execvp`.

## Example 8: FD Sharing After `fork`

```c
int fd = open("data.txt", O_RDONLY);

pid_t pid = fork();

if (pid == 0) {
    char c;
    read(fd, &c, 1);
    printf("child read %c\n", c);
    exit(0);
}

char c;
read(fd, &c, 1);
printf("parent read %c\n", c);

waitpid(pid, NULL, 0);
```

Important:

- Parent and child have separate fd tables.
- But the inherited fd entries can point to the same open-file description.
- The open-file description contains the file offset.
- Therefore, reads can advance a shared offset.
- Output order is nondeterministic, so which process reads the first byte is nondeterministic.

## API Reference: Threads And Monitors

```cpp
std::thread t(function, args...);
t.join();

std::mutex m;
m.lock();
m.unlock();

std::unique_lock<std::mutex> lock(m);
// locks immediately
// unlocks automatically at end of scope

std::condition_variable cv;
cv.wait(lock);
cv.notify_one();
cv.notify_all();
```

Condition-variable rule:

```cpp
std::unique_lock<std::mutex> lock(m);
while (!predicate) {
    cv.wait(lock);
}
// predicate is true here while lock is held
```

Never use `if` for a CV wait predicate.

## Example 9: Lost Update Race

Bad:

```cpp
#include <thread>
#include <vector>

int counter = 0;

void worker() {
    for (int i = 0; i < 100000; i++) {
        counter++;   // not atomic
    }
}

int main() {
    std::thread a(worker);
    std::thread b(worker);

    a.join();
    b.join();
}
```

Why `counter++` is not atomic:

```text
load counter
add 1
store counter
```

Possible bad interleaving:

```text
counter = 0
T1 loads 0
T2 loads 0
T1 stores 1
T2 stores 1
```

Two increments happened, but final value is `1`.

Correct:

```cpp
#include <mutex>

int counter = 0;
std::mutex counter_mutex;

void worker() {
    for (int i = 0; i < 100000; i++) {
        std::unique_lock<std::mutex> lock(counter_mutex);
        counter++;
    } // unlocks here
}
```

## Example 10: Bounded Buffer Monitor

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>

class BoundedBuffer {
private:
    std::mutex m;
    std::condition_variable nonempty;
    std::condition_variable nonfull;
    std::queue<int> q;
    int capacity;

public:
    BoundedBuffer(int cap) : capacity(cap) {}

    void put(int item) {
        std::unique_lock<std::mutex> lock(m);

        while ((int) q.size() == capacity) {
            nonfull.wait(lock);
        }

        q.push(item);

        // State changed: maybe a getter can proceed.
        nonempty.notify_one();
    }

    int get() {
        std::unique_lock<std::mutex> lock(m);

        while (q.empty()) {
            nonempty.wait(lock);
        }

        int item = q.front();
        q.pop();

        // State changed: maybe a putter can proceed.
        nonfull.notify_one();

        return item;
    }
};
```

Pattern:

- Lock at top of monitor method.
- Wait while cannot proceed.
- Change state.
- Notify threads whose predicate may now be true.

## Example 11: Bridge Monitor

Problem shape: cars can cross a one-lane bridge. At most 3 cars may be on the bridge. Cars moving opposite directions cannot be on the bridge at the same time.

```cpp
#include <condition_variable>
#include <mutex>

enum Direction { EAST = 0, WEST = 1 };

class Bridge {
private:
    std::mutex m;
    std::condition_variable cv[2];

    int on_bridge = 0;
    Direction current = EAST;
    int waiting[2] = {0, 0};

public:
    void arrive(Direction dir) {
        std::unique_lock<std::mutex> lock(m);
        waiting[dir]++;

        while ((on_bridge > 0 && current != dir) ||
               on_bridge == 3) {
            cv[dir].wait(lock);
        }

        waiting[dir]--;
        current = dir;
        on_bridge++;
    }

    void leave(Direction dir) {
        std::unique_lock<std::mutex> lock(m);
        on_bridge--;

        if (on_bridge == 0) {
            Direction other = (dir == EAST) ? WEST : EAST;

            if (waiting[other] > 0) {
                current = other;
                cv[other].notify_all();
            } else {
                cv[dir].notify_all();
            }
        } else {
            cv[dir].notify_all();
        }
    }
};
```

What an exam might ask you to improve:

- Add fairness so one direction cannot starve.
- Add a consecutive-car limit.
- Track waiting counts per direction.

## Example 12: Exact Selected Waiters

Problem shape: exactly two hydrogen threads and one oxygen thread form a group. Only selected threads may call `bond()`.

```cpp
#include <condition_variable>
#include <deque>
#include <mutex>

struct Waiter {
    bool assigned = false;
    std::condition_variable cv;
};

class Water {
private:
    std::mutex m;
    std::deque<Waiter *> hydrogens;
    std::deque<Waiter *> oxygens;

    void try_make_group() {
        if (hydrogens.size() >= 2 && oxygens.size() >= 1) {
            Waiter *h1 = hydrogens.front();
            hydrogens.pop_front();

            Waiter *h2 = hydrogens.front();
            hydrogens.pop_front();

            Waiter *o = oxygens.front();
            oxygens.pop_front();

            h1->assigned = true;
            h2->assigned = true;
            o->assigned = true;

            h1->cv.notify_one();
            h2->cv.notify_one();
            o->cv.notify_one();
        }
    }

public:
    void hydrogen() {
        Waiter self;

        std::unique_lock<std::mutex> lock(m);
        hydrogens.push_back(&self);
        try_make_group();

        while (!self.assigned) {
            self.cv.wait(lock);
        }

        lock.unlock();
        bond();
    }

    void oxygen() {
        Waiter self;

        std::unique_lock<std::mutex> lock(m);
        oxygens.push_back(&self);
        try_make_group();

        while (!self.assigned) {
            self.cv.wait(lock);
        }

        lock.unlock();
        bond();
    }
};
```

Syntax:

- `std::deque<Waiter *> hydrogens;` is a queue-like container of pointers.
- `push_back(&self)` stores the address of this thread's waiter record.
- `front()` returns the first pointer.
- `pop_front()` removes it.
- `h1->cv.notify_one()` means `(*h1).cv.notify_one()`.

Why not just counters:

- If you call `notify_all`, extra hydrogens may wake.
- A new arrival may steal a spot from an already-notified thread.
- Exact-selection problems need identity state, not only counts.

## Example 13: All-Or-Nothing Inventory

Problem shape: an order requires multiple item counts atomically. It must not partially reserve one item and wait for another.

```cpp
#include <condition_variable>
#include <map>
#include <mutex>
#include <vector>

class Inventory {
private:
    std::mutex m;
    std::condition_variable changed;
    std::map<int, int> stock;

    bool can_fill(const std::vector<int>& ids,
                  const std::vector<int>& counts) {
        for (size_t i = 0; i < ids.size(); i++) {
            if (stock[ids[i]] < counts[i]) {
                return false;
            }
        }
        return true;
    }

public:
    void add(int id, int count) {
        std::unique_lock<std::mutex> lock(m);
        stock[id] += count;
        changed.notify_all();
    }

    void order(const std::vector<int>& ids,
               const std::vector<int>& counts) {
        std::unique_lock<std::mutex> lock(m);

        while (!can_fill(ids, counts)) {
            changed.wait(lock);
        }

        // Only subtract after all items are known available.
        for (size_t i = 0; i < ids.size(); i++) {
            stock[ids[i]] -= counts[i];
        }
    }
};
```

Why `notify_all`:

- Different orders wait for different item combinations.
- Adding item `7` may help one waiter but not another.

## Example 14: Reusable Barrier

Problem shape: `N` threads wait until all `N` have arrived, then all proceed. The barrier can be reused.

```cpp
#include <condition_variable>
#include <mutex>

class Barrier {
private:
    std::mutex m;
    std::condition_variable cv;
    int arrived = 0;
    int generation = 0;
    int n;

public:
    Barrier(int total) : n(total) {}

    void wait() {
        std::unique_lock<std::mutex> lock(m);
        int my_generation = generation;

        arrived++;

        if (arrived == n) {
            arrived = 0;
            generation++;
            cv.notify_all();
        } else {
            while (generation == my_generation) {
                cv.wait(lock);
            }
        }
    }
};
```

Why `generation` matters:

- Without it, a thread from the next round might pass because of an old notification.
- Reusable barriers need to distinguish rounds.

## Example 15: Deadlock From Lock Ordering

Bad:

```cpp
// Thread 1
lock(A);
lock(B);
// ...
unlock(B);
unlock(A);

// Thread 2
lock(B);
lock(A);
// ...
unlock(A);
unlock(B);
```

Correct:

```cpp
// Every thread uses same order.
lock(A);
lock(B);
// ...
unlock(B);
unlock(A);
```

Reason:

- Bad version can create circular wait.
- Good version removes circular wait by imposing global order.

## Example 16: Base And Bound Translation

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

uintptr_t translate(uintptr_t va, uintptr_t base, uintptr_t bound) {
    if (va >= bound) {
        fprintf(stderr, "address fault\n");
        exit(1);
    }

    return base + va;
}
```

Example:

```text
base = 50000
bound = 20000
VA = 1000
PA = 51000

VA = 25000 -> trap because VA >= bound
```

## Example 17: Page Table Translation

```c
#include <stdint.h>
#include <stdbool.h>

typedef struct {
    bool valid;
    bool readable;
    bool writable;
    uint64_t ppn;
} pte_t;

uint64_t translate(uint64_t va, pte_t *page_table, int page_bits) {
    uint64_t offset_mask = (1ULL << page_bits) - 1;

    uint64_t offset = va & offset_mask;
    uint64_t vpn = va >> page_bits;

    pte_t pte = page_table[vpn];

    if (!pte.valid) {
        page_fault();
    }

    return (pte.ppn << page_bits) | offset;
}
```

Key facts:

- Page size `2^k` means `k` offset bits.
- Offset is copied unchanged from virtual to physical address.
- Page table maps VPN to PPN.

## Example 18: Multilevel Page Table Indexes

For 48-bit virtual addresses, 4KB pages, 8-byte entries:

```text
page size = 4096 = 2^12 -> offset = 12 bits
entries per page-table page = 4096 / 8 = 512 = 2^9
VPN bits = 48 - 12 = 36
levels = 36 / 9 = 4
```

Code to extract indexes:

```c
uint64_t offset = va & 0xfff;

uint64_t l1 = (va >> 39) & 0x1ff;
uint64_t l2 = (va >> 30) & 0x1ff;
uint64_t l3 = (va >> 21) & 0x1ff;
uint64_t l4 = (va >> 12) & 0x1ff;
```

Why `0x1ff`:

- 9 bits all set is binary `111111111`.
- Hex value is `0x1ff`.

## API Reference: Assignment 7 Unix V6

```c
int diskimg_readsector(int dfd, int sectorNum, void *buf);

int inode_iget(const struct unixfilesystem *fs,
               int inumber,
               struct inode *inp);

int inode_getsize(struct inode *inp);

int inode_indexlookup(const struct unixfilesystem *fs,
                      struct inode *inp,
                      int fileBlockIndex);

int file_getblock(const struct unixfilesystem *fs,
                  int inumber,
                  int fileBlockIndex,
                  void *buf);

int directory_findname(const struct unixfilesystem *fs,
                       const char *name,
                       int dirinumber,
                       struct direntv6 *dirEnt);

int pathname_lookup(const struct unixfilesystem *fs,
                    const char *pathname);
```

Useful constants:

```c
#define DISKIMG_SECTOR_SIZE 512
#define INODE_START_SECTOR  2
#define ROOT_INUMBER        1
#define MAX_COMPONENT_LENGTH 14
```

Useful derived facts:

```c
int inodes_per_sector =
    DISKIMG_SECTOR_SIZE / sizeof(struct inode);

int ptrs_per_indirect =
    DISKIMG_SECTOR_SIZE / sizeof(uint16_t); // 256

int dirents_per_block =
    DISKIMG_SECTOR_SIZE / sizeof(struct direntv6); // 32
```

## Example 19: Fetching A Unix V6 Inode

```c
int inode_iget(const struct unixfilesystem *fs,
               int inumber,
               struct inode *inp) {
    struct inode inodes[DISKIMG_SECTOR_SIZE / sizeof(struct inode)];

    int inode_index = inumber - 1;
    int inodes_per_sector =
        DISKIMG_SECTOR_SIZE / sizeof(struct inode);

    int sector =
        INODE_START_SECTOR + inode_index / inodes_per_sector;

    int offset =
        inode_index % inodes_per_sector;

    if (diskimg_readsector(fs->dfd, sector, inodes)
        != DISKIMG_SECTOR_SIZE) {
        return -1;
    }

    *inp = inodes[offset];
    return 0;
}
```

Why `inumber - 1`:

- Unix V6 inode numbers start at 1.
- C arrays start at 0.
- Inumber 1 is stored at inode index 0.

## Example 20: Unix V6 Small/Large Inode Lookup

```c
int inode_indexlookup(const struct unixfilesystem *fs,
                      struct inode *inp,
                      int fileBlockIndex) {
    uint16_t ptrs[DISKIMG_SECTOR_SIZE / sizeof(uint16_t)];
    int n = DISKIMG_SECTOR_SIZE / sizeof(uint16_t);

    if (!(inp->i_mode & ILARG)) {
        // Small inode: i_addr entries are direct data sectors.
        return inp->i_addr[fileBlockIndex];
    }

    if (fileBlockIndex < 7 * n) {
        // Large inode, singly indirect region.
        int which_indirect = fileBlockIndex / n;
        int offset = fileBlockIndex % n;

        int indirect_sector = inp->i_addr[which_indirect];

        if (diskimg_readsector(fs->dfd, indirect_sector, ptrs)
            != DISKIMG_SECTOR_SIZE) {
            return -1;
        }

        return ptrs[offset];
    }

    // Doubly indirect region.
    fileBlockIndex -= 7 * n;

    int outer = fileBlockIndex / n;
    int inner = fileBlockIndex % n;

    uint16_t outer_ptrs[DISKIMG_SECTOR_SIZE / sizeof(uint16_t)];
    uint16_t inner_ptrs[DISKIMG_SECTOR_SIZE / sizeof(uint16_t)];

    if (diskimg_readsector(fs->dfd, inp->i_addr[7], outer_ptrs)
        != DISKIMG_SECTOR_SIZE) {
        return -1;
    }

    int inner_sector = outer_ptrs[outer];

    if (diskimg_readsector(fs->dfd, inner_sector, inner_ptrs)
        != DISKIMG_SECTOR_SIZE) {
        return -1;
    }

    return inner_ptrs[inner];
}
```

Exam traps:

- Small V6 inode: `i_addr[0..7]` are direct data sectors.
- Large V6 inode: `i_addr[0..6]` are singly indirect sectors.
- Large V6 inode: `i_addr[7]` is doubly indirect.
- The doubly-indirect block itself is not a data block.

## Example 21: `file_getblock`

```c
int file_getblock(const struct unixfilesystem *fs,
                  int inumber,
                  int fileBlockIndex,
                  void *buf) {
    struct inode in;

    if (inode_iget(fs, inumber, &in) < 0) {
        return -1;
    }

    int size = inode_getsize(&in);
    int first_byte = fileBlockIndex * DISKIMG_SECTOR_SIZE;

    if (first_byte >= size) {
        return 0;
    }

    int sector = inode_indexlookup(fs, &in, fileBlockIndex);
    if (sector < 0) {
        return -1;
    }

    if (diskimg_readsector(fs->dfd, sector, buf)
        != DISKIMG_SECTOR_SIZE) {
        return -1;
    }

    int remaining = size - first_byte;
    if (remaining < DISKIMG_SECTOR_SIZE) {
        return remaining;
    }

    return DISKIMG_SECTOR_SIZE;
}
```

Why return valid byte count:

- Most blocks are full 512-byte sectors.
- Last file block may contain fewer valid bytes.

## Example 22: Directory Scan

```c
int directory_findname(const struct unixfilesystem *fs,
                       const char *name,
                       int dirinumber,
                       struct direntv6 *dirEnt) {
    struct inode dir_inode;

    if (inode_iget(fs, dirinumber, &dir_inode) < 0) {
        return -1;
    }

    if (!(dir_inode.i_mode & IALLOC)) {
        return -1;
    }

    if ((dir_inode.i_mode & IFMT) != IFDIR) {
        return -1;
    }

    int size = inode_getsize(&dir_inode);
    int nblocks = (size + DISKIMG_SECTOR_SIZE - 1)
                / DISKIMG_SECTOR_SIZE;

    char block[DISKIMG_SECTOR_SIZE];

    for (int b = 0; b < nblocks; b++) {
        int valid = file_getblock(fs, dirinumber, b, block);
        if (valid < 0) {
            return -1;
        }

        struct direntv6 *entries = (struct direntv6 *) block;
        int nentries = valid / sizeof(struct direntv6);

        for (int i = 0; i < nentries; i++) {
            if (entries[i].d_inumber == 0) {
                continue;
            }

            if (strncmp(name, entries[i].d_name,
                        MAX_COMPONENT_LENGTH) == 0) {
                *dirEnt = entries[i];
                return 0;
            }
        }
    }

    return -1;
}
```

Why `strncmp`:

- V6 names are 14 bytes.
- A 14-character name may not end in `'\0'`.

## Example 23: Pathname Lookup

```c
int pathname_lookup(const struct unixfilesystem *fs,
                    const char *pathname) {
    if (pathname[0] != '/') {
        return -1;
    }

    int current = ROOT_INUMBER;

    // Make a mutable copy because strtok modifies the string.
    char path[strlen(pathname) + 1];
    strcpy(path, pathname);

    char *component = strtok(path, "/");

    while (component != NULL) {
        struct direntv6 de;

        if (directory_findname(fs, component, current, &de) < 0) {
            return -1;
        }

        current = de.d_inumber;
        component = strtok(NULL, "/");
    }

    return current;
}
```

Path `/usr/bin/ls`:

```text
root inode 1
  lookup "usr" -> usr inode
  lookup "bin" -> bin inode
  lookup "ls"  -> ls inode
```

## Example 24: Scan All Allocated Inodes

```c
int scan_all_inodes(const struct unixfilesystem *fs) {
    int inodes_per_sector =
        DISKIMG_SECTOR_SIZE / sizeof(struct inode);

    int total_inodes =
        fs->superblock.s_isize * inodes_per_sector;

    for (int inum = 1; inum <= total_inodes; inum++) {
        struct inode in;

        if (inode_iget(fs, inum, &in) < 0) {
            return -1;
        }

        if (!(in.i_mode & IALLOC)) {
            continue;
        }

        // Check whatever property the exam asks for.
    }

    return 0;
}
```

Use when prompt says:

- Find every inode with property X.
- Count files.
- Find whether some disk block is used as metadata.

## Example 25: Find Whether A Block Is An Indirect Block

```c
int is_indirect_block(const struct unixfilesystem *fs,
                      int block_num) {
    int inodes_per_sector =
        DISKIMG_SECTOR_SIZE / sizeof(struct inode);

    int total_inodes =
        fs->superblock.s_isize * inodes_per_sector;

    for (int inum = 1; inum <= total_inodes; inum++) {
        struct inode in;

        if (inode_iget(fs, inum, &in) < 0) {
            return -1;
        }

        if (!(in.i_mode & IALLOC)) {
            continue;
        }

        if (!(in.i_mode & ILARG)) {
            continue;
        }

        // Large inode: i_addr[0..6] are singly indirect blocks.
        for (int i = 0; i < 7; i++) {
            if (in.i_addr[i] == block_num) {
                return 1;
            }
        }

        // i_addr[7] is doubly indirect. The entries inside it
        // point to singly indirect blocks.
        uint16_t ptrs[DISKIMG_SECTOR_SIZE / sizeof(uint16_t)];

        if (diskimg_readsector(fs->dfd, in.i_addr[7], ptrs)
            != DISKIMG_SECTOR_SIZE) {
            return -1;
        }

        for (int i = 0; i < 256; i++) {
            if (ptrs[i] == block_num) {
                return 1;
            }
        }
    }

    return 0;
}
```

Subtle point:

- `in.i_addr[7]` is the doubly-indirect block.
- If the question asks for an indirect block, do not count the doubly-indirect block unless the prompt explicitly includes it.

## Example 26: Hard Link Skeleton

```c
int hard_link(const struct unixfilesystem *fs,
              const char *target_path,
              const char *link_path) {
    int target_inum = pathname_lookup(fs, target_path);
    if (target_inum < 0) {
        return -1;
    }

    struct inode target;
    if (inode_iget(fs, target_inum, &target) < 0) {
        return -1;
    }

    if ((target.i_mode & IFMT) == IFDIR) {
        return -1; // usually reject hard links to directories
    }

    // 1. Split link_path into parent directory path + final name.
    // 2. Look up parent directory inode.
    // 3. Find empty dirent slot in parent directory.
    // 4. Write dirent: d_inumber = target_inum, d_name = final name.
    // 5. Increment target.i_nlink.
    // 6. Write modified directory block and modified inode if prompt asks.

    return 0;
}
```

Concept:

- Hard link creates another directory entry pointing at the same inode.
- It does not copy file data.
- It increments the inode link count.

## Example 27: Upgrade Small V6 Inode To Large

Problem shape: convert a small inode into large format while preserving file contents.

```c
int inode_upgrade(const struct unixfilesystem *fs,
                  struct inode *inp,
                  int new_indirect_sector) {
    uint16_t ptrs[DISKIMG_SECTOR_SIZE / sizeof(uint16_t)];

    // Copy old direct data block addresses into new indirect block.
    for (int i = 0; i < 8; i++) {
        ptrs[i] = inp->i_addr[i];
    }

    // Zero remaining entries.
    for (int i = 8; i < 256; i++) {
        ptrs[i] = 0;
    }

    // Clear old inode address array.
    for (int i = 0; i < 8; i++) {
        inp->i_addr[i] = 0;
    }

    // New large inode points first address at indirect block.
    inp->i_addr[0] = new_indirect_sector;
    inp->i_mode |= ILARG;

    // Exam prompt decides whether this function writes ptrs/inode to disk
    // or only mutates inp and lets caller write it.

    return 0;
}
```

Why contents survive:

- Data blocks do not move.
- Only the addressing structure changes.

## Example 28: Ordered Writes For Create File

Naive create touches:

```text
1. allocate inode
2. initialize inode
3. allocate data block maybe
4. initialize data block maybe
5. add directory entry
6. update free maps
```

Safer ordering idea:

```text
initialize data block
initialize inode
mark inode/block allocated
add directory entry last
```

Why directory entry last:

- Once the directory entry exists, the name can be resolved.
- If it points to an uninitialized inode, the filesystem exposes garbage.

## Example 29: Redo Logging Skeleton

```c
begin_transaction();

log_write("new inode block contents");
log_write("new directory block contents");
log_write("new free bitmap contents");

commit_transaction();

install_to_home_locations();
checkpoint_or_clear_log();
```

Recovery:

```c
if (log_contains_commit()) {
    replay_all_logged_updates();
} else {
    ignore_uncommitted_updates();
}
```

Why replay must be idempotent:

- Crash can happen during recovery.
- Recovery may start again and replay the same committed updates again.
- Replaying twice must have same final effect as replaying once.

## Coding Section Checklist

Before submitting a coding answer:

- Did every `execvp` argument vector end with `NULL`?
- Did the child call `exit` after failed `execvp` or after child-only loop work?
- Did every process close unused pipe ends?
- Did parent close pipe ends too?
- Did parent wait for every child it created?
- Did every shared variable in a predicate use the same mutex?
- Did every condition-variable wait use `while`, not `if`?
- Did notification happen after the state change that may make a predicate true?
- Did exact-selection problems store waiter identity?
- Did all-or-nothing resource code check all resources before subtracting any?
- Did address translation preserve the page offset?
- Did V6 directory code use bounded name comparison?
- Did V6 inode code distinguish small vs large inode addressing?
- Did crash-recovery answers identify the crash point after each disk write?
