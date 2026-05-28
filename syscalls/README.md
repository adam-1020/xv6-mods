# syscalls — New system calls: `usedvp` and `usedpp`

## Description

Adds two new system calls to xv6 that let a user process query the kernel for the number of memory pages it is currently using — both virtual and physical.

## New system calls

### `int usedvp(void)` — Used Virtual Pages

Returns the number of virtual pages occupied by the current process.

**Implementation:**
```c
int usedvp(void) {
    struct proc *p = myproc();
    if(p == 0) return 0;
    uint pages_for_sz = PGROUNDUP(p->sz) / PGSIZE;
    return (int)(pages_for_sz);
}
```

Computes the number of pages needed to cover `p->sz`, rounded up to the nearest page boundary. In xv6, `p->sz` includes the stack, so this reflects the full virtual address space of the process.

### `int usedpp(void)` — Used Physical Pages

Returns the number of physical pages actually mapped in the process page table.

**Implementation:**
```c
int usedpp(void) {
    struct proc *p = myproc();
    if(p == 0 || p->pgdir == 0) return 0;

    pde_t *pgdir = p->pgdir;
    int count = 0;
    for(uint va = 0; va < KERNBASE; va += PGSIZE) {
        pte_t *pte = walkpgdir(pgdir, (char*)va, 0);
        if(pte && (*pte & PTE_P))
            count++;
    }
    return count;
}
```

Walks the entire user address space (from `0` to `KERNBASE = 0x80000000`) and counts every PTE with the `PTE_P` (Present) bit set. This includes the stack page, so `usedpp` will typically return a value one higher than `usedvp`.

**Difference between `usedvp` and `usedpp`:** `usedvp` is a fast estimate derived from `p->sz` (which in xv6 includes the stack). `usedpp` is an exact count via a full page walk. In practice both return the same value — they serve as a cross-check of each other. After `sbrk(4096)` both increase by 1.

## System call path

```
userspace: usedvp()
    → usys.S: SYSCALL(usedvp)  → int $T_SYSCALL with SYS_usedvp
    → syscall.c: syscalls[SYS_usedvp] = sys_usedvp
    → sysproc.c: sys_usedvp() → calls usedvp()
    → vm.c: usedvp() → returns value
```

## Changed files

| File         | Change                                                        |
|--------------|---------------------------------------------------------------|
| `vm.c`       | Implementation of `usedvp()` and `usedpp()`                  |
| `defs.h`     | Declarations `int usedvp(void)` and `int usedpp(void)`       |
| `sysproc.c`  | `sys_usedvp()` and `sys_usedpp()` — syscall wrappers         |
| `syscall.c`  | Registration in the syscall table + `extern` declarations    |
| `syscall.h`  | `#define SYS_usedvp 22` and `#define SYS_usedpp 23`         |
| `usys.S`     | `SYSCALL(usedvp)` and `SYSCALL(usedpp)` macros               |
| `user.h`     | Declarations for user programs                               |
| `test.c`     | Test program                                                  |
| `Makefile`   | Added `_test` to `UPROGS`                                    |

## Test program (`test.c`)

```c
int used_virtual = usedvp();
int used_physical = usedpp();
printf(1, "Number of used virtual pages: %d\n", used_virtual);
printf(1, "Number of used physical pages: %d\n", used_physical);

sbrk(4096);  // allocate one more page

used_virtual = usedvp();
used_physical = usedpp();
printf(1, "After allocating one page:\n");
printf(1, "Number of used virtual pages: %d\n", used_virtual);
printf(1, "Number of used physical pages: %d\n", used_physical);
```

## How to apply

```bash
cd xv6
git apply syscalls.patch
make
```

## How to test

```
$ test
Number of used virtual pages: 3
Number of used physical pages: 3
After allocating one page:
Number of used virtual pages: 4
Number of used physical pages: 4
```

(Exact baseline values depend on the size of the `test` binary.)

## Screenshot

<!-- TODO ss -->