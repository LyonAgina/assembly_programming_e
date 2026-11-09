# 06.asm


## ESP vs RSP

This program is 32-bit, so the stack pointer is ESP.Each push and pop moves ESP by 4 bytes, because a 32-bit register is 4 bytes wide. In 64-bit code the stack pointer is RSP and each push or pop moves it by 8. ESP is also the name for the lower 32 bits of RSP on a 64-bit CPU, but here the whole program runs in 32-bit mode.

## Stack at program start

Under GDB, ESP at the first instruction was `0xffffdb30`, holding `argc = 1`. This data belongs to the OS and is never modified. Everything the program pushes goes below it.

Let `S` be the starting ESP (`0xffffdb30`). Addresses differ between runs because of ASLR, but the offsets from `S` stay the same.

## Step-by-step trace

| # | Instruction | ESP before | ESP after | Effect |
|---|-------------|-----------|-----------|--------|
| 1 | `mov eax, 0x11223344` | S | S | EAX = `0x11223344`. No stack change. |
| 2 | `mov ebx, 0x1234` | S | S | EBX = `0x1234`. No stack change. |
| 3 | `push eax` | S | S - 4 (`...db2c`) | Writes EAX's 4 bytes at `[S-4]`. |
| 4 | `push ebx` | S - 4 | S - 8 (`...db28`) | Writes EBX's 4 bytes at `[S-8]`. |
| 5 | `nop` | S - 8 | S - 8 | Pause point with both values on the stack. |
| 6 | `pop ebx` | S - 8 | S - 4 (`...db2c`) | Reads `[S-8]` = `0x1234` into EBX. |
| 7 | `pop eax` | S - 4 | S (`...db30`) | Reads `[S-4]` = `0x11223344` into EAX. |
| 8 | `mov eax, 1` / `xor ebx, ebx` / `int 0x80` | S | n/a | Exits with status 0. |

### Stack contents at the `nop`

| Address | Value | Note |
|---------|-------|------|
| `0xffffdb28` | `0x00001234` | EBX's value (**ESP points here**, pushed last) |
| `0xffffdb2c` | `0x11223344` | EAX's value (pushed first) |
| `0xffffdb30` | `0x1` | original `argc` |

### Byte layout in memory (little-endian)

The value `0x11223344` is stored with its lowest byte at the lowest address:

```
0xffffdb2c:  44 33 22 11
```

EBX's value `0x1234` also takes a full 4 bytes:

```
0xffffdb28:  34 12 00 00
```

x86 stores multi-byte values this way (little-endian). The CPU reassembles the bytes correctly when the value is popped.

## How the stack is used in this program

The stack is used purely as temporary storage to save and restore registers.The pops are in the reverse order of the pushes (EBX first, then EAX), so each value returns to the register it came from. If the pops were in the same order as the pushes, the values would end up in the wrong registers.

There are no function calls, return addresses, stack frames or local variables.

## Observations from running it

- The program ran to completion with exit code `0`.
- Every push lowered ESP by exactly 4 and every pop raised it by exactly 4, including for the small value `0x1234`, which still occupied a full 4-byte slot.
- The byte dump showed the little-endian layout: `44 33 22 11` for EAX's value.
- The register values looked unchanged after the pops, because nothing modified EAX or EBX between the push and the pop. The pops are visible in ESP returning to `S`, not in the register values.
- After the pops, both values were still in memory below ESP. `pop` moves ESP and does not erase data.
- The pushes and pops were balanced, so ESP ended at its starting value.

## Final state at exit

| Item | Value |
|------|-------|
| ESP | back at the starting value (`S`) |
| EAX | 1 (syscall number for exit; held `0x11223344` after the pops) |
| EBX | 0 (exit status; held `0x1234` after the pops) |
| Stack | balanced, no leaked pushes |