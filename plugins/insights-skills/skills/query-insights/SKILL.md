---
name: query-insights
description: Query the insights database via the `insights query` subcommand using DuckDB SQL
---

# Query Insights Skill

Query the Synadia Insights database (DuckDB) over NATS.

## Querying

```bash
insights query "<SQL>"          # CSV by default; -f json for JSON
echo "<SQL>" | insights query   # pipe SQL via stdin when quoting is awkward
```

Targets the local server by default; use `--nats.*` flags (or `INSIGHTS_NATS_*` env vars) for a remote endpoint. Run `insights query --help` for all flags.

## Discover the schema — don't hardcode it

The schema is self-documenting: every schema, table, view, and column carries a comment with the authoritative semantics (composition, grain, gauge vs counter, units, FK targets). **Discover and read those comments before writing a query — never assume an object or column exists.**

Use the `insights db` subcommands (aligned text table by default; `-f json` for the raw response, which also includes each object's entity-level `table_comment`):

```bash
insights db schemas                              # queryable schemas + object counts
insights db tables  --schema hx                  # tables/views in a schema (with comments)
insights db columns --schema hx --table servers  # columns: type + semantic comment
insights db macros  --schema audit               # audit check macros + signatures
insights db explain "<SQL>"                      # validate/plan a query without running it
```

Omit `--schema`/`--table` to list across everything in scope. (Discovery is scoped to the allowlisted schemas `hx`, `main`, `audit`.)

## Rules that aren't in the catalog

- **Qualify every name with its schema** — `hx` is not on the search path, so an unqualified `server_stats` errors; write `hx.server_stats`. Entities live in `hx`; geo-IP enrichment is `main.ips`; check macros are in `audit`.
- **Query the `hx.<entity>` views for analysis** — they are pre-joined. Fall back to the raw base tables for name resolution and epoch logic: `hx.<entity>_ident` (`pk` → `name`), `hx.<entity>_opts` (config), `hx.<entity>_stats` (per-epoch metrics).
- **Source the current epoch from a `_stats` table** — `epoch` is a Unix timestamp (seconds), and a `_stats` row is guaranteed every epoch an entity exists, so `(SELECT max(epoch) FROM hx.<entity>_stats)` is the reliable "latest".
- **Counters vs gauges** — the column comment says which: counters are monotonic (e.g. `in_msgs`) → diff across epochs for a rate; gauges are point-in-time (e.g. `memory`) → read the latest epoch directly.
- DuckDB SQL dialect (not Postgres/MySQL). Use `name` columns for human-readable output, `pk` for JOINs.

## Epoch scoping

```sql
-- latest epoch
SELECT * FROM hx.servers WHERE epoch = (SELECT max(epoch) FROM hx.server_stats)

-- last hour
SELECT * FROM hx.servers WHERE epoch >= (SELECT max(epoch) - 3600 FROM hx.server_stats)

-- rate of a counter between the two latest consecutive epochs
SELECT curr.name, curr.in_msgs - prev.in_msgs AS msg_delta
FROM hx.servers curr
JOIN hx.servers prev ON curr.pk = prev.pk
WHERE curr.epoch = (SELECT max(epoch) FROM hx.server_stats)
  AND prev.epoch = (SELECT max(epoch) FROM hx.server_stats
                    WHERE epoch < (SELECT max(epoch) FROM hx.server_stats))
```

## Aggregation contexts

Roll a metric up at the level you need — system (all servers/accounts), cluster, server, or account:

```sql
-- system-wide throughput at the latest epoch
SELECT sum(in_msgs) AS in_msgs, sum(out_msgs) AS out_msgs
FROM hx.servers WHERE epoch = (SELECT max(epoch) FROM hx.server_stats)

-- per cluster
SELECT cluster, sum(memory) AS memory, count(*) AS servers
FROM hx.servers WHERE epoch = (SELECT max(epoch) FROM hx.server_stats) GROUP BY cluster

-- per account
SELECT name, sum(conns) AS conns, sum(msgs_sent) AS msgs_sent
FROM hx.accounts WHERE epoch = (SELECT max(epoch) FROM hx.account_stats) GROUP BY name
```

## Audit checks

Findings live in `hx.check_results` (`code`, `severity`, `entity_type`, `entity_pk`, `entity_key`, `epoch`). For ad-hoc analysis, **prefer the `audit.*` macros** — they already return a resolved `entity` name, so you avoid hand-rolling ident JOINs. Discover signatures with `insights db macros --schema audit`, then run directly:

```sql
-- idle streams (verify the signature first via `insights db macros --schema audit`)
SELECT code, entity, idle_minutes
FROM audit.check_opt_idle_007(
  (SELECT max(epoch) FROM hx.server_stats),
  (SELECT max(epoch) - 3600 FROM hx.server_stats))
```

Querying `hx.check_results` directly yields `entity_pk`/`entity_key`, not names. Resolve a name by joining the matching `hx.<entity_type>_ident` on `pk` (e.g. `server` → `hx.server_ident`). Two non-obvious cases: `kvstore`/`objectstore` resolve via `hx.stream_ident`; `service` has no ident table — use `entity_key` directly.

## Examples

```bash
# top 5 connections by messages sent
insights query "SELECT name, lang, msgs_sent FROM hx.conns WHERE epoch = (SELECT max(epoch) FROM hx.conn_stats) ORDER BY msgs_sent DESC LIMIT 5"

# streams with most messages (the view has one row per replica — filter to the leader)
insights query "SELECT name, msgs, bytes FROM hx.streams WHERE epoch = (SELECT max(epoch) FROM hx.stream_replica_stats) AND is_leader ORDER BY msgs DESC LIMIT 10"

# count findings by severity at the latest epoch
insights query "SELECT severity, count(*) AS n FROM hx.check_results WHERE epoch = (SELECT max(epoch) FROM hx.server_stats) GROUP BY severity ORDER BY n DESC"
```
