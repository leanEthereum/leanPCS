---
title: MULFIELD instruction for GF(2^128) multiplication
description: Add a MULFIELD (0x1f) EVM instruction that multiplies two elements of the binary field GF(2^128)
author: Greg Hirsch
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: Core
created: 2026-11-22
---

## Abstract

This EIP introduces a new EVM instruction, `MULFIELD` (`0x1f`), that multiplies two elements of the finite field $\mathrm{GF}(2^{128}) = K[X] / (X^2 + YX + 1)$ over $K = \mathrm{GF}(2)[Y] / (Y^{64} + Y^4 + Y^3 + Y + 1)$. Together with `XOR` (`0x18`) for field addition, `MULFIELD` enables cheap binary-field arithmetic in the EVM, particularly for onchain verification of STARK polynomial interactive oracle proofs (PIOPs) such as those used with leanPCS.

## Motivation

Post-quantum, hash-based STARK verifiers perform two main kinds of work: hashing (for Fiat–Shamir and Merkle proofs) and finite-field arithmetic. When PCS openings are aggregated out of the EVM (as in leanPCS), the remaining onchain verifier—the PIOP—is mostly hashes and field multiplications.

Binary extension fields are simple to specify, map naturally onto bit-level operations, and achieve state-of-the-art proving and verification performance. The EVM already supports binary-field addition via `XOR` (`0x18`), but emulating binary-field multiplication in EVM bytecode is expensive in both gas and code size. Adding a dedicated `MULFIELD` instruction makes binary-field PIOP verifiers compact and cheap to execute without enshrining a full zkVM ISA.

Beyond leanPCS, `MULFIELD` is useful for standalone binary-field STARK/WHIR/Binius verifiers and binary-field Reed–Solomon coding in smart contracts.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Fields

Let:

- $K = \mathrm{GF}(2)[Y] / (Y^{64} + Y^4 + Y^3 + Y + 1)$
- $E = K[X] / (X^2 + YX + 1)$ ($X^2 + YX + 1$ is irreducible over $K$, giving a field of $2^{128}$ elements; leanVM writes $x$ for $Y$ and $y$ for $X$)

An element of $K$ is encoded as a 64-bit unsigned integer whose bit $j$ ($0 \le j \le 63$) is the coefficient of $Y^j$ (bit 0 is the constant term and bit 63 is the coefficient of $Y^{63}$).

An element $a + bX \in E$ (with $a, b \in K$) is encoded as the 256-bit EVM word:

```
a | (b << 64)
```

whose top 128 bits (bits 128 through 255) are zero. In byte encodings, it is represented as that word's low 16 bytes, big-endian (`b || a`, each 8 bytes big-endian).

### `MULFIELD`

| Opcode | Mnemonic | Stack inputs | Stack outputs | Gas cost |
|---|---|---|---|---|
| `0x1f` | `MULFIELD` | `a`, `b` (`a` on top) | `product` | `10` |

Execution semantics:

1. Charge `10` gas. If remaining gas is insufficient, halt exceptionally (out of gas).
2. Pop `a` (top of stack), then pop `b`. If the stack has fewer than 2 items, halt exceptionally (stack underflow).
3. Ignore the top 128 bits of `a` and `b` (interpret `a & (2**128 - 1)` and `b & (2**128 - 1)` as elements of $E$).
4. Compute `product = a * b` in $E$ (multiplication is commutative, so operand order does not affect the result).
5. Push `product` onto the stack, zero-extended to 256 bits (bits 128 through 255 MUST be `0`).

## Rationale

### Why binary fields and this tower

Addition in characteristic 2 is bitwise `XOR`, which the EVM already provides in 3 gas (`0x18`) with no carry propagation. The moduli $Y^{64} + Y^4 + Y^3 + Y + 1$ (the standard irreducible pentanomial of degree 64) and $X^2 + YX + 1$ (irreducible over $K$) have minimal Hamming weight, making both hardware (`PCLMULQDQ` on x86-64, `PMULL` on AArch64) and software implementations fast and concise. This matches the field used by leanVM and leanPCS.

### Why `MULFIELD` instead of a generic `CLMUL`

A generic carry-less multiplication opcode (`CLMUL`) would support multiple binary fields, but multiplying two elements of $E$ from `CLMUL` requires several multiplications, shifts, masks, and `XOR`s for Karatsuba/schoolbook composition and modular reduction. In straight-line PIOP verifiers (which execute hundreds to thousands of field multiplications and must fit under EIP-170's 24 KB contract size limit), folding multiplication and reduction into a single 1-byte instruction saves both execution gas and bytecode size.

### Ignoring the top 128 bits on input and zeroing them on output

Ignoring the top 128 bits of inputs allows contracts to load 16-byte field elements with `CALLDATALOAD` or `MLOAD` (at offset `-16`) and pass them straight into `MULFIELD` without masking first. Guaranteeing that the top 128 bits of the output are zero ensures the result is in canonical form for `EQ`, `XOR`, and memory packing.

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
| Zero | `0x0000000000000000000000000000000000000000000000000000000000000000` | `0x00000000000000000000000000000000ffffffffffffffffffffffffffffffff` | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| Identity | `0x0000000000000000000000000000000000000000000000000000000000000001` | `0x000000000000000000000000000000000123456789abcdeffedcba9876543210` | `0x000000000000000000000000000000000123456789abcdeffedcba9876543210` |
| $Y^{63} \cdot Y = Y^4 + Y^3 + Y + 1$ | `0x0000000000000000000000000000000000000000000000008000000000000000` | `0x0000000000000000000000000000000000000000000000000000000000000002` | `0x000000000000000000000000000000000000000000000000000000000000001b` |
| $X \cdot X = YX + 1$ | `0x0000000000000000000000000000000000000000000000010000000000000000` | `0x0000000000000000000000000000000000000000000000010000000000000000` | `0x0000000000000000000000000000000000000000000000020000000000000001` |
| $X \cdot (YX + 1) = (Y^2 + 1)X + Y$ | `0x0000000000000000000000000000000000000000000000010000000000000000` | `0x0000000000000000000000000000000000000000000000020000000000000001` | `0x0000000000000000000000000000000000000000000000050000000000000002` |
| Top 128 bits ignored | `0xdeadbeefcafebabe0123456789abcdef00000000000000000000000000000003` | `0xffffffffffffffffffffffffffffffff00000000000000000000000000000005` | `0x000000000000000000000000000000000000000000000000000000000000000f` |
| General product | `0x000000000000000000000000000000000123456789abcdef1122334455667788` | `0x00000000000000000000000000000000fedcba98765432108877665544332211` | `0x000000000000000000000000000000006bfe016ed9bc44d6f9b43e74ddd05cd0` |
| Multiplicative inverse | `0x000000000000000000000000000000000123456789abcdef1122334455667788` | `0x000000000000000000000000000000004e2d41e3694f04a51009195a7455a396` | `0x0000000000000000000000000000000000000000000000000000000000000001` |

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

def mulfield(a, b):  # multiplication in E (ignores top 128 bits; top 128 bits of result are 0)
    m = 2**64 - 1
    a0, a1, b0, b1 = a & m, (a >> 64) & m, b & m, (b >> 64) & m
    t = mul_k(a1, b1)  # X^2 = YX + 1
    c0 = mul_k(a0, b0) ^ t
    c1 = mul_k(a0, b1) ^ mul_k(a1, b0) ^ mul_k(t, 2)
    return c0 | (c1 << 64)
```

## Security Considerations

- **Ignored top 128 bits and malleability.** Because `MULFIELD` ignores bits 128–255 of its operands, contracts that read field elements from unmasked 32-byte words (for example from calldata or EIP-8141 `ARBITRARY` signature entries) MUST mask or reject nonzero top 128 bits whenever canonical encoding is required (such as before `EQ` comparisons or to prevent transaction/proof malleability).
- **Denial-of-service / gas pricing.** `MULFIELD` is a pure, fixed-width arithmetic instruction with no memory or state side effects. Its gas cost must be set high enough to prevent worst-case execution or L1 zkEVM proving DoS across client architectures, including hosts without hardware carry-less multiplication instructions.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
