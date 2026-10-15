## Preamble

```
SEP: To Be Assigned
Title: Contract Error Conventions
Author: Pamphile Roy <@tupui>
Status: Draft
Created: 2026-10-15
Updated: 2026-10-15
Version: 0.0.2
Discussion: To Be Assigned
```

## Simple Summary

A convention for the errors that Soroban contracts return: how an error enum is
named, how its codes are numbered so that a code tells a client which standard
defined it and what kind of failure it is, and how a contract handles an error
raised by a contract it calls.

## Dependencies

- [SEP-48](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0048.md)
  — Contract Interface Specification. Error enums are published in the contract
  spec as `SC_SPEC_ENTRY_UDT_ERROR_ENUM_V0`; this SEP constrains what goes into
  them.

This SEP is the first of two companion conventions. The second, covering event
naming and layout, is a separate proposal (see
[Design Rationale](#design-rationale)).

## Motivation

A Soroban contract error reaches a client as `Error(Contract, #N)`: an
`ScError::Contract(u32)` and nothing else. The code carries no name, no
namespace, no kind, and no indication of which contract raised it. A client can
only turn it into something meaningful if it knows the contract's error enum,
and it can only compare errors across contracts if the codes mean the same
thing.

Today they do not:

- [SEP-41](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)
  says a transfer "fails if the balance is less than `amount`" but defines no
  error. SEP-49, 50, 56 and 57 define none either; the last two say only
  "panic".
- Nothing stops two standards, or two libraries, from each claiming the lowest
  codes. A contract that uses both then has two meanings for `1`, and the
  client cannot tell.
- The
  [Stellar documentation](https://developers.stellar.org/docs/build/guides/conventions/error-enum)
  recommends an `Error` enum with explicit `u32` values but gives no rule for
  choosing them.
- The host already classifies its own failures: `ScErrorType` names the
  subsystem and `ScErrorCode` the class (`InvalidInput`, `MissingValue`,
  `ExceededLimit`, …). Contract errors, which are the ones users meet most
  often, have no such class. A wallet cannot tell "you are not allowed" from
  "the input is wrong" from "try again later" without the contract's spec.

Other ecosystems handle this in two ways. Ethereum
([ERC-6093](https://eips.ethereum.org/EIPS/eip-6093)) names errors
`<Domain><Prefix><Subject>` and needs no numbers, because the selector is a
hash of the name and its arguments. Cosmos pairs a codespace string with a
code. The Move standard library encodes a category in the abort code
(`(category << 16) + reason`, with categories such as `INVALID_ARGUMENT`,
`PERMISSION_DENIED` and `NOT_FOUND`, themselves modeled on gRPC's status
codes). Soroban's error value has room for a single `u32` and no codespace, so
both the namespace and the category have to live inside the number. This SEP
defines that.

## Abstract

This SEP defines (1) naming rules for error enums and their variants, (2) a
decimal layout of the contract error code, `N * 1000 + C * 100 + n`, where `N`
is the number of the SEP that defines the error (`0` for a contract's own
errors), `C` is one of ten categories, and `n` is the reason within them, (3)
the categories, and (4) the rules a contract follows when it raises or receives
an error. It does not define the errors of any particular standard; each
standard defines its own, in its own block.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" are to be
interpreted as described in RFC 2119.

### 1. Naming

1. A contract error enum MUST use `#[contracterror]` with `#[repr(u32)]` and an
   explicit discriminant on every variant. Codes MUST NOT depend on source
   order.
2. An enum defined by a SEP MUST be named `<Domain>Error`, where `<Domain>` is
   a PascalCase noun for the standard (`TokenError`, `NonFungibleError`), so
   that it can sit in the same contract spec as other enums. The enum a
   contract defines for its own conditions SHOULD be named `Error`. An enum
   name MUST be unique within a contract spec.
3. A variant name MUST be PascalCase and SHOULD follow `<Prefix><Subject>`,
   after ERC-6093: the prefix says why it failed and the subject says what
   failed. Common prefixes are `Invalid`, `Insufficient`, `Nonexistent`,
   `Exceeded`, `Already`, `Unsupported` and `Not`. The subject is a party or
   item (`Sender`, `Receiver`, `Balance`, `Allowance`, `Token`), never an
   action. The enum name already gives the domain, so a variant MUST NOT repeat
   it (`InsufficientBalance`, not `TokenInsufficientBalance`).
4. Every variant MUST have a doc comment. SEP-48 carries it into the spec, and
   it is what a client shows when it has nothing better.

### 2. Code layout

A contract error code is the decimal number

```
code = N * 1000 + C * 100 + n
```

| Field | Range      | Meaning                                                                       |
| ----- | ---------- | ----------------------------------------------------------------------------- |
| `N`   | `0`–`9999` | Owner. The number of the SEP that defines the error, or `0` for the contract. |
| `C`   | `0`–`9`    | Category (section 3).                                                         |
| `n`   | `1`–`99`   | Reason, assigned sequentially from `1` within the owner and category.         |

So `code / 1000` is the owner and `(code / 100) % 10` is the category, and a
client can read both from `Error(Contract, #41307)` without a table: owner
SEP-41, category `3` (State), reason `7`.

The code space is used as follows. The host's own errors use other
`ScErrorType` values, so every `u32` is available to `ScErrorType::Contract`.

| Range                     | Meaning                                       |
| ------------------------- | --------------------------------------------- |
| `0`                       | Reserved. MUST NOT be used.                   |
| `1` – `999`               | Contract-local (`N = 0`).                     |
| `1,000` – `9,999,999`     | SEP-defined (`N` = SEP number, `1` – `9999`). |
| `10,000,000` – `u32::MAX` | Reserved for future use. MUST NOT be used.    |

**SEP-defined codes.** A SEP numbered `N` owns the codes
`N * 1000 + C * 100 + n` for every `C` and `n`. The SEP number is already
unique and already arbitrated by the SEP process, so no further registry is
needed. A SEP that defines errors MUST assign every code explicitly in its
specification, with the category and reason it chooses, and MUST NOT use a code
outside its block.

**Contract-local codes.** A contract defines its own errors with `N = 0`, so
they lie in `1` – `999` and follow the same category digit: its own
`Validation` error `3` is `103`. A contract that does not care about categories
MAY use `C = 0` and the codes `1` – `99`, which keeps small contracts valid as
they are. The contract is responsible for not reusing a local code for two
meanings, including when it includes a library that defines codes of its own
(see [Design Rationale](#design-rationale)).

**Stability.** Once a code is released it MUST NOT be reassigned to another
meaning or another category, in a SEP or in a contract. A removed variant keeps
its code, retired. A SEP revision MAY add codes to its block and MUST NOT
change existing ones.

### 3. Categories

| `C` | Name          | The failure is that…                                                                                                |
| --- | ------------- | ------------------------------------------------------------------------------------------------------------------- |
| `0` | `Unspecified` | No category below applies, or the author did not assign one.                                                        |
| `1` | `Validation`  | An argument is malformed or out of its valid range, independent of stored state.                                    |
| `2` | `Access`      | The caller or account is not allowed by the contract's own rules: role, owner, allowance, allowlist, blocklist.     |
| `3` | `State`       | Stored state does not permit the operation: missing, already existing, insufficient balance, paused, uninitialized. |
| `4` | `Timing`      | A ledger sequence, timestamp, TTL or deadline does not allow the operation: expired, not yet valid.                 |
| `5` | `Crypto`      | A signature, proof, hash or key is invalid.                                                                         |
| `6` | `Arithmetic`  | A computation is undefined or inexact: overflow, underflow, division by zero, lost precision.                       |
| `7` | `Limit`       | A cap, quota, count or size limit is reached.                                                                       |
| `8` | `Unsupported` | The contract does not support the operation or option requested.                                                    |
| `9` | `Internal`    | The contract's own invariant is violated. The caller did nothing wrong.                                             |

Where more than one category fits, the most specific wins: `Timing`, `Crypto`,
`Arithmetic` and `Limit` over `Validation` or `State`; `Access` over `State`;
`Validation` over `State` when the condition does not depend on stored state.
`Unspecified` is for use only when none fits. A category is part of the code,
so it is chosen once.

`require_auth` failures are host errors (`ScErrorType::Auth`), not contract
errors, and MUST NOT be redefined as `Access` contract errors. `Access` is for
the contract's own rules on top of authentication.

### 4. Raising errors

1. A function defined by a SEP MUST keep the signature the SEP gives it and
   MUST signal an error with `panic_with_error!`. It MUST NOT change its return
   type to `Result` to carry a SEP error, because clients and other contracts
   call the function by its published signature.
2. When a SEP defines an error for a condition, a contract that detects that
   condition in its own code MUST fail with that code. Where more than one
   condition applies at once, which one is reported is unspecified unless the
   SEP says otherwise.
3. A contract MUST publish every error enum it can raise in its SEP-48 spec,
   including the enums of the SEPs it implements.

### 5. Receiving errors

1. A contract that calls another contract and does not catch the failure lets
   it propagate. The code the client sees is the callee's code.
2. A contract that catches a callee failure (`try_` calls, `InvokeError`) and
   receives `InvokeError::Contract(code)` SHOULD propagate `code` unchanged
   when `code` is SEP-defined (`N >= 1`) and the contract has no more specific
   information to add. It SHOULD NOT map the code into its own local range,
   which would discard the owner and category the code carries.
3. A contract that does translate an error MUST document the mapping in its own
   specification.
4. `InvokeError::Contract` carries a code and no contract address. This SEP
   does not define a wrapper that adds the origin (compare
   [ERC-7751](https://eips.ethereum.org/EIPS/eip-7751)); recovering the origin
   is left to client tooling and may be addressed by a later SEP.

### 6. Client behavior

A client that decodes an error SHOULD look the name up in the contract's SEP-48
spec first. When the code is SEP-defined and the client knows that SEP, it MAY
present the SEP's description for the code. When it has neither, it MAY present
a generic message for the category (for example "not permitted" for `Access`).
A client MUST treat a code above `9,999,999` as a failure of unspecified cause
and MUST NOT infer a meaning from a code outside the owner's block.

### Informative example

SEP-41 could adopt this SEP by amendment and define, in its own block:

```rust
#[contracterror]
#[derive(Copy, Clone, Debug, Eq, PartialEq)]
#[repr(u32)]
pub enum TokenError {
    /// The amount is negative. (`C = 1`, Validation)
    InvalidAmount = 41101,
    /// The allowance of `spender` is less than the amount. (`C = 2`, Access)
    InsufficientAllowance = 41201,
    /// The balance of `from` is less than the amount. (`C = 3`, State)
    InsufficientBalance = 41301,
}
```

The codes and names above are illustrative. The actual error sets belong in the
SEPs that define the interfaces, not in this one.

## Design Rationale

**Why the SEP number is the namespace.** The alternatives were (a) a registry
of blocks maintained in this SEP, (b) a single offset such as Anchor's `6000`
for all user errors, (c) a codespace string as in Cosmos, and (d) leaving each
SEP to pick. A registry duplicates what the SEP process already does, since the
SEP number is a unique, arbitrated identifier, and it would force a registry
amendment on every SEP that defines an error. A single offset separates
framework from user errors but does nothing about two standards colliding
inside the user range. A codespace needs room in the error value that
`ScError::Contract` does not have. Leaving it to each SEP is the current state,
and it lets two standards claim the same low codes. Deriving the block from the
SEP number needs no coordination and is readable by a client without a lookup
table.

**Why decimal digits and not bit fields.** The Move standard library packs
`(category << 16) + reason` into the abort code, which is natural where abort
codes are shown in hex. Soroban tooling shows contract errors in decimal
(`Error(Contract, #N)`), where a bit-packed value is unreadable (`65537` for
category 1, reason 1) and a decimal-digit layout is not (`41307`). Decimal
costs range: 99 reasons per owner and category, and 9,999 owners. Neither
should bind: a standard defines tens of errors, not hundreds, and the SEP
numbers are far from 10,000.

**Why these categories.** The set is deliberately small and taken from three
sources, so it agrees with what clients already know:

| `C` | This SEP      | Move standard library                          | gRPC status                                          | Host `ScErrorCode` / `ScErrorType`                                |
| --- | ------------- | ---------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------- |
| `1` | `Validation`  | `INVALID_ARGUMENT`, `OUT_OF_RANGE`             | `INVALID_ARGUMENT`, `OUT_OF_RANGE`                   | `InvalidInput`, `IndexBounds`, `UnexpectedType`, `UnexpectedSize` |
| `2` | `Access`      | `PERMISSION_DENIED`, `UNAUTHENTICATED`         | `PERMISSION_DENIED`, `UNAUTHENTICATED`               | `ScErrorType::Auth` (host-level, not a contract error)            |
| `3` | `State`       | `INVALID_STATE`, `NOT_FOUND`, `ALREADY_EXISTS` | `FAILED_PRECONDITION`, `NOT_FOUND`, `ALREADY_EXISTS` | `MissingValue`, `ExistingValue`, `InvalidAction`                  |
| `4` | `Timing`      | —                                              | —                                                    | —                                                                 |
| `5` | `Crypto`      | —                                              | —                                                    | `ScErrorType::Crypto`                                             |
| `6` | `Arithmetic`  | —                                              | —                                                    | `ArithDomain` (Solidity: `Panic` `0x11`, `0x12`)                  |
| `7` | `Limit`       | `RESOURCE_EXHAUSTED`                           | `RESOURCE_EXHAUSTED`                                 | `ExceededLimit`                                                   |
| `8` | `Unsupported` | `NOT_IMPLEMENTED`                              | `UNIMPLEMENTED`                                      | —                                                                 |
| `9` | `Internal`    | `INTERNAL`                                     | `INTERNAL`                                           | `InternalError`                                                   |

`Timing` is specific to Soroban, where expiry is by ledger sequence and TTL,
and is common enough in contracts (a stale `live_until_ledger`, an expired
signature) to earn its own category. `Crypto` and `Arithmetic` have no
counterpart among the Move and gRPC categories, but are separate in the host
and in Solidity. The most-specific-wins rule is gRPC's. The Move categories
`ABORTED`, `CANCELLED` and `UNAVAILABLE` describe concurrency and transport and
have no meaning in a transaction that either completes or reverts, so they are
left out.

**Why a contract-local range and no library range.** Libraries that ship error
enums are not standards, and the scheme here is defined by SEPs, not by
whatever libraries have already chosen. Two options were considered for a
non-SEP library: (a) give it no reserved space, so that it documents its codes
and the author of a contract that includes it avoids them, as for any other
dependency; and (b) keep a small registered range. This SEP takes (a). The
natural path for a utility that many contracts depend on, such as an
upgradeability or pausing interface, is to become a SEP and take its own block;
SEP-49 already does this for upgradeable contracts.

**Companion SEP.** Event naming and layout are mostly style and are already
partly written down (SEP-41, SEP-50, the SDK's `#[contractevent]`). The error
convention is different in kind: it fixes reserved codes and states MUSTs that
other SEPs are expected to follow. They are proposed separately so that review
of one does not hold up the other.

**Out of scope.** The error sets of individual standards (those are amendments
to SEP-41, SEP-50 and so on), wrapping errors with their origin, and
storage-key or `#[contracttype]` naming. An SDK helper in the style of Move's
`error::invalid_argument(reason)`, which would make categories ergonomic to
write, is a natural follow-up and not a requirement of this SEP.

## Security Concerns

An error code is data returned by the callee. A malicious contract can return
`41301` without being a token, and can declare any category, so a code MUST NOT
be treated as proof of state or used for a security decision about a contract
that is not otherwise trusted. A wallet that shows a SEP-defined or category
message for a code SHOULD attribute it to the contract that raised it and not
present it as a statement of the ledger. A category is a hint for display; a
client MUST NOT retry, resubmit or escalate a transaction automatically because
of it.

A contract that composes several libraries, or a library with a standard, can
still end up with two local meanings for one code. The layout prevents this
among SEP-defined codes and cannot prevent it among local ones, which is why
the contract author is made responsible for them in section 2.

A released code that is later reassigned, or moved to another category, changes
the meaning of failures in transactions already recorded and in clients already
deployed. Hence the stability rule.

## Changelog

- `v0.0.2`: Add the category digit to the code layout and the category table.
  Remove the library registry and all references to existing libraries; the
  contract-local range is now `1` – `999`. Reword the rationale accordingly.
- `v0.0.1`: Initial draft.
