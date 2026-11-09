# 02.asm

## Stack at program start

Under GDB, RSP at the first instruction was `0x7fffffffea20`, holding `argc = 1`. RBP was `0`. This data belongs to the OS and is never modified. Everything the program pushes goes below it.

Let `S` be the starting RSP (`0x7fffffffea20`). Addresses differ between runs because of ASLR, but the offsets from `S` stay the same.

## RSP vs ESP (and RBP vs EBP)

RSP is the 64-bit stack pointer. ESP is its lower 32 bits, used as the stack pointer in 32-bit code. The same goes for RBP (64-bit) and EBP (32-bit). This program is 64-bit, so it uses RSP and RBP, and every push, pop, call and ret moves RSP by 8 bytes. In 32-bit code the same frame pattern would use `push ebp` / `mov ebp, esp`, and the steps would be 4 bytes.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | RBP after | Effect |
|---|-------------|-----------|-----------|-----------|--------|
| 1 | `call work` | S | S - 8 (`...ea18`) | 0 | Pushes return address `0x401005`. Jumps to `work`. |
| 2 | `push rbp` | S - 8 | S - 16 (`...ea10`) | 0 | Saves the caller's RBP (value 0) on the stack. |
| 3 | `mov rbp, rsp` | S - 16 | S - 16 | `...ea10` | RBP now marks the base of this frame. |
| 4 | `sub rsp, 16` | S - 16 | S - 32 (`...ea00`) | `...ea10` | Reserves 16 bytes for two locals. No data written yet. |
| 5 | `mov qword [rbp-8], 100` | S - 32 | S - 32 | `...ea10` | Writes 100 at `...ea08`. RSP unchanged. |
| 6 | `mov qword [rbp-16], 200` | S - 32 | S - 32 | `...ea10` | Writes 200 at `...ea00`. RSP unchanged. |
| 7 | `mov rax, [rbp-8]` | S - 32 | S - 32 | `...ea10` | RAX = 100. |
| 8 | `mov rbx, [rbp-16]` | S - 32 | S - 32 | `...ea10` | RBX = 200. |
| 9 | `nop` | S - 32 | S - 32 | `...ea10` | Pause point to inspect the full frame. |
| 10 | `mov rsp, rbp` | S - 32 | S - 16 (`...ea10`) | `...ea10` | Frees the locals by moving RSP up to RBP. |
| 11 | `pop rbp` | S - 16 | S - 8 (`...ea18`) | 0 | Restores the caller's RBP. |
| 12 | `ret` | S - 8 | S (`...ea20`) | 0 | Pops `0x401005` into RIP. Execution resumes in `_start`. |
| 13 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | n/a | n/a | Exits with status 0. |

## Stack contents at the `nop`

| Address | Value | Meaning |
|---------|-------|---------|
| `0x7fffffffea00` | `200` | local at `[rbp-16]` (**RSP points here**) |
| `0x7fffffffea08` | `100` | local at `[rbp-8]` |
| `0x7fffffffea10` | `0` | saved caller RBP (**RBP points here**) |
| `0x7fffffffea18` | `0x401005` | return address pushed by `call` |
| `0x7fffffffea20` | `1` | original `argc` (OS data) |

## How the stack is used in this program

The stack holds three kinds of data:

1. **Return address.** `call` pushes it automatically. `ret` pops it into RIP so the program knows where to continue.
2. **Saved base pointer.** `push rbp` / `pop rbp` preserve the caller's RBP so the function can use RBP for itself and give it back unchanged.
3. **Local variables.** `sub rsp, 16` reserves two 8-byte slots. They are accessed relative to RBP, so they stay reachable even if RSP moves.

Locals are created by moving RSP once (`sub`), not by pushing. This is why writing to them does not change RSP.

## Observations from running it

- The program ran to completion with exit code `0`.
- `call` dropped RSP by 8 with no `push` in the source, because it pushes the return address itself.
- After `mov rbp, rsp`, RBP equalled RSP. After `sub rsp, 16`, RSP was 16 lower than RBP, which is exactly the space for the two locals.
- RSP did not change when the locals were written or read. Only RBP-relative addressing was used.
- `mov rsp, rbp` freed both locals in one instruction. After the epilogue, the values `100` and `200` were still in memory below RSP, because freeing space only moves RSP and does not erase data.
- After `ret`, RSP was back at `S` and RIP was `0x401005`, confirming the stack was balanced.

## Final state at exit

| Item | Value |
|------|-------|
| RSP | back at the starting value (`S`) |
| RBP | 0 (restored to the caller's value) |
| RAX | 60 (syscall number; held 100 inside `work`) |
| RBX | 200 |
| RDI | 0 (exit status) |
| Stack | balanced, no leaked pushes or frames |