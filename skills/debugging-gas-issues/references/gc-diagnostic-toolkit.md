# gc/bd/dolt diagnostic toolkit

Concrete commands. Read-only / dry-run unless noted. See `safety-floor.md` before anything destructive.

## Confirm the binary matches the source

```bash
git -C <source> log -1 --format='%H %ci %s'    # source HEAD
<binary> version                                # running build
```
If they diverge, your source trace may not reflect the running binary. Rebuild or check out the matching commit first.

## Binary symbol-grep (when config/schema docs lie)

The compiled binary carries its struct tags and Go type names. `grep -ao` works even where `strings` is filtered.

```bash
grep -aoE 'toml:"[^"]+"' "$(which gc)" | sort -u | grep -iE 'patch|rig|agent'   # TOML tags
grep -aoE 'json:"[^"]+"' "$(which gc)" | sort -u                                 # JSON/API tags
grep -aoE '[A-Z][a-zA-Z]+Patch(es)?[a-zA-Z]*' "$(which gc)" | sort -u            # Go type names
grep -aoE '/v[0-9]+/city/[^ ]*' "$(which gc)" | sort -u                          # API path templates
```

## CPU vs load (they are not the same)

Load average counts runnable **plus** uninterruptible-sleep (I/O-wait) tasks. A box can show load 22 at 26% CPU. Measure utilization, not load:

```bash
cat /proc/pressure/cpu /proc/pressure/io      # PSI: cpu `some` high + io `some` low = CPU-bound
mpstat 1 2                                     # %usr+%sys vs %idle, %iowait, %steal
vmstat 1 3                                     # r (run-queue) vs b (blocked); wa, st
nproc; lscpu | grep -E 'On-line|Off-line|Model name'   # vCPU cap vs host (a 6-vCPU guest on a 20-thread host = ~30% of host when pinned)
```
`%steal` high → the hypervisor is throttling. `%iowait` high → I/O-bound, not CPU. Run-queue ≫ cores at low iowait → genuinely CPU-bound.

## Find the real consumer

```bash
pidstat 3 1 | awk '/^Average:/ && $3 ~ /^[0-9]+$/ {cpu[$NF]+=$8} END{for(c in cpu) if(cpu[c]>=2) printf "%7.1f%%  %s\n", cpu[c], c}' | sort -rn
```
Aggregates sustained CPU by command. (`ps` %CPU is lifetime-average — misleading for "now".)

## dolt server introspection (backend: dolt only — confirm first)

Applies to the **dolt sql-server** backend. On a doltlite backend there is no server/port/PROCESSLIST — skip to the doltlite section. Confirm with `jq -r .backend .beads/metadata.json`.

```bash
PORT=$(cat <rig>/.beads/dolt-server.port 2>/dev/null || echo 3307)   # resolution: --port > city dolt.port > port-file > legacy
DSQL(){ DOLT_CLI_PASSWORD='' dolt --host 127.0.0.1 --port "$PORT" --user root --no-tls sql -q "$1"; }

DSQL "SELECT id,db,command,time,LEFT(info,150) FROM information_schema.processlist WHERE command<>'Sleep' ORDER BY time DESC"  # what's running NOW
DSQL "SHOW GLOBAL STATUS WHERE Variable_name IN ('Questions','Com_select','Slow_queries','Threads_connected','Uptime')"        # volume vs slow
DSQL "USE \`<db>\`; EXPLAIN <query>"; DSQL "USE \`<db>\`; SHOW INDEX FROM <table>"                                              # plan + indexes
```
Sample `Questions` twice to get QPS. **`Slow_queries`=0 + huge `Com_select` = a volume problem (gc/bd), not a slow-query problem (gms).**

Map which processes hold connections (find the culprit client):
```bash
ss -tnp 2>/dev/null | awk -v p=":$PORT$" '$5 ~ p' | grep -oE '"[^"]+",pid=[0-9]+' | sort | uniq -c | sort -rn
```

## Bead store: layout, tiers, bloat

Beads live in two tiers: **issues** (durable, non-ephemeral) and **wisps** (ephemeral, TTL-reclaimed by wisp-compact). Bloat is usually closed rows never deleted.

```bash
DSQL "USE \`<db>\`; SELECT issue_type, COUNT(*) n, SUM(status='closed') closed FROM issues GROUP BY issue_type ORDER BY n DESC"
DSQL "USE \`<db>\`; SELECT COUNT(*) FROM issues WHERE status='closed' AND title LIKE 'order:%'"   # leaked order-tracking
gc dolt-cleanup --probe --json     # orphan dbs / stale procs (NEVER --force without go-ahead)
```
Disk reclaim after deleting rows is a **dolt** GC: `CALL DOLT_GC('--full')` (online-safe; quiesce writers first). `gc dolt compact` gates on commit count and skips low-commit/high-churn dbs.

## doltlite (embedded backend — linked or plugin)

No server/port/PROCESSLIST. First confirm which doltlite deployment (see `gas-stack-map.md` → "Which data plane?"):
```bash
jq -r '.backend, (.backend_plugin_command // "—")' .beads/metadata.json   # doltlite + (plugin cmd | —)
pgrep -af 'bd-backend-doltlite|gc-doltlite-fastpath'                      # plugin serve procs (plugin model)
du -sh .beads/doltlite 2>/dev/null                                        # store size on disk
```
Query **through bd** (routed to the backend — works for linked and plugin), not a raw `sqlite3` open of the `.db` (that bypasses the working-set/commit model and can read a stale or locked view):
```bash
bd sql -q "SELECT issue_type, COUNT(*) n, SUM(status='closed') closed FROM issues GROUP BY issue_type ORDER BY n DESC"
```
Disk reclaim / maintenance is a **DoltLite** operation, not `CALL DOLT_GC` over a wire — use the DoltLite SQL functions (`SELECT dolt_gc();`, flatten, maintenance) and mind the store locks. The authoritative references for those, the native read fastpath, and DoltLite lock/maintenance gotchas are the DoltLite contract (`dolthub/doltlite` + README) and the plugin repos (`duncan4123/beads-backend-doltlite`, `duncan4123/gascity`); the `beads-doltlite` pack's `doltlite` skill (if your city imports it) collects them. The CPU-vs-load, binary-grep, store-layout, and dogfood techniques still apply; the server techniques (`PROCESSLIST`, `ss` on a port, `DOLT_GC('--full')` over the wire) do not. **Check the backend before reaching for `SHOW PROCESSLIST`.**
