---
title: leanPCS
description: Cheap verification of STARKs thanks to aggregation of PCS openings
author: Greg Hirsch
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: Core
created: 2026-11-01
requires: 8141, 8288
---

## Abstract

This EIP lets [EIP-8141](./eip-8141.md) frame transactions carry PCS claims ("PCS" for "Polynomial Commitment Scheme"), each stating that a committed multilinear polynomial takes a given value at a given point. It adds one claim-only protocol verifier, `LEANPCS_VERIFIER`. A claim naming it holds when a **glue** contract, which runs the rest of a STARK verification, accepts a message and outputs a hash of the PCS claims it needs, and the block proves those PCS claims. Contracts check such a claim with `HASCLAIM`, like a signature. Paired with the companion [`MULFIELD`](./eip-draft-mulfield.md) instruction for $\mathrm{GF}(2^{128})$ multiplication, this lets users cheaply verify STARKs (post-quantum, hash-based SNARKs) onchain, including STARKs that sign their transaction. All the heavy PCS opening proofs of a block are aggregated into a single STARK with [EIP-8288](./eip-8288.md).

## Motivation

Today, verifying a pre-quantum SNARK onchain costs about 200,000 gas thanks to the elliptic-curve precompiles. Post-quantum STARKs are much larger, and much more expensive to verify. Recursive aggregation amortizes this cost: a single STARK per block proves all the others. [EIP-8288](./eip-8288.md) was initially designed to aggregate whole STARKs, which would require enshrining a STARK-friendly ISA such as RISC-V.

Most of a STARK verification, in proof size and in verification time, is simply PCS openings. With this EIP, Ethereum aggregates this heavy part, and smart contracts run the cheap remaining part of the STARK verification in the EVM. This rest is called the "polynomial interactive oracle proof" (PIOP). The PIOP mostly computes a few hashes (for Fiat-Shamir) and does arithmetic in some finite field. We pick binary fields for their simplicity and their state of the art performance. The EVM can already add field elements with `XOR` (`0x18`). For efficient field multiplication, contracts use the companion [`MULFIELD`](./eip-draft-mulfield.md) instruction.

The goal of this EIP is to get most of the benefits (cheap onchain STARK verification) from [EIP-8288](./eip-8288.md) without the complexity of enshrining a new ISA.


## WARNING (TODO delete this)

- I am not a frame expert
- Gas pricing is not finalized

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

| Constant                       | Value             |
|--------------------------------|-------------------|
| `LEANPCS_VERIFIER`             | TBD               |
| `MAX_PCS_VARIABLES`            | `32`              |
| `MAX_PCS_CLAIMS_PER_STATEMENT` | `256`             |
| `MAX_PCS_PROOF_BYTES`          | TBD               |
| `PCS_STATEMENT_GAS`            | `6000` (provisional) |

EIP-8141 here is its claims-list version (kevaundray/EIPs#7): claims, segregated witnesses, custom verifiers, the pure context, `HASCLAIM` and `dependency_proof` come from it. The recursive STARK and pooled wrappers that carry a sender's aggregated proof (mode 1) come from EIP-8288.

### Fields

Let $K = \mathrm{GF}(2)[Y] / (Y^{64} + Y^4 + Y^3 + Y + 1)$ and $E = K[X] / (X^2 + YX + 1)$, a field of $2^{128}$ elements ($X^2 + YX + 1$ is irreducible over $K$). This is leanVM's field; leanVM writes $x$ for $Y$ and $y$ for $X$. An element of $K$ is encoded as a 64-bit integer whose bit $j$ is the coefficient of $Y^j$. An element $a + bX$ of $E$, with $a, b \in K$, is encoded as the word `a | (b << 64)`, and in bytes as that word's low 16 bytes, big-endian.

### PCS claims

A PCS claim states that the multilinear polynomial $f$ over $K$ in `n` variables committed to as `commitment` takes the value `value` (in $E$) at `point` (in $E^n$), under PCS version `version`:

```
PCS_claim = version (1 byte) || n (1 byte) || commitment (64 bytes) || point_0 || ... || point_{n-1} (16 bytes each) || value (16 bytes)
```

- **The polynomial.** $f(x) = \sum_i f_i \prod_j \ell_j(i, x)$, where $f_i \in K$ is the $i$-th of the $2^n$ committed values, and $\ell_j(i, x)$ is $x_j$ if bit $j$ of $i$ is 1 and $1 - x_j$ otherwise (bit 0 is the least significant).
- **The commitment.** `commitment = root (32 bytes) || ood (32 bytes)`. `root` is the Merkle root of the PCS version's encoding of $f$. `ood = ood_0 || ood_1` holds two out-of-domain answers, $f(z^{(0)})$ and $f(z^{(1)})$, 16 bytes each, where $z^{(k)}_j$ is the element encoded by the last 16 bytes of `keccak256("leanPCS-ood" || version || n || root || k || j)` (`version`, `n`, `k` and `j` are one byte each).
- **Well-formedness.** A PCS claim is well-formed if `version` is supported, `1 <= n <= MAX_PCS_VARIABLES`, and it is `82 + 16 * n` bytes long.

Each PCS version defines a verifier `pcs_verify(PCS_claim, opening_proof)`. It MUST return false if `PCS_claim` is not well-formed, and MUST check, in one opening, $f(z^{(0)})$ = `ood_0`, $f(z^{(1)})$ = `ood_1` and $f$(`point`) = `value`. The first two checks are what tie a commitment to one polynomial: in the list-decoding regime, a root alone fits a list of polynomials, and the answers at $z^{(0)}$ and $z^{(1)}$, two points drawn from the root, single one out.

We start at version `1`: the zero-knowledge variant (ZK-WHIR) of leanVM's WHIR over $E$, at rate 1/2 in the list-decoding regime. Version `1` is not specified here yet, including which `n` it supports; its verifier returns false for any `n` it does not support. The prototype in this repository (`crates/pcs_claim`, at commit `bb2da7f0`) has not moved to $E$ yet: it works over $\mathrm{GF}(2^{192})$ with one out-of-domain answer, supports `15 <= n <= 28`, and is not zero knowledge yet.

### leanPCS claims

A **statement** is the hash of a list of PCS claims `c_0, ..., c_{k-1}`, with `1 <= k <= MAX_PCS_CLAIMS_PER_STATEMENT`:

```
statement = keccak256(keccak256(c_0) || ... || keccak256(c_{k-1}))
```

A statement is **proven** if each `c_i` is well-formed and holds.

A claim naming `LEANPCS_VERIFIER` is `[LEANPCS_VERIFIER, K, msg, gas_limit, witness_len]`, where:

- `K = keccak256(glue || glue_key)`, with `glue` the 20-byte address of a deployed contract and `glue_key` 32 bytes that the glue interprets, for example the hash of an ML-DSA public key or a circuit's verification key.
- `msg` is as in EIP-8141: empty for `compute_sig_hash(tx)`, or an explicit non-zero 32-byte message.
- `gas_limit` is the glue's execution gas and `witness_len` the exact length of the claim's witness. Unlike EIP-8141's other protocol verifiers, `LEANPCS_VERIFIER` takes both from the claim, as EIP-8141 does for custom verifiers, and charges `gas_limit` in full.

Its witness is `statement (32 bytes) || glue (20 bytes) || glue_key (32 bytes) || transcript`. The claim holds if and only if:

1. `statement` is non-zero and `keccak256(glue || glue_key) == K`;
2. `glue` is deployed, non-delegated code, as for an EIP-8141 custom verifier;
3. calling `glue` in EIP-8141's pure context, with calldata `glue_key || resolved_msg || transcript` and gas `gas_limit`, succeeds and returns exactly the 32 bytes `statement`;
4. `statement` is proven.

Checks 1 and 4 need no EVM: nodes read `statement` from the first 32 bytes of the witness.

**Proving statements.** The sender aggregates the openings of a statement, client side, into one STARK with public input `statement`, at most `MAX_PCS_PROOF_BYTES` long. The pooled wrapper carries this proof beside the claim's witness, as an EIP-8288 mode 1 wrapper does, and a node verifies it before reading any state. In a block, statements are proven only by EIP-8288's recursive STARK, which verifies these proofs: each `LEANPCS_VERIFIER` claim adds `statement` to the dependencies the block's aggregate proves, deduplicated, while the claim's own witness stays raw in `dependency_proof`, since validators run its glue. A block whose `dependency_proof` does not prove statements cannot include a `LEANPCS_VERIFIER` claim.

**Gas.** A `LEANPCS_VERIFIER` claim costs `gas_limit + PCS_STATEMENT_GAS` in claim verification gas, which counts toward EIP-8141's `MAX_VERIFY_GAS`, and its witness is priced as a custom verifier's witness. The sender's proof is not charged by the byte: it never reaches a block.

**`MULFIELD`.** This EIP permits `MULFIELD` in EIP-8141's pure context.

### Using leanPCS claims

A contract relies on a glue's result with `HASCLAIM(LEANPCS_VERIFIER, keccak256(glue || glue_key), msg)`. An account protected by a proof stores its `K` and checks `HASCLAIM(LEANPCS_VERIFIER, K, 0)`. An application checks the claim for its own `msg`, for example the hash of its public inputs.

A contract MUST name `LEANPCS_VERIFIER`, not the glue. A claim naming the glue as a custom verifier holds whenever the call succeeds, with no PCS claim proven.

## Rationale

### Benchmarks

These figures were measured under an earlier design of this EIP, with the PIOP in a frame, over the prototype's $\mathrm{GF}(2^{192})$. Under this EIP, the PIOP runs as the glue, its gas is the claim's `gas_limit`, its proof is the claim's witness, and each claim adds `PCS_STATEMENT_GAS`; the figures have to be measured again. *PIOP* is the PIOP's frame, *Total* adds the frame of PCS claims and that design's per-claim gas. *Prover* adds 0.1 s per claim for aggregating, client side, every PCS opening, into a common STARK.

| Application | Prover (Apple M4 Max) | Calldata | PIOP | PCS claims | Total |
|---|---|---|---|---|---|
| Zero-knowledge membership in a Keccak-256 Merkle tree of depth 32 | 1.7 s | 273 bytes | 20,085 gas | 3 | 31,680 gas |
| leanVM verification (here 128 leanXMSS signatures), i.e. any computation thanks to recursion | 3.6 s | 577 bytes | 40,396 gas | 7 | 63,328 gas |

### Why one verifier with a glue

A STARK's PCS claims are computed from its transcript, so a transaction cannot list them before it is built, and cannot list them at all when the STARK signs the transaction. Something must compute them after signing: here, the glue. Making "glue plus PCS openings" one protocol verifier keeps EIP-8141 unchanged:

- **No new concept.** A claim names a verifier, and its key names a program and a public key, as for any other verifier. `HASCLAIM` keeps its meaning and other verifiers keep their rules.
- **Known before execution.** `tx_body` fixes the glue's gas and witness length. Mempools and builders read each statement from its witness, so they know what the block must prove before running any glue.
- **Fails closed.** The protocol compares the glue's output with `statement`. A glue that returns anything else fails, so a proof cannot be skipped.
- **Permissionless.** Anyone can deploy a glue, for ML-DSA, Falcon or an application, without a new fork.

Two alternatives were considered. A custom verifier could call a PCS verifier that records each claim for later: the block learns how many claims to prove only by running the verifier, and a verifier that ignores the call's result passes without a proof. A claim could carry an extra field naming the verifier of a derived claim: that adds a field to every claim and gives other verifiers' return data a meaning.

### Why one statement, proven only in aggregate

The sender aggregates the openings of its statement, client side, into one STARK. The mempool and the block then aggregate one STARK per statement, whatever its number of PCS claims, so a fixed `PCS_STATEMENT_GAS` pays for this work: verifying and aggregating one more STARK, and one dependency of the block. Raw opening proofs are large, so they never reach a block.

### How a proof signs its transaction

An account can be protected by a proof instead of a signature. Such a proof must attest to `compute_sig_hash(tx)`, like a signature. The proof and its PCS claims are computed from this hash, so neither can be in `tx_body`. In a `LEANPCS_VERIFIER` claim with an empty `msg`, they are in the claim's witness, which `compute_sig_hash(tx)` does not cover.

The sender:

1. writes the transaction, with the claim's `gas_limit` and `witness_len`;
2. computes `compute_sig_hash(tx)`;
3. builds the proof, which attests to this hash, and its statement;
4. fills the witness;
5. aggregates the openings into one STARK and broadcasts the transaction.

The glue receives `compute_sig_hash(tx)` as `resolved_msg` and absorbs it into its transcript. Anyone can change the witness bytes, but the claim then holds only with another valid proof for the same key and transaction.

### Why a 128-bit field, with two out-of-domain answers

Contracts hard-code the field through `MULFIELD` and the claim layout, so this choice is final. $E$ is the field leanVM uses: compared with $\mathrm{GF}(2^{192})$, a multiplication takes about half the work and a field element takes 16 bytes of calldata instead of 24. Version `1` builds on leanVM's WHIR, which targets 125 bits of security over $E$. A field of this size is enough because it folds with a random polynomial of degree one instead of a single random field element (Chiesa, Weissenberg, Yogev and Zheng, ePrint 2026/2863), and adds proof of work before its challenges.

A commitment carries two out-of-domain answers because one is not enough in a field of this size. A root alone fits a list of up to $L$ polynomials. With one answer, some two of them give the same answer with probability at most $\binom{L}{2} n / 2^{128}$ per root, about $2^{-111}$ for $n = 32$ and $L = 100$, and a prover can try many roots offline. leanVM adds proof of work before it draws its out-of-domain point. Here the points are hashed from the root with no proof of work, and a second answer at an independent point lowers the bound instead: $\binom{L}{2} (n / 2^{128})^2$ per root, below $2^{-200}$ for any $L$ up to a million, for 16 more bytes per commitment. The migration check in [Upgrades](#upgrades) uses two points for the same reason: with one, a wrong `com'` passes with probability up to $n / 2^{128}$ per attempt, and attempts can be made offline.

### Upgrades

The field cannot change, but the PCS can: a future fork can add a version $v + 1$ that brings the latest performance improvements of PCSs. PCS opnings used to commit to witness can adapted without redeploying legacy smart contracts relying on leanPCS.Commitments stored in contracts, typically circuit verification keys, can be migrated, following this idea of JD:

To migrate a stored commitment `com` (version $v$) to `com'` (version $v + 1$, same `n`), a contract derives two independent points $r^{(0)}, r^{(1)}$ from both commitments by Fiat-Shamir and requires a leanPCS claim, from a migration glue, that `com` and `com'` take the same value at each point. Distinct polynomials pass with probability at most $(n / 2^{128})^2$ per attempt (Schwartz–Zippel). To keep migration trustless, the contract should add `com'` to its accepted commitments rather than replace `com` (see [Security Considerations](#security-considerations)).

## Backwards Compatibility

This EIP adds one protocol verifier to EIP-8141 claims and permits `MULFIELD` in the pure context. It changes no existing transaction, verifier or EIP-8288 dependency format.

## Reference Implementation

### PCS claims

TODO

## Security Considerations

**An explicit message is not tied to its transaction.** A claim with an explicit `msg` only says that the glue accepts `msg` under `glue_key` and that the statement holds. Anyone who sees a pending transaction can copy the claim, its witness and its proof into another transaction. What the proof authorizes (a payment, a call) has to be part of `msg` or of the application's transcript. A claim with an empty `msg` is bound to its transaction through `compute_sig_hash(tx)`, like a signature.

**The glue must bind its inputs.** The protocol checks what the glue returns, not how it computed it. A glue must absorb `glue_key` and `resolved_msg` into its transcript before drawing any challenge, derive every PCS claim itself, and return `statement` only once the PIOP checks pass. A glue that skips any of this accepts proofs it should reject, and every contract trusting its `K` inherits the flaw.

**Absorb the whole commitment.** A root alone can match several polynomials, and `ood` picks out one of them. When a commitment comes from the proof rather than from the contract, the application must hash both `root` and `ood` into its transcript before it draws any challenge that depends on the commitment, and use that `ood` in every claim on that commitment. If it hashes only the root, the prover chooses `ood` after seeing the challenges, and the application's soundness error is multiplied by up to the number of polynomials the root matches.

**Migration can lock a contract to the migrator.** The migration check in [Upgrades](#upgrades) proves that `com'` binds the same polynomial as `com`, so nobody can cheat on the polynomial. It does not prove that anyone else can open `com'`. Opening requires the exact data behind `com'`, and many commitments bind the same polynomial: a migrator can commit with secret blinding (as in ZK-WHIR), or in a code-based PCS like WHIR change one leaf of the encoding to a secret value that opening queries miss. Nobody else knows that secret, so nobody else can produce openings for `com'`. If a contract overwrites `com` with `com'`, only the migrator can open it from then on. Even if the contract keeps `com`, keeping only a single `com'` causes the same lock once the old version is dropped. Migration stays trustless if the contract stores a set of accepted commitments instead of a single one, so anyone can add a commitment they can open before the old version is dropped. Alternatively, a contract can restrict migration to a trusted account.

**Challenges in $E$ give at most 128 bits.** An application draws its challenges from $E$, so a false polynomial identity of degree $d$ still holds at a random challenge with probability up to $d / 2^{128}$ per attempt, which is $2^{-106}$ for $d = 2^{22}$. A random element of $E$ also lies in the subfield $K$ with probability $2^{-64}$, so a check that a prover can pass whenever the challenge lies in $K$ gives at most 64 bits of security. An application that needs more can repeat the check at independent challenges, which multiplies the error probabilities, or add proof of work.

TODO 

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
