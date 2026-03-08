# Solana Auditor Skills

> The ultimate Claude Code skill for Solana smart contract security auditing — 120 attack vectors, 6 parallel agents, DeFi protocol checklists, and adversarial reasoning.

Built in the style of [pashov/skills](https://github.com/pashov/skills) (Solidity) but rebuilt from scratch for **Rust/SVM/Solana**. Aggregates knowledge from 10+ open-source audit and development skill repositories.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Architecture

The same proven architecture as pashov/skills — adapted for Solana's account model, CPI trust boundaries, PDA security, and the Anchor/Native Rust/Pinocchio ecosystem.

### 4-Turn Orchestration

1. **Discover** — find all in-scope `.rs` files, resolve reference paths
2. **Prepare** — bundle codebase + attack vectors + judging rules into per-agent files
3. **Spawn** — launch 4–6 agents in parallel (vector scan, adversarial reasoning, protocol analysis)
4. **Report** — merge, deduplicate by root cause, sort by confidence, format

### 120 Attack Vectors (4 reference files)

| File | Vectors | Focus Areas |
| --- | --- | --- |
| [attack-vectors-1](solana-auditor/references/attack-vectors/attack-vectors-1.md) | 1–30 | Signer checks, ownership, discriminators, account constraints, data matching, reinitialization, writable flags, Token-2022 mint authority, sysvar spoofing, instruction introspection |
| [attack-vectors-2](solana-auditor/references/attack-vectors/attack-vectors-2.md) | 31–60 | PDA derivation (bumps, sharing, collisions, seeds), CPI safety (arbitrary CPI, signer pass-through, stale data, return values), invoke_signed, Token-2022 transfer hooks, cross-program reentrancy |
| [attack-vectors-3](solana-auditor/references/attack-vectors/attack-vectors-3.md) | 61–90 | Integer overflow/underflow, precision loss, rounding direction, first-depositor inflation, fee bypass, dust DoS, token decimals, state lifecycle (close, revival, realloc), coupled fields, time units |
| [attack-vectors-4](solana-auditor/references/attack-vectors/attack-vectors-4.md) | 91–120 | Oracle manipulation (staleness, confidence, spoofing), flash loan attacks, vault share inflation, staking reward gaming, liquidation economics, compute DoS, program upgrade authority, governance, bridges |

### 3 Specialized Agent Types

| Agent | Mode | Model | Approach |
| --- | --- | --- | --- |
| Vector Scan (x4) | Default + Deep | Sonnet | Systematic triage of ~30 vectors each against full codebase |
| Adversarial Reasoning | Deep only | Opus | Free-form exploit hunting with Feynman questioning, state inconsistency analysis, invariant hunting |
| Solana Protocol | Deep only | Opus | Domain-specific checklists for lending, AMM, vaults, staking, bridges, governance, proxies, session keys |

### Quality Controls

- **FP Gate:** 3-check filter (concrete path, reachable entry, no existing guard)
- **Confidence Scoring:** Base 100 with deductions for privileged callers (-25), partial paths (-20), self-contained impact (-15), token assumptions (-10), external preconditions (-10)
- **Threshold:** Findings below 75 confidence reported without fix suggestions
- **Framework-aware:** Works with Anchor, native Rust, and Pinocchio

---

## Install & Run

Works with **Claude Code CLI**, the **VS Code Claude extension**, and **Cursor**.

**Claude Code CLI:**

```bash
git clone https://github.com/sanbir/solana-auditor-skills.git && mkdir -p ~/.claude/commands && cp -r solana-auditor-skills/solana-auditor ~/.claude/commands/solana-auditor
```

**Cursor:**

```bash
git clone https://github.com/sanbir/solana-auditor-skills.git && mkdir -p ~/.cursor/skills && cp -r solana-auditor-skills/solana-auditor ~/.cursor/skills/solana-auditor
```

The skill is then invocable as `/solana-auditor`. See the [skill README](solana-auditor/README.md) for usage.

**Update to latest:** `cd` into the cloned repo and run:

```bash
git pull
# Claude Code CLI:
cp -r solana-auditor/ ~/.claude/commands/solana-auditor
# Cursor:
cp -r solana-auditor/ ~/.cursor/skills/solana-auditor
```

---

## Skills

| Skill | Description |
| --- | --- |
| [solana-auditor](solana-auditor/) | 120-vector security audit with 4–6 parallel agents, DeFi protocol checklists, and adversarial reasoning |

---

## What's Included

### 120 Attack Vectors (4 reference files)

Organized by attack surface:

**Account Validation & Authorization (V1–V30):** Missing signer checks, ownership spoofing, type cosplay, discriminator bypass, reinitialization, `init_if_needed` frontrunning, `has_one` constraint gaps, writable flag abuse, `UncheckedAccount` without manual validation, `remaining_accounts` injection, account revival, duplicate mutable accounts, rent exemption, sysvar spoofing, instruction introspection abuse, token mint/authority validation, mint close authority.

**PDA, CPI & Cross-Program Security (V31–V60):** Non-canonical bumps, PDA sharing, seed concatenation collisions, cross-type seed collisions, purpose isolation, arbitrary CPI, `invoke` vs `invoke_signed` confusion, signer pass-through, SOL balance drain via CPI, post-CPI ownership change, stale data after CPI (missing `reload()`), CPI return values ignored, Token-2022 incompatibility, transfer hook gaps, cross-program reentrancy, address lookup tables, durable nonces.

**Arithmetic, Tokens & State Management (V61–V90):** Integer overflow/underflow, division-before-multiplication, unsafe casting, rounding direction exploitation, first-depositor vault inflation, round-trip profit, saturating math misuse, slippage not enforced, lamport invariant violation, fee bypass on alternate paths, pre/post-fee confusion, token decimals mismatch, coupled field inconsistency, counter drift, time unit mismatch, account realloc without zero-init, unbounded collection DoS, Token-2022 interest/fee extensions, unsafe Rust.

**Oracle, DeFi & Platform-Level (V91–V120):** Stale oracle price, confidence interval not validated, fake oracle account, retroactive pricing, on-chain price as slippage reference, flash loan manipulation, vault share inflation, staking reward index bugs, flash stake/unstake capture, reward dilution via direct transfer, precision loss zeroing small stakers, cooldown griefing, liquidation incentive gaps, self-liquidation profit, interest during pause, compute budget DoS, unbounded logs, supply inflation/deflation, program upgrade authority.

### Protocol Checklists (74 items across 8 domains)

| Domain | Items | Key Checks |
| --- | --- | --- |
| Lending/Borrowing | 14 | Health factor includes accrued interest, liquidation incentive covers gas, self-liquidation not profitable, bad debt socialization |
| AMM/DEX | 10 | Slippage from calldata not on-chain, deadline enforced, multi-hop protection, LP value not from raw balance |
| Vault/Token Accounting | 10 | First-depositor mitigated, rounding direction correct, round-trip not profitable, share price not manipulable |
| Staking/Rewards | 10 | Reward accumulator updated before balance change, no flash stake capture, precision doesn't zero small stakers |
| Bridge/Cross-Chain | 9 | Message replay protection, source validation, rate limits, decimal conversion, supply invariant |
| Governance | 6 | Vote weight from past slot, timelock, quorum, no double-voting via transfer |
| Proxy/Upgradeable | 8 | Multi-sig upgrade authority, timelock, storage append-only, verifiable build |
| Session Keys/AA | 7 | Bounded permissions, revocable, replay protection, no self-escalation |

---

## Attributions

This skill aggregates knowledge from the following open-source repositories. We are grateful to all contributors.

### Architecture Inspiration

| Repository | Author | Contribution |
| --- | --- | --- |
| [pashov/skills](https://github.com/pashov/skills) | Pashov Audit Group | Parallelized agent orchestration pattern, FP gate, confidence scoring, vector-scan and adversarial-reasoning agent design, report formatting — adapted from Solidity to Solana |

### Solana Security Knowledge

| Repository | Author | Contribution |
| --- | --- | --- |
| [ciphernova-skills/safe-solana-builder](https://github.com/ciphernova-skills/safe-solana-builder) | CipherNova / Frank Castle | 20 comprehensive security rules (account validation, PDA security, CPI safety, arithmetic, Token-2022, oracle, fees, state management, clock/timing), Anchor-specific and native Rust patterns, security checklist methodology |
| [trailofbits/building-secure-contracts](https://github.com/trailofbits/building-secure-contracts) | Trail of Bits | 6 critical Solana vulnerability patterns (arbitrary CPI, improper PDA validation, missing ownership/signer checks, sysvar spoofing, instruction introspection), Solana-specific lint rules, detection patterns with code examples |
| [tenequm/skills](https://github.com/tenequm/skills) | Tenequm | 15 detailed vulnerability patterns with exploit scenarios and secure alternatives (signer validation, overflow, PDA substitution, type cosplay, account reloading, closing, lamports, CPI, duplicates, bump canonicalization, precision loss, init_if_needed, stale oracle) |
| [nicholasgasior/solana-dev-skill](https://github.com/nicholasgasior/solana-dev-skill) | Nicholas Gasior | 9 vulnerability categories with Anchor/Pinocchio code examples, program-side checklist (39 items), client-side checklist (7 items), security review questions, framework-specific prevention patterns |
| [solana-claude-config](https://github.com/nicholasgasior/solana-claude-config) | Nicholas Gasior | 13-step audit workflow, Anchor architect agent (PDA architecture, token programs, CPI patterns), Anchor engineer agent (modern patterns, constraint patterns, testing), comprehensive Anchor rules (446 lines), security checklists |
| [aeither/solana-anchor-claude-skill](https://github.com/aeither/solana-anchor-claude-skill) | Aeither | Anchor development coding guidelines, platform terminology, Anchor version best practices, project structure conventions, PDA management patterns, space calculation methodology |
| [nicholasgasior/dot-context](https://github.com/nicholasgasior/dot-context) | Nicholas Gasior | 11 Solana/Anchor audit check categories (account constraints, PDA safety, CPI safety, deserialization, error handling, token operations, system accounts, type cosplay, closing accounts), 10 detailed vulnerability knowledge base files |

### Methodology & Agents

| Repository | Author | Contribution |
| --- | --- | --- |
| [sainikethan/nemesis-auditor](https://github.com/sainikethan/nemesis-auditor) | Nemesis | Feynman questioning strategy, state inconsistency analysis methodology — adapted for adversarial reasoning agent |
| [carni-ships/SolidSecs](https://github.com/carni-ships/SolidSecs) | SolidSecs | Protocol-specific checklist approach (lending, AMM, vault, staking, bridge, governance, proxy, account abstraction) — adapted for Solana protocol agent |
| [auditmos/skills](https://github.com/auditmos/skills) | Auditmos | Lending protocol vulnerability patterns, liquidation mechanics, staking reward edge cases — adapted for DeFi attack vectors |

### DeFi Protocol Knowledge

| Repository | Author | Contribution |
| --- | --- | --- |
| [sendaifun/skills](https://github.com/sendaifun/skills) | SendAI | Solana ecosystem protocol integration patterns (Jupiter, Drift, Kamino, Helius) |
| [ethskills.com](https://ethskills.com) | EthSkills | Cross-chain audit checklist methodology, protocol composability patterns |
| [kadenzipfel/scv-scan](https://github.com/kadenzipfel/scv-scan) | Kaden Zipfel | 4-phase systematic audit methodology (load → sweep → validate → report) |

---

## License

[MIT](LICENSE) — see individual attribution repos for their respective licenses.
