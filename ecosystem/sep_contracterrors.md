## Preamble

```
SEP: To Be Assigned
Title: Contract Error Conventions
Author: Pamphile Roy <@tupui>
Status: Draft
Created: 2026-10-15
Updated: 2026-10-16
Version: 0.1.0
Discussion: To Be Assigned
```

## Simple Summary

A convention for the errors that Soroban contracts return: a reserved range of
standard error codes with fixed names and meanings (seeded with the codes the
host's built-in contracts already use), a rule that a contract's own codes stay
outside that range, naming rules for error enums, and rules for what a contract
does with an error raised by a contract it calls.

## Dependencies

- [SEP-48](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0048.md)
  — Contract Interface Specification. Error enums are published in the contract
  spec as `SC_SPEC_ENTRY_UDT_ERROR_ENUM_V0`; this SEP constrains what goes into
  them.
- [SEP-41](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md)
  — Soroban Token Interface. The Stellar Asset Contract is its reference
  implementation, and its error codes are the seed of the standard range.
- [SEP-47](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0047.md)
  — Contract Interface Discovery. A contract declares that it follows this SEP
  in its `sep` meta entry.

## Motivation

A Soroban contract error reaches a client as `Error(Contract, #N)`:
`ScError::Contract(u32)` and nothing else. The value carries no name, no
namespace, no kind, and no indication of which contract raised it. Several
facts about the platform make this a problem in practice.

**The host already defines error codes, and they are not written down as a
standard.** The built-in contracts (the Stellar Asset Contract and the built-in
classic-account contract) share one `ContractError` enum in `rs-soroban-env`
with codes `2`–`15`: `Unauthorized` is `4`, `NegativeAmount` is `8`,
`Allowance` is `9`, `Balance` is `10`, `Overflow` is `12`, and so on. These
codes are emitted today, appear in recorded transactions, and cannot change. A
comment in the source says one enum is shared by all built-ins because "it's
not clear how to distinguish them". That is the missing-namespace problem, met
by the host itself. Meanwhile the SDK's token interface documentation says that
the interface "does not define any set of standard errors", and SEP-41, 49, 50,
56 and 57 define none, so a wallet cannot name `#10` for any token other than
by special-casing the Stellar Asset Contract.

**Codes are not namespaced, and they travel.** A plain cross-contract call
returns the callee's error to the caller's caller unchanged. A `try_` call
hands the caller the raw contract code, and the SDK converts it into the
_caller's_ error enum by number alone. Two things follow:

- A contract that calls a token through a `try_` client and defines a variant
  with a code in `1`–`15` of its own relabels the token's failures as its own.
  The SDK's own `errors` example defines `LimitReached = 1`.
- A client that receives `Error(Contract, #10)` from a nested call decodes it
  against the top-level contract's spec, because that is the only spec it has.

**The only namespace is off-chain.** SEP-48 gives every spec entry a `name` and
a `lib`. Neither is in the error value.

Other ecosystems separate the _class_ of an error from the specific error, and
separate platform and standard-library codes from user codes. Solana has
built-in `ProgramError` variants (`InvalidArgument`, `InsufficientFunds`, …)
and `Custom(u32)`. Substrate has `DispatchError` variants (`BadOrigin`,
`Token(..)`, `Arithmetic(..)`) and `Module { index, error }`. CosmWasm has an
`ErrorKind` on its standard error. TON reserves `0`–`127` for the VM,
`128`–`255` for the standard library and `256`–`65535` for developers; Anchor
reserves everything below `6000` for the framework. Ethereum names errors in
the ABI ([ERC-6093](https://eips.ethereum.org/EIPS/eip-6093)) and needs no
numbers. Soroban's error value has room for a single `u32`, so the separation
has to be a partition of that number and a table of what is in it. This SEP
defines both.

## Abstract

This SEP reserves the codes `0`–`99` as the **standard range**, defines the
table of standard errors (names, codes, classes) and how it grows, requires a
contract's own error codes to be `100` or above, gives naming rules for error
enums and variants, and states how a contract and a client treat an error that
came from another contract.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" are to be
interpreted as described in RFC 2119.

### 1. Code ranges

The host's own errors use other `ScErrorType` values, so every `u32` is
available to `ScErrorType::Contract`. This SEP partitions it as follows.

| Range              | Meaning                                                   |
| ------------------ | --------------------------------------------------------- |
| `0` – `99`         | Standard range. Defined by the table in section 2.        |
| `100` – `u32::MAX` | Contract range. Defined by each contract in its own spec. |

A contract MUST NOT use a code in the standard range for a meaning other than
the one the table gives it, and MUST NOT define a code in the standard range
that the table does not list. A code in the standard range that the table does
not list is reserved.

A standard code means the same thing in every contract. A contract code means
something only relative to the contract that raised it.

### 2. Standard errors

Each standard error has a code, a canonical name, and a class. The class is
metadata of the table; it is not encoded in the number.

| Class         | The failure is that…                                                             |
| ------------- | -------------------------------------------------------------------------------- |
| `Validation`  | An argument is malformed or out of its valid range, independent of stored state. |
| `Access`      | The caller or account is not allowed by the contract's own rules.                |
| `State`       | Stored state does not permit the operation.                                      |
| `Timing`      | A ledger sequence, timestamp, TTL or deadline does not allow the operation.      |
| `Crypto`      | A signature, proof, hash or key is invalid.                                      |
| `Arithmetic`  | A computation is undefined or inexact.                                           |
| `Limit`       | A cap, quota, count or size limit is reached.                                    |
| `Unsupported` | The contract does not support the operation or option requested.                 |
| `Internal`    | The contract's own invariant is violated; the caller did nothing wrong.          |

Where more than one class fits, the most specific wins: `Timing`, `Crypto`,
`Arithmetic` and `Limit` over `Validation` or `State`; `Access` over `State`.

**Seed.** Codes `1`–`15` are those of the host's built-in `ContractError`. They
keep their codes; this SEP gives them canonical names and classes. Code `0` is
never assigned.

| Code | Name                         | Class         | Raised when…                                                                | Host name                    |
| ---- | ---------------------------- | ------------- | --------------------------------------------------------------------------- | ---------------------------- |
| `0`  | —                            | —             | Never assigned.                                                             | —                            |
| `1`  | —                            | —             | Never assigned (the host retired its internal error).                       | —                            |
| `2`  | `UnsupportedOperation`       | `Unsupported` | The contract does not support the operation.                                | `OperationNotSupportedError` |
| `3`  | `AlreadyInitialized`         | `State`       | The contract or token is already initialized.                               | `AlreadyInitializedError`    |
| `4`  | `Unauthorized`               | `Access`      | The caller is not allowed to perform the operation.                         | `UnauthorizedError`          |
| `5`  | `AuthenticationFailed`       | `Access`      | An account's signatures do not satisfy its signers and thresholds.          | `AuthenticationError`        |
| `6`  | `AccountMissing`             | `State`       | A required account does not exist.                                          | `AccountMissingError`        |
| `7`  | `AccountNotClassic`          | `Validation`  | An address that must be a classic account is not.                           | `AccountIsNotClassic`        |
| `8`  | `NegativeAmount`             | `Validation`  | An amount is negative.                                                      | `NegativeAmountError`        |
| `9`  | `InvalidAllowance`           | `State`       | The allowance is less than the amount, or its expiration ledger is invalid. | `AllowanceError`             |
| `10` | `InsufficientBalance`        | `State`       | The balance is less than the amount.                                        | `BalanceError`               |
| `11` | `BalanceDeauthorized`        | `Access`      | The balance is not authorized to move.                                      | `BalanceDeauthorizedError`   |
| `12` | `Overflow`                   | `Arithmetic`  | An amount or balance would overflow.                                        | `OverflowError`              |
| `13` | `TrustlineMissing`           | `State`       | A required trustline does not exist.                                        | `TrustlineMissingError`      |
| `14` | `InsufficientAccountReserve` | `State`       | The account would fall below its minimum reserve.                           | `InsufficientAccountReserve` |
| `15` | `TooManySubentries`          | `Limit`       | The account would exceed its subentry limit.                                | `TooManyAccountSubentries`   |

**Additions.** Codes `16` onward cover the conditions that recur across
deployed Soroban contracts and that the host does not already cover (see
[Design Rationale](#design-rationale) for the survey they come from).

| Code | Name              | Class        | Raised when…                                                     |
| ---- | ----------------- | ------------ | ---------------------------------------------------------------- |
| `16` | `NotInitialized`  | `State`      | The contract has not been initialized.                           |
| `17` | `NotFound`        | `State`      | A requested entry does not exist.                                |
| `18` | `AlreadyExists`   | `State`      | An entry to be created already exists.                           |
| `19` | `InvalidAmount`   | `Validation` | An amount is zero or outside the range the operation accepts.    |
| `20` | `InvalidArgument` | `Validation` | An argument is invalid and no more specific code fits.           |
| `21` | `Expired`         | `Timing`     | A deadline, expiration ledger or TTL has passed.                 |
| `22` | `Paused`          | `State`      | The contract or operation is paused.                             |
| `23` | `NotPaused`       | `State`      | The operation requires the contract to be paused, and it is not. |
| `24` | `InvalidProof`    | `Crypto`     | A proof does not verify.                                         |
| `25` | `InternalError`   | `Internal`   | An invariant of the contract is violated.                        |

**Growth.** A standard code is added by amendment to this SEP, assigned the
next free code, with a canonical name and a class. A condition qualifies when
it recurs across independent contracts; one standard's specific condition does
not. A code is never reassigned and never moved to another class.

**Use by standards.** A SEP that defines a function whose failure is covered by
a standard error MUST use that code. SEP-41 token implementations MUST use `10`
for a balance that is too low and `9` for an allowance that is too low, as the
Stellar Asset Contract does. A SEP that needs a condition the table lacks adds
it to the table by amendment; it MUST NOT define a new meaning for a code in
the standard range.

### 3. Contract errors

1. A contract's own error codes MUST be `100` or above. A contract MUST NOT
   define a local meaning for a code in `0`–`99`.
2. When a condition the contract detects is in the standard table, the contract
   MUST use the standard code and name, not define a local duplicate. A
   contract MAY declare the standard errors it raises as variants of its own
   error enum (`InsufficientBalance = 10`), or in a separate enum; either way
   the name and code MUST match the table.
3. A contract error enum MUST use `#[contracterror]` with `#[repr(u32)]` and an
   explicit discriminant on every variant. Codes MUST NOT depend on source
   order.
4. An error enum a contract defines for its own conditions SHOULD be named
   `Error`. An enum name MUST be unique within a contract spec, so an enum
   shared across contracts SHOULD be named `<Domain>Error`.
5. A variant name MUST be PascalCase and SHOULD follow `<Prefix><Subject>`,
   after ERC-6093: the prefix says why it failed (`Invalid`, `Insufficient`,
   `Nonexistent`, `Exceeded`, `Already`, `Unsupported`, `Not`) and the subject
   says what failed (`Sender`, `Balance`, `Allowance`, `Token`), never an
   action. The enum name already gives the domain, so a variant MUST NOT repeat
   it.
6. Every variant SHOULD have a doc comment. SEP-48 carries it into the spec,
   and it is what a client shows when it has nothing better.
7. A function defined by a SEP SHOULD signal an error with `panic_with_error!`
   rather than change its return type to `Result`, so that its spec entry stays
   identical to the one the SEP gives.
8. A contract that follows this SEP MUST list it in its SEP-47 `sep` meta
   entry. Clients use that entry to decide whether the standard table applies
   (section 5).
9. `require_auth` failures are host errors (`ScErrorType::Auth`) and MUST NOT
   be redefined as contract errors. `Unauthorized` and `AuthenticationFailed`
   are for the contract's own checks on top of authorization.
10. Once released, a contract code MUST NOT be reassigned to another meaning. A
    removed variant keeps its code, retired.

### 4. Errors from other contracts

This section describes the platform behavior it relies on, then the rules.

**Platform behavior.** A cross-contract `call` that fails returns the callee's
error to the caller's caller unchanged. A `try_call` that fails with a contract
error hands the caller the raw code; any other recoverable host error is
reduced to one generic error (`Context`, `InvalidAction`), and unrecoverable
ones abort. The SDK's `TryFrom` for an error enum matches the bare number, and
accepts it if the enum has a variant with that number, whichever contract
raised it. The originating contract is recorded only in diagnostic events.

**Rules.**

1. A contract that catches a failure from another contract MUST NOT convert a
   contract-range code into one of its own enum's variants by number, unless
   that enum is the callee's (for instance the one generated from the callee's
   spec). Doing so relabels the callee's failure. Converting a standard-range
   code by number is safe, since its meaning does not depend on the contract.
2. A contract that catches a failure and re-raises SHOULD pass a standard-range
   code through unchanged, and SHOULD map a contract-range code to a code of
   its own, since the callee's code has no meaning outside the callee.
3. A contract that does not catch the failure lets the callee's error
   propagate, and its clients see the callee's code. Because conforming
   contracts keep their own codes out of the standard range, a standard-range
   code raised by a conforming contract is never ambiguous, wherever it
   surfaces.

### 5. Client behavior

1. A client SHOULD first look the code up in the SEP-48 spec of the contract it
   invoked.
2. When the spec has no variant with that code, the code is in `0`–`99`, and
   the invoked contract lists this SEP in its SEP-47 `sep` meta entry, the
   client MAY decode it against the standard table and present the canonical
   name or a message for the class. A contract that does not declare this SEP
   may use the standard range for its own meanings, so the table MUST NOT be
   applied to it.
3. When the failure came from a nested call, only diagnostic events identify
   the contract that raised it. A client without them MUST NOT present a name
   decoded from a contract-range code as certain.
4. A client MUST treat a code it cannot resolve as a failure of unspecified
   cause, and MUST NOT infer a meaning from the number.

## Design Rationale

**Why a reserved range and a table, not structure inside the number.** The
platform does not enforce anything about the number, and the host's own codes
mix classes (`Access` is `4` and `5`, `State` is `3`, `6`, `10` and `13`). A
decimal-digit or bit-packed layout (Move packs `(category << 16) + reason`)
would not coexist with those fixed codes and would be advisory for everything
else. The usual solution elsewhere is the one used here: the class is a
separate attribute (a variant, an `ErrorKind`, a table column) and the
platform's own codes live in a reserved layer below the user's. TON and Anchor
draw the line with a numeric boundary; Solana, Substrate and CosmWasm with
typed variants. Soroban can only do the first, and the table supplies the
second.

**Why seed from the host.** The codes `2`–`15` are in production, emitted by
the reference SEP-41 implementation, and cannot change. A standard that put its
own numbers there would disagree with every Stellar Asset Contract transaction
already on the ledger. Adopting them makes every SEP-41 token decode like the
Stellar Asset Contract, with no wallet special case.

**Why one table across standards.** ERC-6093 prefixes every error with its
standard (`ERC20InsufficientBalance`, `ERC721InsufficientApproval`,
`ERC1155InsufficientBalance`), because two Solidity errors with the same name
and different parameters cannot be declared together. A Soroban contract error
carries no parameters, so one code can serve every standard: an insufficient
balance is `10` in a fungible token, a vault or a regulated token alike.

**Why not Solana's model.** Solana keeps the runtime's errors and program
errors disjoint by bit range (built-in `ProgramError` values occupy the upper
32 bits, `Custom(u32)` the lower), and reserves nothing between programs. SPL
Token's `TokenError` starts at `0` (`InsufficientFunds` is `1`, `Overflow` is
`14`, `AccountFrozen` is `17`), and a wallet can read those codes because every
SPL token is the same program. A Soroban token is its own contract, so the role
of the shared token program falls to a standard, and the standard has to fix
the codes, as ERC-6093 fixes the names on Ethereum.

**Prior practice on Soroban.** The error enums of 38 public Soroban projects
(194 enums, 1,942 variants) were surveyed for this SEP. Three habits dominate:
numbering from `0` or `1` (103 enums), one block of `100` per contract or
module (Phoenix, Soroswap, Aquarius and others), and copying the host's codes.
Outside that last group, no condition has the same code in two independent
projects; code `1` alone means `Unauthorized`, `AlreadyInitialized`, an
internal error or `NotInitialized` depending on the project. The group that
copies the host is the precedent for this SEP: Blend and the Script3 contracts
(governor, fee vault) use the host's `1`–`12` for the same conditions and put
their own codes at `100` or above. The conditions that recur most across
projects are `AlreadyInitialized` (22 projects), `NotFound` (20), `Overflow`
(19), `InsufficientBalance` (18), `Unauthorized` (17), `NegativeAmount` (17),
`AlreadyExists` (14), `Expired` (14), `InvalidAmount` (13) and `NotInitialized`
(12). The table covers each of them.

**Why the contract range starts at `100`.** A code a contract defines in the
standard range is indistinguishable from a standard code that arrived through a
nested call, so the range has to be reserved. `100` is the boundary existing
code already respects most: 13 of the 38 surveyed projects use no code below
`100` other than the host's own, against 6 for a boundary at `1000`. A hundred
codes leave room for the table, which holds 25 entries for conditions that
recur across all of those projects. Contracts that use `1`, `2`, `3` for their
own errors keep working; they do not conform, they do not declare this SEP, and
clients do not apply the table to them.

**Why no per-SEP numeric blocks.** Giving each SEP a block (for instance
`SEP number * 1000`) would namespace SEP-defined errors, but nothing enforces
it: the SDK accepts any number, and a contract can define any code. A block
that cannot be enforced adds a rule without adding a guarantee. Conditions that
are common go into the standard table; conditions that are specific to one
standard stay in that contract's own range, where SEP-48 names them.

**Why classes are not in the number.** No consumer needs to derive the class
from the number: a client that knows the standard table has the class, and a
client that does not has nothing to derive it from for contract codes either
way. Putting it in the number would also fix the class of the host's existing
codes by position, which they do not respect.

**Classes.** The set is small and taken from sources clients already know:

| Class         | Move standard library                          | gRPC status                                          | Host `ScErrorCode` / `ScErrorType`                                |
| ------------- | ---------------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------- |
| `Validation`  | `INVALID_ARGUMENT`, `OUT_OF_RANGE`             | `INVALID_ARGUMENT`, `OUT_OF_RANGE`                   | `InvalidInput`, `IndexBounds`, `UnexpectedType`, `UnexpectedSize` |
| `Access`      | `PERMISSION_DENIED`, `UNAUTHENTICATED`         | `PERMISSION_DENIED`, `UNAUTHENTICATED`               | `ScErrorType::Auth` (host-level)                                  |
| `State`       | `INVALID_STATE`, `NOT_FOUND`, `ALREADY_EXISTS` | `FAILED_PRECONDITION`, `NOT_FOUND`, `ALREADY_EXISTS` | `MissingValue`, `ExistingValue`, `InvalidAction`                  |
| `Timing`      | —                                              | —                                                    | —                                                                 |
| `Crypto`      | —                                              | —                                                    | `ScErrorType::Crypto`                                             |
| `Arithmetic`  | —                                              | —                                                    | `ArithDomain` (Solidity `Panic` `0x11`, `0x12`)                   |
| `Limit`       | `RESOURCE_EXHAUSTED`                           | `RESOURCE_EXHAUSTED`                                 | `ExceededLimit`                                                   |
| `Unsupported` | `NOT_IMPLEMENTED`                              | `UNIMPLEMENTED`                                      | —                                                                 |
| `Internal`    | `INTERNAL`                                     | `INTERNAL`                                           | `InternalError`                                                   |

`Timing` is specific to Soroban, where expiry is by ledger sequence and TTL.
The Move categories `ABORTED`, `CANCELLED` and `UNAVAILABLE` describe
concurrency and transport and have no meaning in a transaction that either
completes or reverts.

**Out of scope.** Wrapping errors with their origin (compare
[ERC-7751](https://eips.ethereum.org/EIPS/eip-7751)).

## Security Concerns

An error code is data returned by the callee. A malicious contract can return
`10` without being a token, so a code MUST NOT be treated as proof of state or
used for a security decision about a contract that is not otherwise trusted. A
wallet that shows a name or a class for a code SHOULD attribute it to the
contract that raised it, and a class is a hint for display: a client MUST NOT
retry, resubmit or escalate a transaction automatically because of it.

Contract-range codes remain ambiguous when they propagate: two contracts can
use the same code for different things, and a client without diagnostic events
cannot tell which raised it. Section 4 reduces the chance that a contract
creates this ambiguity itself; it cannot remove it.

A released code that is later reassigned, or moved to another class, changes
the meaning of failures in transactions already recorded and in clients already
deployed. Hence the stability rules.

## Changelog

- `v0.1.0`: Initial draft.
