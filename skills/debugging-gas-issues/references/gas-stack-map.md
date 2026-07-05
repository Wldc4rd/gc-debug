# Gas* stack map — which layer owns this bug?

The symptom surfaces high in the stack (a `gc` command feels slow, an agent stalls), but the bug usually lives lower. Walk *down* until the behavior is owned, then trace and PR *there*.

## The stack

| Layer | Repo | What it owns | Local source / where it lives |
|---|---|---|---|
| gc / gastown | `gastownhall/gascity`, `gastownhall/gastown` | Supervisor, controllers, sessions, hooks, reconcilers, orders, formulas, pack imports, the bd/dolt bridge (`internal/beads`) | your gascity source checkout; confirm vs the running `gc` (see below) |
| bd / beads | `gastownhall/beads` | The issue tracker, jsonl↔dolt sync, federation/credentials, the `bd` CLI | your beads source checkout; **the consuming `go.mod` pins which fork+version is real** |
| packs | `gastownhall/gascity-packs` (+ community pack repos) | Agent prompts, formulas, orders, skills, scripts | `.gc/system/packs/*`, `packs/*`, imported pack repos under `~/.gc/cache/repos/*` |
| dolt | `dolthub/dolt` | The data plane: storage (noms/chunks), commits, GC, the sql-server, `dolt` CLI | the module in the consumer's build cache; the running `dolt sql-server` |
| go-mysql-server (gms) | `dolthub/go-mysql-server` | The SQL engine embedded in dolt: query planning, index selection, joins, expression eval | a dolt dependency (check dolt's `go.mod`) |
| vitess | `dolthub/vitess` | MySQL wire protocol + SQL parser used by gms | a gms dependency |
| driver | `dolthub/driver` | The Go `database/sql` driver gc/bd use to talk to a dolt server | a gc/bd dependency |
| doltlite | `dolthub/doltlite` + the beads **backend plugin** (`bd-backend-doltlite`) & gc **fastpath** (`gc-doltlite-fastpath`) | SQLite-backed version-controlled store — an alternative to dolt; **no mysql sql-server, no port**. In the *plugin* deployment bd/gc are plain (unlinked) and talk to a `…serve` subprocess over stdio | `.beads/doltlite/*.db`; the co-located plugin binaries (e.g. `~/.local/lib/beads-plugin/`); `.beads/metadata.json` names them |

## Ground-truth sources (consult before tracing or changing behavior)

- **Code** — the repos above (`gastownhall/*`, `dolthub/*`); the consuming `go.mod` pins the exact linked version (see the rule below).
- **Documented behavior + contracts** — the official Gas City docs, **https://docs.gascityhall.com/llms-full.txt** (the whole corpus in one file). This is where a feature's *intended* behavior and a record's *documented purpose* live — exactly the consumer/contract surface a delete/mutate fix must not break. (gc#2929: `docs/tutorials/07-orders.md` documented that `gc order history` reads the order-tracking beads — the ledger contract the deletion erased; consulting the docs would have surfaced it.)

## Which layer owns this symptom?

| Symptom | Likely owner(s) | Confirm by |
|---|---|---|
| `dolt`/`bd` process pinning CPU | **dolt/gms** (heavy/looping query) OR **gc/bd** (query *volume* — too many calls, or scanning a bloated store) | `SHOW PROCESSLIST` (what query); global `Questions`/`Com_select` over time (volume); `pidstat` (the real consumer); map clients with `ss` |
| Slow `bd list`/query | **gc/bd** (no filter / unbounded / bloated table / no projection) → **gms** only if EXPLAIN shows a scan despite an index | `EXPLAIN`; `SHOW INDEX`; row counts; whether bd pushes the filter to SQL vs filters in-process |
| OOM / RSS climb | **dolt** (server memory) OR **bd** (forking the dolt CLI per op — loads the whole DB) | `pgrep`/`pidstat` for forks; guard logs; which process's RSS climbs |
| Store/noms bloat on disk | **gc/bd** churn (rows written faster than reclaimed) — reclaim is **dolt** (`DOLT_GC`) | row counts by type/status; noms dir size; commit count vs `gc dolt compact`'s gate |
| Stalls / "controller stalled" | **gc** (reconciler/config-load) OR host (CPU/IO saturation) — *not* the data plane by default | load **vs** CPU% (PSI/mpstat); run-queue vs iowait; per-import `git status` cost |
| Wrong/duplicated query plans, index ignored | **gms** (planner) — a genuine upstream gms issue | `EXPLAIN` shows a full scan with an index present + stats populated |

**Rule of thumb:** a *volume* problem (too many cheap queries, a bloated table) is almost always **gc/bd**; a *per-query* problem (one query is expensive/mis-planned) points at **dolt/gms**. Prove which with `Slow_queries`, `Questions`-rate, and `EXPLAIN` before you blame the engine.

## The go.mod-is-the-source-of-truth rule

Forks and mirrors exist (e.g. `bd` appears under both `gastownhall/beads` and a personal fork; local "patched" binaries drift from any repo). **The consuming project's `go.mod` names the exact module + version that is actually linked.** Trace and PR *that* one. Before trusting any local source checkout, confirm it matches the running binary:

```bash
git -C <source> log -1 --format='%H %ci %s'   # source HEAD
<binary> version                               # running version/commit
# mismatch -> your trace may not reflect the binary; rebuild or check out the right commit
```

## Which data plane? — detect the backend FIRST (don't assume mysql)

A beads store can sit on any of several data-plane backends, and their diagnostics differ completely — so identify the store's backend and **trace to that layer's repo** (the stack table above). `.beads/metadata.json` is the ground truth — read it *before* reaching for `SHOW PROCESSLIST`. The common ones are below; the list isn't closed (others may exist), so key off metadata, not a fixed set:

```bash
jq -r '.backend, (.backend_plugin_command // "—")' .beads/metadata.json  # backend + plugin cmd (or —)
ls -d .beads/dolt .beads/doltlite 2>/dev/null                            # which store dir exists
pgrep -af 'dolt sql-server|bd-backend-doltlite|gc-doltlite-fastpath'     # which serve procs are live
```

- **dolt (sql-server)** — `backend: dolt`; a MySQL-protocol server on a TCP port, store under `.beads/dolt`. Diagnose with `SHOW PROCESSLIST`, `information_schema.processlist`, global status, `ss`/`lsof` on the port. Port resolution: `--port` > city `dolt.port` > `<rig>/.beads/dolt-server.port` > legacy default.
- **doltlite, linked** — `backend: doltlite`, **no** `backend_plugin_command`; DoltLite compiled *into* bd/gc. Version-controlled, SQLite-backed; **no server, no port, no PROCESSLIST.**
- **doltlite, backend-plugin** — `backend: doltlite` **with** `backend_plugin_command` set. bd/gc are plain (unlinked) and launch `bd-backend-doltlite serve` (bd storage, over stdio, per-invocation) + `gc-doltlite-fastpath serve` (gc's always-on read fastpath). Store is `.beads/doltlite/*.db`; the plugin binaries are co-located (metadata names them). **No TCP server** — you cannot `SHOW PROCESSLIST` or `ss` a port. Inspect via `bd sql` (routed through the plugin), the `…serve` processes, the store dir on disk, and the DoltLite SQL maintenance functions. Deep commands + gotchas (`dolt_gc`, flatten, maintenance, locks, native read fastpath) live in the DoltLite contract itself (`dolthub/doltlite` + its README) and the plugin repos (`duncan4123/beads-backend-doltlite`, `duncan4123/gascity`). A city that imports the `beads-doltlite` pack (`gastownhall/gascity` `//examples/beads-doltlite`) also gets its `doltlite` skill as a convenience layer over those.

Detect which backend a given store uses and **trace to that owning layer (and its repo)** — don't assume. On a shared box the backends can coexist **per city** (one city on one backend, a throwaway on another), so detect per store, never assume per host.

See `gc-diagnostic-toolkit.md` for the concrete commands per backend.
