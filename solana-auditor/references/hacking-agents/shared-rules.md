# Shared Scan Rules

## Reading

Your bundle has two sections:

1. **Core source** (inline) — read in parallel chunks (offset + limit), compute offsets from the line count in your prompt.
2. **Peripheral file manifest** — file paths under `# Peripheral Files (read on demand)`. Read only those relevant to your specialty.

When matching function names, check both the Rust function name and the Anchor instruction handler name (which may differ via `#[instruction]` or snake_case convention). For native programs, check `process_instruction` dispatch arms and the individual handler functions.

## Cross-program patterns

When you find a bug in one instruction handler, **weaponize that pattern across every other handler and module in the bundle.** Search by function name AND by code pattern. Finding missing signer validation in `deposit()` means you check every other handler's account validation — missing a repeat instance is an audit failure.

After scanning: escalate every finding to its worst exploitable variant (DoS may hide fund theft). Then revisit every function where you found something and attack the other branches.

## Do not report

Admin-only functions doing admin things (guarded by `has_one`, `Signer` constraint on authority, or manual `key == stored_authority` checks). Standard Anchor safety features (8-byte discriminators, automatic owner checks on `Account<'info, T>`). Self-harm-only bugs. "Authority can rug" without a concrete mechanism. Missing event emissions (Anchor `emit!`). Compute-unit micro-optimizations.

## Output

Return structured blocks only — no preamble, no narration. Exception: vector scan agent outputs its classification block first.

FINDINGs have concrete, unguarded, exploitable attack paths. LEADs have real code smells with partial paths — default to LEAD over dropping.

**Every FINDING must have a `proof:` field** — concrete values, traces, or state sequences from the actual code. No proof = LEAD, no exceptions.

**One vulnerability per item.** Same root cause = one item. Different fixes needed = separate items.

```
FINDING | program: Name | handler: func | bug_class: kebab-tag | group_key: Program | handler | bug-class
path: caller → handler → state change → impact
proof: concrete values/trace demonstrating the bug
description: one sentence
fix: one-sentence suggestion

LEAD | program: Name | handler: func | bug_class: kebab-tag | group_key: Program | handler | bug-class
code_smells: what you found
description: one sentence explaining trail and what remains unverified
```

The `group_key` enables deduplication: `ProgramName | handlerName | bug_class`. Agents may add custom fields.
