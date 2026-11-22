---
title: MULFIELD instruction for GF(2^192) multiplication
description: Add a MULFIELD (0x1f) EVM instruction that multiplies two elements of the binary field GF(2^192)
author: Greg Hirsch
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: Core
created: 2026-11-22
---

## Abstract

This EIP introduces a new EVM instruction, `MULFIELD` (`0x1f`), that multiplies two elements of the finite field $\mathrm{GF}(2^{192}) = K[X] / (X^3 + X + 1)$ over $K = \mathrm{GF}(2)[Y] / (Y^{64} + Y^4 + Y^3 + Y + 1)$. Together with `XOR` (`0x18`) for field addition, `MULFIELD` enables cheap binary-field arithmetic in the EVM, particularly for onchain verification of STARK polynomial interactive oracle proofs (PIOPs) such as those used with leanPCS.

## Motivation

Post-quantum, hash-based STARK verifiers perform two main kinds of work: hashing (for Fiat–Shamir and Merkle proofs) and finite-field arithmetic. When PCS openings are aggregated out of the EVM (as in leanPCS), the remaining onchain verifier—the PIOP—is mostly hashes and field multiplications.

Binary extension fields are simple to specify, map naturally onto bit-level operations, and achieve state-of-the-art proving and verification performance. The EVM already supports binary-field addition via `XOR` (`0x18`), but emulating binary-field multiplication in EVM bytecode is expensive in both gas and code size. Adding a dedicated `MULFIELD` instruction makes binary-field PIOP verifiers compact and cheap to execute without enshrining a full zkVM ISA.

Beyond leanPCS, `MULFIELD` is useful for standalone binary-field STARK/WHIR/Binius verifiers and binary-field Reed–Solomon coding in smart contracts.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Fields

Let:

- $K = \mathrm{GF}(2)[Y] / (Y^{64} + Y^4 + Y^3 + Y + 1)$
- $E = K[X] / (X^3 + X + 1)$

An element of $K$ is encoded as a 64-bit unsigned integer whose bit $j$ ($0 \le j \le 63$) is the coefficient of $Y^j$ (bit 0 is the constant term and bit 63 is the coefficient of $Y^{63}$).

An element $a + bX + cX^2 \in E$ (with $a, b, c \in K$) is encoded as the 256-bit EVM word:

```
a | (b << 64) | (c << 128)
```

whose top 64 bits (bits 192 through 255) are zero. In byte encodings, it is represented as that word's low 24 bytes, big-endian (`c || b || a`, each 8 bytes big-endian).

### `MULFIELD`

| Opcode | Mnemonic | Stack inputs | Stack outputs | Gas cost |
|---|---|---|---|---|
| `0x1f` | `MULFIELD` | `a`, `b` (`a` on top) | `product` | `10` |

Execution semantics:

1. Charge `10` gas. If remaining gas is insufficient, halt exceptionally (out of gas).
2. Pop `a` (top of stack), then pop `b`. If the stack has fewer than 2 items, halt exceptionally (stack underflow).
3. Ignore the top 64 bits of `a` and `b` (interpret `a & (2**192 - 1)` and `b & (2**192 - 1)` as elements of $E$).
4. Compute `product = a * b` in $E$ (multiplication is commutative, so operand order does not affect the result).
5. Push `product` onto the stack, zero-extended to 256 bits (bits 192 through 255 MUST be `0`).

## Rationale

### Why binary fields and this tower

Addition in characteristic 2 is bitwise `XOR`, which the EVM already provides in 3 gas (`0x18`) with no carry propagation. The moduli $Y^{64} + Y^4 + Y^3 + Y + 1$ (the standard irreducible pentanomial of degree 64) and $X^3 + X + 1$ (irreducible trinomial over $K$) have minimal Hamming weight, making both hardware (`PCLMULQDQ` on x86-64, `PMULL` on AArch64) and software implementations fast and concise. This matches the field used by leanVM and leanPCS.

### Why `MULFIELD` instead of a generic `CLMUL`

A generic carry-less multiplication opcode (`CLMUL`) would support multiple binary fields, but multiplying two elements of $E$ from `CLMUL` requires several multiplications, shifts, masks, and `XOR`s for Karatsuba/schoolbook composition and modular reduction. In straight-line PIOP verifiers (which execute hundreds to thousands of field multiplications and must fit under EIP-170's 24 KB contract size limit), folding multiplication and reduction into a single 1-byte instruction saves both execution gas and bytecode size.

### Ignoring the top 64 bits on input and zeroing them on output

Ignoring the top 64 bits of inputs allows contracts to load 24-byte field elements with `CALLDATALOAD` or `MLOAD` (at offset `-8`) and pass them straight into `MULFIELD` without masking first. Guaranteeing that the top 64 bits of the output are zero ensures the result is in canonical form for `EQ`, `XOR`, and memory packing.

### Opcode assignment

Opcode `0x1f` is unassigned and sits at the end of the `0x10`–`0x1f` comparison and bitwise block, alongside `XOR` (`0x18`) and `CLZ` (`0x1e`, EIP-7939).

### Gas cost

A constant cost of `10` gas is comparable to `MUL` (`5` gas) and `MULMOD` (`8` gas) on CPUs with native carry-less multiplication (`PCLMULQDQ` / `PMULL`). Final pricing should also account for software fallbacks on hosts without hardware CLMUL and for proving cost in L1 zkEVMs (especially non-CLMUL ISAs).

## Backwards Compatibility

`MULFIELD` uses opcode `0x1f`, which was previously undefined and caused an exceptional halt consuming all available gas. Existing contracts that execute `0x1f` as an invalid opcode will observe different behavior; no standard compiler emits `0x1f`.

## Test Cases

All values are 256-bit unsigned integers in big-endian hexadecimal.

| Case | `a` | `b` | `MULFIELD(a, b)` |
|---|---|---|---|
| Zero | `0x0000000000000000000000000000000000000000000000000000000000000000` | `0x0000000000000000ffffffffffffffffffffffffffffffffffffffffffffffff` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| Identity | `0x0000000000000000000000000000000000000000000000000000000000000001` | `0x00000000000000000123456789abcdeffedcba98765432100011223344556677` | `0x00000000000000000123456789abcdeffedcba98765432100011223344556677` |
| $Y^{63} \cdot Y = Y^4 + Y^3 + Y + 1$ | `0x0000000000000000000000000000000000000000000000008000000000000000` | `0x0000000000000000000000000000000000000000000000000000000000000002` | `0x000000000000000000000000000000000000000000000000000000000000001b` |
| $X^2 \cdot X = X + 1$ | `0x0000000000000000000000000000000100000000000000000000000000000000` | `0x0000000000000000000000000000000000000000000000010000000000000000` | `0x0000000000000000000000000000000000000000000000010000000000000001` |
| $X^2 \cdot X^2 = X^2 + X$ | `0x0000000000000000000000000000000100000000000000000000000000000000` | `0x0000000000000000000000000000000100000000000000000000000000000000` | `0x0000000000000000000000000000000100000000000000010000000000000000` |
| Top 64 bits ignored | `0xdeadbeefcafebabe000000000000000000000000000000000000000000000003` | `0xffffffffffffffff000000000000000000000000000000000000000000000005` | `0x000000000000000000000000000000000000000000000000000000000000000f` |
| General product | `0x00000000000000000123456789abcdef112233445566778899aabbccddeeff00` | `0x0000000000000000fedcba9876543210887766554433221100ffeeddccbbaa99` | `0x0000000000000000b7fa7ec40bf6739ae1eb1e61ec96623c9ed9a693b7c0b836` |
| Multiplicative inverse | `0x00000000000000000123456789abcdef112233445566778899aabbccddeeff00` | `0x0000000000000000915de5cffad350ef0a12bb42f109902ebdda05ac352d7c15` | `0x0000000000000000000000000000000000000000000000000000000000000001` |

## Reference Implementation

```python
def mul_k(a, b):  # multiplication in K (a, b in [0, 2**64 - 1])
    r = 0
    for i in range(64):
        if (b >> i) & 1:
            r ^= a << i
    for i in range(126, 63, -1):  # reduce by Y^64 + Y^4 + Y^3 + Y + 1
        if (r >> i) & 1:
            r ^= ((1 << 64) | 0b11011) << (i - 64)
    return r

def mulfield(a, b):  # multiplication in E (ignores top 64 bits; top 64 bits of result are 0)
    x = [(a >> (64 * i)) & (2**64 - 1) for i in range(3)]
    y = [(b >> (64 * i)) & (2**64 - 1) for i in range(3)]
    c = [0] * 5
    for i in range(3):
        for j in range(3):
            c[i + j] ^= mul_k(x[i], y[j])
    for d in (4, 3):  # X^d = X^(d-2) + X^(d-3)
        c[d - 2] ^= c[d]
        c[d - 3] ^= c[d]
    return c[0] | (c[1] << 64) | (c[2] << 128)
```

## Security Considerations

- **Ignored top 64 bits and malleability.** Because `MULFIELD` ignores bits 192–255 of its operands, contracts that read field elements from unmasked 32-byte words (for example from calldata or EIP-8141 `ARBITRARY` signature entries) MUST mask or reject nonzero top 64 bits whenever canonical encoding is required (such as before `EQ` comparisons or to prevent transaction/proof malleability).
- **Denial-of-service / gas pricing.** `MULFIELD` is a pure, fixed-width arithmetic instruction with no memory or state side effects. Its gas cost must be set high enough to prevent worst-case execution or L1 zkEVM proving DoS across client architectures, including hosts without hardware carry-less multiplication instructions.
- **Permanence of the field.** Unlike versioned PCS schemes in leanPCS, an EVM opcode's field cannot be upgraded in place once deployed; changing the field later would require allocating a new opcode.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
