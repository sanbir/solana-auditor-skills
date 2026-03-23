---
name: solana-auditor
description: Security audit of Solana/Rust programs while you develop. Trigger on "audit", "check this program", "review for security". Modes - default (full repo) or a specific filename.
---

# Solana Program Security Audit

You are the orchestrator of a parallelized Solana smart contract security audit.

## Mode Selection

**Exclude pattern:** skip directories `tests/`, `test/`, `migrations/`, `scripts/`, `target/`, `node_modules/` and files matching `*_test.rs`, `*_tests.rs`, `test_*.rs`, `tests.rs`, `mod.rs` (unless it contains instruction handlers).

- **Default** (no arguments): scan all `.rs` files in the program directory using the exclude pattern. Use Bash `find` (not Glob).
- **`$filename ...`**: scan the specified file(s) only.

**Flags:**

- `--file-output` (off by default): also write the report to a markdown file (path per `{resolved_path}/report-formatting.md`). Never write a report file unless explicitly passed.
- `--deep`: also spawn the Solana protocol analysis agent (Agent 9, opus). Use for thorough reviews of DeFi protocols. Slower and more costly.

## Orchestration

**Turn 1 — Discover.** Print the banner, then make these parallel tool calls in one message:

a. Bash `find` for in-scope `.rs` files per mode selection
b. Glob for `**/references/attack-vectors/attack-vectors-1.md` — extract the `references/` directory (two levels up) as `{resolved_path}`
c. ToolSearch `select:Agent`
d. Read the local `VERSION` file from the same directory as this skill
e. Bash `curl -sf https://raw.githubusercontent.com/sanbir/solana-auditor-skills/main/solana-auditor/VERSION`
f. Bash `mktemp -d /tmp/audit-XXXXXX` → store as `{bundle_dir}`

If the remote VERSION fetch succeeds and differs from local, print `⚠️ You are not using the latest version. Please upgrade for best security coverage. See https://github.com/sanbir/solana-auditor-skills`. If it fails, skip silently.

**Turn 2 — Prepare.** In one message, make parallel tool calls: (a) Read `{resolved_path}/report-formatting.md`, (b) Read `{resolved_path}/judging.md`.

Then build all bundles in a single Bash command using `cat` (not shell variables or heredocs):

1. `{bundle_dir}/source.md` — ALL in-scope `.rs` files, each with a `### path` header and fenced code block.
2. Agent bundles = `source.md` + agent-specific files:

| Bundle               | Appended files (relative to `{resolved_path}`)                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `agent-1-bundle.md`  | `attack-vectors/attack-vectors-1.md` + `attack-vectors/attack-vectors-2.md` + `attack-vectors/attack-vectors-3.md` + `attack-vectors/attack-vectors-4.md` + `attack-vectors/attack-vectors-5.md` + `hacking-agents/vector-scan-agent.md` + `hacking-agents/shared-rules.md` |
| `agent-2-bundle.md`  | `hacking-agents/math-precision-agent.md` + `hacking-agents/shared-rules.md`                                     |
| `agent-3-bundle.md`  | `hacking-agents/access-control-agent.md` + `hacking-agents/shared-rules.md`                                     |
| `agent-4-bundle.md`  | `hacking-agents/economic-security-agent.md` + `hacking-agents/shared-rules.md`                                  |
| `agent-5-bundle.md`  | `hacking-agents/execution-trace-agent.md` + `hacking-agents/shared-rules.md`                                    |
| `agent-6-bundle.md`  | `hacking-agents/invariant-agent.md` + `hacking-agents/shared-rules.md`                                          |
| `agent-7-bundle.md`  | `hacking-agents/periphery-agent.md` + `hacking-agents/shared-rules.md`                                          |
| `agent-8-bundle.md`  | `hacking-agents/first-principles-agent.md` + `hacking-agents/shared-rules.md`                                   |

Print line counts for every bundle and `source.md`. Do NOT inline file content into agent prompts.

**Turn 3 — Spawn.** In one message, spawn all 8 agents as parallel foreground Agent calls. Prompt template (substitute real values):

```
Your bundle file is {bundle_dir}/agent-N-bundle.md (XXXX lines).
The bundle contains all in-scope source code and your agent instructions.
Read the bundle fully before producing findings.
```

If `--deep` is set, also spawn Agent 9 (Solana protocol analysis) with `model: "opus"`. Agent 9 receives the in-scope `.rs` file paths and the instruction: your reference directory is `{resolved_path}`. Read `{resolved_path}/hacking-agents/solana-protocol-agent.md` for your full instructions.

**Turn 4 — Deduplicate, validate & output.** Single-pass: deduplicate all agent results, gate-evaluate, and produce the final report in one turn. Do NOT print an intermediate dedup list — go straight to the report.

1. **Deduplicate.** Parse every FINDING and LEAD from all agents. Group by `group_key` field (format: `Program | handler | bug-class`). Exact-match first; then merge synonymous bug_class tags sharing the same program and handler. Keep the best version per group, number sequentially, annotate `[agents: N]`.

   Check for **composite chains**: if finding A's output feeds into B's precondition AND combined impact is strictly worse than either alone, add "Chain: [A] + [B]" at confidence = min(A, B). Most audits have 0-2.

2. **Gate evaluation.** Run each deduplicated finding through the four gates in `judging.md` (do not skip or reorder). Evaluate each finding exactly once — do not revisit after verdict.

   **Single-pass protocol:** evaluate every relevant code path ONCE in fixed order (initialize → deposit/stake → process/swap → withdraw/unstake → claim → close). One-line verdict per path: `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`. Commit after all paths — do not re-examine. `UNCERTAIN` = `ALLOWS`.

3. **Lead promotion & rejection guardrails.**
   - Promote LEAD → FINDING (confidence 75) if: complete exploit chain traced in source, OR `[agents: 2+]` demoted (not rejected) the same issue.
   - `[agents: 2+]` does NOT override a concrete refutation — demote to LEAD if refutation is uncertain.
   - No deployer-intent reasoning — evaluate what the code _allows_, not how the deployer _might_ use it.

4. **Fix verification** (confidence >= 80 only): trace the attack with fix applied; verify no new DoS, CPI failures, or broken invariants; list all locations if the pattern repeats. If no safe fix exists, omit it with a note.

5. **Format and print** per `report-formatting.md`. Exclude rejected items. If `--file-output`: also write to file.

## Banner

Before doing anything else, print this exactly:

```

███████╗ ██████╗ ██╗      █████╗ ███╗   ██╗ █████╗      █████╗ ██╗   ██╗██████╗ ██╗████████╗ ██████╗ ██████╗
██╔════╝██╔═══██╗██║     ██╔══██╗████╗  ██║██╔══██╗    ██╔══██╗██║   ██║██╔══██╗██║╚══██╔══╝██╔═══██╗██╔══██╗
███████╗██║   ██║██║     ███████║██╔██╗ ██║███████║    ███████║██║   ██║██║  ██║██║   ██║   ██║   ██║██████╔╝
╚════██║██║   ██║██║     ██╔══██║██║╚██╗██║██╔══██║    ██╔══██║██║   ██║██║  ██║██║   ██║   ██║   ██║██╔══██╗
███████║╚██████╔╝███████╗██║  ██║██║ ╚████║██║  ██║    ██║  ██║╚██████╔╝██████╔╝██║   ██║   ╚██████╔╝██║  ██║
╚══════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝    ╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝

```
