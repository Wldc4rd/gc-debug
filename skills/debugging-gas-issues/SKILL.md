---
name: debugging-gas-issues
description: Use when a Gas City / Gas Town issue (gc, bd/beads, dolt, go-mysql-server, vitess, doltlite, driver, packs) is complex, intermittent, or a performance problem — high CPU, slow bd/queries, OOM, store/noms bloat, stalls, full-table scans — before theorizing, working around, or escalating, and when the fix should land as an upstream PR.
---

# Debugging Gas* Issues

## Overview

Many "gc problems" are actually **bd, dolt, go-mysql-server, vitess, doltlite, driver, or pack** problems. **The layer that shows the symptom is usually not the layer that owns the bug.** This skill traces a complex Gas* issue across the dependency stack to its real root, proves the fix on real data, and lands it as a mergeable upstream PR.

**Core principle:** Trace from the source, not the symptom. A green unit test proves *behavior*, not *scale* — verify on real data before you call it fixed. And a fix that **deletes or mutates shared state must preserve what *else* reads it** — audit the consumers, not just the symptom.

**REQUIRED BACKGROUND:** This skill is `superpowers:systematic-debugging` (no fix without a root cause) and `superpowers:test-driven-development` (the failing test) applied to the Gas* stack, with the concrete gc/bd/dolt toolkit and an upstream-PR finish.

## When to use

- High CPU, slow `bd`/queries, OOM, store/noms bloat, stalls, full-table scans, or intermittent failures anywhere in gc · bd/beads · dolt · gms · vitess · doltlite · driver · packs.
- **Before** theorizing, applying a workaround, or escalating.
- When you intend to fix it *properly* (an upstream PR), not just patch locally.

Not for: trivial config typos, already-documented issues, or single-fact lookups.

## Find the owning layer FIRST

The symptom's layer is rarely the bug's layer. Before tracing, locate the layer that **owns** the behavior — then trace *there*:

```
gc        gastownhall/gascity · gastownhall/gastown
 ├─ bd     gastownhall/beads   (gc go.mod pins the fork+version — that IS the source-of-truth)
 ├─ packs  gastownhall/gascity-packs
 └─ data plane:
     ├─ dolt      dolthub/dolt            (mysql-protocol, sql-server, port-based)
     │   └─ gms   dolthub/go-mysql-server (the SQL engine — planning, indexes)
     │       └─ vitess  dolthub/vitess    (MySQL protocol/parser)
     ├─ doltlite  dolthub/doltlite        (SQLite-backed; linked OR backend-plugin = bd-backend-doltlite + gc-doltlite-fastpath — no sql-server/port)
     └─ driver    dolthub/driver
```

See `references/gas-stack-map.md` for the **"which layer owns this symptom"** heuristics, the upstream repo to PR per layer, the dolt-vs-doltlite split, and the **ground-truth sources** — the repos (code) plus the official docs at **https://docs.gascityhall.com/llms-full.txt** (documented behavior/contracts). Patch the *owning* layer, not the one that surfaced the pain.

## The arc

1. **Capture state first.** Snapshot config/metrics/logs before touching anything. (`references/safety-floor.md`)
2. **Gather evidence at the boundaries.** Measure the *exact* resource/path; don't infer it. **Read the owning component's own logs** — services record their decisions (lifecycle outcomes, deferrals, gate/skip/drop reasons); the root is often a log line, not a deduction. (`references/gc-diagnostic-toolkit.md`)
3. **Search prior art — by symptom — before you dig.** Check the owning repos' Issues + PRs (open *and* closed) for the symptom; a known issue may already carry the diagnosis, a workaround, or a fix in flight. Don't re-trace what's filed — build on it.
4. **Locate the owning layer** (above) and **source-trace to the exact lines** — first confirm the binary matches the source (`git log -1` vs the running version), or the trace lies.
5. **RCA discipline.** Coincidence → correlation → causation. State a *falsifiable* hypothesis and name your bias. "It's flaky" is not a diagnosis.
6. **Diagnose WHERE the time/resource goes.** When something is slow or stuck, measure where it spends its time — don't retry-bigger or bump the timeout.
7. **Challenge the diagnosis — try to *break* it before you build on it.** The moment a cause looks found is the moment to attack it, not adopt it; skipping this is how a trace becomes a rabbit trail chasing the first fishy thing. State the root-cause claim in one line, then: **(a)** write the *disconfirming prediction* — what you'd observe if this were **not** the cause — and go look for it; **(b)** list the alternatives you have **not** excluded; **(c)** show causation, not correlation — does removing *this* actually resolve it, and does it **reproduce** reliably? **(d)** a fix is proven only **past the failure's own recurrence window**, not at first green. Best run as a **skeptic pass**: have a *second* agent argue *against* the diagnosis — a fresh adversary refutes what a motivated debugger won't. Scale the rigor to the cost of acting — a reversible poke needs little; an irreversible or shared-state change needs the full gauntlet. (`references/challenging-the-diagnosis.md`)
8. **Map the blast radius before you change shared state.** If the fix DELETES, MUTATES, or REPURPOSES persistent/shared state (beads, rows, labels, columns, cursors, files), enumerate **every consumer first** — grep the code *and the docs* (incl. the official docs, https://docs.gascityhall.com/llms-full.txt) for who reads it. State that looks like dead bookkeeping may be a ledger, a cursor, or a last-run marker; deleting it erases semantics other code depends on. Don't change it until it's provably unreferenced or its consumers are migrated. If it serves more than one purpose, **separate the concerns** — don't blunt-delete.
9. **Patch + a failing test (TDD).** Watch it fail for the right reason, then write the minimal fix. For a delete/mutate fix, the test must cover the *semantics the touched data serves* (each consumer from the step above), not just the mechanics of your change.
10. **Dogfood on real / production-scale data BEFORE the PR.** A green unit test proves behavior, not scale. Run the fix against real data and confirm the *actual* effect — **and that nothing downstream regressed.**
11. **Land it upstream.** Rebase onto the owning repo's `main`; **first search the repo by your fix / root-cause** — an existing or closed PR may carry prior art, the maintainer's preferred shape, or why it wasn't done; link/build on it, don't dup. Then file a thorough PR to the **right** repo. (`references/upstreaming-fixes.md`)

## Iron laws

- **Search the owning repo's Issues/PRs first** — by *symptom* before investigating, by *fix* before filing. Don't re-trace or re-file what already exists.
- **Read the component's own decision-logs early** — services log *why* they acted (outcomes, deferrals, gate/skip reasons). A chain of refuted hypotheses means you skipped this.
- Source-trace before you theorize.
- No fix without a root cause.
- A falsifiable hypothesis or it's not a diagnosis — name the bias.
- **A diagnosis is a suspect, not a verdict** — try to disprove it before you act on it. Name the *single observation that would refute it*, then go find that observation. A fresh skeptic beats a motivated debugger.
- **Verify on real, production-scale data before claiming done.**
- **Green ≠ cured** — a fix is proven only once it survives the failure's *own recurrence window*, not at first passing test.
- **Before deleting or mutating shared state, enumerate its consumers** — test the semantics it serves, not just your change.
- Nothing destructive without explicit go-ahead; capture state first.

## Red flags — STOP

Every row below is a real rationalization from a live trace, and every one was wrong:

| Rationalization | Reality |
|---|---|
| "Load average is 22, so CPU is maxed." | Load ≠ CPU. Measure utilization (PSI / mpstat / iowait / steal). It was a vCPU cap, not saturation. |
| "It's the obvious cause" → skip the trace. | Source-trace first. The symptom layer rarely owns the bug. |
| "The unit test passes — ship the PR." | Green unit test = behavior, not scale. Dogfooding on real data caught **three** further bugs. |
| "Batching will make it faster." | Measure the bottleneck first. The small-N timing already showed per-item cost dominated; batching barely helped. |
| "The run timed out — bump the timeout / retry bigger." | Diagnose *where* the time goes. It was stuck in an unbounded list query, not the work. |
| "Patch the layer showing the symptom." | Traverse to the owning layer. A gc-CPU symptom had a gc-lifecycle owner — but the identical symptom could live in gms. |
| "These rows are just dead bookkeeping — safe to delete." | Prove it. Grep every reader first. Bloat another subsystem queries is a *ledger*, not garbage — deleting it erases history / last-run / cursor state. (gc PR #2929 was CHANGES_REQUESTED for exactly this — see `references/worked-examples.md`.) |
| "I have a cleaner theory now." (your 2nd, 3rd…) | A *chain* of refuted hypotheses = you're pattern-matching the symptom, not measuring. Stop. Grep the **owning component's own decision-logs** (lifecycle outcomes, deferrals, gate/skip/drop reasons) — the root is often one line you haven't read. A single `deferred_by_wake_budget` line ended a 3-theory chain (debounce → busy-detector → multiline-paste). See worked example #3. |
| "The agent is CPU-starved / hung." | Read the **process state**, don't infer it. Asleep in `ep_poll` (`S` / `wchan`) at 2% CPU = idle/waiting, not starved or hung — even on a genuinely saturated box. `ps -o stat,pcpu,wchan,etime` before you blame CPU. |
| "Found the fishy thing — that's the bug." | A suspicious signal is a **lead, not a verdict**. Write what you'd see if it *weren't* the cause and go look; confirm that removing it actually resolves *and reproduces*. Adopting the first fishy thing is exactly how a trace becomes a rabbit trail. |
| "It went green — it's resolved." | Green ≠ cured. Prove it **past the failure's own recurrence window** — a stopgap re-breaks after the first clean interval. (A doltlite guard was announced "resolved / 15-of-15," then reframed as a *stopgap* with a second cause still open.) |

**Any of these means: stop and go back to the trace.**

## References

- `references/challenging-the-diagnosis.md` — the skeptic pass + the challenge checklist (disconfirming prediction, un-excluded alternatives, causation-not-correlation, proof past the recurrence window), the confidence ladder, and calibrating rigor to the cost of acting.
- `references/gas-stack-map.md` — the stack, which-layer-owns-this, upstream repos, dolt vs doltlite, the go.mod-source-of-truth rule.
- `references/gc-diagnostic-toolkit.md` — binary symbol-grep, dolt `SHOW PROCESSLIST` + connection mapping, CPU-vs-load, bead-store/tier layout, doltlite inspection.
- `references/upstreaming-fixes.md` — patch → prove → dogfood → rebase → PR (to the right repo) + a PR-body template.
- `references/worked-examples.md` — three real traces, end to end, each showing the layer hop.
- `references/safety-floor.md` — capture-state-first; the don't-without-go-ahead list.
