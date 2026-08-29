---
name: confound-check
description: Stress-test causal claims for confounds before anyone believes them. Use when an analysis concludes that one thing caused, drove, lifted, hurt, or explained another; when reviewing A/B test results, observational analyses, backtests, or metric-movement postmortems; when designing a study, experiment, or backtest so controls get decided upfront instead of defended afterward; and whenever causal language ("drove", "because of", "the impact of", "X% lift from", "attributable to") appears over data that was not randomized. Also use to audit queries, notebooks, and pipelines for structural bugs that manufacture confounds — lookahead bias, survivorship, mix shift, post-treatment controls.
---

# Confound Check

## The rule

A confound is any explanation, other than the claimed cause, that produces the same
observed pattern.

Two consequences drive everything below:

1. **Naming a confound is worthless without the observation that would distinguish it.**
   Every rival explanation gets a discriminating test, or it stays live and the claim
   gets softened. "This could be confounded by seasonality" is not a finding. "If it were
   seasonality, last year's same-week data would show the same bump — check it" is.

2. **If you cannot name the units that did *not* get the treatment, and say why they were
   comparable, there is no causal estimate.** There is a description with a causal verb
   attached to it.

## Mode

Route on what already exists. Do not run the full pass when a shorter one is called for.

| Situation | Mode | Run |
|---|---|---|
| Nothing measured yet | **Design** | §1, §2, §6, then write the §4 ledger as the pre-registration: arms, controls, the confounds being accepted, and the checks that will decide each one |
| A result exists and is being reviewed | **Review** | §1–§8, full |
| Causal language appeared mid-analysis | **Guardrail** | §1 and §2 only, no ledger. If §2 produces an untested list, stop and surface it *before* the conclusion is written down |

Guardrail triggers — fire on these in any draft, comment, commit message, or chart title
over non-randomized data: *drove, caused, led to, resulted in, because of, thanks to,
the impact of, X% lift from, responsible for, attributable to, moved the needle on*.
Softer tells: a time series with a vertical line at a launch date; a self-selected group
compared to "everyone else"; the phrase "we controlled for" with no list of what.

---

## §1 Restate as a causal claim

Force the claim into this shape:

> **[Treatment]** applied to **[units]** changed **[outcome]** by **[magnitude]** over **[window]**.

An empty slot is itself the finding — report it and stop rather than proceeding:

- **No treatment** → descriptive claim wearing causal clothes. Correlation reported, causation implied.
- **No units, or "our users"** → population undefined, so selection is unconstrained.
- **No window** → likely a before/after with a floating boundary, chosen after seeing the data.
- **No magnitude, only direction** → often means the effect did not survive being quantified.

## §2 Name the comparison, then diff it

This is the highest-yield step. Every causal claim is implicitly a comparison. Write both
arms out, then list **everything the arms differ by other than the treatment.**

That list *is* the confound list. Produce it before consulting any catalog.

| Comparison type | Arms | Where the differences hide |
|---|---|---|
| Treated vs. untreated | who got it / who didn't | how units were selected into treatment |
| Before vs. after | pre-window / post-window | everything else that changed on that date |
| Adopters vs. non-adopters | used feature / didn't | adoption is an outcome, not an assignment |
| Cohort vs. cohort | joined Jan / joined Jun | acquisition channel, price, product version |
| In-sample vs. out-of-sample | fitted / held out | regime, liquidity, universe composition |
| Region vs. region | A / B | rollout order is rarely random |

If the honest answer to "how did units end up in the treated arm?" is *they chose*,
*we picked them*, or *whoever was left*, selection is the primary confound and every
downstream number inherits it.

## §3 Generate rivals

Run all eight generators against the claim. Do not free-associate; the point of a fixed
list is catching the one nobody thought of.

| # | Generator | Ask |
|---|---|---|
| 1 | **Selection** | How did units enter the treated group? Who is missing from the data entirely? |
| 2 | **Common cause** | What third thing moves both treatment and outcome? |
| 3 | **Reverse / simultaneity** | Could the outcome have caused the treatment, or both move together? |
| 4 | **Concurrent change** | What else happened in that window? Releases, pricing, seasonality, macro, a competitor, a holiday, a marketing push |
| 5 | **Mix shift** | Did the *composition* of units change, so every segment held steady while the aggregate moved? (Simpson's paradox) |
| 6 | **Measurement** | Did the definition, logging, instrumentation, or collection of the outcome change? Did it change *because* of the treatment? |
| 7 | **Attrition / survivorship** | Who dropped out, and was dropping out related to treatment or outcome? Who never made it into the dataset? |
| 8 | **Researcher degrees of freedom** | How many specifications, windows, segments, or thresholds were tried before this one? Was the cut chosen before or after seeing results? |

Generator 5 and generator 8 are the two most often skipped and the two that most often
turn out to be the whole story.

**Every generator produces a written line** — either a named rival or `N/A — <reason>`.
Silence is not a pass. A generator with no line is a generator that was skipped, and the
output must make the count visible: eight lines in, eight lines out. Partial compliance
with multi-part instructions is the documented failure mode of exactly this kind of
checklist, and it is invisible unless the format forces the tally.

**Regression to the mean** deserves its own check whenever units were selected on an
extreme value. "We targeted our worst-performing stores and they improved" is the
canonical false positive: the worst performers improve without any intervention, because
they were partly unlucky. If selection used an extreme of the same metric being tracked,
assume regression to the mean until a control group proves otherwise.

## §4 Write the ledger

For each surviving rival, first state the discriminator:

> If **[rival]** is the real story, we would also see **[specific observable]**.

Good discriminating observables are ones the claimed cause does *not* predict. If both
stories predict the same thing, the test is worthless.

Then commit it to a ledger **before** running anything. A rival argued away in prose is not
resolved; a rival is resolved when a named check produced a named result.

```markdown
# Confound ledger: <claim in one line>

Claim: [treatment] on [units] changed [outcome] by [magnitude] over [window]
Comparison: [arm A] vs [arm B]

- [ ] GATE-1: assignment ratio matches the intended split
  CHECK: <command computing the chi-square on observed vs. intended>
  EXPECT: /p = 0\.[1-9]/
  EVIDENCE: pending

- [ ] R1 [Killer]: treated units are self-selected on intent
  CHECK: <command producing standardized differences on pre-treatment covariates>
  EXPECT: /max SMD = 0\.0[0-9]/
  EVIDENCE: pending

- [ ] R2 [Material]: aggregate moved by mix shift, not within-segment change
  CHECK: <command running the fixed-weight decomposition>
  EXPECT: /within-segment share: 9[0-9]%/
  EVIDENCE: pending

ABANDON: R4 no pre-treatment data exists for this cohort; untestable, carried as a live limitation.
```

Rules that give the ledger teeth:

- **A checkbox is a claim; evidence is the proof.** A checked box whose `EVIDENCE` still
  reads `pending` counts as **unmet — and worse than unchecked**, because it asserts a
  clearance that was never earned.
- **`EXPECT` must be decisive.** Match a string that can appear only when the rival is
  ruled *out*. A pattern that appears either way tests nothing.
- **Never silently drop a rival.** If it cannot be tested, write
  `ABANDON: <id> <non-empty reason>` and surface it in the verdict. An abandoned rival is
  resolved as *unresolvable*, not as absent.
- **Measure figures independently.** Do not paste the analyst's reported n, effect size, or
  p-value into `EXPECT` and call the match a confirmation. That tests whether they can copy,
  not whether the number is right.
- **Manual entries are allowed** where no command can decide the outcome — quote the
  deciding lines or cite `file:line` as evidence. Never paste a log.

Skip the ledger for a guardrail interruption or a claim nobody will act on. Use it when
quiet incompleteness would cost something.

### Positive-control your discriminators

Before trusting any check that reports **absence** of a confound, verify the check can
detect that confound when it is present. Otherwise "we found no imbalance" is
indistinguishable from "our imbalance test is broken."

A balance table showing perfect balance because a bad join dropped 90% of rows reads
exactly like a clean randomization. Establish the positive control by injecting a known
imbalance, running the check on a period or cohort where the confound is known to exist, or
verifying the row counts the check actually consumed. Absence of evidence earns its keep
only from an instrument shown to be capable of producing evidence.

## §5 Gates first, then rank

### Gates

Some checks are preconditions, not rival explanations. They carry no "share of the effect"
to estimate — a failure says the measurement apparatus is broken, so no number produced by
it means anything. **Do not tier a gate.** One failed gate forces `Confounded` by itself.

| Gate | Fails when |
|---|---|
| Sample ratio mismatch | Observed split differs significantly from intended |
| Negative-control period | The analysis finds an effect in a pre-treatment window |
| Placebo treatment | A randomly assigned fake treatment produces an effect |
| Metric definition stability | The outcome was defined or logged differently across the compared periods |

Gates enter the ledger as `GATE-n` entries and are checked first. Report a failure as
`GATE FAILED — [name] — [debug action]`, above the ranked list, and say the ranking is moot
until it is fixed. Debug the apparatus; do not interpret the result.

### Ranking

Not every confound matters. Rank each surviving rival by **how much of the estimated effect
it could plausibly absorb**, not by how easy it is to name.

**Merge before ranking.** Two rivals that share one root cause and resolve with one action
are one finding. Name it for the defect, not the symptom. Split them only when the fixes
genuinely differ — data that was never knowable at the time and data that was knowable but
later revised are one finding if one point-in-time source fixes both, two if not.

| Tier | Meaning |
|---|---|
| **Killer** | Could plausibly account for the entire effect. Blocks the conclusion. |
| **Material** | Could absorb a meaningful share. Effect direction may survive; magnitude is unreliable. |
| **Noted** | Real but small relative to the effect. Mention and move on. |

A confound that could shift the estimate by 5% when the measured effect is 300% is
`Noted`. Say so and stop discussing it. Confound review that treats every rival as equally
threatening is as useless as review that finds none.

## §6 Audit the controls themselves

The failure mode nobody looks for: the adjustment is the bug.

| Bug | What happens | Rule |
|---|---|---|
| **Post-treatment control** | Adjusting for something the treatment caused removes the effect being measured | Only adjust for variables determined *before* treatment |
| **Mediator control** | Same, in the case where the variable is the mechanism | If the treatment works *through* it, do not control for it |
| **Collider control** | Adjusting for a common *effect* of two variables **creates** a spurious association | Never condition on a downstream common effect |
| **Bad-control chains** | Proxy variables that are themselves post-treatment | Check the timing of every control, not just its name |

Practical check: for each control variable, ask *when was this value determined, relative
to treatment?* Anything measured after treatment assignment is suspect until proven
otherwise. "We controlled for it" is not reassuring when the control is downstream.

If the analysis uses no adjustment or control variables at all, **say so explicitly** —
"no controls used, §6 N/A". A silent §6 is indistinguishable from a skipped one.

## §7 Verdict

Output one of four, with the reason attached:

- **Not a causal claim** — §1 failed. State what the analysis actually supports.
- **Confounded** — a gate failed, or a `Killer` is live. Name it, name the test that would
  settle it.
- **Survivable** — `Material` rivals remain but the effect's direction holds. State the
  caveat that must travel with the number, verbatim, wherever the number goes.
- **Clean** — comparison is defensible, rivals tested. State what was checked, so the
  claim is auditable later.

Paste the ledger tally — **N met / N unmet / N abandoned** — and surface every `ABANDON`
line with its reason. Do not compose a verdict while a required entry is unmet; an unmet
ledger *is* the finding.

**Re-measure every number at report time.** Any figure appearing in the verdict — effect
size, sample size, p-value, chi-square, share absorbed — gets re-derived from source as the
verdict is written, not quoted from earlier in the session. Numbers stated from memory are
where otherwise-sound reviews go wrong, and a confound review that misquotes the effect it
is adjudicating has no standing.

Always list what could *not* be checked. An unchecked rival is not an absent one.

## §8 Independent re-verification

Self-certification is weakest exactly where it matters most. Whoever ran the analysis has
already internalized its causal story, and that is the state in which §2 and §3 get answered
from the narrative instead of from the data.

For a claim that will drive a decision — a ship, a capital allocation, a published result —
re-run the ledger's checks in a fresh context that has **not** read the analyst's writeup.
Give it the claim, the data, and the ledger; withhold the argument. Then compare the two
evidence columns. A rival cleared in one pass and live in the other is a finding about the
review, not only about the analysis.

Re-running is the point. Reading a ledger's status is not re-executing its checks.

---

## Observational data

| Confound | Tell | Discriminating test |
|---|---|---|
| Self-selection into treatment | "Users who used X retained better" | Compare pre-treatment covariates across arms; find an instrument or a natural experiment |
| Regression to the mean | Units picked on an extreme of the tracked metric | Untreated control selected the same way, same window |
| Mix shift / Simpson's | Aggregate moves, no segment moves | Decompose: within-segment change vs. reweighting. Fixed-weight the mix and recompute |
| Concurrent change | Any before/after with a launch date | Negative-control outcome; a comparable population that got no treatment |
| Differential attrition | Sample size shrinks unequally across arms | Compare dropout rates and dropout characteristics by arm |
| Immortal time | Treated group had to survive long enough to be treated | Align time zero; treat exposure as time-varying |
| Ecological fallacy | Group-level correlation used for individual claim | Re-run at the unit level |
| Measurement change | Metric definition or logging changed mid-window | Recompute both periods under one definition |
| Reverse causation | Plausible either direction | Lead-lag ordering; does treatment precede outcome unit by unit? |

## Experiments and A/B tests

Randomization buys a lot, but it is a claim about the assignment process, not a
guarantee about the data that came back. Verify it was delivered.

| Confound | Tell | Discriminating test |
|---|---|---|
| **Sample ratio mismatch** | Arm sizes off from intended split | Chi-square on observed vs. expected split. **A significant SRM invalidates the test — debug it, do not interpret the result** |
| Assignment leakage / crossover | Users see both variants | Check assignment stability per unit; count crossovers |
| Interference between arms | Marketplace, social, or shared-resource product | Cluster-randomize; check whether control outcomes moved vs. pre-period |
| Novelty / primacy | Effect decays or grows monotonically after launch | Plot effect by day since exposure; hold out a late cohort |
| Peeking / optional stopping | Test stopped when it hit significance | Sequential-testing correction; check the stopping rule was pre-set |
| Multiple comparisons | Many metrics or segments scanned, one reported | Count every test run; correct, or pre-register the primary metric |
| Trigger/analysis mismatch | Analysis on triggered users, assignment at session start | Analyze intent-to-treat on the assigned population |
| Unequal exposure windows | Arms ran over different calendar days | Restrict to a common window |
| Confounded instrumentation | Logging shipped with the treatment | Check the control arm's event volume against the pre-period |

## Backtests and trading research

The trading-specific confound is that the "alpha" is a repackaged known exposure.
Assume it until the factor regression says otherwise.

| Confound | Tell | Discriminating test |
|---|---|---|
| **Factor confound** | Strategy long a style without meaning to | Regress strategy returns on market, size, value, momentum, sector, liquidity. Is alpha still there after? |
| Lookahead bias | Data used at time *t* that was not knowable at *t* | Point-in-time data; check every join key and every restated field |
| Survivorship | Universe built from names that exist today | Rebuild the universe as of each rebalance date, delistings included |
| Restatement bias | Fundamentals as currently reported | Use as-first-reported vintages |
| Regime confound | Test window is one regime | Split by regime; report per-regime, not pooled |
| Strategy selection | Many variants tried, best reported | Report the count of variants tested; deflate the Sharpe; hold out a period never touched |
| Overlapping samples | Multi-day holds sampled daily | Correct for serial correlation; effective sample size, not row count |
| Cost omission | Gross returns quoted | Model spread, impact, borrow, financing. Check whether edge survives at realistic size |
| Common liquidity driver | Signal and return both track volume/volatility | Control for the liquidity variable; test in a liquidity-matched subsample |

---

## Executable diagnostics

When code and data are available, run these rather than reasoning about them. Each maps
to a rival from §3.

| Diagnostic | Catches | Method |
|---|---|---|
| **Balance table** | Selection | Compare pre-treatment covariate distributions across arms |
| **SRM test** | Broken assignment | Chi-square, observed vs. intended split |
| **Negative-control outcome** | Concurrent change | Pick a metric the treatment cannot affect. If it moved, something else did |
| **Negative-control period** | Concurrent change, spurious fit | Run the identical analysis on a pre-treatment window. Effect should be zero |
| **Placebo treatment** | Spurious pipeline | Assign a fake treatment at random. Effect should be zero |
| **Mix decomposition** | Simpson's | Split aggregate change into within-segment vs. reweighting |
| **Permutation / shuffle** | Overfitting, researcher DoF | Shuffle labels many times; where does the real result sit in the null distribution? |
| **Leave-one-segment-out** | Single-driver effects | Drop each segment; does the conclusion survive all drops? |
| **Factor regression** | Factor confound | Regress returns on known factors; test whether intercept ≠ 0 |

Two of these — negative-control period and placebo treatment — catch bugs in the analysis
pipeline itself, not just confounds in the world. Run them even when the design looks
clean.

## Auditing code and queries

Structural mistakes that manufacture confounds, found by reading the pipeline:

- Joins on a current-value dimension table where a point-in-time one was needed
- Filters applied after treatment assignment (`WHERE status = 'active'` silently drops
  differential attrition)
- Universe built from a table that only holds live entities
- Feature columns computed over a window that includes the label period
- Control variables computed from post-treatment timestamps
- `dropna()` applied across a frame, dropping units non-randomly
- Metric definitions that changed in a dbt model between the two compared periods
- Any date boundary that is a literal, and was edited more than once

---

## Reviewer discipline

- **Do not list every textbook confound.** Only rivals that are plausible for *this*
  design, ranked. An exhaustive list signals that no thinking happened.
- **Do not accept "we controlled for it"** without checking when each control was
  determined relative to treatment. §6 exists because this is where reviews go soft.
- **A large effect does not excuse a broken comparison.** A large effect from a broken
  comparison is a large bias.
- **Do not manufacture doubt.** A well-randomized experiment with a clean SRM and a
  pre-registered metric deserves `Clean`. Reflexive skepticism is as unhelpful as
  reflexive belief, and it trains people to ignore the review.
- **Give the fix, not just the flaw.** Every `Killer` and `Material` finding ends with the
  specific check, control group, or data source that would resolve it.
- **Do not stop at the first two.** Finding a `Killer` early is the moment the remaining
  generators get skipped. Complete all eight regardless; the second `Killer` changes the
  fix, not just the verdict.

---

The ledger, `ABANDON`, positive-controlled absence checks, and the report-time numbers rule
adapt the enforcement model of the MIT-licensed
[unlazy](https://github.com/Leonxlnx/unlazy) skill by Leonxlnx — prose cannot enforce prose,
so completion claims belong in a file and get decided by commands.
