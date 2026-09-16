---
name: query-insights
description: Query an Insights database over NATS with the `insights query` and `insights db` subcommands using DuckDB SQL, including selecting a system on a multi-system deployment
---

# Query Insights Skill

Query the Synadia Insights database (DuckDB) over NATS.

## Querying

```bash
insights query "<SQL>"          # CSV by default; -f json for JSON
echo "<SQL>" | insights query   # pipe SQL via stdin when quoting is awkward
```

Targets the local server by default; use `--nats.*` flags (or `INSIGHTS_NATS_*` env vars) for a remote endpoint, and `--system` to pick which monitored system answers (see below). Run `insights query --help` for all flags.

## Targeting a system

One deployment can hold several monitored systems, each with **its own database**. Every query goes to exactly one, named in the call:

```bash
insights system list                    # what is reachable, and on which node
insights query --system west "<SQL>"    # or export INSIGHTS_SYSTEM=west
```

Resolution is flag → `INSIGHTS_SYSTEM` → the config file's `system.id`. With nothing naming a target the CLI runs one discovery round before dispatching: exactly one system answering is used; **several answering is refused rather than guessed**; none answering falls back once to the pre-multi-system subjects.

```
multiple systems are reachable — select one with --system <id> (or set INSIGHTS_SYSTEM):
```

So on a multi-system deployment, name the system. It is also faster — a named target skips the two-second discovery window — and it never falls back, so a wrong id reports the silence instead of quietly answering from elsewhere:

```
$INS.sys.nope.db.query request: nats: no responders available for request
```

Discovery is per invocation, so a system that appears later needs no restart or reconfiguration to become queryable.

## Cross-system questions are a fan-out, not a join

Each system has its own DuckDB instance. **No SQL spans them** — there is no schema qualifier, no `ATTACH`, no federated view, and a system id is not a column. A question about several systems is answered by running the query once per system and merging the results:

```bash
for sys in $(insights system list -f json \
      | jq -r '.[].systems[] | select(.roles | index("catalog")) | .id' | sort -u); do
  insights query --system "$sys" -f json "<SQL>" | jq --arg s "$sys" 'map(. + {system: $s})'
done | jq -s add
```

**Filter to the `catalog` role.** A system's scrape side and catalog side can live in different processes, and only the catalog answers queries. On a collector deployment every system is advertised twice — once by the collector as `scrape`, once by the indexing node as `catalog` — so an unfiltered list queries each system twice. Worse, if the indexing node is down the system is still advertised, by its collector, and querying it fails:

```
$INS.sys.east.db.query request: nats: no responders available for request
```

The filter turns that into an empty list, which is the truthful answer: nothing can be queried right now.

Two things to carry into any merge:

- **Stamp the system id yourself.** Rows do not carry one, so a merged result is unattributable without it.
- **`pk` is per system.** Two systems routinely both have a server with `pk` 7, so rows may only be matched across systems on the `(system, pk)` pair — never `pk` alone.

Epochs are also per system: scrape intervals differ, so `max(epoch)` in one system is unrelated to `max(epoch)` in another. Resolve the window separately per system rather than computing one and reusing it.

## Discover the schema — don't hardcode it

The schema is self-documenting: every schema, table, view, and column carries a comment with the authoritative semantics (composition, grain, gauge vs counter, units, FK targets). **Discover and read those comments before writing a query — never assume an object or column exists.**

Use the `insights db` subcommands (aligned text table by default; `-f json` for the raw response, which also includes each object's entity-level `table_comment`):

```bash
insights db schemas                              # queryable schemas + object counts
insights db tables  --schema hx                  # tables/views in a schema (with comments)
insights db columns --schema hx --table servers  # columns: type + semantic comment
insights db macros  --schema checks              # check macros + signatures
insights db explain "<SQL>"                      # validate/plan a query without running it
insights db explain --analyze "<SQL>"            # execute + profile it (EXPLAIN ANALYZE, runtime timing)
```

Omit `--schema`/`--table` to list across everything in scope. (Discovery is scoped to the allowlisted schemas `hx`, `main`, `checks`, `contexts`.)

## Rules that aren't in the catalog

- **Always bound the result** — every `*_stats` table and `hx.<entity>` view holds one row *per entity per epoch*, so an unscoped `SELECT * FROM hx.conns` scans all retained history (millions of rows). Scope by epoch (see below) and/or add a `LIMIT`. The streaming endpoint aborts a scan past a row backstop with a message telling you to scope it — treat that as a signal to add an epoch filter, not to raise the cap.
- **Qualify every name with its schema** — `hx` is not on the search path, so an unqualified `server_stats` errors; write `hx.server_stats`. Entities live in `hx`; geo-IP enrichment is `main.ips`; check macros are in `checks`.
- **Query the epoch-scoped `hx.<entity>` views for analysis** — they are pre-joined; keep the `WHERE epoch = ...` on them. Fall back to the raw base tables for name resolution and epoch logic: `hx.<entity>_ident` (`pk` → `name`), `hx.<entity>_opts` (config), `hx.<entity>_stats` (per-epoch metrics).
- **`epoch` is a `TIMESTAMP WITH TIME ZONE`, not a number** — every `epoch` column in `hx` is a timestamp, so windowing is interval arithmetic: `max(epoch) - INTERVAL 1 HOUR`. `max(epoch) - 3600` is a binder error.
- **Source the current epoch from a `_stats` table** — a `_stats` row is guaranteed every epoch an entity exists, so `(SELECT max(epoch) FROM hx.<entity>_stats)` is the reliable "latest". Scrape cycles are not evenly spaced and epochs go missing, so derive the previous epoch with `lag(epoch) OVER (...)` rather than subtracting a fixed interval.
- **Counters vs gauges** — the column comment says which: counters are monotonic (e.g. `in_msgs`) → diff across epochs for a rate; gauges are point-in-time (e.g. `memory`) → read the latest epoch directly.
- DuckDB SQL dialect (not Postgres/MySQL). Use `name` columns for human-readable output, `pk` for JOINs.
- **`raft_group` names record the replica count at creation, not the current one.** A group named
  `S-R1F-…` on a stream that has since been scaled to R3 keeps the `R1F` name forever, so the prefix
  is not evidence of the replica factor. Read `num_replicas`, or count the distinct servers holding
  a replica, and ignore the name.
- **Absent rows are ambiguous** — a missing `_stats` row for one entity in one epoch can mean the
  entity is gone, or just that a single monitoring reply was lost. The endpoints are scraped
  independently, so cross-check another table for the same server and epoch (`hx.raft_groups`
  against `hx.server_stats`, say): rows from one and not the other means a lost reply, not an
  outage. Corroborate with counters — `in_msgs` advancing across the gap proves the server was up.
- **The entity views are not exactly one row per stats row.** `hx.servers`, `hx.streams` and
  `hx.consumers` attach the config valid at each stats epoch; two `_opts` rows tied on the same
  epoch are both kept and duplicate that stats row, and a stats row older than the entity's first
  config is dropped entirely. Both are rare and both are silent, so verify a total against the base
  `_stats` table before reporting it. `hx.accounts` is different again: it attaches the *latest*
  config overall, so historical rows carry present-day limits and claims.

## One entity, several rows per epoch

JetStream replica state is reported from more than one vantage point, so `hx.streams` and
`hx.consumers` hold up to three kinds of row per entity per epoch:

| Row kind             | `is_leader` | `peer_server_pk` | What is actually observed         |
| -------------------- | ----------- | ---------------- | --------------------------------- |
| Leader self-report   | true        | 0                | everything                        |
| Follower self-report | false       | 0                | its own message and sequence state |
| Leader-reported peer | false       | non-zero         | `is_current`, `is_offline`, `active`, `lag` |

Everything outside a row's set is a zero value, not an observation.

- **Filter `is_leader` for one row per entity** — otherwise a `sum()` adds every replica's copy plus
  a set of zeros. It is a filter, not a guarantee: some epochs carry no leader row at all (973
  stream-epochs in a 3-hour production window), so a leader-filtered count runs slightly under.
- **Replica health lives only on `peer_server_pk != 0` rows** — a self-reported row hardcodes
  `is_current = true` and `is_offline = false` whatever the replica is really doing, so
  `WHERE NOT is_current` over the whole table finds nothing but leader-reported peers by accident.
- **A raft group has one `hx.raft_group_ident` row per server holding a replica.** `group_id` is the
  *server* peer hash (`getHash(serverName)`, 8 chars), identical for every group on that server;
  `group_name` is the group. Count groups with `count(DISTINCT group_name)` within an account.
  `raft_group_stats.leader` is a peer hash too, and `server_pk` lives on the stats row, not the ident
  row, so resolving a leader to a server takes a second hop:

```sql
-- which server leads each raft group at the latest epoch
SELECT i.group_name, ls.server_pk AS leader_server_pk
FROM hx.raft_group_stats s
JOIN hx.raft_group_ident i  ON i.pk = s.raft_group_pk
JOIN hx.raft_group_ident li ON li.group_name = i.group_name AND li.group_id = s.leader
JOIN hx.raft_group_stats ls ON ls.raft_group_pk = li.pk AND ls.epoch = s.epoch
WHERE s.epoch = (SELECT max(epoch) FROM hx.raft_group_stats) AND s.leader <> ''
```

## Rolling up across servers

Most per-server metrics are not additive, because the fleet sees each message more than once.

- **`hx.servers.in_msgs` / `out_msgs` / `in_bytes` / `out_bytes` include route, gateway and leaf
  traffic.** Summing them fleet-wide counts a message once per server that handled it. Measured
  between two adjacent epochs on a production fleet, route traffic alone was 78% of the fleet
  `in_msgs` delta. There is no client-only column on the server; attribute publishes with
  `hx.conns.msgs_recv` instead.
- **Account totals carry the same transports** — `hx.accounts.msgs_recv` / `msgs_sent` also include
  route, gateway and leaf traffic, but the account tables break them out, so client-only is
  `msgs_recv - route_msgs_recv - gateway_msgs_recv - leaf_msgs_recv`. On a production fleet the
  transports were 63% of account `msgs_recv`.
- **Route rows are symmetric.** Both ends report the same link, so sum `in_*` or `out_*`, never both.
- **`out_msgs` counts deliveries, `in_msgs` counts publishes** — the ratio is fan-out; they are not
  meant to balance.
- **`routes` is connections, `remotes` is peers.** Pooling and per-account pinned routes open several
  connections per peer. For peers use `remotes`, or `count(DISTINCT remote_server_pk)` on `hx.routes`.
- **Account rows fan out per server** — `hx.accounts` has one row per account *per server*, so
  reduce per `(name, epoch)` before summing anything across the fleet.
- **Gateways are unidirectional connections.** `is_outbound` splits them; real message flow is
  `out_*` on outbound rows and `in_*` on inbound ones, and the opposite direction is protocol
  chatter. Interest state (`hx.gateway_account_stats`) exists for outbound rows only.

## Columns whose name misleads

The catalog comments are authoritative and say all of this, but these are the ones a reader assumes
they already understand:

| Column | It is actually |
| ------ | -------------- |
| `conn_stats.msgs_sent` / `bytes_sent` | what the server **delivered to** the client; `*_recv` is what the client published |
| `account_stats.total_conns` | a gauge, always `conns + leafnodes` — not cumulative, so it cannot give churn |
| `consumer_replica_stats.num_redelivered` | outstanding un-acked redeliveries; it decreases, so never diff it |
| `server_sublist_stats.cache_hit_rate` | cumulative since process start, and can exceed 1.0 — for a window, diff `cache_hit_rate * num_matches` and `num_matches` |
| `servers.subscriptions`, `accounts.subs` | sublist size including interest propagated in over routes/gateways/leafs; client subscriptions are `sum(hx.conns.num_subs)` |
| `stream_replica_stats.bytes` | message record size (subject + headers + overhead), not payload and not disk footprint |
| `server_stats.cpu` | a one-second sample taken by nats-server, not an average over the epoch interval |
| `server_health_stats.status` | `error` whenever any assigned JetStream asset is behind, so it is common rather than exceptional — 39% of server-epochs in a 3-hour production window, almost all of them consumers catching up |

## Epoch scoping

```sql
-- latest epoch
SELECT * FROM hx.servers WHERE epoch = (SELECT max(epoch) FROM hx.server_stats)

-- last hour (epoch is a timestamp, so the window is an interval)
SELECT * FROM hx.servers WHERE epoch >= (SELECT max(epoch) - INTERVAL 1 HOUR FROM hx.server_stats)

-- per-second rate of a counter, gap-safe: epochs are irregular and some go missing,
-- so take the previous epoch each row actually had and divide by the real elapsed time
SELECT name, epoch,
       (in_msgs - prev_in_msgs) / date_diff('second', prev_epoch, epoch) AS msgs_per_sec
FROM (
  SELECT s.server_pk, i.name, s.epoch, s.in_msgs,
         lag(s.in_msgs) OVER (PARTITION BY s.server_pk ORDER BY s.epoch) AS prev_in_msgs,
         lag(s.epoch)   OVER (PARTITION BY s.server_pk ORDER BY s.epoch) AS prev_epoch
  FROM hx.server_stats s JOIN hx.server_ident i ON i.pk = s.server_pk
  WHERE s.epoch >= (SELECT max(epoch) - INTERVAL 1 HOUR FROM hx.server_stats)
)
WHERE prev_epoch IS NOT NULL AND in_msgs >= prev_in_msgs  -- drop restarts, which reset counters
```

A restart mints a new `server_pk`, so partitioning by `pk` already keeps a counter reset out of most
deltas; the guard covers the rest.

## Aggregation contexts

Roll a metric up at the level you need — system (all servers/accounts), cluster, server, or account:

```sql
-- per cluster: memory and server count are genuinely additive
SELECT cluster, sum(memory) AS memory, count(*) AS servers
FROM hx.servers WHERE epoch = (SELECT max(epoch) FROM hx.server_stats) GROUP BY cluster

-- per account: hx.accounts has one row per account per server, so group by name to
-- fold the servers together; msgs_recv includes route/gateway/leaf, so net them out
SELECT name, sum(conns) AS conns,
       sum(msgs_recv - route_msgs_recv - gateway_msgs_recv - leaf_msgs_recv) AS client_published
FROM hx.accounts WHERE epoch = (SELECT max(epoch) FROM hx.account_stats) GROUP BY name

-- fleet-wide client publishes: attribute to connections, not to hx.servers.in_msgs,
-- which also counts every route, gateway and leaf hop
SELECT sum(msgs_recv) AS client_msgs_published
FROM hx.conns WHERE epoch = (SELECT max(epoch) FROM hx.conn_stats) AND kind = 'Client'
```

`sum(hx.servers.in_msgs)` looks like fleet throughput and is not — see **Rolling up across servers**.

## Checks

Findings live in `hx.check_findings` (`code`, `severity`, `entity_type`, `entity_pk`, `entity_key`, `epoch`). For ad-hoc analysis, **prefer the `checks.*` macros** — they already return a resolved `entity` name, so you avoid hand-rolling ident JOINs. Discover signatures with `insights db macros --schema checks`, then run directly:

A check code like `consumer-001` becomes the macro `checks.consumer_001` — hyphens to
underscores, nothing else. Every macro takes `(epoch_start, epoch_end)` **in that order** and filters
`epoch BETWEEN epoch_start AND epoch_end`, so passing the window backwards returns nothing rather
than an error.

```sql
-- consumer replicas reported offline in the last hour
SELECT code, entity, remote_name, epoch
FROM checks.consumer_001(
  (SELECT max(epoch) - INTERVAL 1 HOUR FROM hx.server_stats),
  (SELECT max(epoch) FROM hx.server_stats))
```

Querying `hx.check_findings` directly yields `entity_pk`/`entity_key`, not names. Resolve a name by joining the matching `hx.<entity_type>_ident` on `pk` (e.g. `server` → `hx.server_ident`). Two non-obvious cases: `kvstore`/`objectstore` resolve via `hx.stream_ident`; `service` has no ident table — use `entity_key` directly.

## Examples

```bash
# top 5 connections by messages sent
insights query "SELECT name, lang, msgs_sent FROM hx.conns WHERE epoch = (SELECT max(epoch) FROM hx.conn_stats) ORDER BY msgs_sent DESC LIMIT 5"

# streams with most messages (the view has one row per replica — filter to the leader)
insights query "SELECT name, msgs, bytes FROM hx.streams WHERE epoch = (SELECT max(epoch) FROM hx.stream_replica_stats) AND is_leader ORDER BY msgs DESC LIMIT 10"

# count findings by severity at the latest epoch
insights query "SELECT severity, count(*) AS n FROM hx.check_findings WHERE epoch = (SELECT max(epoch) FROM hx.server_stats) GROUP BY severity ORDER BY n DESC"
```
