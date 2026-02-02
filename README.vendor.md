# Certora Vendor Patch — Summary of Changes

This document summarizes all modifications made to Anchor by the Certora patch for formal verification. Enable the patch by building with the **`certora`** feature (e.g. `cargo build --features certora`).

---

## 1. Feature flag

- **`lang/Cargo.toml`**: New feature `certora` that enables:
  - `anchor-attribute-event/certora`
  - `anchor-attribute-error/certora`
- **`lang/attribute/error/Cargo.toml`**: New feature `certora` (no deps).
- **`lang/attribute/event/Cargo.toml`**: New feature `certora` (no deps).
- **`spl/Cargo.toml`**: New feature `certora` (no deps).

---

## 2. Error attribute (`lang/attribute/error`)

- **`lang/attribute/error/src/lib.rs`**
  - Default `#[error]` macro (when **not** `certora`): unchanged; creates errors with source tracking.
  - When **`certora`** is enabled: `#[error]` creates errors with **source tracking disabled** (`create_error(error_code, false, None)` instead of `true`), simplifying the error type for the verifier.

---

## 3. Event attribute (`lang/attribute/event`)

- **`lang/attribute/event/src/lib.rs`**
  - Default `emit!` (when **not** `certora`): unchanged; emits the event.
  - When **`certora`** is enabled: **`emit!` is a no-op** — it expands to `()` so event emission is skipped during verification.

---

## 4. Require macros (`lang/src/lib.rs`)

When **`certora`** is enabled (via the alternate error module and macro behavior), all of the following **panic on failure** instead of returning `Err(...)`:

- `require!`
- `require_eq!`
- `require_neq!`
- `require_keys_eq!`
- `require_keys_neq!`
- `require_gt!`
- `require_gte!`

So failed invariants are modeled as panics for the verifier rather than as `Result` errors.

---

## 5. Error module swap (`lang/src/lib.rs` + `lang/src/error+certora.rs`)

- **`lang/src/lib.rs`**
  - `pub mod error` is conditionally compiled: with **`certora`** it uses **`error+certora.rs`** instead of `error.rs` (`#[cfg_attr(feature = "certora", path = "error+certora.rs")]`).

- **`lang/src/error+certora.rs`** (Certora-only error implementation)
  - **Unboxed `Error`**: `Error::AnchorError` and `Error::ProgramError` hold **by value** (`AnchorError`, `ProgramErrorWithOrigin`) instead of `Box<...>` to avoid heap and simplify verification.
  - **Reduced reporting**:
    - `Display` for `Error`, `ProgramErrorWithOrigin`, and related types is a no-op (no string output).
    - `log()` on `ProgramErrorWithOrigin` and `AnchorError` is a no-op.
    - `with_account_name`, `with_source`, `with_pubkeys`, `with_values` on `Error` (and analogous methods elsewhere) are no-ops and just return `self`.

---

## 6. Account and program types (for mocking / verification)

- **`lang/src/accounts/account.rs`**
  - **`Account<T>`**: Fields `account` and `info` are made **`pub`** so the verifier or tests can construct/mock account data.

- **`lang/src/accounts/interface_account.rs`**
  - When **`certora`** is enabled: new **`InterfaceAccount::new_unchecked(account: Account<'a, T>)`** to build an `InterfaceAccount` from an `Account` without full validation (for mocking).

- **`lang/src/accounts/program.rs`**
  - **`Program<T>`**: Fields `info` and `_phantom` are made **`pub`** for the same mocking/verification purposes.

---

## 7. SPL token types (`spl`)

- **`spl/src/token_interface.rs`**
  - When **`certora`** is enabled:
    - **`TokenAccount::new_unchecked(inner: spl_token_2022::state::Account)`** — construct a `TokenAccount` from raw state for verification.
    - **`Mint::new_unchecked(inner: spl_token_2022::state::Mint)`** — construct a `Mint` from raw state for verification.

---

## Summary table

| Area              | Change |
|-------------------|--------|
| Features          | `certora` feature in `lang`, `anchor-attribute-error`, `anchor-attribute-event`, `spl`. |
| `#[error]`        | With `certora`: no source tracking. |
| `emit!`           | With `certora`: no-op. |
| Require macros    | With `certora`: panic instead of `Err`. |
| Error module      | With `certora`: `error+certora.rs` — unboxed `Error`, no-op Display/log/with_*. |
| `Account<T>`      | `account` and `info` made `pub`. |
| `Program<T>`      | `info` and `_phantom` made `pub`. |
| `InterfaceAccount`| With `certora`: `new_unchecked(Account)`. |
| SPL               | With `certora`: `TokenAccount::new_unchecked`, `Mint::new_unchecked`. |
