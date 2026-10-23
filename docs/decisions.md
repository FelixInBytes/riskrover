# RiskRover — Decision Log

One entry per decision reached while sharpening the idea. Each records what we chose, why,
what we rejected, and whether it can be cheaply reversed. Vocabulary is defined in
[`../CONTEXT.md`](../CONTEXT.md).

Status at time of writing: **design only. No code written.** The first build step (D15) is
agreed but deliberately deferred.

---

## D1 — Domain: risk-based test *strategy*, not financial risk

**Decision.** RiskRover consumes historical e2e test reports and produces a test-run policy
for CI. The initial reading of the name (portfolio risk reporting) was abandoned.

**Why.** The originating goal is to identify flaky/failing/healthy tests and decide what
should run and how often — an orchestration problem over test history.

**Rejected.** A local-first finance CLI that reads a holdings file and prints a risk report.

**Reversible.** N/A — this is the premise.

---

## D2 — Advise only; no write access to CI

**Decision.** RiskRover computes and publishes a policy. It does not trigger runs, does not
own scheduling, and carries no CI credentials.

**Why.** Acting on CI means owning concurrency, cancellation, dedupe, and partial-failure
semantics — a distributed-systems problem that would swamp the actual interesting work (the
deciding algorithm). Advising keeps the hard part a pure function over history.

**Rejected.** Advise + trigger; a blocking gate; a full orchestrator that dispatches runs.

**Reversible.** Yes, in principle — but adding "act" later is additive, not a rewrite.

---

## D3 — Input: JUnit XML from a single CI

**Decision.** Ingest JUnit XML artifacts plus commit SHA and per-test attempt information.
Normalize to `(repo, suite, test_id, status, duration, attempt, commit, run_timestamp)`.

**Why.** De-facto interchange format; Pytest/Jest/JUnit/Playwright/Go/.NET all emit it, so
multi-framework support is nearly free and needs no justification.

**Rejected.** Raw console logs (a parser project first); a bespoke JSON export (no reason to
invent a schema when a standard one exists); multi-CI support (per-provider adapters).

**Reversible.** Yes for additional sources; the normalized record is the stable part.

---

## D4 — Output: per-test score plus tiered policy; "the line" = threshold

**Decision.** Each test gets a `score` in `[0,1]`; configured thresholds cut that range into
tiers. Thresholds are config, reported per response as `thresholds_used`.

**Why.** When someone asks "why is this running on every PR?", the answer must be
`score 0.83 ≥ must_run line 0.80`, not "it was near the top". A line is only meaningful if
it can be pointed at. The original phrase "set a line for a CI chain" was clarified to mean
a policy line, not a single pass/fail gate.

**Rejected.** A single gate threshold (discards per-test detail); budgeted selection
("best tests in 20 minutes" — powerful, but needs trustworthy durations and hides the
decision inside an optimiser); a bare frequency schedule.

**Reversible.** Thresholds yes. The score+tier shape would be a breaking API change.

---

## D5 — Signals: report-derived only, behind a `Signal` seam

**Decision.** `score = f(signals)` where each signal is an independent module contributing
evidence: flakiness, recent failure rate, trend, time since last green, duration. A manual
criticality override is allowed. Change-awareness is deferred and will be added as one more
signal.

**Why.** The question is not *which* signals but whether more can be added without rewriting
the scorer. One deep module — `score(history) -> Score` — keeps each signal's quirks behind
a small interface.

**Rejected.** Change-aware scoring in v1 (needs coverage maps or a maintained test→path
mapping first — a data-collection project before it is an algorithm project); failure-rate
only (ignores trend and criticality); deriving criticality automatically from repo metadata.

**Reversible.** Yes — this is the point of the seam.

---

## D6 — Storage and deployment: one binary, embedded SQLite

**Decision.** A single binary with an `ingest` path and a `plan` query path, storing
normalized history in SQLite on disk, deployed as one container.

**Why.** CI artifact retention is typically 30–90 days, so RiskRover must own its history;
that is the feature that makes a stable score possible. At ~300 tests × 90 nights (~27k
rows) this is four orders of magnitude below where SQLite is a constraint, and the whole
store stays a single copyable file that can be used to reproduce a policy decision.

**Rejected.** A stateless CLI re-reading raw XML each invocation (reprocessing cost, and it
only sees what the job handed it); service + Postgres (more ops at a scale we do not have);
a library with no persistence opinion.

**Reversible.** Yes — the store is a seam, not a rewrite.

---

## D7 — Integration: live HTTP endpoint, with evidence age

**Decision.** CI queries `GET /plan?repo=<repo>&window=<window>`. The response carries, per
test, `score`, `tier`, `health`, `signals`, `n`, `thresholds_used`, plus `generated_at` and
the age of the newest ingested data.

**Why.** SQLite lives with the service and CI runners are ephemeral and concurrent, so a
binary inside the job would need the database mounted over the network. Serving live also
makes staleness visible: if ingest dies, the answer can fail loudly instead of silently
getting older. A committed policy file cannot distinguish fresh from stale.

**Rejected.** A generated `policy.json` file (easiest arbitrary-CI wiring, but a stale policy
is indistinguishable from a fresh one); a CLI invoked in the job (locking and availability
problems); both file and endpoint (redundant for now).

**Reversible.** Yes; the file variant could be added as a rendering of the same policy.

---

## D8 — Scope: multi-repo within one org, one instance

**Decision.** `repo` is a first-class column and a required query parameter. Ingest reads a
configured list of repo→artifact-source pairs. No tenant isolation, no per-org auth.

**Why.** A team's end-to-end suite rarely lives in a single repo — API e2e, browser e2e, and
a shared harness are commonly separate. Retrofitting a repo dimension later means re-keying
every stored record, and faithfully impossible for history that never recorded it.

**Rejected.** Single-repo only (would need the dimension anyway); multi-tenant SaaS
(a business decision that changes auth and isolation entirely).

**Reversible.** Adding tenants later is additive; removing the repo key would not be.

---

## D9 — Two axes: `score` and `health`

**Decision.** Model `health ∈ {healthy, flaky, broken, unknown}` independently of `score`.
`health` is derived from the *pattern* of outcomes; `score` from their magnitude. Tier is
derived from both, with `health` gating.

**Why.** `pass fail pass` is proof of nondeterminism; `fail fail fail` is proof of
determinism — deterministically bad. Rate cannot separate them. On a single failure-rate
score both land in `must_run`, permanently paying for a test that cannot pass.

**Rejected.** One score only (a dead test and a flake become indistinguishable); health
verdict only (no ordering within a class); human-assigned labels (defers the core problem).

**Reversible.** No, not really — everything downstream consumes both axes.

---

## D10 — Cold start: no-op until proven, with backfill

**Decision.** Insufficient evidence ⇒ `health = unknown`, `tier = baseline`. Every signal
carries `n`. Backfill whatever artifacts CI still retains so the window fills immediately.

**Why.** If CI currently runs everything nightly, `baseline` *is* current behaviour, so the
tool's first act changes nothing. That buys a safe observation period. Propagating `n`
prevents a 3-observation score from being trusted like a 300-observation one.

**Rejected.** Default everything to `must_run` and let evidence demote (no savings until
tests earn their way down; quarantining requires having already been burned); refuse to
answer until warm (honest but useless for a month); mirror the current CI config.

**Reversible.** Thresholds yes; "cold start is a no-op" is argued as an invariant.

---

## D11 — Constraint: one attempt per test per night (fact, not choice)

**Observed.** Each nightly job runs every test exactly once. No retries, no attempts
recorded, no sharding.

**Consequence.** Flakiness can never be *proven* within a run; it is inferred across nights.
`pass fail pass` across three commits is indistinguishable from a regression that arrived on
one commit and was fixed on the next.

**Mitigation.** The ≥2-flip rule (D12), and eventually change-awareness (D5) to discount
flips that coincide with relevant source changes.

---

## D12 — `flaky` requires ≥2 flips

**Decision.** `flaky` needs ≥2 outcome transitions within the window. One flip is reported
(`flips: 1`) but does not change the tier.

**Why.** A single flip is fully explained by a normal break-then-fix regression. Two or more
flips means the test recovered *and* failed again with no fix cycle in between, which a
clean regression cannot produce. Costs detection latency; buys a low false-positive rate.

**Rejected.** One flip (mislabels break-then-fix regressions); confirm-by-re-probe (needs a
re-run mechanism the CI does not have); human confirmation before any flaky verdict
(safer, but makes quarantine manual and graveyards accumulate).

**Reversible.** Yes — it is a configurable threshold.

---

## D13 — Test identity: exact key, no rename detection

**Decision.** Identity is `(repo, suite, classname, name)`. `file` is stored as metadata,
not part of the key. A rename produces a new test with `health = unknown`,
`tier = baseline`, and a fresh `first_seen`.

**Why.** Rename detection is guesswork whose failure mode is silently attaching a flake's
history to an unrelated healthy test — misattribution nobody notices until the policy is
wrong. An honest reset costs two weeks of evidence on one test; a bad match poisons the
score invisibly. `file` is excluded because it is optional in JUnit and emitters disagree on
including it, which would split identities based on reporter config.

**Rejected.** `file` in the key; fuzzy rename matching; human-maintained stable IDs
(most robust, but demands annotation buy-in and code changes).

**Reversible.** Yes, at the cost of re-keying stored history.

---

## D14 — Tiers: four, with an automatic exit from quarantine

**Decision.** `must_run`, `elevated`, `baseline`, `quarantine`. Quarantine exits
automatically after K consecutive passes; history is preserved. **No `retire` tier.**

**Why.** The useful advice is graded, not binary: "run this more often than baseline" is the
original ask. Quarantine without an exit is a one-way door and is how these tools die — a
silent graveyard nobody trusts. An auto-exit rule makes the policy self-healing and makes
quarantine politically acceptable ("paused, and here is what un-pauses it"). `retire` is
rejected outright: a test that has not failed recently is the most plausible catcher of the
next regression, and deletion advice has silent, unbounded downside against a few minutes of
CI saved.

**Rejected.** Five tiers including `retire`; three tiers (loses graded frequency); sticky
manual quarantine release.

**Reversible.** Adding a tier later is additive.

---

## D15 — First build step: throwaway scoring spike

**Decision.** Before any service code, write a single-file Rust binary that replays a
directory of real JUnit XMLs in date order and prints
`test_id | runs | fails | flips | health | score | tier`. **Deliberately deferred — not
started.**

**Why.** Every threshold so far (2 flips, 3 failures, 10 observations, 30 nights) is an
invented default. Reading the tier column against tests known to be flaky or known-good
falsifies them in an hour, cheaply. Once the scorer is welded to SQLite and an HTTP
response, that experiment is expensive. The spike is explicitly disposable: its job is to
answer a question, not to become `src/scoring.rs`.

**Rejected.** A thin vertical slice first (deployable early, but spends days on plumbing
before learning whether the idea works); ingest-and-model first; building full v1 and
tuning in production.

**Reversible.** Yes — it is throwaway by construction.

---

## Defaults adopted without asking (ratify or change these)

These were assumed while writing `CONTEXT.md`. None has been confirmed by the owner.

| Default | Value | Note |
|---|---|---|
| Window | 30 nights, per-repo configurable | Matches typical nightly cadence |
| Minimum observations for any verdict | 10 | Below this ⇒ `unknown` |
| `broken` threshold | ≥3 consecutive trailing failures | Conservative; configurable |
| Quarantine exit | ≥5 consecutive passes | K in D14, unvalidated |
| Tier boundaries | `must_run` 0.80 / `elevated` 0.55 / `quarantine` 0.70 | Invented; the spike exists to falsify these |
| Ingest trigger | Cron inside the container | Alternative: CI pushes artifacts to an endpoint |
| Auth | Single static bearer token | Multi-tenant auth is out of scope per D8 |
| Unreachable-service fallback | Consumer treats it as `baseline` | Documented, not enforced |

---

## Open questions

1. **Change-awareness source.** Coverage maps (reliable, expensive to collect) or a
   maintained test→path mapping (cheap, drifts). This determines whether the signal is
   trustworthy or theatre.
2. **Artifact fetch.** CI API token + pagination vs a push endpoint. Affects ingest design
   more than any other open question.
3. **Score formula.** Weighted sum of normalized signals vs a learned/calibrated model. A
   weighted sum is explainable, which invariant 4 demands; the weights still need a story.
4. **Criticality override.** Where it lives (config file? repo annotation?) and whether it
   can dominate the score cap.
5. **Is this a product?** One org today; the answer changes auth, isolation, and everything
   D8 deliberately deferred.
