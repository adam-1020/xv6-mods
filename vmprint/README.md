# vmprint — Page table visualizer

## Description

Implements `vmprint()` in xv6 — a kernel function that prints the full two-level page table structure (page directory + page tables) for any process. The function is called automatically on every `exec()` and `exit()`, letting you observe how the address space changes throughout a process's lifetime.

## xv6 memory architecture (i386, 32-bit)

xv6 uses a two-level page table:

```
Virtual Address (32 bit)
┌──────────┬──────────┬────────────┐
│  PD idx  │  PT idx  │   offset   │
│  10 bit  │  10 bit  │   12 bit   │
└──────────┴──────────┴────────────┘
     │            │
     ▼            ▼
Page Directory  Page Table   Physical Page
(1024 entries) (1024 entries)  (4096 bytes)
```

- **Page Directory (PD)**: 1024 × 4 B = 4 KB; each entry points to a Page Table
- **Page Table (PT)**: 1024 × 4 B = 4 KB; each entry points to a physical page
- **Flags**: `PTE_P` (Present), `PTE_W` (Writable), `PTE_U` (User-accessible)

## Implementation

### `vmprint(pde_t *pgdir)` — `vm.c`

Iterates over the first 512 Page Directory entries (lower half of the address space — user space). For each present PDE it prints:

```
pgdir pa 0x<physical address of page directory>
.. <i>: pde pa 0x<PT address> flags 0x<flags>
.. .. <j>: pte pa 0x<page address> va 0x<virtual address> flags 0x<flags>
```

The `..` and `.. ..` notation reflects nesting depth (PDE → PTE).

**Why 512 instead of 1024?** The upper half (entries 512–1023) is the kernel mapping — identical in every process. There is no need to print it when inspecting user-space layout.

### Kernel call sites

**`exec.c`** — after loading the new process image:
```c
cprintf("exec - ");
vmprint(curproc->pgdir);
```

**`proc.c`** — inside `exit()`:
```c
cprintf("exit - ");
vmprint(curproc->pgdir);
```

This prints the full address-space layout at both ends of a process's life.

### `testvm.c` (new user program)

Allocates 5 pages (5 × 4 KB) via `sbrk(4096)` in a loop, then exits. Lets you verify that `vmprint` shows the growing number of PTE entries after each allocation.

## Changed files

| File        | Change                                                     |
|-------------|------------------------------------------------------------|
| `vm.c`      | New `vmprint()` function                                   |
| `defs.h`    | Declaration `void vmprint(pde_t *pgdir)`                   |
| `exec.c`    | Call `vmprint()` after loading the process                 |
| `proc.c`    | Call `vmprint()` inside `exit()`                           |
| `testvm.c`  | New user-space test program                                |
| `Makefile`  | Added `_testvm` to `UPROGS`             |

## How to apply

```bash
cd xv6
git apply vmprint.patch
make
```

## How to test

```
$ testvm
```

In the QEMU kernel log you will see:

```
exec - pgdir pa 0x...
.. 0: pde pa 0x... flags 0x...
.. .. 0: pte pa 0x... va 0x0 flags 0x...
...
exit - pgdir pa 0x...
.. 0: pde pa 0x... flags 0x...
...
```

The PTE count at `exit` should be 5 higher than at `exec` (5 allocations × 1 page each).

## Screenshot

<!-- TODO ss -->