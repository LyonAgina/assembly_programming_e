# Division

## Program 1: div1 (100 divided by 7)

```nasm
mov ax, [dividend]  ; ax = 100
mov bl, [divisor]   ; bl = 7
div bl              ; al = 14 (quotient), ah = 2 (remainder)
```

Result: `al = 14`, `ah = 2`

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Undefined | The instruction does not report a carry. Dividing 100 by 7 fits in eight bits, and the instruction would raise an error instead if it did not. |
| Overflow flag | Undefined | The instruction does not report overflow. A quotient that is too large causes a divide error, not a flag change. |
| Sign flag | Undefined | It is not copied from the quotient. The quotient 14 is positive, but the flag is not updated from it. |
| Zero flag | Undefined | The quotient is 14, and the flag is not updated from it either way. |
| Parity flag | Undefined | It is not calculated from the quotient `0x0E` or the remainder `0x02`. |
| Auxiliary carry flag | Undefined | No nibble carry is calculated for a division. |

---

## Program 2: div3 (300000000 divided by 1000)

```nasm
mov eax, [dividend]  ; eax = 300000000
mov edx, [highpart]  ; edx = 0
mov ebx, [divisor]   ; ebx = 1000
div ebx              ; eax = 300000 (quotient), edx = 0 (remainder)
```

Result: `eax = 300000`, `edx = 0`

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Undefined | The instruction does not report a carry. The quotient fits in 32 bits, so no error occurs. |
| Overflow flag | Undefined | A quotient too large for `eax` would cause a divide error, not set this flag. |
| Sign flag | Undefined | It is not copied from the quotient, which is positive. |
| Zero flag | Undefined | The remainder is 0, but the flag is not set from it. To test for a zero remainder, compare `edx` with zero. |
| Parity flag | Undefined | It is not calculated from the quotient or remainder. |
| Auxiliary carry flag | Undefined | No nibble carry is calculated for a division. |

---