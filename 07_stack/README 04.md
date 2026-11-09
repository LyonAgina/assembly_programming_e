# 04.asm


## ESP vs RSP: why this file uses ESP

This program is 32-bit, so the stack pointer is **ESP** and the frame pointer is **EBP**. Each push, pop, call and ret moves ESP by **4 bytes** (in 64-bit code RSP moves by 8). ESP is also the lower 32 bits of RSP on a 64-bit CPU, but here the whole program runs in 32-bit mode.

## Stack at program start

Under GDB, ESP at the first instruction was `0xffffdb30`, holding `argc = 1`. EBP was `0`. This data belongs to the OS and is never modified. Everything the program pushes goes below it.

Let `S` be the starting ESP (`0xffffdb30`). Addresses differ between runs because of ASLR, but the offsets from `S` stay the same.

## Step-by-step trace

| # | Instruction | ESP before | ESP after | EBP after | Effect |
|---|-------------|-----------|-----------|-----------|--------|
| 1 | `call work` | S | S - 4 (`...db2c`) | 0 | Pushes return address `0x08049005`. Jumps to `work`. |
| 2 | `push ebp` | S - 4 | S - 8 (`...db28`) | 0 | Saves the caller's EBP (value 0) on the stack. |
| 3 | `mov ebp, esp` | S - 8 | S - 8 | `...db28` | EBP now marks the base of this frame. |
| 4 | `sub esp, 8` | S - 8 | S - 16 (`...db20`) | `...db28` | Reserves 8 bytes for two locals. No data written yet. |
| 5 | `mov dword [ebp-4], 100` | S - 16 | S - 16 | `...db28` | Writes 100 at `...db24`. ESP unchanged. |
| 6 | `mov dword [ebp-8], 200` | S - 16 | S - 16 | `...db28` | Writes 200 at `...db20`. ESP unchanged. |
| 7 | `mov eax, [ebp-4]` | S - 16 | S - 16 | `...db28` | EAX = 100. |
| 8 | `mov ebx, [ebp-8]` | S - 16 | S - 16 | `...db28` | EBX = 200. |
| 9 | `nop` | S - 16 | S - 16 | `...db28` | Pause point to inspect the full frame. |
| 10 | `mov esp, ebp` | S - 16 | S - 8 (`...db28`) | `...db28` | Frees the locals by moving ESP up to EBP. |
| 11 | `pop ebp` | S - 8 | S - 4 (`...db2c`) | 0 | Restores the caller's EBP. |
| 12 | `ret` | S - 4 | S (`...db30`) | 0 | Pops `0x08049005` into EIP. Execution resumes in `_start`. |
| 13 | `mov eax, 1` / `xor ebx, ebx` / `int 0x80` | S | n/a | n/a | Exits with status 0. |

## Stack contents at the `nop`

| Address | Value | Meaning |
|---------|-------|---------|
| `0xffffdb20` | `200` | local at `[ebp-8]` (**ESP points here**) |
| `0xffffdb24` | `100` | local at `[ebp-4]` |
| `0xffffdb28` | `0` | saved caller EBP (**EBP points here**) |
| `0xffffdb2c` | `0x08049005` | return address pushed by `call` |
| `0xffffdb30` | `1` | original `argc` (OS data) |

## How the stack is used in this program

The stack holds three kinds of data:

1. **Return address.** `call` pushes it automatically (4 bytes). `ret` pops it into EIP.
2. **Saved base pointer.** `push ebp` / `pop ebp` preserve the caller's EBP so the function can use EBP for itself and give it back unchanged.
3. **Local variables.** `sub esp, 8` reserves two 4-byte slots. They are accessed relative to EBP, so they stay reachable even if ESP moves.

Locals are created by moving ESP once, not by pushing, so writing to them does not change ESP.

## Observations from running it

- The program ran to completion with exit code `0`.
- Every stack operation moved ESP by exactly 4 bytes, half the step size of the 64-bit version.
- `call` dropped ESP by 4 with no `push` in the source, because it pushes the return address itself.
- After `mov ebp, esp`, EBP equalled ESP. After `sub esp, 8`, ESP was 8 lower than EBP, exactly the space for the two locals.
- ESP did not change when the locals were written or read.
- `mov esp, ebp` freed both locals in one instruction. After the epilogue, `100` was still in memory below ESP, because freeing space only moves ESP and does not erase data.
- EBX held 200 inside `work`, but `xor ebx, ebx` cleared it before `int 0x80`. The exit status comes from EBX, so it is 0.
- After `ret`, ESP was back at `S` and EIP was `0x08049005`, confirming the stack was balanced.

## Final state at exit

| Item | Value |
|------|-------|
| ESP | back at the starting value (`S`) |
| EBP | 0 (restored to the caller's value) |
| EAX | 1 (syscall number for exit) |
| EBX | 0 (exit status) |
| Stack | balanced, no leaked pushes or frames |