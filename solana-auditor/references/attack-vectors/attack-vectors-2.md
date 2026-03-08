# Attack Vectors Reference — PDA, CPI & Cross-Program Security (2/4)

> Part 2 of 4 · Vectors 31–60 of 120 total
> Covers: PDA derivation, seed security, CPI safety, invoke_signed, signer escalation, program validation, stale data

Each vector follows the format:
- **D:** Description — what makes it exploitable
- **FP:** False-positive conditions — mitigations that would make it NOT a finding

---

**31. Non-Canonical Bump Seed**

- **D:** PDA created or verified using `create_program_address` with a user-supplied bump instead of the canonical bump from `find_program_address`. Multiple valid bumps exist for the same seeds — attacker can derive alternate PDAs, fragmenting state or bypassing intended PDA checks.
- **FP:** Anchor `seeds` + `bump` constraint used (automatic canonical bump). Native: `find_program_address` used for derivation. Canonical bump stored in account data and reused.

**32. PDA Sharing — Missing User-Specific Seed**

- **D:** PDA seeds lack a user-specific component (e.g., `user.key()`). All users share the same PDA, meaning one user's action can affect another user's state. Classic example: a global vault PDA instead of per-user vaults.
- **FP:** Seeds include `user.key().as_ref()` or equivalent unique identifier. Design explicitly requires a shared/global account (e.g., protocol config). Access control prevents unauthorized state changes on shared accounts.

**33. Seed Concatenation Collision**

- **D:** PDA seeds constructed from variable-length user inputs without delimiters or fixed-length encoding. Seeds `["AB", "C"]` and `["A", "BC"]` produce the same PDA — attacker finds a collision to access another user's account.
- **FP:** All seeds are fixed-length (pubkeys, u8 arrays). Variable-length seeds use canonical delimiters or length prefixes. Seeds are hashed before use.

**34. Seed Collision Across Account Types**

- **D:** Different account types (vault, escrow, config) use seeds with no unique type prefix. An attacker's "vault" PDA collides with a legitimate "escrow" PDA, enabling cross-type access.
- **FP:** Unique string prefixes per type: `b"vault"`, `b"escrow"`, `b"config"`. Seeds include account type discriminator.

**35. PDA Purpose Isolation Violation**

- **D:** Single PDA used across multiple logical domains or external programs. If one domain is compromised, the shared PDA grants access to all domains.
- **FP:** Each distinct capability (vault, escrow, staking) uses a distinct PDA with distinct seeds. Program-specific seeds prevent cross-program PDA reuse.

**36. Exposed PDA Seeds — User-Controllable Derivation**

- **D:** PDA seeds entirely composed of user-controlled inputs without any program-controlled components. Attacker can pre-compute and pre-initialize PDAs to front-run legitimate users.
- **FP:** Seeds include program-controlled values (authority pubkey, protocol nonce). `init` constraint prevents re-initialization. Seeds include the payer's key.

**37. Forced Seed De-Bump**

- **D:** Program accepts a bump seed from the user and doesn't verify it's canonical. Attacker provides bump = 0 or a non-canonical bump, causing `create_program_address` to fail or derive a different address than expected.
- **FP:** Bump stored on-chain and reused. Anchor `bump` constraint validates canonical bump automatically. `find_program_address` used to derive and verify.

**38. Arbitrary CPI — Unvalidated Program ID**

- **D:** `invoke()` or `invoke_signed()` called with a program ID from an unvalidated `AccountInfo`. Attacker passes a malicious program that mimics the expected interface — returns success without performing the operation, or performs a different operation (e.g., reverse transfer).
- **FP:** Anchor `Program<'info, Token>` type used (automatic validation). Native: `require_keys_eq!(program.key(), expected_program::ID)`. Program ID hardcoded in CPI call.

**39. CPI Without Signer Seeds — invoke vs invoke_signed Confusion**

- **D:** `invoke()` used where `invoke_signed()` is required because a PDA needs to sign the CPI. The CPI fails silently or panics. Conversely, `invoke_signed()` used unnecessarily, potentially escalating signer privileges.
- **FP:** PDA signer correctly identified — `invoke_signed` used with proper seeds. Non-PDA CPI correctly uses `invoke`. Anchor CPI context correctly constructed.

**40. Signer Pass-Through in CPI**

- **D:** All accounts from the current instruction passed into a CPI call without filtering. Signer accounts retain their signer privilege in the CPI — a malicious callee can use the signer authority to perform unauthorized actions.
- **FP:** Only necessary accounts passed to CPI. Signer accounts explicitly filtered out unless required. Account isolation via user-specific PDAs limits blast radius.

**41. SOL Balance Drain via CPI**

- **D:** Signer account passed to an external CPI. The callee program can spend SOL from the signer (Solana has no `msg.value` equivalent — any signer can be drained). No balance check before/after CPI.
- **FP:** `signer.lamports()` recorded before CPI, verified after: `balance_before <= balance_after + max_spendable`. CPI target is a trusted, verified program. PDA used instead of user signer for CPI authority.

**42. Post-CPI Ownership Change**

- **D:** An attacker-controlled program uses the `assign` instruction during CPI to change an account's owner. After the CPI returns, the account is no longer owned by the expected program, but the caller doesn't re-verify.
- **FP:** Account owner re-checked after CPI: `account.owner == expected_program`. CPI target is a trusted program (SPL Token, System Program). Account is a PDA owned by the calling program.

**43. Stale Data After CPI — Missing reload()**

- **D:** Account data deserialized before a CPI, then used after the CPI without calling `reload()`. The CPI may have modified the on-chain state, but the in-memory struct still holds the pre-CPI values. Decisions based on stale balances enable double-spends or over-withdrawals.
- **FP:** `ctx.accounts.account.reload()?` called after every CPI that modifies shared accounts. Account only read after all CPIs complete. No CPI modifies the account in question.

**44. CPI Return Value Ignored**

- **D:** CPI invocation result not propagated with `?`. If the inner call fails, the outer instruction continues executing with an inconsistent state (e.g., transfer failed but balance decremented).
- **FP:** All CPI calls wrapped with `?` operator. Anchor CPI helpers (`token::transfer(ctx, amount)?`) used. Native: `invoke(...)?.` or explicit match on result.

**45. Unnecessary Accounts Passed to CPI**

- **D:** More accounts than needed passed to a CPI call. The callee gains read/write access to accounts it shouldn't touch, expanding the attack surface.
- **FP:** CPI account lists contain only the minimum required accounts. Anchor CPI context structs enforce exact account requirements.

**46. invoke_signed with Incorrect Seeds**

- **D:** `invoke_signed` called with wrong seeds or wrong bump, causing PDA signature to fail. In some cases, the wrong seeds derive a valid but unintended PDA, causing the wrong account to sign.
- **FP:** Seeds match the PDA derivation exactly. Stored canonical bump used. Anchor `CpiContext::new_with_signer` with correct signer seeds.

**47. CPI to System Program — Unintended Account Creation**

- **D:** CPI to System Program's `create_account` or `transfer` without verifying the destination account. Attacker forces creation of accounts at unexpected addresses or redirects SOL transfers.
- **FP:** Destination account validated as expected PDA or known address. Account creation uses PDA seeds (deterministic). Transfer destination is a protocol-controlled account.

**48. Missing Program ID Check on Token Program**

- **D:** Token operations (transfer, mint, burn) performed via CPI without distinguishing between SPL Token (legacy) and Token-2022. Using the wrong program ID causes silent failures or fund loss when Token-2022 mints are involved.
- **FP:** Anchor `Interface<'info, TokenInterface>` or `InterfaceAccount` used. Native: dynamic program ID detection based on mint owner. `transfer_checked` used with correct program.

**49. Token-2022 Incompatibility — Legacy transfer Used**

- **D:** `anchor_spl::token::transfer` (or native equivalent) hardcodes the legacy Token Program ID. When used with Token-2022 mints, the CPI fails or misbehaves, causing DoS or fund loss.
- **FP:** `transfer_checked` used for all token operations. `InterfaceAccount` types detect correct program. Mint and decimals provided in transfer (required by `transfer_checked`).

**50. Token-2022 Transfer Hook Not Accounted For**

- **D:** Token-2022 mint has a transfer hook extension, but the program doesn't pass the required extra accounts for the hook CPI. Transfer silently fails or reverts.
- **FP:** Transfer hook accounts resolved and passed via `remaining_accounts`. Program checks for transfer hook extension on the mint. Only legacy tokens supported (documented and enforced).

**51. CPI Privilege Escalation via invoke_signed**

- **D:** `invoke_signed` extends signer privileges to accounts that shouldn't be signers. Attacker exploits the elevated privilege to authorize operations on accounts they don't control.
- **FP:** Only PDA accounts given signer privilege via `invoke_signed`. Non-PDA accounts explicitly not included in signer seeds. Minimum necessary privileges granted.

**52. Missing CPI Program ID Validation in Anchor**

- **D:** Anchor instruction uses `/// CHECK:` on a program account instead of `Program<'info, T>`. The program ID is never validated, enabling arbitrary CPI.
- **FP:** `Program<'info, Token>` or equivalent typed program account used. Manual `require_keys_eq!` check on program key. `address` constraint on the program account.

**53. Cross-Program Reentrancy via CPI Callback**

- **D:** Program makes a CPI to an external program that calls back into the original program before the first call completes. State is partially updated — the callback sees inconsistent state and can exploit it.
- **FP:** State fully updated before any CPI (checks-effects-interactions pattern). Reentrancy guard flag set before CPI, checked on entry. No external CPI to untrusted programs.

**54. PDA Bump Not Stored — Recomputation Cost**

- **D:** Canonical bump not stored in account data, forcing `find_program_address` on every instruction. This wastes ~2000 CU per call. While not a security vulnerability itself, it can push complex transactions over the compute budget, causing DoS.
- **FP:** Bump stored as `pub bump: u8` in account struct. `create_program_address` used with stored bump for re-derivation. Compute budget adequate for the operation.

**55. PDA Used as Signer Without Ownership Verification**

- **D:** PDA derived from user-controlled seeds used as a signer, but the program doesn't verify it owns the PDA. Attacker derives a PDA that belongs to their malicious program and uses it to sign unauthorized CPIs.
- **FP:** PDA verified as owned by the current program before use as signer. Seeds include program-specific constants. Anchor `seeds` constraint verifies PDA derivation.

**56. CPI to Upgradeable Program Without Freeze Check**

- **D:** CPI target is an upgradeable program that could be maliciously upgraded between the time the call is validated and executed. An upgrade changes the program's behavior, potentially converting a safe CPI into a malicious one.
- **FP:** CPI target is a non-upgradeable (frozen) program. Program upgrade authority validated as trusted. Immutable programs (SPL Token, System Program) used.

**57. Address Lookup Table Contains Signer**

- **D:** Signer pubkey included in an Address Lookup Table (ALT). Signer pubkeys must always be inline in the transaction — inclusion in ALT breaks transaction signing validation.
- **FP:** Only non-signer accounts in ALT. Signer accounts always included directly in transaction. ALT not used.

**58. Durable Nonce Not First Instruction**

- **D:** `AdvanceNonceAccount` instruction placed after other instructions in a transaction using durable nonces. The nonce doesn't advance, potentially allowing transaction replay.
- **FP:** `AdvanceNonceAccount` is the first instruction. Durable nonces not used. Transaction uses recent blockhash instead.

**59. Cross-Program State Desync**

- **D:** Program reads state from an external program's account, caches it, then makes decisions based on the cached value. Between the read and the decision, another instruction modifies the external state. The program acts on stale cross-program data.
- **FP:** State read and decision in the same instruction with no intervening CPI. External state re-read after any CPI. Atomic transaction design ensures consistency.

**60. Unvalidated remaining_accounts Used in CPI**

- **D:** `remaining_accounts` passed directly into a CPI call without validation. Attacker injects malicious accounts into the remaining accounts list, which the CPI callee processes as legitimate.
- **FP:** Each remaining account validated (owner, key, type) before CPI. remaining_accounts not passed to CPI. CPI uses only named, validated accounts.
