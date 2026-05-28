# colors — Colored text output in the xv6 console

## Description

Adds colored text output to the CGA (VGA text mode) console in xv6. User programs can change the foreground color of printed text by embedding a simple escape sequence in the byte stream passed to `write()`.

## How it works

The CGA framebuffer stores each character as two bytes: the ASCII byte and an attribute byte (color). By default xv6 hardcoded attribute `0x07` (light grey on black). This patch adds a global `cons_attr` variable and escape-sequence handling inside `consputc()`.

### Escape sequence

```
%X
```

Where `X` is a single hexadecimal digit (`0`–`9`, `a`–`f`, `A`–`F`) selecting one of the 16 CGA foreground colors:

| Code | Color         | Code | Color           |
|------|---------------|------|-----------------|
| `0`  | Black         | `8`  | Dark grey       |
| `1`  | Blue          | `9`  | Light blue      |
| `2`  | Green         | `a`  | Light green     |
| `3`  | Cyan          | `b`  | Light cyan      |
| `4`  | Red           | `c`  | Light red       |
| `5`  | Magenta       | `d`  | Light magenta   |
| `6`  | Brown         | `e`  | Yellow          |
| `7`  | Light grey (default) | `f` | White    |

The `%X` sequence is **consumed** — it does not appear on screen. It changes the global color attribute for all subsequent characters until the next `%X`.

### `%` as a literal character

If the character after `%` is not a hex digit, both `%` and the following character are printed normally. This means existing programs using `printf` with `%d`, `%s`, etc. continue to work unchanged (since `cprintf` never passes its format string through `write()`).

## Changed files

### `console.c`

- `cons_attr` — static variable holding the current CGA attribute byte
- `set_cons_color(int idx)` — sets `cons_attr` from color index 0–15
- `hexchar_to_index(int c)` — converts a hex character to 0–15, returns -1 if not hex
- `cgaputc()` — now uses `cons_attr` instead of the hardcoded `0x0700`
- `consputc()` — added a one-state machine (`pending_pct`): waits for the char after `%`, either changes color or prints both characters

### `hello.c` (new file)

Demo program — prints `"Hello, World"` 16 times, once in each CGA color (colors 0–15).

### `Makefile`

- Added `_hello` to `UPROGS`
- Includes the base build fixes

## How to apply

```bash
cd xv6-public
git apply colors.patch
make clean
make qemu
```

## How to test

After booting QEMU:

```
$ hello
```

You should see `Hello, World` printed 16 times, each line in a different color.

You can also test from your own C code:

```c
write(1, "%4Red text\n", 10);    // %4 = red
write(1, "%2Green text\n", 12);  // %2 = green
write(1, "%7Back to grey\n", 14);
```

## Screenshot

<img width="1544" height="1030" alt="Screenshot From 2026-05-28 11-41-12" src="https://github.com/user-attachments/assets/fdc5f0a9-4ed0-49e3-a1e0-400932bdd32c" />
