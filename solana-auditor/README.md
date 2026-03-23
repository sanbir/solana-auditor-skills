# Solana Auditor

A security agent for **Solana programs**.

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
/solana-auditor deep
/solana-auditor programs/vault/src/lib.rs
/solana-auditor --file-output
```

## Coverage

- **105 attack vectors** tuned for Solana program security
- **Parallel scan agents** for rapid first-pass triage
- **Deep mode** for adversarial reasoning and Solana protocol analysis

## What It Looks For

- missing signer / writable / owner checks
- PDA seed collisions, bump misuse, and close/reinit bugs
- CPI trust errors and stale-account assumptions after CPI
- Token and Token-2022 quirks, transfer-hook exposure, and authority mixups
- initialization frontruns and unsafe authority rotation
- oracle / fee / slippage / liquidation logic bugs
- compute- or loop-driven denial of service

## Tips

- **Target instruction handlers and account-validation code first.** Those files usually contain the real trust boundaries.
- **Use `deep` for CPI-heavy, PDA-heavy, liquidation-sensitive, or oracle-dependent programs.** The extra pass pays off when state changes span several accounts and handlers.
