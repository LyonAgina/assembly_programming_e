# 01.asm


## Stack at program start

Linux sets up the stack before `_start` runs. Under GDB, RSP at the first instruction was `0x7fffffffea20`. The memory there holds:

| Address        | Value              | Meaning                    |
|----------------|--------------------|----------------------------|
| `0x7fffffffea20` | `0x1`            | `argc` (one argument)      |
| `0x7fffffffea28` | `0x7fffffffec82` | pointer to `argv[0]`       |
| `0x7fffffffea30` | `0x0`            | `NULL` terminator of argv  |
| `0x7fffffffea38` | `0x7fffffffec9e` | first environment pointer  |

This data belongs to the OS and the program never modifies it. All of the program's pushes go to addresses **below** `...ea20`.

The base address differs between machines and runs because of ASLR and environment size. What matters is the offsets from the starting RSP.

## Step-by-step trace

Let `S` be the starting RSP (`0x7fffffffea20` in my run).

| # | Instruction      | RSP before | RSP after | Effect |
|---|------------------|-----------|-----------|--------|
| 1 | `mov rax, 100`   | S         | S         | RAX = 100. No stack change. |
| 2 | `mov rbx, 200`   | S         | S         | RBX = 200. No stack change. |
| 3 | `push rax`       | S         | S - 8 (`...ea18`) | Writes `100` (`0x64`) at `[S-8]`. |
| 4 | `push rbx`       | S - 8     | S - 16 (`...ea10`) | Writes `200` (`0xc8`) at `[S-16]`. |
| 5 | `nop`            | S - 16    | S - 16    | Pause point to inspect the stack with 2 values on it. |
| 6 | `pop rcx`        | S - 16    | S - 8 (`...ea18`) | Reads `[S-16]` = 200 into RCX. |
| 7 | `pop rdx`        | S - 8     | S (`...ea20`) | Reads `[S-8]` = 100 into RDX. |
| 8 | `nop`            | S         | S         | Pause point. RSP is back at its start value. |
| 9 | `mov rax, 60`    | S         | S         | Syscall number for `exit`. |
| 10 | `xor rdi, rdi`  | S         | S         | RDI = 0, the exit status. |
| 11 | `syscall`       | S         | n/a       | Process ends. |

### Stack contents at the first `nop` (after both pushes)

| Address          | Value  | Note                |
|------------------|--------|---------------------|
| `0x7fffffffea10` | `0xc8` (200) | **RSP points here** (top) |
| `0x7fffffffea18` | `0x64` (100) | pushed first        |
| `0x7fffffffea20` | `0x1`        | original `argc`     |

`200` sits on top because it was pushed last.

### Why RCX and RDX end up with these values

The pops come out in reverse order of the pushes:

- `pop rcx` takes the top value, which is `200`.
- `pop rdx` takes the next value, which is `100`.

So the final registers are RCX = 200 and RDX = 100. RAX and RBX held 100 and 200 originally, so the pair has effectively been **swapped in order** when copied into RCX and RDX.

## RSP vs ESP

This program is 64-bit, so the stack pointer is **RSP** (64 bits). **ESP** is just the lower 32 bits of RSP. In 64-bit code every `push` and `pop` moves RSP by 8 bytes, and ESP changes along with it because it is part of the same register.

In a 32-bit program, the stack pointer is ESP and each push/pop moves it by 4 bytes instead of 8. The behaviour is otherwise the same: the stack grows downward, and ESP always points at the top value.

| Mode   | Stack pointer | Size of one push/pop |
|--------|---------------|----------------------|
| 64-bit | RSP           | 8 bytes              |
| 32-bit | ESP           | 4 bytes              |

This program uses only RSP. It never reads or writes ESP directly.

## How the stack is used in this program

The stack here is used purely as **temporary storage**. The two values are parked in memory and then restored. There are:

- no function calls, so no return addresses are pushed,
- no local variables or stack frames,
- no saved or restored registers beyond this push/pop demo.

## Observations from running it

- The program ran to completion and returned exit code `0`.
- Single-stepping in GDB confirmed that RSP drops by exactly 8 on each `push` and rises by exactly 8 on each `pop`.
- After both pops, RSP returned to exactly its starting value. This shows the pushes and pops are balanced, which is essential for a correct stack.
- After the pops, the values `0xc8` and `0x64` were still present in memory at `...ea10` and `...ea18`. `pop` only moves RSP and does not erase data. That memory is now free and would be overwritten by the next `push`.
- The disassembly shows `mov rax, 100` was encoded as `mov eax, 0x64`. NASM picks the shorter 32-bit form, and writing to EAX zero-extends into RAX, so the result is the same.

## Final state at exit

| Item | Value |
|------|-------|
| RSP  | back at the starting value (`S`) |
| RAX  | 60 (syscall number) |
| RBX  | 200 |
| RCX  | 200 |
| RDX  | 100 |
| RDI  | 0 (exit status) |
| Stack | balanced, with no leaked pushes |