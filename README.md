# confound-check

**Stress-test causal claims before anyone believes them.**

A skill for AI agents that reviews analyses claiming one thing caused another — A/B tests,
observational studies, backtests, metric-movement postmortems — and reports which rival
explanations are still live, ranked, each with the specific check that would settle it.

Works with Claude Code, Claude Desktop, Cowork, and anything else that reads `SKILL.md`.

---

## The idea

Most confound review fails in one of two directions: it finds nothing, or it lists every
textbook bias and ranks none of them. Both get ignored.

This skill is built on one rule:

> **Naming a confound is worthless without the observation that would distinguish it.**

"This might be seasonality" is not a finding. "If it were seasonality, last year's same
week shows the same bump — here is the query" is. Every rival gets a discriminating test or
it stays live and the claim gets softened.

The second rule does most of the remaining work:

> **If you cannot name the units that did *not* get the treatment, and say why they were
> comparable, there is no causal estimate.**

## What it does

| Step | |
|---|---|
| §1 | Force the claim into `[treatment] on [units] changed [outcome] by [magnitude] over [window]`. An empty slot is itself the finding. |
| §2 | Write both arms of the comparison, then list what they differ by *besides* the treatment. That list is the confound list. |
| §3 | Run eight fixed rival generators. Every one produces a written line — `N/A — <reason>` counts, silence does not. |
| §4 | Commit each rival to a ledger with a runnable `CHECK`, a decisive `EXPECT`, and recorded `EVIDENCE`. |
| §5 | Gates (pass/fail preconditions) first; then rank surviving rivals by how much of the effect they could absorb. |
| §6 | Audit the controls themselves — post-treatment, mediator, and collider adjustment are bugs, not rigor. |
| §7 | Verdict: Not a causal claim / Confounded / Survivable / Clean, with the ledger tally. |
| §8 | Re-run the ledger in a fresh context that never read the analyst's argument. |

Ships with confound catalogs for observational data, experiments, and backtests; a set of
executable diagnostics; and a list of pipeline bugs that manufacture confounds
(lookahead joins, post-assignment filters, universes built from live-entities-only tables).

## Why the ledger

A rival argued away in prose is not resolved. The ledger makes clearance earnable:

```markdown
- [ ] R1 [Killer]: treated units are self-selected on intent
  CHECK: <command producing standardized differences on pre-treatment covariates>
  EXPECT: /max SMD = 0\.0[0-9]/
  EVIDENCE: pending
```

A checked box whose `EVIDENCE` still reads `pending` counts as **unmet — and worse than
unchecked**, because it asserts a clearance nobody earned. A rival that cannot be tested
gets `ABANDON: <id> <reason>` and is surfaced in the verdict. It is never silently dropped.

Two rules keep the checks honest:

- **Positive-control every absence.** Before trusting "no imbalance found," verify the check
  can detect imbalance when it is there. A balance table that looks clean because a bad join
  dropped 90% of rows is indistinguishable from clean randomization.
- **Re-measure every number at report time.** Any figure in the verdict gets re-derived from
  source as the verdict is written, never quoted from memory.

## Install

### Claude Desktop / claude.ai (personal)

Settings → **Customize → Skills** → upload `confound-check.zip`.

### Claude Code — personal (all your projects)

```bash
git clone https://github.com/<you>/confound-check ~/.claude/skills/confound-check
```

### Claude Code — project (shared with your team via git)

```bash
mkdir -p .claude/skills
git clone https://github.com/<you>/confound-check .claude/skills/confound-check
rm -rf .claude/skills/confound-check/.git
git add .claude/skills/confound-check && git commit -m "Add confound-check skill"
```

Project skills load from `.claude/skills/` in the directory Claude Code starts in and every
parent up to the repo root, so a skill at the root is picked up from any subdirectory.

### Organization-wide (Team / Enterprise, admin only)

**Organization settings → Skills → + Add**, and upload `confound-check.zip`.

Requires **Code execution and file creation** and **Skills** to be toggled on in the same
settings page. Once uploaded it provisions to everyone in the organization immediately,
enabled by default, and each user can toggle it off individually.

### Anything else

`SKILL.md` is plain markdown. Paste it as a system prompt, a Cursor rule, or a preamble.

## When it fires

On causal language over data that was not randomized — *drove, caused, led to, because of,
the impact of, X% lift from, attributable to* — and on the softer tells: a time series with
a vertical line at a launch date, a self-selected group compared to "everyone else", the
phrase "we controlled for" with no list of what.

It also runs in **design mode** before a study, where the ledger becomes the
pre-registration, and as a **guardrail** that interrupts mid-analysis before a conclusion
gets written down.

## What it will not do

Manufacture doubt. A well-randomized experiment with a clean sample ratio and a
pre-registered metric gets `Clean`, and the review says what it checked. Reflexive
skepticism trains people to ignore the reviewer, which costs more than it saves.

## Credit

The ledger, `ABANDON`, positive-controlled absence checks, and the report-time numbers rule
adapt the enforcement model of [unlazy](https://github.com/Leonxlnx/unlazy) by Leonxlnx
(MIT) — prose cannot enforce prose, so completion claims belong in a file and get decided by
commands.

## License

MIT
