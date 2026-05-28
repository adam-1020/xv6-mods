# xv6-mods — Kernel extensions for xv6

A collection of independent patches that extend the [xv6](https://github.com/mit-pdos/xv6-public) teaching operating system with new kernel features. Each patch is self-contained and can be applied on its own.

All patches target xv6 for i386 (32-bit x86) and have been tested on **GCC 10+ / binutils 2.36+** on a modern Linux host.

---

## Repository layout

```
xv6-mods/
├── patch/              # Build fixes for modern toolchains (standalone)
├── colors/             # Colored console output
├── semaphores/         # Counting semaphores (IPC)
├── shm/                # Shared memory page
├── syscalls/           # usedvp / usedpp — page usage queries
└── vmprint/            # Page table visualizer
```

---

## Patches

### `patch/` — Base Build Fixes

Fixes compilation on modern toolchains. Useful if you only want the build fix without any other feature.

- Removes `-Werror` and adds `-mno-sse` to `CFLAGS`
- Strips `.note.gnu.property` sections from `initcode.o` and `ulib.o` (emitted by newer binutils, breaks the bootloader)

```bash
git apply patch/xv6.patch
```

---

### `colors/` — Colored Console Output

Adds colored text output to the CGA console. User programs embed a `%X` escape sequence (where `X` is a hex digit `0`–`f`) in the byte stream passed to `write()` to switch the foreground color.

```c
write(1, "%cLight red text\n", 16);   // %c = light red
write(1, "%2Green text\n", 12);       // %2 = green
write(1, "%7Back to grey\n", 14);     // %7 = default
```

Includes a `hello` demo program that prints `Hello, World` once in each of the 16 CGA colors.

```bash
git apply colors/colors.patch
```

---

### `semaphores/` — Counting Semaphores

Implements counting semaphores as a kernel-level IPC primitive, exposed through three new system calls:

| Call | Description |
|------|-------------|
| `sem_init(id, val)` | Initialize semaphore `id` with value `val` |
| `sem_down(id)` | Decrement (block if zero) |
| `sem_up(id)` | Increment and wake one waiter |

The kernel holds a table of 10 semaphores (`NSEM = 10`), shared across all processes and identified by index. `sem_down` uses xv6's `sleep()` with a `while` loop to handle spurious wakeups correctly.

```bash
git apply semaphores/semaphores.patch
```

---

### `shm/` — Shared Memory

Maps a single shared physical page into any process that calls `shmget()`, always at the fixed virtual address `0x40000000`. Forked children inherit the mapping without copying the page; the physical frame is freed only when the last attached process exits.

```c
shmget();                   // map shared page
int *shared = (int*)0x40000000;
*shared = 42;               // visible to all attached processes
```

```bash
git apply shm/shm.patch
```

---

### `syscalls/` — Page Usage Queries

Adds two system calls for inspecting a process's memory usage:

| Call | Returns |
|------|---------|
| `usedvp()` | Number of virtual pages (derived from `p->sz`, rounded up) |
| `usedpp()` | Number of physical pages (exact count via full page-table walk) |

Both return the same value in practice and serve as a cross-check of each other. After `sbrk(4096)` both increase by 1.

```bash
git apply syscalls/syscalls.patch
```

---

### `vmprint/` — Page Table Visualizer

Implements `vmprint(pde_t *pgdir)` — a kernel function that prints the full two-level page table (page directory → page tables → physical pages) for any process. Called automatically on every `exec()` and `exit()`:

```
exec - pgdir pa 0x3fa000
.. 0: pde pa 0x3f9000 flags 0x7
.. .. 0: pte pa 0x3f8000 va 0x0 flags 0x7
...
```

Includes `testvm`, a user program that allocates 5 pages via `sbrk` so you can watch the PTE count grow between `exec` and `exit`.

```bash
git apply vmprint/vmprint.patch
```

---

## Downloading and applying a patch

```bash
git clone https://github.com/mit-pdos/xv6-public.git
cd xv6-public
git apply ../xv6-mods/<patch-dir>/<patch-name>.patch
make
qemu-system-i386 -nographic -drive file=fs.img,index=1,media=disk,format=raw -drive file=xv6.img,index=0,media=disk,format=raw
```

---

## Syscall number map

| Number | Name | Patch |
|--------|------|-------|
| 22 | `sem_init` / `usedvp` / `shmget` | semaphores / syscalls / shm |
| 23 | `sem_down` / `usedpp` | semaphores / syscalls |
| 24 | `sem_up` | semaphores |

---

## Requirements

- `i386-elf` cross-compiler (`gcc`, `ld`, `objcopy`, `objdump`)
- `qemu-system-i386`
- Linux host with GCC 10+ and binutils 2.36+