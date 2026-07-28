# Feedback rounds

One file per improvement round, named `<YYYYMMDD>-<slug>.md`.

Each round starts from **evidence produced by a real run** — a Cadence cycle report
(`<runDir>/report.md`, rendered by `/cadence:report`) brought back to this repo — and is
worked through interactively with the user: findings are traced from the symptom to the
rule that caused them, the user picks what to address and confirms anything that touches
a safety invariant, and only then are edits applied, versioned, and tagged.

A round file records:

- **Source** — which run/report the findings came from (cycle slug, dates, plugin version).
- **Findings** — symptom → evidence (PR / comment URL / SHA / timestamp) → the rule or
  file that produced it → class (behavior defect · missing capability · doc gap ·
  not-the-plugin).
- **Changed** — what was edited for each accepted finding, and the resulting version.
- **Not changed** — what was declined or deferred, **with the reasoning**, and what
  evidence would change the call next time.

That last section is the point: it's what makes the next cycle's report comparable to
this one instead of re-litigating the same questions.

The full loop is documented in the repo's `CLAUDE.md` ("Evolving a plugin from a cycle
report") and, user-facing, in
[the Cadence README](../../plugins/cadence/README.md#closing-the-loop-the-report-is-how-cadence-evolves).
