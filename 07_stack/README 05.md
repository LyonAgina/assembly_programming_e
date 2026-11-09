# 05.asm

## Stack at program start

Under GDB, RSP at the first instruction was `0x7fffffffea20`, holding `argc = 1`. This data belongs to the OS and is never modified. Everything the program pushes goes below it.

Let `S` be the starting RSP (`0x7fffffffea20`). Addresses differ between runs because of ASLR, but the offsets from `S` stay the same.

## RSP vs ESP

RSP is the 64-bit stack pointer. ESP is its lower 32 bits and is the stack pointer in 32-bit code. This program is 64-bit, so each push and pop moves RSP by 8 bytes, even when the value is small. In 32-bit code, ESP would move by 4 bytes per push or pop.

## Step-by-step trace

| # | Instruction | RSP before | RSP after | Effect |
|---|-------------|-----------|-----------|--------|
| 1 | `mov rax, 0x1122334455667788` | S | S | RAX = `0x1122334455667788`. No stack change. |
| 2 | `mov rbx, 0x1234` | S | S | RBX = `0x1234`. No stack change. |
| 3 | `push rax` | S | S - 8 (`...ea18`) | Writes RAX's 8 bytes at `[S-8]`. |
| 4 | `push rbx` | S - 8 | S - 16 (`...ea10`) | Writes RBX's 8 bytes at `[S-16]`. |
| 5 | `nop` | S - 16 | S - 16 | Pause point with both values on the stack. |
| 6 | `pop rbx` | S - 16 | S - 8 (`...ea18`) | Reads `[S-16]` = `0x1234` into RBX. |
| 7 | `pop rax` | S - 8 | S (`...ea20`) | Reads `[S-8]` = `0x1122334455667788` into RAX. |
| 8 | `mov rax, 60` / `xor rdi, rdi` / `syscall` | S | n/a | Exits with status 0. |

### Stack contents at the `nop`

| Address | Value | Note |
|---------|-------|------|
| `0x7fffffffea10` | `0x0000000000001234` | RBX's value (**RSP points here**, pushed last) |
| `0x7fffffffea18` | `0x1122334455667788` | RAX's value (pushed first) |
| `0x7fffffffea20` | `0x1` | original `argc` |

### Byte layout in memory (little-endian)

The 64-bit value `0x1122334455667788` is stored with its lowest byte at the lowest address:

```
0x7fffffffea18:  88 77 66 55 44 33 22 11
```

RBX's value `0x1234` also takes a full 8 bytes:

```
0x7fffffffea10:  34 12 00 00 00 00 00 00
```

x86 stores multi-byte values this way (little-endian). The CPU reassembles the bytes correctly when the value is popped.

## How the stack is used in this program

The stack is used purely as temporary storage to save and restore registers.The pops are in the reverse order of the pushes (RBX first, then RAX), so each value returns to the register it came from. If the pops were in the same order as the pushes, the values would end up in the wrong registers.

There are no function calls, return addresses, stack frames or local variables.

## Observations from running it

- The program ran to completion with exit code `0`.
- Every push lowered RSP by exactly 8 and every pop raised it by exactly 8, including for the small value `0x1234`, which still occupied a full 8-byte slot.
- The byte dump showed the little-endian layout: `88 77 66 55 44 33 22 11` for RAX's value.
- The register values looked unchanged after the pops, because nothing modified RAX or RBX between the push and the pop. The pops are visible in RSP returning to `S`, not in the register values.
- After the pops, both values were still in memory below RSP. `pop` moves RSP and does not erase data.
- The pushes and pops were balanced, so RSP ended at its starting value.

## Final state at exit

| Item | Value |
|------|-------|
| RSP | back at the starting value (`S`) |
| RAX | 60 (syscall number; held `0x1122334455667788` after the pops) |
| RBX | `0x1234` |
| RDI | 0 (exit status) |
| Stack | balanced, no leaked pushes |