# 03.asm

## Stack at program start

Under GDB, RSP at the first instruction was `0x7fffffffea20`, holding `argc = 1`. This data belongs to the OS and is never modified. Everything the program pushes goes below it.

Let `S` be the starting RSP (`0x7fffffffea20`). Addresses differ between runs because of ASLR, but the offsets from `S` stay the same.

## RSP vs ESP

RSP is the 64-bit stack pointer. ESP is its lower 32 bits and is the stack pointer in 32-bit code. This program is 64-bit, so `call` and `ret` move RSP by 8 bytes. In 32-bit code the same instructions would move ESP by 4 bytes and push a 4-byte return address.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | Effect |
|---|-------------|-----------|-----------|--------|
| 1 | `mov rax, 10` | S | S | RAX = 10. No stack change. |
| 2 | `call calculate` | S | S - 8 (`...ea18`) | Pushes return address `0x40100a`. RIP = `0x401015` (start of `calculate`). |
| 3 | `add rax, 5` | S - 8 | S - 8 | RAX = 15. No stack change. |
| 4 | `ret` | S - 8 | S (`...ea20`) | Pops `0x40100a` into RIP. Execution resumes at `after_call`. |
| 5 | `nop` | S | S | RAX is 15 here. |
| 6 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | n/a | Exits with status 0. |

### Stack contents inside `calculate`

| Address | Value | Meaning |
|---------|-------|---------|
| `0x7fffffffea18` | `0x40100a` | return address pushed by `call` (**RSP points here**) |
| `0x7fffffffea20` | `1` | original `argc` (OS data) |

## How the stack is used in this program

The stack remembers where to return to. `call` pushes the address of the next instruction and jumps to the function. `ret` pops that address into RIP, so execution continues where it left off.

There are:

- no `push` or `pop` instructions,
- no stack frame (no RBP setup),
- no local variables,
- no saved registers.

The value is passed in and out through RAX instead of the stack.

## Observations from running it

- The program ran to completion with exit code `0`.
- `call` dropped RSP by exactly 8 and wrote `0x40100a`, the address of `after_call`, at the new top of the stack.
- `add rax, 5` changed RAX from 10 to 15 and did not change RSP.
- `ret` raised RSP by exactly 8 and loaded `0x40100a` into RIP, so execution resumed at `after_call`.
- After `ret`, the old return address was still in memory at `...ea18`. `ret` only moves RSP and does not erase it.
- RAX was 15 at `after_call`, confirmed in GDB. The exit code is 0 because the exit status comes from RDI, not RAX.
- RSP returned to its starting value, so the call was balanced.

## Final state at exit

| Item | Value |
|------|-------|
| RSP | back at the starting value (`S`) |
| RAX | 60 (syscall number; held 15 after the call) |
| RDI | 0 (exit status) |
| Stack | balanced, no leaked pushes |