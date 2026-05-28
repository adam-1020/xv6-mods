# semaphores — Counting semaphores in xv6

## Description

Implements counting semaphores in xv6 as an inter-process synchronization primitive. The semaphore logic lives entirely in the kernel (`sysproc.c`) without a separate source file. Semaphores are exposed to user programs through three new system calls: `sem_init`, `sem_down`, and `sem_up`.

## Concept

A semaphore is an integer counter protected by a spinlock. Processes use it to coordinate execution:

- **`sem_down`** (P / wait): if the counter > 0 — decrements and continues; if = 0 — puts the process to sleep until someone calls `sem_up`
- **`sem_up`** (V / signal): increments the counter and wakes one waiting process

## Architecture

### Semaphore table

```c
#define NSEM 10
struct semaphore semtab[NSEM];
```

The kernel holds a global table of 10 semaphores (indices 0–9), shared among all processes. A semaphore is identified simply by its index into this table.

### `struct semaphore`

```c
struct semaphore {
    int value;
    struct spinlock lock;
};
```

Each semaphore has a counter and a spinlock that makes operations atomic.

### Initialization

`semaphoreinit()` is called from `main.c` during kernel boot — initializes the spinlocks and zeroes all counters.

## System call implementations

### `sem_init(int id, int value)`

Sets the counter of semaphore `id` to `value`. Must be called before first use.

### `sem_down(int id)`

```c
acquire(&semtab[id].lock);
while(semtab[id].value == 0)
    sleep(&semtab[id], &semtab[id].lock);
semtab[id].value--;
release(&semtab[id].lock);
```

Uses xv6's `sleep()` — atomically releases the lock while sleeping, preventing the race condition between checking the counter and going to sleep. The `while` loop instead of `if` guards against spurious wakeups.

### `sem_up(int id)`

```c
acquire(&semtab[id].lock);
semtab[id].value++;
wakeup(&semtab[id]);
release(&semtab[id].lock);
```

`wakeup()` wakes all processes sleeping on channel `&semtab[id]` — which is why `sem_down` uses a `while` loop.

## System call path

```
userspace: sem_down(id)
    → usys.S: SYSCALL(sem_down)
    → syscall.c: syscalls[SYS_sem_down] = sys_sem_down
    → sysproc.c: sys_sem_down() → argint(0, &id) → sem_down(id)
```

## Changed files

| File          | Change                                                        |
|---------------|---------------------------------------------------------------|
| `sysproc.c`   | `sys_sem_init/down/up` — syscall wrappers + semaphore logic  |
| `defs.h`      | Declarations: `semaphoreinit`, `sem_init`, `sem_down`, `sem_up` |
| `main.c`      | Call `semaphoreinit()` during boot                            |
| `syscall.c`   | Registration of 3 new syscalls                                |
| `syscall.h`   | `SYS_sem_init = 22`, `SYS_sem_down = 23`, `SYS_sem_up = 24`  |
| `usys.S`      | `SYSCALL` macros for the 3 new syscalls                       |
| `user.h`      | Declarations for user programs                                |
| `testsem.c`   | New test program                                              |
| `Makefile`    | Added `semaphore.o` to OBJS, `_testsem` to UPROGS            |

## Test program (`testsem.c`)

Creates 2 child processes and synchronizes their output using semaphores:

```
Expected output:
I am parent and I should print first
I am child 1       (or child 2 — order between children is not guaranteed)
Only one child has printed by now
I am child 2       (or child 1)
Both children have printed by now
```

Synchronization scheme:
1. Both children start blocked on `sem_down(CHILDSEM)` (initialized to 0)
2. Parent sleeps 2 ticks, then prints
3. Parent does `sem_up(CHILDSEM)` — wakes exactly one child
4. That child prints and does `sem_up(PARENTSEM)`
5. Parent waits on `sem_down(PARENTSEM)`, confirms one child printed, then wakes the second
6. Second child prints and signals parent again

## How to apply

```bash
cd xv6
git apply semaphores.patch
make
```

## How to test

```
$ testsem
I am parent and I should print first
I am child 1
Only one child has printed by now
I am child 2
Both children have printed by now
```

## Screenshot

<!-- TODO ss -->