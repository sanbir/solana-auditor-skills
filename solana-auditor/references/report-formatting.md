# Report Formatting

## Report Path

Save the report to `assets/findings/{project-name}-solana-ai-audit-report-{timestamp}.md` where `{project-name}` is the repo root basename and `{timestamp}` is `YYYYMMDD-HHMMSS` at scan time.

## Output Format

````
# 🔐 Security Review — <ProgramName or repo name>

---

## Scope

|                                  |                                                        |
| -------------------------------- | ------------------------------------------------------ |
| **Mode**                         | ALL / default / filename                               |
| **Framework**                    | Anchor / Native Rust / Pinocchio                       |
| **Files reviewed**               | `File1.rs` · `File2.rs`<br>`File3.rs` · `File4.rs`    | <!-- list every file, 3 per line -->
| **Attack vectors checked**       | 120 (across N agents)                                  |
| **Agents deployed**              | N vector-scan + adversarial + protocol                 |
| **Confidence threshold (1-100)** | N                                                      |

---

## Findings

[95] **1. <Title>**

`program::instruction_handler` · Confidence: 95

**Description**
<The vulnerable code pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

[82] **2. <Title>**

`program::instruction_handler` · Confidence: 82

**Description**
<The vulnerable code pattern and why it is exploitable, in 1 short sentence>

**Fix**

```diff
- vulnerable line(s)
+ fixed line(s)
```
---

< ... all findings >

---

Findings List

| # | Confidence | Title |
|---|---|---|
| 1 | [95] | <title> |
| 2 | [82] | <title> |
| | | **Below Confidence Threshold** |
| 3 | [75] | <title> |
| 4 | [60] | <title> |

---

> ⚠️ This review was performed by an AI assistant. AI analysis can never verify the complete absence of vulnerabilities and no guarantee of security is given. Team security reviews, bug bounty programs, and on-chain monitoring are strongly recommended.

````

**Rules:** Follow the template above exactly. Sort findings by confidence (highest first). Findings below the threshold get a description but no **Fix** block. Draft findings directly in report format — do not re-generate.
