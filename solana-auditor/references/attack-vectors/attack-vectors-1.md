# Attack Vectors Reference — Account Validation & Authorization (1/4)

> Part 1 of 4 · Vectors 1–30 of 120 total
> Covers: signer checks, ownership, discriminators, account constraints, data matching, writable flags, reinitialization

Each vector follows the format:
- **D:** Description — what makes it exploitable
- **FP:** False-positive conditions — mitigations that would make it NOT a finding

---

**1. Missing Signer Check**

- **D:** Authority account used in a privileged instruction (withdraw, transfer, admin update) without verifying `is_signer`. Any account address can be passed without a signature, allowing unauthorized execution.
- **FP:** Anchor `Signer<'info>` type used. Native: explicit `if !account.is_signer` check present. Pinocchio: `is_signer()` validated.

**2. Missing Owner Check on Deserialized Account**

- **D:** Account data deserialized via `try_from_slice` or manual parsing without first checking `account.owner == expected_program_id`. Attacker crafts a fake account with identical data layout owned by a malicious program, spoofing balances or authorities.
- **FP:** Anchor `Account<'info, T>` used (automatic owner check). Native: explicit owner comparison before deserialization. Pinocchio: `is_owned_by()` check present.

**3. Type Cosplay — Missing Discriminator Check**

- **D:** Account struct deserialized without verifying its 8-byte discriminator. Attacker passes an `Admin` account where a `User` account is expected (or vice versa), bypassing privilege checks because the data layouts partially overlap.
- **FP:** Anchor `Account<'info, T>` or `#[account]` macro used (automatic discriminator). Native: manual discriminator byte check at offset 0. Zero-copy: `AccountLoader<'info, T>` with discriminator validation.

**4. Reinitialization Attack**

- **D:** Initialization instruction can be called on an already-initialized account, overwriting the authority field. Attacker reinitializes to become the new owner and drains controlled assets.
- **FP:** Anchor `init` constraint used (prevents reinit by checking discriminator + owner). Native: explicit `is_initialized` flag checked before setup logic. `init_if_needed` used with additional validation of existing state.

**5. init_if_needed Without State Validation**

- **D:** `init_if_needed` constraint used without checking existing account state when the account already exists. Attacker pre-initializes the account with harmful state (e.g., wrong authority) before the legitimate user calls the instruction.
- **FP:** Code validates existing data fields (owner, authority) when account already exists. `init` used instead of `init_if_needed`. Frontrunning protection via PDA seeds that include the payer's key.

**6. Missing has_one Constraint — Data Mismatch**

- **D:** Account relationship not validated — e.g., `vault.authority != signer.key()` not enforced. Attacker passes a vault they control instead of the victim's vault, or passes a mismatched token account.
- **FP:** Anchor `has_one = authority` constraint present. Native: manual pubkey comparison between stored field and provided account. Constraint logic validates cross-account relationships.

**7. Missing Writable Check**

- **D:** Account modified without being marked as writable in the transaction. In older runtime versions, this could cause silent state corruption. Even in newer versions, missing writable annotation in Anchor's `#[account(mut)]` means the framework won't serialize changes back.
- **FP:** Anchor `#[account(mut)]` present on all modified accounts. Native: explicit `is_writable` check before mutation.

**8. UncheckedAccount Without Manual Validation**

- **D:** `UncheckedAccount<'info>` or raw `AccountInfo<'info>` used without any manual ownership, signer, or discriminator checks. This is the most permissive account type — attacker can pass any account.
- **FP:** Manual checks present after `/// CHECK:` comment (owner, signer, key comparison, data validation). Account only used for reading lamport balance or key comparison (no deserialization).

**9. remaining_accounts Without Validation**

- **D:** `ctx.remaining_accounts` iterated without ownership, signer, type, or data checks. These accounts bypass Anchor's compile-time constraint system entirely — easiest injection point for malicious accounts.
- **FP:** Full validation loop present: owner check, discriminator check, and signer/writable checks applied to each remaining account. Non-zero data length check present.

**10. Missing Account Close Constraint — Data Accessible After Close**

- **D:** Account closed by only zeroing lamports without using Anchor's `close` constraint. Data remains readable within the same transaction. Attacker reads sensitive data or reuses the "closed" account in subsequent instructions.
- **FP:** Anchor `close = recipient` constraint used. Native: full close sequence (zero data → drain lamports → assign to System Program).

**11. Account Revival Attack**

- **D:** Account closed by draining lamports but not reassigning ownership to the System Program. Attacker refunds rent to the account address, "reviving" it with stale or manipulated data within the same transaction.
- **FP:** Proper close: data zeroed, lamports drained, owner set to System Program. Anchor `close` constraint used. Lamport check (`> 0`) before processing accounts.

**12. Operations on Closed Accounts**

- **D:** Instruction reads or writes to an account that was closed in a prior instruction within the same transaction. Account data is still accessible until the transaction completes, leading to inconsistent state.
- **FP:** Lamport balance checked (`> 0`) before operating on the account. Discriminator checked (closed accounts have zeroed discriminator). Transaction design prevents close + use in same tx.

**13. Duplicate Mutable Accounts**

- **D:** Same account passed for two different mutable parameters (e.g., `from` and `to` in a transfer). The last serialized write wins, effectively enabling free "transfers" to self that bypass balance checks or double state mutations.
- **FP:** Explicit constraint: `from.key() != to.key()`. Anchor constraint with `@ ErrorCode::SameAccount`. Single-reference pattern used when updating different fields of the same account.

**14. Missing Rent Exemption Check**

- **D:** New account funded with insufficient lamports — below the rent-exempt threshold. Account becomes eligible for garbage collection by the runtime, causing permanent data loss.
- **FP:** Anchor `init` with `payer` handles rent automatically. Native: `Rent::get()?.minimum_balance(data_len)` checked. System Program `create_account` called with correct lamports.

**15. Unintended Account Closure via close Constraint**

- **D:** `close` constraint applied to an account that should persist, or close destination set to an attacker-controllable address. Attacker triggers the close path and receives the rent lamports.
- **FP:** Close constraint only on accounts explicitly designed to be closeable. Close destination is a trusted address (original payer, program-controlled PDA). Access control on the close instruction.

**16. Missing Token Account Mint Validation**

- **D:** Token account used in a transfer without validating its `mint` field matches the expected mint. Attacker passes a token account for a different (worthless) mint, receiving valuable tokens in return.
- **FP:** Anchor `#[account(token::mint = expected_mint)]` constraint. Native: manual `token_account.mint == expected_mint` comparison. `has_one = mint` on the state account.

**17. Missing Token Account Authority Validation**

- **D:** Token account's `owner` (authority) field not validated against the expected authority. Attacker passes their own token account where the program expects a protocol-controlled account, redirecting funds.
- **FP:** Anchor `#[account(token::authority = expected_authority)]` constraint. `has_one = authority` on the vault state. Manual pubkey comparison of token account owner field.

**18. Sysvar Account Spoofing**

- **D:** Sysvar account (Clock, Rent, Instructions, SlotHashes) passed by user without verifying its public key matches the known sysvar address. Attacker passes a fake account with manipulated timestamps, rent values, or instruction data. (Wormhole exploit vector.)
- **FP:** Sysvar accessed via `Clock::get()?` or `Rent::get()?` (syscall, not account). Anchor `Sysvar<'info, Clock>` type used. Manual address comparison: `account.key == &sysvar::clock::ID`.

**19. Instruction Introspection with Absolute Index**

- **D:** `load_instruction_at(0, ...)` or `load_instruction_at_checked(N, ...)` used with an absolute index. Attacker crafts a transaction where the same instruction at index 0 is used to validate multiple program calls, bypassing intended one-time checks.
- **FP:** Relative indexing used: `get_instruction_relative(-1, ...)` or `load_current_index_checked()` + offset. Correlation validation between current and referenced instructions (same program ID, same accounts).

**20. Unchecked load_instruction_at (Pre-1.8.1)**

- **D:** `load_instruction_at()` (unchecked version) used instead of `load_instruction_at_checked()`. On Solana < 1.8.1, the sysvar account is not validated, allowing complete instruction spoofing.
- **FP:** `load_instruction_at_checked()` used. Solana runtime >= 1.8.1. Manual sysvar address validation before call.

**21. Missing Account Data Length Check**

- **D:** Account data deserialized without checking `account.data_len()` matches the expected struct size. Truncated or oversized data causes deserialization errors or reads garbage bytes.
- **FP:** Anchor handles this automatically via `Account<'info, T>`. Native: explicit `if account.data_len() != expected_size` check. Borsh deserialization with proper error handling.

**22. Account Confusion — System Program as Token Program**

- **D:** System Program account passed where Token Program is expected (or vice versa). Without program ID validation, the CPI call either fails silently or executes unexpected logic.
- **FP:** Anchor `Program<'info, Token>` or `Program<'info, System>` types used. Explicit `require_keys_eq!` against known program IDs.

**23. Missing Writable Requirement on PDA Signer**

- **D:** PDA used as a signer in CPI via `invoke_signed` but the target instruction expects the PDA account to be writable (e.g., for lamport transfer). Missing writable flag causes silent CPI failure.
- **FP:** PDA account marked as writable in both the `AccountMeta` and the Anchor `#[account(mut)]`. Transaction-level writable flag set correctly.

**24. Authority Transfer Without Timelock**

- **D:** Admin/authority can be transferred in a single instruction with no timelock, two-step process, or governance approval. A compromised key immediately takes full control of the protocol.
- **FP:** Two-step transfer: `propose_authority` + `accept_authority`. Timelock or governance vote required. Multi-sig authority.

**25. Missing Constraint on Config Account Update**

- **D:** Protocol config account (fee recipient, fee rate, pause flag) updated without sufficient access control or range validation. Attacker or compromised admin redirects fees to attacker-controlled address or sets fees to 100%.
- **FP:** Access control on config update instruction. Range validation on numeric parameters (`fee_bps <= MAX_FEE`). Fee recipient validated as a known protocol address.

**26. Frontrunnable Account Initialization**

- **D:** Account initialization uses predictable PDA seeds without including the payer's key. Attacker frontruns the legitimate initialization with their own authority, gaining control of the account.
- **FP:** PDA seeds include `payer.key()` or `authority.key()`. `init` constraint ensures single initialization. Seeds include unpredictable components.

**27. Missing is_initialized Flag Check (Native)**

- **D:** In native Rust programs without Anchor, no `is_initialized` boolean flag checked before processing an account. Attacker passes an uninitialized (zeroed) account, causing default/zero values to be treated as valid state.
- **FP:** Discriminator check (non-zero first bytes). Explicit `is_initialized` flag checked. Account owned by System Program rejected (uninitialized accounts are owned by System Program).

**28. Unconstrained Mint Authority**

- **D:** Mint authority not validated during token operations. Attacker mints arbitrary tokens if they can invoke the mint instruction without proper authority checks, inflating supply.
- **FP:** `mint::authority = expected_authority` constraint. Native: mint authority field compared to signer. CPI to token program includes authority validation.

**29. Unconstrained Freeze Authority**

- **D:** Token mint created with a freeze authority that is not set to `None` or a trusted address. Holder of freeze authority can freeze any token account at will, griefing users.
- **FP:** Freeze authority set to `None` at mint initialization. Freeze authority is a governance-controlled address. Token design explicitly requires freeze capability.

**30. Missing Mint Close Authority Validation**

- **D:** Mint close authority not set to `None` during initialization. A mint with close authority can be closed and re-initialized at the same address with different decimals, breaking all downstream accounting.
- **FP:** `mint_close_authority` asserted as `None` during init. Mint close authority is a protocol-controlled PDA. Token-2022 mint authority extensions explicitly managed.
