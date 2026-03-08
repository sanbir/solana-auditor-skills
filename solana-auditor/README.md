# Solana Auditor

The ultimate AI-powered security audit skill for Solana — 120 attack vectors, 4 parallel scan agents, adversarial reasoning, and DeFi protocol analysis.

Built for:

- **Solana devs** who want a security check before every commit
- **Security researchers** looking for fast wins before a manual review
- **Auditors** who want systematic vector coverage as a first pass

Not a substitute for a formal audit — but the most comprehensive AI check you can run on Solana programs.

## What's Inside

- **120 attack vectors** across 4 reference files — covering account validation, PDA security, CPI trust boundaries, arithmetic safety, token operations (SPL + Token-2022), state lifecycle, oracle manipulation, DeFi protocol economics, and more
- **4 parallel vector-scan agents** — each assigned ~30 vectors, scanning the full codebase simultaneously
- **Adversarial reasoning agent** (DEEP mode) — free-form exploit hunting using Feynman questioning, state inconsistency analysis, and invariant hunting
- **Solana protocol agent** (DEEP mode) — domain-specific checklists for lending, AMM/DEX, vaults, staking, bridges, governance, proxies, and session keys
- **False-positive gate** — every finding must pass 3 checks (concrete path, reachable entry point, no existing guard)
- **Confidence scoring** — base 100 with deductions for privileged callers, partial paths, self-contained impact, token assumptions, and external preconditions
- **Framework-aware** — works with Anchor, native Rust, and Pinocchio programs

## Usage

```bash
# Scan the full repo (default — 4 agents)
/solana-auditor

# Full repo + adversarial reasoning + protocol analysis (6 agents)
/solana-auditor deep

# Review specific file(s)
/solana-auditor programs/vault/src/lib.rs
/solana-auditor programs/vault/src/instructions/deposit.rs programs/vault/src/instructions/withdraw.rs

# Write report to a markdown file (terminal-only by default)
/solana-auditor --file-output
```

## Known Limitations

**Codebase size.** Works best up to ~2,500 lines of Rust. Past ~5,000 lines, triage accuracy and mid-bundle recall drop noticeably. For large codebases, run per program rather than everything at once.

**What AI misses.** AI is strong at pattern matching — missing account validations, unchecked arithmetic, known CPI pitfalls. It struggles with relational reasoning: multi-transaction state setups, specification/invariant bugs, cross-protocol composability, game-theory attacks, and off-chain assumptions. AI catches what humans forget to check. Humans catch what AI cannot reason about. You need both.
