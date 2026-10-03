# Subtraction:

## Program 1: sub1 (50 minus 80)

```nasm
mov al, [num1]      ; al = 50 (0x32)
sub al, [num2]      ; al = 50 - 80 = -30 (0xE2)
mov [result], al
```

Result: `al = 0xE2` (226 as unsigned, -30 as signed)

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Set | 50 is smaller than 80, so a borrow was needed. As unsigned, the result wrapped around to 226. |
| Overflow flag | Cleared | As signed, 50 - 80 = -30, which fits in the byte range of -128 to 127. |
| Sign flag | Set | Bit 7 of `0xE2` (`11100010`) is 1, so the result is negative. |
| Zero flag | Cleared | The result is not zero. |
| Parity flag | Set | The low byte `11100010` has four 1-bits, which is an even count. |
| Auxiliary carry flag | Cleared | The lowest four bits are `0x2 - 0x0`, which needs no borrow. |

---

## Program 2: sub3 (1000 minus 2000)

```nasm
mov ax, [num1]      ; ax = 1000 (0x03E8)
sub ax, [num2]      ; ax = 1000 - 2000 = -1000 (0xFC18)
mov [result], ax
```

Result: `ax = 0xFC18` (64536 as unsigned, -1000 as signed)

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Set | 1000 is smaller than 2000, so a borrow was needed. As unsigned, the result wrapped around to 64536. |
| Overflow flag | Cleared | As signed, 1000 - 2000 = -1000, which fits in the word range of -32768 to 32767. |
| Sign flag | Set | Bit 15 of `0xFC18` is 1, so the result is negative. |
| Zero flag | Cleared | The result is not zero. |
| Parity flag | Set | The low byte `0x18` (`00011000`) has two 1-bits, which is an even count. |
| Auxiliary carry flag | Cleared | The lowest four bits are `0x8 - 0x0`, which needs no borrow. |

---
