# Multiplication

## Program 1: mul1 (25 times 10)

```nasm
mov al, [num1]      ; al = 25
mul byte [num2]     ; ax = al * 10 = 250
mov [result], ax
```

Result: `ax = 250` (`0x00FA`). The upper half is `ah = 0`.

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Cleared | The upper half (`ah`) is zero, so the result of 250 fits in the lower half (`al`). |
| Overflow flag | Cleared | Same reason as the carry flag: no significant bits ended up in the upper half. |
| Sign flag | Undefined | Not calculated by `mul`. Bit 7 of `al` is 1 (`0xFA`), but the flag does not reflect it. |
| Zero flag | Undefined | Not calculated by `mul`. |
| Parity flag | Undefined | Not calculated by `mul`. |
| Auxiliary carry flag | Undefined | Not calculated by `mul`. |

---

## Program 2: mul2 (3000 times 200)

```nasm
mov ax, [num1]      ; ax = 3000
mul word [num2]     ; dx:ax = 3000 * 200 = 600000
mov [result], ax    ; lower 16 bits
mov [result+2], dx  ; upper 16 bits
```

Result: `dx = 9`, `ax = 0x27C0`, which together make `0x000927C0` (600000).

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Set | The upper half (`dx`) is 9, which is not zero. The result needs 20 bits and does not fit in `ax` alone. |
| Overflow flag | Set | Same reason as the carry flag: significant bits ended up in the upper half. |
| Sign flag | Undefined | Not calculated by `mul`. |
| Zero flag | Undefined | Not calculated by `mul`. |
| Parity flag | Undefined | Not calculated by `mul`. |
| Auxiliary carry flag | Undefined | Not calculated by `mul`. |

---