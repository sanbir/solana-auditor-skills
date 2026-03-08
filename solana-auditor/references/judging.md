# Finding Validation

Each finding passes a false-positive gate, then gets a confidence score (how certain you are it is real).

## FP Gate

Every finding must pass all three checks. If any check fails, drop the finding — do not score or report it.

1. You can trace a concrete attack path: caller → instruction handler → state change → loss/impact. Evaluate what the code _allows_, not what the deployer _might choose_.
2. The entry point is reachable by the attacker (check Anchor constraints, `Signer` types, `has_one`, access control, PDA-only instructions).
3. No existing guard already prevents the attack (Anchor constraints, `require!`, manual checks, `if`-revert, reentrancy flags, etc.).

## Confidence Score

Confidence measures certainty that the finding is real and exploitable — not how severe it is. Every finding that passes the FP gate starts at **100**.

**Deductions (apply all that fit):**

- Privileged caller required (admin, authority, upgrade authority, multi-sig) → **-25**.
- Attack path is partial (general idea is sound but cannot write exact caller → instruction → state change → outcome) → **-20**.
- Impact is self-contained (only affects the attacker's own funds, no spillover to other users) → **-15**.
- Requires specific token behavior (Token-2022 extensions, transfer hooks, interest-bearing, fee-on-transfer) that may not apply to whitelisted tokens → **-10**.
- Requires external precondition (oracle failure, bridge delay, Solana runtime version constraint) → **-10**.

Confidence indicator: `[score]` (e.g., `[95]`, `[75]`, `[60]`).

Findings below the confidence threshold (default 75) are still included in the report table but do not get a **Fix** section — description only.

## Do Not Report

- Anything a linter, compiler, or seasoned Rust developer would dismiss — INFO-level notes, gas/CU micro-optimizations, naming, documentation, redundant comments.
- Admin/authority can set fees, parameters, or pause — these are by-design privileges, not vulnerabilities.
- Missing event emissions or insufficient logging.
- Centralization observations without a concrete exploit path (e.g., "upgrade authority could rug" with no specific mechanism beyond trust assumptions).
- Theoretical issues requiring implausible preconditions (e.g., compromised Solana validator, >50% token supply held by attacker). Note: common token behaviors (Token-2022 transfer hooks, fee-on-transfer, interest-bearing) are NOT implausible — if the code accepts arbitrary tokens, these are valid attack surfaces.
