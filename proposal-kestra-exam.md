# Proposal: `kestra-exam` — a spec-derived exam as the pre-delivery gate

**Status: request for reaction. Nothing described here is built.**
This document lives on a branch of my fork (`arkaphat/kestra`). It is not an upstream issue and not
a PR — it exists so you can shoot holes in the shape *before* I spend five PRs on it. If the shape
is wrong, or the delivery plan is wrong, saying so now costs you a comment and saves me the work.

---

## The gap this aims at

`design-principles.md` already names the territory, in the repo's own words:

> TDD only closes half the gap: a test is only as strong as the spec it was derived from. An edge
> case the spec never considered produces a test that never considered it either … That residual
> belongs to spec review / spec-to-test traceability, not to the stage machine.

The stage machine enforces *how* work happens — write-scope allowlist, test-hash freeze, commit per
stage, every decision from a command that actually ran. What no artifact in the repo does today is
check the delivered feature against **what was asked for**, independently of what was built. The
frozen tests are derived from the spec by the same pass that plans the build, at the same
granularity as the implementation, and they are the exit criteria for the stages that produce that
implementation.

`kestra-exam` adds one more mechanical, independently derived source of evidence at the end, in
service of the same claim the repo already makes: **it turns trust in the AI into trust in
evidence**, and it narrows and surfaces where you still have to trust it.

**What it explicitly does not claim.** Its coverage is *spec-derived* and nothing more. It does not
cover runtime invariants or the guards that enforce them — `design-principles.md` states that no
mechanical gate can carry that obligation, and the exam does not try to. A spec that never
considered a case produces an exam that never considers it either; the exam moves the check to a
second, independent derivation from the requirement text, not to omniscience.

---

## What `kestra-exam` is

A skill that, immediately after `kestra-spec` writes `0-spec.md`, produces **two artifacts**:

1. **An exam script** — a requirement-level **black-box e2e script**: executable, red/green, run at
   the external seam the spec declares. Not a checklist. Not a white-box unit suite. One check per
   acceptance criterion, never replicating the frozen tests' granularity — different level, so the
   overlap with `generate-tests` is minimal by construction.
2. **A single manifest** — one row per check: AC id → check id → class → provenance flag →
   red-proof result + failure signature → unexaminable reason. The verdict contract (pass/fail per
   check + evidence + AC coverage summary) rides as its final section, so it is a schema, not a
   third file. The raw red-proof log sits beside it as an attachment.

### Derived from the requirement surface only — blind to build planning

The blindness comes from an explicit read rule, not from where in the chain it runs. The exam may
read only: Functional Requirements, Edge Cases & Error States, Runtime Invariants, the AC Coverage
Map restricted to its `AC` + `Source` columns, and a new `## External Interface` section declaring
the user-facing seam. It must **not** read Files to Touch, the Codebase Survey, Solution
Architecture, or the Coverage Map's `Covered by (files/steps)` column — those are implementation
shape and would leak it into the exam.

### Red-first, with an honest failure signature

- **Check #0 is a harness smoke**, so "harness broken" is distinguishable from "feature fails."
- Checks carry a class: **`must-flip`** (red at creation → green at the gate; new behavior, and for
  a bug fix red *is* the repro) and **`must-hold`** (legitimately green at creation → must still be
  green; regression guards, not red-first failures).
- Red-proof records a **failure signature** separating *behavioral* red (harness reached the seam,
  assertion failed) from *infrastructure* red (crash before the seam) — an infrastructure red is
  not proof the check can ever go green.
- A **`must-flip` check that is born green is flagged `unproven` in the verdict** — never silently
  counted as passing, never auto-demoted to `must-hold`. The verdict may still go green; its
  evidence quality is stated rather than laundered.
- Red-proof runs in a disposable clone, so nothing lands in the working repo.

### Stored outside every worktree

The exam lives in a user-level exams directory outside every worktree, keyed by the repo's `origin`
URL and the feature slug — no `origin` remote is a hard stop at exam creation, with no fallback
naming. The only thing inside the project is a pointer on the tracker — an issue tracker of the
user's choosing, where the feature's spec ticket already lives; this repo imposes no tracker
requirement today, and standalone use stays tracker-free — opting into the exam (and the wider
chain) is what presupposes one. That pointer records path + content hash, latest-wins,
and regeneration edits it in place rather than opening a second one. Nothing about the exam lands in
`state.json` — not even an exists-flag; the gate learns an exam should exist from the spec's
recorded mode prediction.

Honest posture, stated up front: **nothing mechanically prevents a build-side agent from reading
the exam.** What is achievable is best-effort leak detection and tamper evidence, with the residuals
named rather than papered over. If you think the hiding isn't worth the machinery, that is exactly
the kind of pushback this document is for.

### Staleness refusal

Each derived artifact is anchored to a **triple**: the raise commit's SHA, a hash of the requirement
surface, and the version of the extractor that produced that hash (so a change to the hasher itself
can't quietly pass as "unchanged"). *Raise commit* = the second of the two commits `kestra-spec`
would make when materializing a spec — commit 1 records the source text verbatim, commit 2 (the
"raise") lands the pass's own sharpened additions; that second SHA is what the anchor points at.
The gate **refuses to emit a verdict when the exam is stale** relative to the spec, so a green
verdict can never certify against a superseded requirement. Prose, typo, and *provision-layer*
edits — the implementation-shaped sections (Codebase Survey, Files to Touch, Solution Architecture),
as opposed to the requirement surface — change no hash and regenerate nothing; regeneration is
delta-scoped by the AC→check map, so one changed AC does not re-prove the whole exam.

---

## Opt-in, and no new hard dependency

The exam is **opt-in**, keyed to `kestra-build`'s **full mode**. Mode is decided inside
`kestra-build` — later than the exam is authored, and with overrides — so `kestra-spec` records a
**mode prediction as a fact** in `0-spec.md`, and both `kestra-exam` and `kestra-build` follow that
fact without re-litigating it. A `lite` run has no exam, deliberately.

This keeps the repo's standing rule intact: **no hard cross-skill dependencies.** Running
`kestra-build` standalone over a hand-written `0-spec.md`, with no exam anywhere, keeps working
exactly as it does today. The chain's extra rigor is available, never mandatory.

Who *runs* the exam is deliberately unbound. `kestra-exam`'s `SKILL.md` documents the gate
procedure — the sweeps (the leak-detection searches that procedure defines: three over the repo,
one over the tracker, looking for the exam's content having leaked into the build side), the hash
comparison against the pointer record, the hard fail on more than one pointer — as the text an
eventual runner executes. **Building that runner is not part of this
work**, and I'd rather say so than imply a gate that doesn't exist.

---

## Delivery: five waves, five PRs

Each wave is one PR against `develop`, landing at the bar this repo already keeps: skill definition
+ every paired README surface (`workflow/README.md` ↔ `README-th.md`, and the group table in the
root `README.md` ↔ `README-th.md`) + an `install.sh` `SKILLS` entry + a dated eval directory under
`workflow/evals/` with measured numbers. Stdlib-only, no third-party dependencies, per the standing
rule.

| Wave | What lands | Skill files touched |
|---|---|---|
| **1** | **Measure first.** One instrumented re-run of the current-shape `kestra-spec` pass on a realistic fixture, with peak-context instrumentation, plus the thresholds that would tell us the one-pass shape has outgrown itself. | **None** — eval only |
| **2** | `kestra-spec`, exam-relevant slice: the three spec-side preconditions the exam depends on (`## External Interface`, a `Source` column in the AC Coverage Map, the recorded mode-prediction fact), plus the shared requirement-surface extractor and the `validate_spec.py` checks that enforce them. **Also in this PR** (not exam-driven): the input contract moves to a human-vetted spec ticket — a tracker issue holding the feature's intent, written by a companion skill that lives outside this repo and vetted by a human before any agent consumes it — and the in-chain clarifying-pass fallback is retired, materialization becomes two commits, and the template gains a `## Exit Criteria` section carrying the stop condition and `progress:` fragments for loop-shaped checks (single-shot checks carry none). | `kestra-spec` (+ the validator script) |
| **3** | **`kestra-exam` itself** — the new skill: read rule, exam script, manifest, red-first classes, anchor fields | new skill |
| **4** | `kestra-build` + `kestra-run`, exam-relevant slice: the staleness machinery — the `workflow.yaml` anchor, the pre-spawn surface check, mismatch as a hard stop in the same class as a test-hash mismatch. **Also in this PR** (not exam-driven): `kestra-build` folds a sliced ticket set — that same out-of-repo companion process also breaks the vetted spec ticket into one tracker ticket per slice — into one workflow and embeds those ticket bodies into stage briefs at build time, copies each `progress:` metric from the spec into the owning stage's `exit_criteria`, and `kestra-run` gains a new enumerated stop condition — two consecutive rounds with no movement on that metric escalates to `reworking`. | `kestra-build`, `kestra-run` |
| **5** | One real feature run end-to-end through the whole chain, dogfooded, on a feature that actually trips the full-mode condition table | none — eval only |

Waves 2 and 4 are the two that carry more than the exam needs — the "also in this PR" items come
from the same design effort and would land in the same diffs, so they are on the table here too and
reaction to them is just as welcome.

Wave 1 is first on purpose. The consolidation into a single-pass `kestra-spec` was a deliberate
decision with a date on it; if the pass has since grown too heavy, that should be re-opened by
measured numbers before anything downstream is reshaped — not by assertion. If Wave 1's numbers
trip the threshold, I stop and bring the seam question back here before Wave 2.

Wave 3's eval proves what is inspectable without a runner: derivation from the requirement surface,
**the exam script actually executing red/green** (check #0 smoke plus recorded red-proof
signatures — an eval a checklist could pass is not evidence), the manifest and anchor fields,
delta-scoped regeneration, refusal on a stale anchor, and `unproven` flagging.

---

## What I'm asking

Your reaction, before any of it is built. Specifically:

1. **The exam's form.** Is a requirement-level black-box e2e script at the declared seam the right
   artifact, or does it duplicate enough of `verify` that you'd rather it were something smaller?
2. **The hiding.** Exam outside every worktree, tracker pointer only, best-effort leak detection.
   Worth the machinery, or over-built for a solo workflow?
3. **The requirement surface boundary.** Does the exam derive from the Given-When-Then
   `## Acceptance Criteria` rows or from the Coverage Map's paraphrase of them — and do
   `## Business Rules` count? This decides what "stale" means, and I'd rather it were decided once,
   out loud, than settled by whichever extractor I wrote first.
4. **The wave plan.** Five stacked PRs is a lot to ask of a solo review. I can collapse Wave 1's
   eval into Wave 2's PR (the ordering of the measurement is what matters, not a separate
   round-trip), or split differently. Your call is cheaper to take now than after Wave 2 merges.
5. **Anything here that collides with a direction you already have for the repo.** I've matched the
   existing vocabulary where I could — stop conditions, provenance, `write_scope`, freeze, mode —
   rather than inventing parallel terms, but you know the intent behind those better than I do.

If the answer is "not this," that's a fine outcome for this document. It's cheaper than five PRs.
