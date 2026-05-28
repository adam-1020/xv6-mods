# shm — Shared Memory in xv6

## Description

Implements a single shared memory page in xv6, accessible by multiple processes at the same fixed virtual address. Any process can call `shmget()` to map the shared page into its address space. When a process forks, the child automatically inherits access to the shared page — without copying it — allowing parent and child to communicate through the same physical memory.

## Concept

Shared memory is the fastest form of inter-process communication: instead of exchanging data through pipes or system calls, multiple processes read and write directly to the same physical page. In this implementation there is exactly **one** shared page in the system, identified by a fixed virtual address (`0x40000000`). Any process that calls `shmget()` gets that page mapped at that address.

## Architecture

### Global state (kernel, `vm.c`)

```c
static char *shm_kaddr = 0;       // kernel virtual address of the shared page
static int   shm_refcount = 0;    // number of processes currently attached
static struct spinlock shm_lock;  // protects both fields above
```

- `shm_kaddr` — the shared page is allocated with `kalloc()` the first time any process calls `shmget()`. The pointer is kept here so subsequent callers map the same physical frame.
- `shm_refcount` — tracks how many processes have the page mapped. When the last process exits, the page is freed with `kfree()`.

### Fixed virtual address

```c
// mmu.h
#define SHMVA 0x40000000
```

Every process that attaches sees the shared data at `(void*)0x40000000`. Using a fixed address avoids the need to communicate the mapping location between processes.

## System call: `int shmget(void)`

Maps the shared page into the calling process's address space at `SHMVA`.

```c
int shmget(void) {
    struct proc *p = myproc();
    acquire(&shm_lock);

    if(shm_kaddr == 0) {           // first caller: allocate the page
        shm_kaddr = kalloc();
        memset(shm_kaddr, 0, PGSIZE);
    }
    pte_t *pte = walkpgdir(p->pgdir, (char*)SHMVA, 0);
    if(pte && (*pte & PTE_P)) {    // already mapped in this process
        release(&shm_lock);
        return 0;
    }
    mappages(p->pgdir, (void*)SHMVA, PGSIZE, V2P(shm_kaddr), PTE_W|PTE_U);
    shm_refcount++;
    release(&shm_lock);
    return 0;
}
```

Returns `0` on success, `-1` on allocation failure.

## Fork behaviour (`copyuvm`)

When a process forks, xv6 normally copies every page. The shared page is treated differently: instead of copying, `copyuvm` maps the **same physical frame** into the child's page table and increments `shm_refcount`.

```c
if(i == SHMVA) {
    acquire(&shm_lock);
    shm_refcount++;
    release(&shm_lock);
    mappages(d, (void*)SHMVA, PGSIZE, pa, flags);
    continue;          // skip the kalloc/memmove that normal pages use
}
```

## Exit behaviour (`freevm`)

When a process exits, its page table is freed. Before the regular teardown, `freevm` checks for the shared mapping and decrements the reference counter:

```c
pte = walkpgdir(pgdir, (char*)SHMVA, 0);
if(pte && (*pte & PTE_P)) {
    *pte = 0;                       // unmap from this process
    acquire(&shm_lock);
    shm_refcount--;
    if(shm_refcount == 0) {         // last user: free the physical page
        kfree(shm_kaddr);
        shm_kaddr = 0;
    }
    release(&shm_lock);
}
```

## System call path

```
userspace: shmget()
    → usys.S: SYSCALL(shmget)
    → syscall.c: syscalls[SYS_shmget] = sys_shmget
    → sysproc.c: sys_shmget() → shmget()
    → vm.c: shmget() → maps shared page, returns 0
```

## Changed files

| File         | Change                                                                    |
|--------------|---------------------------------------------------------------------------|
| `vm.c`       | `shm_kaddr`, `shm_refcount`, `shm_lock`; `shminit()`, `shmget()`; fork/exit hooks |
| `defs.h`     | Declaration `void shminit(void)`                                          |
| `main.c`     | Call `shminit()` during kernel boot                                       |
| `mmu.h`      | `#define SHMVA 0x40000000`                                                |
| `sysproc.c`  | `sys_shmget()` — syscall wrapper                                          |
| `syscall.c`  | Registration: `[SYS_shmget] sys_shmget`                                  |
| `syscall.h`  | `#define SYS_shmget 22`                                                   |
| `usys.S`     | `SYSCALL(shmget)`                                                         |
| `user.h`     | Declaration `int shmget(void)` for user programs                          |
| `test.c`     | New test program                                                          |
| `Makefile`   | Added `_test` to `UPROGS`                                                 |

## Test program (`test.c`)

```c
shmget();                   // parent maps the shared page
int *x = (int*)SHM_VA;

if (fork() == 0) {
    shmget();               // child maps the same page (no copy)
    sleep(100);             // give parent time to write
    printf(1, "child: %d\n", *x);
} else {
    *x = 1234;              // parent writes through shared memory
    wait();
}
exit();
```

Expected output:

```
child: 1234
```

The child reads `1234` — written by the parent — directly from the shared physical page, with no explicit IPC.

## How to apply

```bash
cd xv6
git apply shm.patch
make
```

## How to test

After booting QEMU:

```
$ test
child: 1234
```

## Screenshot

<!-- TODO screenshot -->