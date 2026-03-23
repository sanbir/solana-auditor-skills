# Finding Validation

Every finding passes four sequential gates. Fail any gate → **rejected** or **demoted** to lead. Later gates are not evaluated for failed findings.

## Gate 1 — Refutation

Construct the strongest argument that the finding is wrong. Find the Anchor constraint, `require!`, manual check, or PDA-gated path that kills the attack — quote the exact line and trace how it blocks the claimed step.

- Concrete refutation (specific guard blocks exact claimed step) → **REJECTED** (or **DEMOTE** if code smell remains)
- Speculative refutation ("probably wouldn't happen") → **clears**, continue

## Gate 2 — Reachability

Prove the vulnerable state exists in a live deployment.

- Structurally impossible (enforced invariant, Anchor constraint prevents it) → **REJECTED**
- Requires privileged actions outside normal operation → **DEMOTE**
- Achievable through normal usage or common token behaviors → **clears**, continue

## Gate 3 — Trigger

Prove an unprivileged actor executes the attack.

- Only trusted roles (authority, admin, upgrade authority) can trigger → **DEMOTE**
- Costs exceed extraction → **REJECTED**
- Unprivileged actor triggers profitably → **clears**, continue

## Gate 4 — Impact

Prove material harm to an identifiable victim.

- Self-harm only → **REJECTED**
- Dust-level, no compounding → **DEMOTE**
- Material loss to identifiable victim → **CONFIRMED**

## Confidence

Start at **100**, deduct: partial attack path **-20**, bounded non-compounding impact **-15**, requires specific (but achievable) state **-10**. Confidence >= 80 gets description + fix. Below 80 gets description only.

## Safe patterns (do not flag)

- Anchor 8-byte discriminators (automatic type safety)
- `has_one` constraints on authority fields (ownership check)
- `seeds` + `bump` with canonical bump stored in account (PDA validation)
- `checked_add`/`checked_sub`/`checked_mul` with error propagation
- `Account<'info, T>` with correct type (automatic owner + discriminator check)
- `Signer<'info>` on authority accounts (signer enforcement)
- Two-step authority transfer pattern
- Consistent protocol-favoring rounding unless compounding or zero-rounding
- `close = destination` constraint (proper account closing)

## Lead promotion

Before finalizing leads, promote where warranted:

- **Cross-handler echo.** Same root cause confirmed as FINDING in one handler → promote in every handler where the identical pattern appears.
- **Multi-agent convergence.** 2+ agents flagged same area, lead was demoted (not rejected) → promote to FINDING at confidence 75.
- **Partial-path completion.** Only weakness is incomplete trace but path is reachable and unguarded → promote to FINDING at confidence 75, description only.

## Leads

High-signal trails for manual investigation. No confidence score, no fix — title, code smells, and what remains unverified.

## Do Not Report

Linter/compiler issues, compute-unit micro-opts, naming, doc comments. Admin privileges by design (`has_one = authority`). Missing event emissions (`emit!`). Centralization without exploit path. Implausible preconditions (but Token-2022 transfer fees, transfer hooks, interest-bearing ARE plausible for programs accepting arbitrary tokens).
