# ADD

## Program 1: add2 (32000 + 500)

```nasm
mov ax, [num1]      ; AX = 32000 (0x7D00)
add ax, [num2]      ; AX = 32000 + 500 = 32500 (0x7EF4)
mov [result], ax
```

Result: `AX = 0x7EF4` (32500)

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Cleared | There is no carry out of bit 15. As unsigned, 32500 fits in 16 bits (max 65535). |
| Overflow flag | Cleared | Two positive numbers gave a positive result. As signed, 32500 fits in 16 bits (max 32767), so there is no signed overflow. |
| Sign flag | Cleared | Bit 15 of the result is 0, so the result is not negative. |
| Zero Flag | Cleared | The result is not zero. |
| Parity flag| Cleared | The low byte is `0xF4` = `11110100`, which has five 1-bits (odd), so parity is not set. |
| Auxillary Carry flag| Cleared | Adding the low nibbles `0x0 + 0x4` gives `0x4`, with no carry out of bit 3. |

The result fits both as unsigned and as signed, so Carry Flag and Overflow are both cleared.

---

## Program 2: add3 (0xFFFF + 1, then ADC)

```nasm
mov ax, [num1]      ; AX = 0xFFFF
add ax, [num2]      ; AX = 0xFFFF + 1 -> 0x0000, carry out
adc ax, 0           ; AX = AX + 0 + CF
mov [result], ax
```

### Step 1: `add ax, [num2]`

Result: `AX = 0x0000`

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Set | The true sum is `0x10000`, which needs 17 bits. The 17th bit is carried out of bit 15, so as unsigned the result is wrapped around. |
| Overflow flag| Cleared | As signed, `0xFFFF` is -1, and -1 + 1 = 0 is correct. Adding numbers of opposite signs can never overflow. |
| Sign flag| Cleared | Bit 15 of the result is 0. |
| Zero flag | Set | The 16-bit result is exactly zero. The carry is not part of the result. |
| Parity flag| Set | The low byte is `0x00`, which has zero 1-bits (an even count). |
| Auxillary carry flag | Set | The low nibbles `0xF + 0x1` = `0x10`, which carries out of bit 3. |

### Step 2: `adc ax, 0`

`ADC` adds the source plus the carry flag, so this computes `0x0000 + 0 + CF(1)`.

Result: `AX = 0x0001`

| Flag | Status | Why |
|------|--------|-----|
| Carry flag | Cleared | `0 + 0 + 1 = 1` does not carry out of bit 15. ADC used up the old carry and produced a new one. |
| Overflow flag| Cleared | A small positive result with no sign change, so no signed overflow. |
| Sign flag| Cleared | Bit 15 of the result is 0. |
| Zero Flag| Cleared | The result is 1, not zero. |
| Parity flag| Cleared | The low byte `0x01` has one 1-bit (odd). |
| Auxillary carry flag| Cleared | The low nibble `0 + 0 + 1 = 1` does not carry out of bit 3. |

Final value: `result = 1`. The carry from the first addition was added into the second one, which is how multi-word addition works.

---
