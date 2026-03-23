# CLAUDE.md

Instructions for Claude when contributing to this repository.

## What This Repo Is

A library of Claude AI skills focused on **Solana program security auditing**.

This repo is not a general Rust framework helper. Contributions should sharpen vulnerability detection for real Solana runtime issues.

## Structure

```
solana-auditor/ # Security review of Solana programs while you develop.
CLAUDE.md       # This file (read by Claude Code)
```

## Rules

- Keep guidance specific to Solana accounts, PDAs, CPI boundaries, authorities, and runtime semantics.
- Prefer concrete signer / writable / owner / seed / reload / Token-2022 risks over generic smart-contract advice.
- No fabricated examples - outputs must reflect real model responses.
- No secrets, API keys, or personal data.
