# RiskRover — Context

The single source of truth for what RiskRover is and what its words mean.
Read this before touching code; if a term here conflicts with code, the code is wrong.

Decisions and their rationale live in [`docs/decisions.md`](docs/decisions.md).

---

## What it is

RiskRover ingests historical end-to-end test reports (JUnit XML from a single CI) and
derives a per-test **risk policy**: which tests must run on every pipeline, which deserve
to run more often, which are safe on the baseline schedule, and which are flaky enough to
be quarantined.

It is **advise-only**. It computes and publishes the policy over HTTP. It never triggers
runs, never mutates CI configuration, and never holds CI credentials.

Despite the name, this has nothing to do with financial risk. "Risk" here means
*the chance that a test's current result is untrustworthy or that skipping it hides a
defect*. The name predates the domain and was kept.

## What it is not

- **Not a test runner.** It consumes results, never produces them.
- **Not a CI gate.** It does not pass or fail anything. A consumer may gate on its output;
  that is the consumer's decision, not RiskRover's.
- **Not a dashboard.** It answers "what should run?", not "show me graphs".
- **Not a deletion advisor.** It will never recommend removing a test. See invariant 2.
- **Not a rename tracker.** A renamed test is a new test. See invariant 7.
- **Not change-aware (yet).** It knows nothing about which source files changed. This is the
  single biggest limitation and the planned next increment.

---

## Ubiquitous language

| Term | Meaning |
|---|---|
| **Repo** | One source repository. First-class key: every stored record and every query carries one. |
| **Suite** | The report-producing group within a repo (one JUnit file / one nightly job). Part of a test's identity. |
| **Test** | A single test case, identified by `(repo, suite, classname, name)`. |
| **Test key** | That 4-tuple. Exact match only. Never fuzzy-matched. |
| **Night** | One execution of the nightly job for a repo. The unit of ingestion. Carries a commit SHA and a timestamp. |
| **Observation** | One `(test, night)` outcome: pass or fail, plus duration. **Exactly one per test per night** — the CI does not retry. |
| **Window** | The rolling set of nights used to score a test. Default 30 nights; per-repo configurable. |
| **Flip** | A transition between differing outcomes in a test's observation sequence: `pass→fail` or `fail→pass`. |
| **Health** | The *verdict* about a test's determinism: `healthy`, `flaky`, `broken`, `unknown`. Derived from the **pattern** of outcomes, not the rate. |
| **Score** | A number in `[0,1]` expressing how much the test currently deserves attention/execution. Derived from signals. Orthogonal to `health`. |
| **Signal** | One independent, named piece of evidence contributing to the score (e.g. `flip_rate`, `recent_failure_rate`). Every signal carries `n`. |
| **`n`** | Sample size behind a signal or verdict. Propagated to the consumer so a 3-observation score is never mistaken for a 300-observation one. |
| **The line** | A configured threshold that cuts the score range into tiers. The line is always reported in the response as `thresholds_used`, never implicit. |
| **Tier** | The policy verdict: `must_run`, `elevated`, `baseline`, `quarantine`. |
| **Policy / Plan** | The full response: a tier per test, each with score, health, signals, and the thresholds used. |
| **Evidence age** | Time since the newest ingested night. Published with every policy so staleness is visible. |

### Health

| Verdict | Rule | Meaning |
|---|---|---|
| `healthy` | No flips, not currently failing | Passing consistently |
| `flaky` | ≥2 flips in window | Nondeterministic. Cannot be trusted as a gate. |
| `broken` | ≥3 consecutive trailing failures | Deterministically failing. Not flaky — needs a fix or a decision. |
| `unknown` | <10 observations, or otherwise insufficient evidence | No claim. Defaults to `baseline`. |

A single flip is *reported* (`flips: 1`) but does not change the verdict. A single flip is
fully explained by a normal break-then-fix regression.

### Tiers

| Tier | Meaning |
|---|---|
| `must_run` | Run on every pipeline. |
| `elevated` | Run more often than the baseline schedule. |
| `baseline` | Run on the existing schedule. |
| `quarantine` | Flaky: pause recommended execution; do not let it gate. **Exits automatically** after ≥K consecutive passes; history is preserved. |

---

## Invariants

These are the rules that make the tool trustworthy. Breaking one is a bug, not a trade-off.

1. **Cold start is a no-op.** With insufficient history, the answer must be
   `health = unknown`, `tier = baseline`. Deploying RiskRover must not change CI behaviour
   on day one. A tool whose first act is "run everything forever" is indistinguishable
   from a broken tool.
2. **Never recommend deletion.** A test that has not failed in the window is not
   evidence that it is useless — it is the most plausible candidate for catching the next
   regression. Absence of failure is not absence of value.
3. **`broken` never reaches `must_run`.** A deterministically failing test must not be
   promoted for more execution; that pays full cost for zero information.
4. **Every tier ships its evidence.** A response carries `score`, `health`, `signals`,
   `n`, and `thresholds_used`. A tier without its evidence is an oracle, and oracles lose
   trust the first time they surprise someone.
5. **`flaky` requires ≥2 flips.** One flip is a normal break-then-fix regression. Calling
   healthy tests flaky is the fastest way to lose the team's trust.
6. **Quarantine has an exit.** No one-way doors. Automatic re-entry after K consecutive
   passes, with history preserved so the score does not reset.
7. **A rename is a reset, and it is visible.** New `first_seen`, `health = unknown`. No
   fuzzy matching: silently attaching one test's history to another is worse than an
   honest loss of evidence.
8. **Advise only.** No CI writes, no triggered runs, no credentials. The consumer decides
   what to do with advice.
9. **Staleness is explicit.** Every policy carries `generated_at` and evidence age. A stale
   answer must never be mistakable for a fresh one.

---

## The model: two axes, deliberately

`health` and `score` are orthogonal because a single number cannot separate these two tests,
both failing ~40% of nights:

```
A: pass fail pass fail pass   → flaky    → quarantine / characterise
B: fail fail fail fail fail   → broken   → fix or make a decision
```

A failure-rate-only score promotes both to `must_run`, forever paying for a test that
cannot pass. `health` comes from the *pattern*; `score` comes from the *magnitude*. The tier
is derived from both, with `health` acting as a gate.

## Pipeline

```
JUnit XML artifacts ──ingest──▶ SQLite (normalized history) ──score──▶ policy ──HTTP──▶ CI
```

- **Ingest** pulls artifacts for a configured list of repos, normalizes them into
  observations, and appends. It owns the history so the policy survives CI artifact
  expiry (typically 30–90 days).
- **Score** is a pure function of `(test history, config)`. No side effects, no I/O. This is
  the part worth testing hardest.
- **Query** serves the policy over HTTP. Embedded SQLite, one binary, one container.

### Response contract (shape, not final)

```
GET /plan?repo=<repo>&window=30d

{
  "repo": "api",
  "generated_at": "...",
  "evidence_age_hours": 4.2,
  "thresholds_used": { "must_run": 0.80, "elevated": 0.55, "quarantine": 0.70 },
  "tests": [
    {
      "test_id": "api::e2e::checkout.spec::completes purchase",
      "tier": "must_run",
      "score": 0.83,
      "health": "healthy",
      "n": 30,
      "first_seen": "...",
      "signals": { "flip_rate": {...}, "recent_failure_rate": {...} }
    }
  ]
}
```

## Known weaknesses

| Weakness | Consequence | Resolution path |
|---|---|---|
| No change-awareness | Cannot distinguish "unexplained flake" from "regression that got fixed twice". Flips are a weaker signal than they could be. | Next increment: a `changed_since_last_run` signal, from coverage maps or a maintained test→path mapping. |
| One attempt per night | Flakiness is never *proven*, only inferred across nights. | Same as above; a re-probe ("run this 5×") advice is a possible later addition. |
| Thresholds are invented defaults | 2 flips, 3 failures, 10 observations, 30 nights are plausible, not validated. | The pre-build spike exists to falsify them against real reports. |
| Rename resets history | Recovering costs ~2 weeks of evidence for that test. | Accepted; visibility is the mitigation. |
| Availability coupling | CI integration makes RiskRover's uptime a CI concern. | Consumers must define a fallback (default: treat unreachable as `baseline`). |
| Single tenant | No isolation or per-org auth. | Out of scope until it becomes a product question. |

## Next step (not started)

A throwaway single-file Rust spike: replay a directory of real JUnit XMLs in date order,
print `test_id | runs | fails | flips | health | score | tier`, and read it against tests
known to be flaky or known-good. Its only job is to falsify the invented thresholds. It is
deleted afterwards; the real `score` module is then built behind a proper interface.
