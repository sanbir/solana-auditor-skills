# Solana Auditor

A security agent for **Solana programs**.

Attribution: this fork keeps the v2 packaging and audit workflow lineage from [pashov/skills](https://github.com/pashov/skills), adapted for Solana programs.

Built for:

- **Program developers** who want a security pass before shipping instruction changes
- **Security researchers** who need rapid coverage over handlers, PDAs, and CPIs
- **Auditors** who want a structured first pass over account validation and state transitions

It is not a substitute for a full audit. It is the fast pass you should run before you trust a program.

## Demo

_Portrayed below: running the skill in a terminal workflow_

![Running solana-auditor in terminal](../static/skill_pag.gif)

## Usage

```bash
/solana-auditor
/solana-auditor --deep
/solana-auditor programs/vault/src/lib.rs
/solana-auditor --file-output
```

## Architecture (v3)

8 specialized parallel agents (sonnet) + optional protocol agent (opus for --deep):

| Agent | Focus |
|-------|-------|
| 1. Vector Scan | All attack vectors from vector bundle |
| 2. Math Precision | Arithmetic, rounding, `as` truncation, decimals |
| 3. Access Control | Signer, owner, PDA authority, initialization |
| 4. Economic Security | Oracles, token quirks, CPI trust, value extraction |
| 5. Execution Trace | Post-CPI staleness, serialization, remaining_accounts |
| 6. Invariant | Conservation laws, state couplings, round-trips |
| 7. Periphery | Utility modules, helpers, serialization code |
| 8. First Principles | Assumption extraction and violation |
| 9. Protocol (--deep) | DeFi-specific checklists (lending, AMM, vault, staking, bridge, governance) |

## Coverage

- **105+ attack vectors** tuned for Solana program security
- **8 specialized hacking agents** for deep parallel analysis
- **4-gate validation** with confidence scoring and lead tracking
- **--deep mode** adds protocol-specific DeFi analysis

## What It Looks For

- missing signer / writable / owner checks and type cosplay
- PDA seed collisions, bump misuse, and close/reinit bugs
- CPI trust errors, stale-account assumptions after CPI, return value ignoring
- Token and Token-2022 quirks, transfer-hook exposure, and authority mixups
- `as` truncation, saturating math abuse, f64 in financial logic
- initialization frontruns and unsafe authority rotation
- oracle / fee / slippage / liquidation logic bugs
- remaining_accounts injection and instruction introspection bypass
- broken invariants, state coupling gaps, round-trip exploits
- compute- or loop-driven denial of service

## Tips

- **Target instruction handlers and account-validation code first.** Those files usually contain the real trust boundaries.
- **Use `--deep` for CPI-heavy, PDA-heavy, liquidation-sensitive, or oracle-dependent programs.** The extra pass pays off when state changes span several accounts and handlers.
