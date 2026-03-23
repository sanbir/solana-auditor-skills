# Contributing to Solana Auditor Skills

## Pull Request Process

1. Fork the repo and create a branch from `main`.
2. Make your changes - attack vectors, agent prompts, report formatting, or documentation.
3. Ensure your branch is up to date with `main` before opening a PR.
4. Do not edit `VERSION` - it is bumped automatically on merge via CI.
5. Fill in the PR template. A maintainer will review within 5 business days.

### PR checklist

- [ ] No API keys, tokens, or sensitive data
- [ ] No fabricated examples - outputs must reflect real model responses
- [ ] Skill works with Claude Code CLI, VS Code, and Cursor

## What to Contribute

- **Attack vectors** - add new vectors to `solana-auditor/references/attack-vectors/attack-vectors-*.md` following the existing local format.
- **Agent prompts** - improve triage accuracy, reduce false positives, tighten output format.
- **Report formatting** - improve the output structure or fix template issues.
- **Bug fixes** - if the skill produces incorrect output, open an issue or PR with a fix.

## Reporting Bugs

Use the [Bug Report](.github/ISSUE_TEMPLATE/bug_report.md) issue template and include:

- Which skill is affected and how you invoked it.
- The model used.
- The input you gave and the output you got.
- What you expected instead.
