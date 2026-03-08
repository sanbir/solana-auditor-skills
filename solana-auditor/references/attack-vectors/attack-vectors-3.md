# Attack Vectors Reference — Arithmetic, Tokens & State Management (3/4)

> Part 3 of 4 · Vectors 61–90 of 120 total
> Covers: integer safety, precision loss, token operations, state lifecycle, account closing, fee logic, dust attacks

Each vector follows the format:
- **D:** Description — what makes it exploitable
- **FP:** False-positive conditions — mitigations that would make it NOT a finding

---

**61. Integer Overflow via Unchecked Arithmetic**

- **D:** Standard `+`, `-`, `*` operators used on `u64`/`u128` financial values. In release mode, Rust wraps on overflow — a deposit of `u64::MAX - 100` plus `200` wraps to `99`, effectively destroying funds or creating tokens from nothing.
- **FP:** `.checked_add()`, `.checked_sub()`, `.checked_mul()`, `.checked_div()` used with `?` propagation. `overflow-checks = true` in Cargo.toml `[profile.release]`. Anchor's `require!` with checked math.

**62. Integer Underflow on Balance Subtraction**

- **D:** Balance subtraction without underflow check: `vault.balance -= amount` where `amount > balance`. Wraps to a massive positive value, crediting the attacker with near-unlimited funds.
- **FP:** `.checked_sub()` used. `require!(vault.balance >= amount)` check before subtraction. Anchor constraint validation.

**63. Division Before Multiplication — Precision Loss**

- **D:** `(amount / total_supply) * price` truncates the division result before multiplying, losing precision permanently. Attacker exploits by using amounts that truncate to zero, getting free operations.
- **FP:** Multiplication performed first: `(amount * price) / total_supply`. Higher-precision intermediate type (u128) used. Decimal/fixed-point library used for calculations.

**64. Division by Zero**

- **D:** Division operation where the divisor can be zero (e.g., `total_supply`, `pool_balance`, `shares_outstanding`). Causes a panic that aborts the transaction, enabling DoS.
- **FP:** Explicit zero check: `require!(divisor > 0)`. `.checked_div()` used. Early return or special case when divisor is zero.

**65. Unsafe Integer Casting — Type Narrowing**

- **D:** Wider type cast to narrower type without bounds check: `value as u32` where `value: u64`. Silently truncates, causing incorrect amounts in transfers, fees, or state updates.
- **FP:** Explicit bounds check: `require!(val <= u32::MAX as u64)`. `try_into()` with error handling used. Consistent types throughout (no narrowing needed).

**66. Rounding Direction Exploitation**

- **D:** Share/token calculations always round in the user's favor. On deposits, rounding up gives slightly more shares; on withdrawals, rounding up gives slightly more tokens. Repeated small operations slowly drain the pool.
- **FP:** Rounding direction favors the protocol: round down on deposits (fewer shares), round up on withdrawals (fewer tokens returned). `floor` for minting, `ceil` for burning.

**67. First Depositor Vault Inflation Attack**

- **D:** First depositor mints shares, then donates tokens directly to the vault, inflating the share price. Subsequent depositors' deposits are rounded down to zero shares due to the inflated price, and the first depositor steals their tokens.
- **FP:** Virtual offset: vault starts with non-zero virtual shares/assets. Minimum deposit enforced. Dead shares minted on initialization. `require!(shares > 0)` on deposit.

**68. Round-Trip Profit — Deposit/Withdraw Arbitrage**

- **D:** Due to inconsistent rounding between deposit and withdraw operations, a user can deposit and immediately withdraw for a net profit. Repeated round-trips drain the pool.
- **FP:** Rounding consistently favors the pool in both directions. Minimum lock period between deposit and withdraw. Round-trip test: `deposit(X) → withdraw(all) ≤ X`.

**69. Saturating Math Misuse**

- **D:** `.saturating_sub()` used where underflow should be an error, not silently clamped to zero. A health factor or balance clamped to zero instead of reverting enables invalid state transitions.
- **FP:** `saturating_sub` only used where a floor of zero is semantically correct (e.g., remaining time). `.checked_sub()` used for financial calculations. Explicit `require!` before subtraction.

**70. Price Slippage Not Enforced**

- **D:** Swap, purchase, or pricing function lacks a user-provided `min_amount_out` or `max_price` parameter. MEV bots sandwich the transaction, manipulating price between submission and execution.
- **FP:** `expected_price` or `min_amount_out` parameter required. Slippage tolerance enforced with `require!`. Deadline parameter prevents stale execution.

**71. Lamport Balance Invariant Violation**

- **D:** Custom logic creates or destroys lamports — violating Solana's invariant that total lamports across all accounts in an instruction must remain constant. Can cause runtime errors or exploitable accounting discrepancies.
- **FP:** All lamport transfers are balanced: sum of debits equals sum of credits. System Program used for lamport transfers. Manual lamport math verified with assertions.

**72. Rent Lamports Sent to Arbitrary Destination**

- **D:** When closing an account, rent-exempt lamports transferred to a user-specified destination without validation. Attacker redirects rent from protocol accounts to themselves.
- **FP:** Close destination hardcoded to original payer or program-controlled address. Anchor `close = known_recipient`. Validated recipient address.

**73. Token Dust Account Poisoning**

- **D:** Attacker deposits a tiny (dust) amount into a token account, preventing it from being closed (close requires zero balance). This permanently blocks account closure, leaking rent and preventing state cleanup.
- **FP:** Dust swept or burned before close. Dust threshold defined — operations below threshold rejected. `close_account` with force flag if balance is below dust threshold.

**74. Fee Bypass on Alternative Code Path**

- **D:** Protocol fee applied on one code path (e.g., normal withdrawal) but not on another (e.g., emergency withdrawal, single-asset exit). Attacker uses the fee-free path to avoid paying fees.
- **FP:** Fees applied on every exit/withdrawal path. Single fee calculation function used across all paths. Fee-free paths have access control (e.g., admin-only emergency).

**75. Pre-Fee / Post-Fee Amount Confusion**

- **D:** Fee calculated on the pre-fee amount but the capacity check uses the post-fee amount (or vice versa). This creates accounting mismatches — either overfilling beyond capacity or under-charging fees.
- **FP:** Consistent amount used: either pre-fee throughout or post-fee throughout. Fee deducted atomically with the principal operation. Clear naming: `amount_before_fee`, `amount_after_fee`.

**76. Fee Deduction Not Atomic with Transfer**

- **D:** Fee calculated and deducted in a separate instruction from the transfer. Attacker skips the fee instruction or reorders instructions to avoid payment.
- **FP:** Fee deducted in the same instruction as the transfer. Atomic transaction design. Fee deducted from the transferred amount (not a separate call).

**77. Token Decimals Mismatch**

- **D:** Token operations assume a specific decimal count (e.g., 6 for USDC) without reading the mint's actual `decimals` field. When a token with different decimals is used, amounts are off by orders of magnitude.
- **FP:** `mint.decimals` read and used in all calculations. `transfer_checked` used (requires decimals parameter). Decimal normalization applied.

**78. Missing Transfer Amount Validation**

- **D:** Transfer instruction accepts `amount = 0` without validation. Zero-amount transfers can be used to trigger side effects (events, state updates, reward snapshots) without actual economic commitment.
- **FP:** `require!(amount > 0)` check on all transfer/deposit/withdraw instructions. Minimum amount enforced. Zero-amount short-circuits to no-op.

**79. Coupled State Fields Not Reset Atomically**

- **D:** Account has logically coupled fields (e.g., `shares_pending` + `total_shares`, `rewards_owed` + `last_claim_time`). On close or completion, one field is reset but the other isn't, leaving the account in an inconsistent state exploitable in future operations.
- **FP:** All coupled fields reset in the same instruction. Struct method that resets all related fields atomically. Close constraint zeros entire account data.

**80. Counter Drift — Statistic Not Updated Atomically**

- **D:** Global counters (total_deposits, total_users, volume) updated in a separate step from the operation that triggers them. If the update is skipped (error, reorder), counters drift, breaking protocol invariants.
- **FP:** Counters updated in the same instruction as the triggering operation. Atomic increment: `counter = counter.checked_add(1)?`. Counter can be re-derived from on-chain state if needed.

**81. Time Unit Mismatch — Slots vs Seconds**

- **D:** One part of the code uses slot numbers, another uses Unix timestamps (seconds), but they're compared directly. A vesting window in seconds compared to slot-based timestamps can unlock 4× earlier than intended.
- **FP:** Single canonical time unit used throughout. Explicit scale factor applied when converting. Field names annotated: `_slot`, `_timestamp_secs`. `Clock::get()?.unix_timestamp` used consistently.

**82. Stale Clock — Using Cached Timestamp**

- **D:** `Clock::get()` called once, timestamp cached, then used across multiple operations within the instruction. For most programs this is fine, but if the instruction spans multiple CPI calls that depend on ordering, the cached value may not reflect the expected time context.
- **FP:** `Clock::get()` called once per instruction (normal and expected). Timestamp used only for comparison, not for absolute scheduling. No time-sensitive CPI interleaving.

**83. Account Data Realloc Without Zero-Init**

- **D:** Account data reallocated to a larger size without zero-initializing the new bytes. Old data from the memory allocator may leak into the new space, causing unpredictable behavior.
- **FP:** Anchor `realloc` with `zero` = true constraint. Native: `memset` on new bytes. New space explicitly initialized before use.

**84. Unbounded Collection — Compute DoS**

- **D:** Instruction iterates over a variable-length collection (vector, remaining_accounts, linked list) without an upper bound. Attacker grows the collection until iteration exceeds the compute budget, permanently DoS-ing the instruction.
- **FP:** Fixed upper bound on collection size. Pagination pattern used. `SetComputeUnitLimit` budgeted for worst case. Iteration short-circuits or processes in batches.

**85. Self-Transfer Inflates Fee Claims**

- **D:** Transfer function allows `source == destination`. A self-transfer that triggers fee calculation or reward snapshot allows the user to accumulate fees/rewards without actual economic activity.
- **FP:** `require!(source.key() != destination.key())` check. Self-transfer short-circuits to no-op. Fee calculation skipped for zero-net-movement operations.

**86. Missing Preprocessing on Share Transfer**

- **D:** Shares or LP tokens transferred between users without settling pending fees/rewards on both source and destination first. Destination receives fees they never earned; source loses fees they're owed.
- **FP:** Both source and destination preprocessed (fees settled, reward accumulators snapshotted) before share transfer. Automatic settlement via transfer hook. Fee settlement in same instruction.

**87. Expired Account Not Closeable**

- **D:** Time-limited accounts (offers, escrows, locks) have no expiry-based close mechanism. Expired accounts leak rent indefinitely and can be griefed by keeping them alive.
- **FP:** Anyone can close expired accounts (not just creator). Expiry check: `require!(clock.unix_timestamp >= expiry)`. Automatic cleanup via crank or keeper.

**88. Token-2022 Interest-Bearing Token Not Accounted For**

- **D:** Token-2022 interest-bearing extension modifies the effective balance over time, but the program reads the raw balance. Calculations based on raw balance are incorrect, leading to under/over-accounting.
- **FP:** Interest-bearing extension detected and effective balance calculated. Only standard SPL tokens supported (documented and enforced). Token-2022 extensions explicitly handled.

**89. Token-2022 Transfer Fee Not Accounted For**

- **D:** Token-2022 transfer fee extension deducts a fee on every transfer, but the program assumes the full amount arrives at the destination. Accounting becomes inconsistent — the program credits more than was received.
- **FP:** Transfer fee extension detected. Post-transfer balance checked instead of using input amount. `transfer_checked` with fee calculation. Only non-fee tokens supported (enforced).

**90. Unsafe Rust Block Without Justification**

- **D:** `unsafe` block used for performance optimization (zero-copy, raw pointer access) without proper bounds checking. Memory corruption or out-of-bounds read can lead to arbitrary state manipulation.
- **FP:** No `unsafe` blocks in the codebase. `unsafe` blocks have explicit safety comments and bounds checks. `bytemuck` or `zerocopy` used instead of raw pointer manipulation.
