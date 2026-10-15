## Preamble

```
SEP: To Be Assigned
Title: Contract Error Conventions
Author: Pamphile Roy <@tupui>
Status: Draft
Created: 2026-10-15
Updated: 2026-10-15
Version: 0.0.1
Discussion: To Be Assigned
```

## Simple Summary

A convention for the errors that Soroban contracts return: how an error enum is
named, how its codes are numbered so that two standards, two libraries, or a
standard and a library never use the same code for different things, and how a
contract handles an error raised by a contract it calls.

## Dependencies

- [SEP-48](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0048.md)
  — Contract Interface Specification. Error enums are published in the contract
  spec as `SC_SPEC_ENTRY_UDT_ERROR_ENUM_V0`; this SEP constrains what goes into
  them.

This SEP is the first of two companion conventions. The second, covering event
naming and layout, is a separate proposal because it changes on a different
cadence (see [Design Rationale](#design-rationale)).

## Motivation

A Soroban contract error reaches a client as `Error(Contract, #N)`: an
`ScError::Contract(u32)` and nothing else. The code carries no name, no
namespace, and no indication of which contract raised it. A client can only
turn it into something meaningful if it knows the contract's error enum, and it
can only compare errors across contracts if the codes mean the same thing.

Today they do not:

- [SEP-41](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)
  says a transfer "fails if the balance is less than `amount`" but defines no
  error. SEP-49, 50, 56 and 57 define none either; the last two say only
  "panic".
- Nothing stops two standards from each claiming the lowest codes. A contract
  that implements both would then have two meanings for `1`, and the client
  cannot tell.
- OpenZeppelin's Stellar library allocates blocks per module (Fungible `1XX`,
  Non-Fungible `2XX`, RWA `3XX`, Vault `4XX`; Pausable `10XX`, Ownable `21XX`,
  and so on). That avoids collisions inside the library, but the same condition
  has different codes in different modules (`InsufficientBalance` is `100` in
  Fungible and `300` in RWA), the blocks are not coordinated with any SEP, and
  a wallet cannot assume them for a token that does not use the library.
- The
  [Stellar documentation](https://developers.stellar.org/docs/build/guides/conventions/error-enum)
  recommends an `Error` enum with explicit `u32` values but gives no rule for
  choosing them.

Other ecosystems solve this by namespacing. Ethereum
([ERC-6093](https://eips.ethereum.org/EIPS/eip-6093)) names errors
`<Domain><Prefix><Subject>` and needs no numbers, because the selector is a
hash of the name and its arguments. Cosmos pairs a codespace string with a
code. Anchor reserves `6000` and above for programs. Soroban's error value has
room for a single `u32` and no codespace, so the namespace has to live inside
the number. This SEP defines that.

## Abstract

This SEP defines (1) naming rules for error enums and their variants, (2) a
partition of the `u32` contract-error space into a contract-local range, a
library-registry range, and a range derived from SEP numbers, (3) the rules a
contract follows when it raises or receives an error, and (4) a registry,
maintained in this document, of the library ranges. It does not define the
errors of any particular standard; each standard defines its own, using the
range this SEP derives for it.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" are to be
interpreted as described in RFC 2119.

### 1. Naming

1. A contract error enum MUST use `#[contracterror]` with `#[repr(u32)]` and an
   explicit discriminant on every variant. Codes MUST NOT depend on source
   order.
2. An enum defined by a SEP or a library MUST be named `<Domain>Error`, where
   `<Domain>` is a PascalCase noun for the standard or module (`TokenError`,
   `NonFungibleError`, `PausableError`), so that it can sit in the same
   contract spec as other enums. The enum a contract defines for its own
   conditions SHOULD be named `Error`. An enum name MUST be unique within a
   contract spec.
3. A variant name MUST be PascalCase and SHOULD follow `<Prefix><Subject>`,
   after ERC-6093: the prefix says why it failed and the subject says what
   failed. Common prefixes are `Invalid`, `Insufficient`, `Nonexistent`,
   `Exceeded`, `Already`, `Unsupported` and `Not`. The subject is a party or
   item (`Sender`, `Receiver`, `Balance`, `Allowance`, `Token`), never an
   action. The enum name already gives the domain, so a variant MUST NOT repeat
   it (`InsufficientBalance`, not `TokenInsufficientBalance`).
4. Every variant MUST have a doc comment. SEP-48 carries it into the spec, and
   it is what a client shows when it has nothing better.

### 2. Numbering

The `u32` value space is partitioned as follows. The host's own errors use
other `ScErrorType` values, so every `u32` is available to
`ScErrorType::Contract`.

| Range                        | Owner        | Rule                                                      |
| ---------------------------- | ------------ | --------------------------------------------------------- |
| `0`                          | —            | Reserved. MUST NOT be used.                               |
| `1` – `99`                   | The contract | Local errors. Libraries and SEPs MUST NOT use this range. |
| `100` – `39,999`             | Libraries    | Allocated in the registry (section 5), in blocks of 100.  |
| `40,000` – `999,999,999`     | SEPs         | Derived from the SEP number (below).                      |
| `1,000,000,000` – `u32::MAX` | The contract | Local errors, extended. Same rule as `1` – `99`.          |

**SEP-derived codes.** A SEP numbered `N` (`N >= 40`) owns the codes
`N * 1000 + n` for `n` in `1` – `999`. SEP-41 owns `41001` – `41999`, SEP-50
owns `50001` – `50999`. The SEP number is already unique and already arbitrated
by the SEP process, so no further registry is needed, and a reader can recover
the standard from a code (`code / 1000`). A SEP that defines errors MUST define
them in its own block, MUST assign `n` values explicitly in its specification,
and MUST NOT use a code outside its block. Within a block, `n` is assigned
sequentially from `1`.

A contract that implements several standards therefore has disjoint codes by
construction. A condition that occurs in two standards (for example an
insufficient balance in SEP-41 and in SEP-56) has one code per standard; they
do not alias each other.

**Stability.** Once a code is released it MUST NOT be reassigned to another
meaning, in a SEP, in a library, or in a contract. A removed variant keeps its
code, retired. A SEP revision MAY add codes to its block and MUST NOT change
existing ones.

### 3. Raising errors

1. A function defined by a SEP MUST keep the signature the SEP gives it and
   MUST signal an error with `panic_with_error!`. It MUST NOT change its return
   type to `Result` to carry a SEP error, because clients and other contracts
   call the function by its published signature.
2. When a SEP defines an error for a condition, a contract that detects that
   condition in its own code MUST fail with that code. Where more than one
   condition applies at once, which one is reported is unspecified unless the
   SEP says otherwise.
3. Authorization failures raised by `require_auth` are host errors
   (`ScErrorType::Auth`) and MUST NOT be redefined as contract errors.
4. A contract MUST publish every error enum it can raise in its SEP-48 spec,
   including the enums of the SEPs and libraries it implements.

### 4. Receiving errors

1. A contract that calls another contract and does not catch the failure lets
   it propagate. The code the client sees is the callee's code.
2. A contract that catches a callee failure (`try_` calls, `InvokeError`) and
   receives `InvokeError::Contract(code)` SHOULD propagate `code` unchanged
   when `code` falls in a SEP-derived or registered library range and the
   contract has no more specific information to add. It SHOULD NOT map the code
   into its own local range, which would discard the namespace this SEP
   provides.
3. A contract that does translate an error MUST document the mapping in its own
   specification.
4. `InvokeError::Contract` carries a code and no contract address. This SEP
   does not define a wrapper that adds the origin (compare
   [ERC-7751](https://eips.ethereum.org/EIPS/eip-7751)); recovering the origin
   is left to client tooling and may be addressed by a later SEP.

### 5. Library registry

A library that publishes error enums reused across many contracts SHOULD claim
a block of the library range by pull request to this SEP. A block is 100 codes
aligned on a multiple of 100. A library MUST NOT use a block it has not
claimed. The table is seeded with the allocation published by OpenZeppelin
Stellar Contracts at the time of writing; their codes are unchanged by this
SEP.

| Block           | Library              | Module             |
| --------------- | -------------------- | ------------------ |
| `100` – `199`   | OpenZeppelin Stellar | Fungible           |
| `200` – `299`   | OpenZeppelin Stellar | Non-Fungible       |
| `300` – `399`   | OpenZeppelin Stellar | RWA                |
| `400` – `499`   | OpenZeppelin Stellar | Vault              |
| `1000` – `1099` | OpenZeppelin Stellar | Pausable           |
| `1100` – `1199` | OpenZeppelin Stellar | Upgradeable        |
| `1300` – `1399` | OpenZeppelin Stellar | Merkle Distributor |
| `1400` – `1499` | OpenZeppelin Stellar | Crypto             |
| `1500` – `1599` | OpenZeppelin Stellar | Math               |
| `2000` – `2099` | OpenZeppelin Stellar | Access Control     |
| `2100` – `2199` | OpenZeppelin Stellar | Ownable            |
| `2200` – `2299` | OpenZeppelin Stellar | Role Transfer      |
| `3000` – `3999` | OpenZeppelin Stellar | Accounts           |
| `4000` – `4999` | OpenZeppelin Stellar | Governance         |
| `5000` – `5999` | OpenZeppelin Stellar | Fee Abstraction    |

A block in this table is owned by the library for as long as the library
publishes it. A library MAY return a block it no longer needs by pull request.

### 6. Client behavior

A client that decodes an error SHOULD look the name up in the contract's SEP-48
spec first. When the code is in a SEP-derived range and the client knows that
SEP, it MAY present the SEP's description for the code. A client MUST treat a
code it cannot resolve as a failure of unspecified cause and MUST NOT infer a
meaning from a code outside the owner's block.

### Informative example

SEP-41 could adopt this SEP by amendment and define, in its own block:

```rust
#[contracterror]
#[derive(Copy, Clone, Debug, Eq, PartialEq)]
#[repr(u32)]
pub enum TokenError {
    /// The balance of `from` is less than the amount.
    InsufficientBalance = 41001,
    /// The allowance of `spender` is less than the amount.
    InsufficientAllowance = 41002,
    /// The amount is negative.
    InvalidAmount = 41003,
}
```

The codes and names above are illustrative. The actual error sets belong in the
SEPs that define the interfaces, not in this one.

## Design Rationale

**Why the SEP number.** The alternatives were (a) a registry of blocks for
every standard, as OpenZeppelin does for its modules, (b) a single offset such
as Anchor's `6000`, and (c) leaving each SEP to pick. A registry for standards
duplicates what the SEP process already does, since the SEP number is a unique,
arbitrated identifier, and it would force a registry amendment on every SEP
that defines an error. A single offset separates framework from user errors but
does nothing about two libraries colliding inside the user range. Leaving it to
each SEP is the current state, and it lets two standards claim the same low
codes. Deriving the block from the SEP number needs no coordination and is
checkable by a client without a lookup table.

**Why a library registry.** Libraries are not SEPs and already ship codes
(OpenZeppelin's, in particular). Breaking them to fit a SEP-derived scheme
would cost more than registering them, so the library range keeps their
allocation and adds a place for others to claim blocks.

**Why a local range.** A contract's own conditions need no registry and no SEP.
Keeping `1`–`99` for them (and the range at the top for contracts that need
more) makes the common single-contract case need no allocation at all. The cost
is that a standard cannot use the low codes for itself; it takes its derived
block instead.

**Why not a codespace.** `ScError::Contract` is a single `u32`. A codespace
would need an XDR change, a protocol upgrade and new tooling, to gain what a
partition of the existing space already gives.

**Why name rules at all.** Names are not on-chain, but they are in the SEP-48
spec and in every SDK that generates bindings. Two enums named `Error` in one
spec are ambiguous, and `<Prefix><Subject>` keeps names predictable across
standards in the way ERC-6093 does.

**Companion SEP.** Event naming and layout are mostly style and are already
partly written down (SEP-41, SEP-50, the SDK's `#[contractevent]`). This SEP
needs a living registry and a named owner for arbitration; an event style guide
does not. Keeping them apart means a registry amendment does not reopen the
event rules.

**Out of scope.** The error sets of individual standards (those are amendments
to SEP-41, SEP-50 and so on), wrapping errors with their origin, and
storage-key or `#[contracttype]` naming.

## Security Concerns

An error code is data returned by the callee. A malicious contract can return
`41001` without being a token, so a code MUST NOT be treated as proof of state
or used for a security decision about a contract that is not otherwise trusted.
A wallet that shows a SEP-derived description for a code SHOULD attribute it to
the contract that raised it and not present it as a statement of the ledger.

Collisions in a composite contract make a client decode one condition as
another. The partition in section 2 exists to prevent this; contracts and
libraries that step outside their range reintroduce it.

A released code that is later reassigned changes the meaning of failures in
transactions already recorded and in clients already deployed. Hence the
stability rule.

## Changelog

- `v0.0.1`: Initial draft.
