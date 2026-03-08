# Attack Vectors Reference — Oracle, DeFi & Platform-Level (4/4)

> Part 4 of 4 · Vectors 91–120 of 120 total
> Covers: oracle manipulation, DeFi protocol patterns, staking/rewards, compute budget, logging, input validation, protocol economics

Each vector follows the format:
- **D:** Description — what makes it exploitable
- **FP:** False-positive conditions — mitigations that would make it NOT a finding

---

**91. Stale Oracle Price**

- **D:** Oracle price feed used without checking the `publish_time` or `last_update` timestamp. Oracle may have stopped updating hours ago — attacker exploits stale price to buy/sell at favorable outdated rates.
- **FP:** Staleness check: `require!(clock.unix_timestamp - price.publish_time <= MAX_AGE_SECONDS)`. `MAX_AGE_SECONDS` is admin-configurable. Price rejected if older than threshold.

**92. Oracle Confidence Interval Not Validated**

- **D:** Oracle price used without checking the confidence interval. Wide confidence (high `conf / price` ratio) means the price is unreliable — acting on it enables price manipulation or unfavorable trades.
- **FP:** Confidence check: `require!(price.conf * 100 / price.price <= MAX_CONF_PCT)`. Threshold is configurable. Price rejected if confidence is too wide.

**93. Oracle Status Not Checked**

- **D:** Oracle account read without verifying the price status (e.g., `PriceStatus::Trading`). Non-trading status (halted, unknown) may contain stale or invalid prices.
- **FP:** Status check: `require!(price_feed.status == PriceStatus::Trading)`. Error returned for non-trading status.

**94. Fake Oracle Account — Missing Owner Validation**

- **D:** Oracle account deserialized without checking its owner matches the oracle program (e.g., Pyth, Switchboard). Attacker creates a fake oracle account with manipulated price data.
- **FP:** `require_keys_eq!(*oracle.owner, PYTH_PROGRAM_ID)`. Anchor `Account<'info, PriceAccount>` with owner constraint. Oracle account address hardcoded or stored in validated config.

**95. Retroactive Oracle Pricing**

- **D:** Current oracle price used to settle positions that were opened at a different price. Instead of storing the reference price at open time, the program uses the live price at settlement, enabling manipulation.
- **FP:** Reference price stored in position account at open time. Settlement uses stored reference price. Price updates only affect new positions.

**96. On-Chain Price as Slippage Reference**

- **D:** Slippage protection calculated using an on-chain price (oracle, pool spot price) instead of a user-provided expected price. Attacker manipulates the on-chain price via flash loan, then the "slippage check" uses the manipulated value.
- **FP:** Slippage parameter provided by user off-chain (`min_amount_out`, `max_price`). TWAP used instead of spot price. Flash-loan-resistant price source.

**97. Flash Loan Price Manipulation**

- **D:** Protocol uses spot pool reserves or AMM price for valuation. Attacker takes a flash loan, manipulates pool reserves to move the price, executes the vulnerable operation at the manipulated price, then repays.
- **FP:** TWAP or oracle price used instead of spot. Manipulation-resistant price source. Flash loan detection (same-slot check). Minimum holding period.

**98. Vault Share Inflation — First Depositor Attack**

- **D:** Empty vault/pool allows first depositor to mint 1 share, then donate directly to inflate share price. Second depositor's deposit truncates to 0 shares — first depositor redeems for both deposits.
- **FP:** Virtual shares/assets offset in vault math. Minimum first deposit enforced. Dead shares minted to burn address on init. `require!(minted_shares > 0)`.

**99. Staking Reward Index Not Updated Before Balance Change**

- **D:** Staking contract doesn't update `rewardPerToken` or equivalent accumulator before stake/unstake operations. New staker gets credit for rewards earned before they staked; unstaker loses pending rewards.
- **FP:** Reward accumulator updated before any balance change. `update_rewards()` called at start of stake/unstake. Checkpoint pattern implemented.

**100. Flash Stake/Unstake Reward Capture**

- **D:** No minimum staking duration — attacker flash-deposits before a reward distribution, captures the reward, and immediately withdraws. Gets rewards without any real staking commitment.
- **FP:** Minimum staking/lockup period enforced. Reward distribution pro-rated over time. Snapshot-based rewards from past block/slot.

**101. Reward Dilution via Direct Transfer**

- **D:** Reward calculation based on token balance (`token_account.amount`) rather than internal accounting. Attacker transfers tokens directly to the reward pool, diluting all stakers' reward rates or manipulating the reward-per-token ratio.
- **FP:** Internal accounting tracks deposits separately from raw balance. Rewards calculated from `total_staked` state variable, not balance. Direct transfers don't affect reward math.

**102. Precision Loss Zeroing Small Stakers**

- **D:** Reward calculation for small stakers rounds to zero due to integer division: `(small_stake * reward_rate) / total_stake = 0`. Small stakers permanently earn zero rewards while their stake still dilutes others.
- **FP:** High-precision accumulator (u128 or fixed-point) used. Minimum stake enforced above precision threshold. Accumulated reward tracking prevents rounding to zero.

**103. Cooldown/Unstake Period Griefable by Dust**

- **D:** Unstaking cooldown resets on any new deposit. Attacker sends dust deposits to victim's staking position, perpetually resetting their cooldown and locking their funds.
- **FP:** Cooldown tracks per-deposit or doesn't reset on new deposits. Only the staker themselves can modify their position. Dust deposits below threshold rejected.

**104. Liquidation Incentive Insufficient for Small Positions**

- **D:** Liquidation bonus (percentage-based) on small/dust positions doesn't cover the gas/transaction cost for liquidators. Positions become permanently unliquidatable, accumulating bad debt.
- **FP:** Minimum position size enforced. Fixed minimum liquidation bonus in addition to percentage. Dust position auto-liquidation by protocol.

**105. Self-Liquidation Profitable**

- **D:** User can liquidate their own position and profit from the liquidation bonus. The bonus exceeds the penalty, creating a risk-free arbitrage that drains protocol reserves.
- **FP:** Self-liquidation prohibited (`liquidator != borrower`). Liquidation bonus < penalty. Health factor check prevents liquidation of healthy positions.

**106. Interest Accrual During Protocol Pause**

- **D:** Protocol pauses operations (deposits, withdrawals) but interest continues accruing. When unpaused, users face unexpected interest charges or liquidation from interest accumulated during pause.
- **FP:** Interest accrual paused alongside operations. Accumulated interest during pause forgiven or capped. Pause doesn't affect user positions.

**107. Compute Budget Exhaustion DoS**

- **D:** Instruction requires more compute units than the default 200K (or even the maximum 1.4M) due to complex calculations, large iterations, or multiple CPIs. Transaction always fails, permanently DoS-ing the functionality.
- **FP:** `SetComputeUnitLimit` called with adequate budget. Operations batched to fit within compute limits. Iteration bounded. Complex math optimized.

**108. Unbounded Log Output — Silent Truncation**

- **D:** Program emits large log messages that exceed Solana's ~10KB per-transaction log limit. Logs are silently truncated, losing critical audit trail data. Not exploitable directly but masks attacks.
- **FP:** Log messages kept concise. Critical data emitted as structured events (fixed-size). State persisted on-chain, not only in logs.

**109. Vec Initialization Bug — Comma vs Semicolon**

- **D:** `vec![0, N]` (comma) used instead of `vec![0; N]` (semicolon). Creates a two-element vector `[0, N]` instead of N zeroes. Accessing index 2+ panics, causing DoS.
- **FP:** `vec![0; N]` (semicolon) used correctly. Fixed-size arrays used instead of Vec. No dynamic Vec initialization.

**110. Unconstrained Mint — Supply Inflation**

- **D:** Mint instruction callable without proper authority validation, allowing anyone to mint tokens. Inflates supply, devaluing all existing tokens.
- **FP:** Mint authority validated as signer. `mint::authority = expected` constraint. Supply cap enforced.

**111. Unconstrained Burn — Supply Deflation**

- **D:** Burn instruction callable on any user's tokens without their authorization. Attacker burns other users' tokens, causing permanent fund loss.
- **FP:** Token account owner must be signer for burn. `token::authority = signer` constraint. Only self-burn allowed.

**112. Missing Input Amount Validation**

- **D:** Instruction accepts amounts without bounds checking — amounts of 0, `u64::MAX`, or values outside protocol's operational range. Zero amounts trigger side effects without commitment; max amounts overflow calculations.
- **FP:** `require!(amount > 0 && amount <= MAX_AMOUNT)` on all user inputs. Minimum and maximum amounts enforced. Protocol-specific bounds validated.

**113. Unconstrained Fee Recipient Update**

- **D:** Fee recipient address updateable by admin without validation. Compromised admin redirects all protocol fees to attacker address.
- **FP:** Fee recipient update has timelock. New recipient validated against allowlist. Multi-sig required for updates.

**114. Protocol Config Allows Zero-Fee Path**

- **D:** Admin can set fee to 0%, creating a zero-fee path that drains protocol revenue or enables wash trading without cost.
- **FP:** Minimum fee enforced: `require!(fee_bps >= MIN_FEE_BPS)`. Fee changes require governance. Zero fee only in specific contexts (e.g., whitelisted addresses).

**115. Missing same-asset Check in Swap**

- **D:** Swap function accepts `input_mint == output_mint`. Same-token swap can be exploited to manipulate fee accounting or pool invariants without actual economic activity.
- **FP:** `require!(input_mint != output_mint)` check. Same-asset short-circuits to no-op. Pool invariant checked after swap.

**116. Unchecked Realloc — Account Data Overflow**

- **D:** Account data reallocated beyond the 10MB limit or without proper space calculation. Realloc to smaller size truncates data; realloc to larger size without paying rent causes runtime error.
- **FP:** Realloc size calculated correctly. Rent difference paid. Anchor `realloc` constraint with proper space calculation. Size bounds checked.

**117. Event Logging Inconsistency**

- **D:** Critical state-changing events not emitted or emitted with incorrect data. Off-chain indexers miss state changes, causing UI inconsistencies or delayed responses. Not directly exploitable but enables masked attacks.
- **FP:** Events emitted for all state changes. Event data matches actual state changes. Structured events with all relevant fields.

**118. Unchecked Return Data from CPI**

- **D:** CPI return data assumed to be in a specific format without validation. Malicious program returns unexpected data format, causing deserialization error or misinterpreted values.
- **FP:** CPI target is a trusted program. Return data parsed with proper error handling. `sol_get_return_data()` result validated.

**119. Missing Deadline on Time-Sensitive Operations**

- **D:** Swap, deposit, or other time-sensitive operation has no deadline parameter. Transaction sits in mempool, executes hours later at stale prices or unfavorable conditions.
- **FP:** `deadline` or `valid_until` parameter required. `require!(clock.unix_timestamp <= deadline)` check. Transaction recentness enforced.

**120. Program Upgrade Authority Not Secured**

- **D:** Program's upgrade authority is a single hot wallet. Compromised key can deploy malicious code, draining all protocol funds. This is the highest-impact vector for upgradeable programs.
- **FP:** Upgrade authority is a multi-sig or governance-controlled address. Program is immutable (upgrade authority set to `None`). Timelock on upgrades. Verifiable build deployed.
