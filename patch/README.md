# xv6 — Base Build Fixes

## Description

A base patch that fixes compilation issues with the original xv6 on modern toolchains (GCC 10+, binutils 2.36+). Without these fixes the project will not compile on a contemporary Linux system.

## Changes

### `Makefile`

**Compiler flags (`CFLAGS`)**

Removed `-Werror`, added `-mno-sse`:

```diff
-CFLAGS = ... -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer
+CFLAGS = ... -Wall -MD -mno-sse -ggdb -m32 -fno-omit-frame-pointer
```

- `-Werror` turned warnings into errors. Newer GCC generates more warnings than the original xv6 was written for, so the build would fail immediately.
- `-mno-sse` disables SSE instructions. xv6 does not save/restore `MXCSR` or XMM registers during context switches, so any SSE use would corrupt floating-point state across processes.

**`initcode` — strip `.note.gnu.property` section**

```diff
+$(OBJCOPY) --remove-section .note.gnu.property initcode.o
```

Newer binutils emit this section automatically. In `initcode` (the binary shellcode loaded by the kernel) this extra section pushes the binary past one disk sector, breaking the xv6 bootloader's assumptions.

**`ulib` — strip `.note.gnu.property` section**

```diff
+$(OBJCOPY) --remove-section .note.gnu.property ulib.o
```

Same issue when linking user-space programs against `ulib.o`.

## Dependencies

None — this patch is included as part of all subsequent patches.

## How to apply

```bash
cd xv6
git apply xv6.patch
make
```