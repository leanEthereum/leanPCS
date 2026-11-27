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

This EIP lets [EIP-8141](./eip-8141.md) transactions use STARKs (post-quantum, hash-based SNARKs) cheaply, including STARKs that sign their transaction. A STARK check splits into a cheap part (the PIOP), run by a **glue** contract in the EVM, and a heavy part (PCS openings, "PCS" for "Polynomial Commitment Scheme"), which every block aggregates into one STARK with [EIP-8288](./eip-8288.md). One new claim-only verifier, `LEANPCS_VERIFIER`, ties the two together, so contracts check a whole STARK with `HASCLAIM`, like a signature. The companion [`MULFIELD`](./eip-draft-mulfield.md) instruction makes the PIOP's $\mathrm{GF}(2^{128})$ arithmetic cheap.

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
| `PCS_VK_HASH`                  | TBD               |

EIP-8141 here is its claims-list version (kevaundray/EIPs#7): claims, segregated witnesses, custom verifiers, the pure context, `HASCLAIM`, the pooled wrapper and `dependency_proof` come from it. The recursive STARK comes from EIP-8288, which this EIP requires to be active.

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

A STARK check has a cheap part, the PIOP, and a heavy part, the PCS openings. A **glue** contract runs the PIOP in the EVM and outputs a **statement**, the hash of the PCS claims that the openings must prove. The block proves all statements with one recursive STARK ([EIP-8288](./eip-8288.md)). The protocol verifier `LEANPCS_VERIFIER` ties the two together, so a contract checks the whole STARK with one `HASCLAIM`.

```
statement = keccak256(keccak256(c_0) || ... || keccak256(c_{k-1}))   # PCS claims c_i, 1 <= k <= MAX_PCS_CLAIMS_PER_STATEMENT
glue_id   = keccak256(glue || glue_key)                                # glue: 20-byte address, glue_key: 32 bytes
```

A statement is **true** if every `c_i` is well-formed and holds. `glue_key` is whatever the glue interprets, for example the hash of an ML-DSA public key or a circuit's verification key.

#### The claim

A leanPCS claim is the EIP-8141 claim `[LEANPCS_VERIFIER, glue_id, msg, gas_limit, witness_len]`. A transaction is invalid if one breaks these bounds:

- `glue_id` is 32 bytes and names the glue and its key;
- `msg` is as in EIP-8141: empty for `compute_sig_hash(tx)`, or an explicit non-zero 32-byte message;
- `gas_limit` is the glue's execution gas, charged in full, with `0 < gas_limit < 2**64`;
- `witness_len` is the exact length of the witness, with `84 <= witness_len <= MAX_CLAIM_WITNESS_BYTES`.

Its witness is `statement (32 bytes) || glue (20 bytes) || glue_key (32 bytes) || transcript`. The claim holds if and only if:

```python
def leanpcs_claim_holds(claim, witness, resolved_msg, state, proven) -> bool:
    statement, glue, glue_key, transcript = witness[:32], witness[32:52], witness[52:84], witness[84:]
    # No EVM needed for these two checks.
    if keccak256(glue + glue_key) != claim.key:
        return False
    if statement not in proven:
        return False
    # Run the glue like an EIP-8141 custom verifier (pure context, own access sets,
    # entry access charged from gas_limit), and require it to return the statement.
    if is_precompile(glue) or code_is_empty_or_delegated(state, glue):
        return False
    ok, output = execute_pure_verifier(glue, glue_key + resolved_msg + transcript, claim.gas_limit, state)
    return ok and output == statement
```

`state` is the block's pre-state, or the pool's current state in a mempool. `proven` is the set of statements that a valid proof covers: the transaction's `statements_proof` in a mempool, the block's `aggregate` in a block.

#### Proving statements

- **Dependency.** Each statement is the EIP-8288 dependency `(LEANSTARK_SCHEME, statement, PCS_VK_HASH)`, read from the witness rather than declared in a frame. `PCS_VK_HASH` names one relation, fixed by the fork: given PCS claims `c_0, ..., c_{k-1}` and their opening proofs, `pcs_verify(c_i, opening_proof_i)` holds for every `i` and the `c_i` hash to `statement`. EIP-8288's recursive STARK proves these dependencies from this relation, or from inner recursive STARKs, under `AGGREGATED_VK`.
- **Coverage.** The statements `D` of a transaction or a block are the dependencies of its leanPCS witnesses, deduplicated and sorted; in a block, `D` also includes EIP-8288's `dependencies(block)`. A proof **covers** `D` if it verifies under `AGGREGATED_VK` with public inputs `[get_deps_hash(D), AGGREGATED_VK]`.
- **One digest encoding.** This EIP replaces EIP-8288's `get_deps_hash` for every list, in headers, wrappers, recursion and discards: `get_deps_hash(deps) = DEPS_HASH(concat(scheme || data_hash || verification_key_hash for each triple))`, with `scheme` as 31 zero bytes then the scheme byte, so lists of different schemes never share an encoding. Frames naming `LEANSTARK_SCHEME` stay invalid, and EIP-8288's leanSTARK limits and gas do not apply to statements.

#### Transport

- **Pooled wrapper.** A transaction with a leanPCS claim travels in pooled wrapper version `1`, `FRAME_TX_TYPE || rlp([tx_body, 1, witnesses, blob_sidecar, statements_proof])`. `statements_proof`, at most `MAX_PCS_PROOF_BYTES`, covers the transaction's statements. At EIP-8141's acceptance step 2, a node rejects a version `0` wrapper of such a transaction and a version `1` wrapper of any other. At step 5, before reading any state, it checks every `glue_id` and the `statements_proof`. At step 6, it runs each glue. A failing wrapper is rejected; its `tx_hash` is not marked invalid.
- **Block.** A block with a leanPCS claim carries `dependency_proof` version `1`, `0x01 || rlp([witnesses, aggregate])`, canonical with no trailing bytes. `witnesses` is as in version `0`, one raw witness per dependency in `block_dependencies(block)`, leanPCS witnesses included, since validators run their glue. `aggregate`, at most `MAX_PCS_PROOF_BYTES`, is EIP-8288's dependency proof and MUST cover the block's `D`, with `block_deps_hash = get_deps_hash(D)`. Otherwise, or with any other `dependency_proof` version, the block is invalid.

#### Changes to EIP-8141

- **Gas and length.** Unlike EIP-8141's other protocol verifiers, `LEANPCS_VERIFIER` takes its gas and witness length from the claim: its `resolve_claim_gas` is `gas_limit + PCS_STATEMENT_GAS`, and its `resolve_witness_len` is `witness_len`. `PCS_STATEMENT_GAS` pays for verifying and aggregating one statement. The witness is priced as a custom verifier's witness. `statements_proof` is not charged by the byte: it never reaches a block.
- **Evaluation.** Unlike the others, a leanPCS claim is never evaluated without the EVM: it always runs its glue.
- **`MULFIELD`** is permitted in the pure context.

### Using leanPCS claims

- **A proof that signs its transaction.** An account stores its `glue_id` and checks `HASCLAIM(LEANPCS_VERIFIER, glue_id, 0)`. The glue runs the PIOP against `compute_sig_hash(tx)`, so it must fit the account's validation gas.
- **An application proof.** An application whose PIOP does not depend on `compute_sig_hash(tx)` can run the PIOP in its own frame, with state access and no validation gas limit. It uses an **echo glue**: a contract `ECHO_GLUE`, pinned by the application, that always returns bytes 32 to 63 of its calldata (`resolved_msg`). The transaction declares `[LEANPCS_VERIFIER, keccak256(ECHO_GLUE || bytes32(0)), statement, gas_limit, 84]` with witness `statement || ECHO_GLUE || bytes32(0)`. The application checks the PIOP, computes `statement` itself and checks `HASCLAIM(LEANPCS_VERIFIER, keccak256(ECHO_GLUE || bytes32(0)), statement)`. Such a claim says only that `statement` is true, never that anything is authorized.

A contract MUST name `LEANPCS_VERIFIER`, never the glue: a claim naming the glue as a custom verifier holds whenever the call succeeds, with no PCS claim proven. An echo that can return anything other than `resolved_msg` lets a forger pass another true statement under the application's `msg`.

## Rationale

### Benchmarks

These figures were measured under an earlier design of this EIP, with the PIOP in a frame, over the prototype's $\mathrm{GF}(2^{192})$. Under this EIP, the PIOP runs as the glue, its gas is the claim's `gas_limit`, its proof is the claim's witness, and each claim adds `PCS_STATEMENT_GAS`; the figures have to be measured again. *PIOP* is the PIOP's frame, *Total* adds the frame of PCS claims and that design's per-claim gas. *Prover* adds 0.1 s per claim for aggregating, client side, every PCS opening, into a common STARK.

| Application | Prover (Apple M4 Max) | Calldata | PIOP | PCS claims | Total |
|---|---|---|---|---|---|
| Zero-knowledge membership in a Keccak-256 Merkle tree of depth 32 | 1.7 s | 273 bytes | 20,085 gas | 3 | 31,680 gas |
| leanVM verification (here 128 leanXMSS signatures), i.e. any computation thanks to recursion | 3.6 s | 577 bytes | 40,396 gas | 7 | 63,328 gas |

### Why one verifier with a glue

A STARK's PCS claims are computed from its transcript, so a transaction cannot list them before it is built, and cannot list them at all when the STARK signs the transaction. Something must compute them after signing: here, the glue. Making "glue plus PCS openings" one protocol verifier changes no transaction format:

- **Few new concepts.** One verifier, its glue and its statement. A claim names a verifier, and its key names a program and a public key, as for any other verifier. `HASCLAIM` keeps its meaning and other verifiers keep their rules.
- **Known before execution.** `tx_body` fixes the glue's gas and witness length. Mempools and builders read each statement from its witness, so they know what the block must prove before running any glue.
- **Fails closed.** The protocol compares the glue's output with `statement`. A glue that returns anything else fails, so a proof cannot be skipped.
- **Permissionless.** Anyone can deploy a glue, for ML-DSA, Falcon or an application, without a new fork.

Two alternatives were considered. A custom verifier could call a PCS verifier that records each claim for later: the block learns how many claims to prove only by running the verifier, and a verifier that ignores the call's result passes without a proof. A claim could carry an extra field naming the verifier of a derived claim: that adds a field to every claim and gives other verifiers' return data a meaning.

### Why one statement, proven only in aggregate

The sender aggregates the openings of its transaction's statements, client side, into one STARK, `statements_proof`. The mempool and the block then aggregate one STARK per transaction, whatever its number of PCS claims, so a fixed `PCS_STATEMENT_GAS` per statement pays for this work: verifying and aggregating one more STARK, and one dependency of the block. Raw opening proofs are large, so they never reach a block.

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

The field cannot change, but the PCS can: a future fork can add a version $v + 1$ that brings the latest performance improvements of PCSs. PCS openings used to commit to witnesses can be adapted without redeploying legacy smart contracts relying on leanPCS. Commitments stored in contracts, typically circuit verification keys, can be migrated, following this idea of JD:

To migrate a stored commitment `com` (version $v$) to `com'` (version $v + 1$, same `n`), a contract derives two independent points $r^{(0)}, r^{(1)}$ from both commitments by Fiat-Shamir, takes from the prover the values $y^{(0)}, y^{(1)}$, and requires a claim (with the echo glue of [Using leanPCS claims](#using-leanpcs-claims)) whose statement hashes the four PCS claims that `com` and `com'` both take value $y^{(k)}$ at $r^{(k)}$. Distinct polynomials pass with probability at most $(n / 2^{128})^2$ per attempt (Schwartz–Zippel). To keep migration trustless, the contract should add `com'` to its accepted commitments rather than replace `com` (see [Security Considerations](#security-considerations)).

## Backwards Compatibility

This EIP adds one protocol verifier that, unlike EIP-8141's others, runs EVM code and reads state. It amends EIP-8141's claim constraints, `resolve_claim_gas`, `resolve_witness_len`, `verify_dependency`, direct evaluation of protocol verifiers, and acceptance steps 2, 5 and 6, and adds pooled wrapper version `1` and `dependency_proof` version `1`. It replaces EIP-8288's `get_deps_hash`. It changes no transaction format and no existing verifier.

## Reference Implementation

### PCS claims

TODO

## Security Considerations

**An explicit message is not tied to its transaction.** A claim with an explicit `msg` only says that the glue accepts `msg` under `glue_key` and that the statement holds. Anyone who sees a pending transaction can copy the claim, its witness and its proof into another transaction. What the proof authorizes (a payment, a call) has to be part of `msg` or of the application's transcript, with a domain, the action and a nonce the application consumes. A claim with an empty `msg` is bound to its transaction through `compute_sig_hash(tx)`, like a signature.

**The glue must bind its inputs.** The protocol checks what the glue returns, not how it computed it. A glue must absorb `glue_key` and `resolved_msg` into its transcript before drawing any challenge, derive every PCS claim itself, and return `statement` only once the PIOP checks pass. Its result must not depend on its gas, which the transaction chooses. A glue that skips any of this accepts proofs it should reject, and every contract trusting its `glue_id` inherits the flaw. With the echo glue, these duties fall on the application's frame.

**Absorb the whole commitment.** A root alone can match several polynomials, and `ood` picks out one of them. When a commitment comes from the proof rather than from the contract, the application must hash both `root` and `ood` into its transcript before it draws any challenge that depends on the commitment, and use that `ood` in every claim on that commitment. If it hashes only the root, the prover chooses `ood` after seeing the challenges, and the application's soundness error is multiplied by up to the number of polynomials the root matches.

**Migration can lock a contract to the migrator.** The migration check in [Upgrades](#upgrades) proves that `com'` binds the same polynomial as `com`, so nobody can cheat on the polynomial. It does not prove that anyone else can open `com'`. Opening requires the exact data behind `com'`, and many commitments bind the same polynomial: a migrator can commit with secret blinding (as in ZK-WHIR), or in a code-based PCS like WHIR change one leaf of the encoding to a secret value that opening queries miss. Nobody else knows that secret, so nobody else can produce openings for `com'`. If a contract overwrites `com` with `com'`, only the migrator can open it from then on. Even if the contract keeps `com`, keeping only a single `com'` causes the same lock once the old version is dropped. Migration stays trustless if the contract stores a set of accepted commitments instead of a single one, so anyone can add a commitment they can open before the old version is dropped. Alternatively, a contract can restrict migration to a trusted account.

**Challenges in $E$ give at most 128 bits.** An application draws its challenges from $E$, so a false polynomial identity of degree $d$ still holds at a random challenge with probability up to $d / 2^{128}$ per attempt, which is $2^{-106}$ for $d = 2^{22}$. A random element of $E$ also lies in the subfield $K$ with probability $2^{-64}$, so a check that a prover can pass whenever the challenge lies in $K$ gives at most 64 bits of security. An application that needs more can repeat the check at independent challenges, which multiplies the error probabilities, or add proof of work.

TODO 

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
