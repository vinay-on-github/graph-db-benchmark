# Methodology — what is measured and why it's fair

This document explains how the benchmark measures the three criteria
(performance, cost, ability to write complex queries), the deliberate design
choices that keep the Neo4j vs Neptune comparison honest, and the caveats you
should keep in mind when reading the numbers.

---

## The three criteria

### 1. Performance

Two things are timed:

- **Query latency.** Each query in the catalog runs `BENCH_ITER` times
  (default 6). The first `BENCH_WARMUP` runs (default 1) are recorded
  separately as **cold** (caches empty, plan not yet compiled). The remaining
  runs are the **warm** sample, from which the report shows the mean and the
  raw JSON keeps p50/p95/p99 and standard deviation. Warm mean is the headline
  number because it reflects steady-state behaviour; the cold number tells you
  about first-hit and planning cost.

- **Load throughput.** The wall-clock time to load all nodes and then all
  relationships, reported separately (`load_seconds_nodes`, `load_seconds_rels`,
  `load_seconds_total`).

Timing is measured client-side with `time.perf_counter()` around each
`client.run(...)` call, so it includes the network round-trip and result
materialisation — i.e. what an application actually experiences.

### 2. Cost

A 3-year total-cost-of-ownership model (`bench/cost_model.py`) with three
scenarios: Neptune Provisioned, Neptune Serverless, and self-hosted Neo4j on
EC2. Every input — instance hourly rate, I/O volume, storage, ops overhead,
optional savings-plan discount — is a named field in an `Assumptions` dataclass
with the 2026 list price sourced in a comment. The model is meant to be
**edited to match your workload**, not taken as a universal verdict. The cost
model does not require a database connection and always runs.

### 3. Ability to write complex queries

This is the qualitative core, made concrete. The catalog (`bench/queries.py`)
holds 12 lineage questions spanning categories: simple scan, filter,
multi-hop, bounded lineage, unbounded recursion, shortest path, path
enumeration, anti-join (governance gap), aggregation, and neighbourhood
extraction. **Each query is authored twice** — idiomatic Neo4j Cypher and
Neptune-compatible openCypher — and every result records whether it actually
ran. When Neptune cannot express a construct, that shows up as a
`not supported` cell in the report rather than being hidden. The gaps this
surfaces (verified against AWS documentation):

- **`shortestPath()` / `allShortestPaths()`** — not supported by Neptune
  openCypher. Query `q07` carries a rewrite that enumerates bounded paths and
  takes the shortest, which is more expensive and can miss paths beyond the cap.
  Its `neptune_native = False` flag is the finding.
- **APOC procedures** (e.g. `apoc.path.subgraphNodes`) — Neo4j-only. Query
  `q12` uses APOC on Neo4j and a plain bounded traversal on Neptune.
- **`EXISTS { ... }` anti-join subqueries** — used on Neo4j in `q08`; rewritten
  for Neptune as `OPTIONAL MATCH` + `count() = 0`.
- **Unbounded variable-length paths and NOT-EXISTS patterns** — `q05` uses a
  truly unbounded `*` on Neo4j; the Neptune variant caps depth and derives
  "root" nodes by out-degree instead.

So "ability to write complex queries" is answered on two axes at once: *can the
engine express the query at all* (the compatibility column) and *how fast is it
when it can* (the latency column).

---

## Why these design choices keep it fair

**One client, both engines.** Both Neo4j and Neptune speak the Bolt protocol,
so the same `neo4j` Python driver drives both (`bench/clients.py`). The
serialisation path, connection handling, and result materialisation are
therefore identical; differences in the numbers come from the databases, not
from two different client libraries. (A boto3 HTTPS transport is included as an
alternative for environments where Bolt is blocked or IAM signing is required;
it is slightly slower per call because of request signing, which is why Bolt is
the default for the comparison.)

**Identical load path — deliberately not the native bulk loaders.** Both loads
use the same batched `UNWIND` Cypher (`bench/loader.py`). This is a conscious
choice: Neo4j's `LOAD CSV` / `neo4j-admin import` and Neptune's S3 bulk loader
are engine-specific, tuned very differently, and would make "load time"
incomparable. Sending identical batched writes over the same protocol measures
the same thing on both sides. The trade-off: these numbers represent
*application-style incremental loading*, **not** the fastest possible bulk
ingest. For a real multi-million-node initial load you would use each engine's
native bulk loader — noted here so nobody mistakes this for a bulk-ingest
benchmark.

**Compatibility failures are results, not errors.** The runner catches a failed
query and records it with its error text and `supported = false` instead of
aborting the suite. A query Neptune cannot run is a data point about query
capability, which is exactly one of the three criteria.

**Deterministic dataset.** The synthetic graph is fully seeded
(`BENCH_SEED`, default 42), so every run — and every machine — builds a
byte-identical graph. Comparisons across runs and across the two databases are
therefore stable.

---

## Caveats — read the numbers with these in mind

**Network asymmetry (the big one).** In the default setup Neo4j runs in a local
Docker container (sub-millisecond network) while Neptune is reached across the
internet through an SSH tunnel to AWS. Every Neptune query therefore pays a
laptop→AWS round-trip that Neo4j does not. Two ways to read this honestly:

- If you care about *"what will my app feel like talking to a managed cloud DB
  from where it runs"*, the tunnelled numbers are informative — but then you
  should also run your app from the same network as the DB.
- If you care about *engine compute alone*, run the benchmark **from an EC2
  instance in the same region/VPC as Neptune** (Section 7 of the Neptune setup
  guide), and compare against Neo4j on a comparable EC2 instance. The bench
  records enough (cold vs warm, per-query) to reason about where time goes.

The report notes which side was run and does not pretend the environments are
identical.

**Small graph by laptop standards.** The defaults (~24k nodes / ~71k edges) fit
comfortably in memory on both engines, which flatters in-memory latency. Scale
`BENCH_N_FIELDS` up to stress traversal and memory behaviour if that matches
your real graph.

**Cost model is assumptions, not prophecy.** The default $500/month ops line for
self-hosted Neo4j, the 200M-I/O/month figure for Neptune, and the chosen
instance types dominate the result. Change them to your reality before drawing
conclusions.

**List prices, one region.** Cost figures are `us-east-1` list prices verified
in 2026; other regions, committed-use discounts, and enterprise agreements will
differ.

---

## Optional: running on a real open dataset

The default graph is synthetic-but-realistic and a precise match for the
capital-markets lineage scenario. To satisfy the "run on an openly available
large dataset" requirement with genuinely external data, `bench/real_dataset.py`
can build the graph from a **Stack Exchange data dump** (CC-BY-SA, published on
the Internet Archive).

**The mapping.** A Q&A site is already a graph with the same query *shapes* as
data lineage — only the labels change:

| Lineage concept | Stack Exchange analogue |
|---|---|
| `(:Field)` node | `(:Post)` node (question or answer) |
| `DERIVES_FROM` (one-directional dependency) | `ANSWERS` (answer → its question, via `ParentId`) |
| lineage/related links | `LINKED_TO` (from `PostLinks.xml`: related/duplicate) |
| impact / blast-radius traversal | reachable posts within N hops of `LINKED_TO` |
| shortest lineage path | shortest `LINKED_TO` path between two posts (same Neptune gap) |
| fan-out hotspot | most-linked posts (degree ranking) |

So the same benchmark categories — filter, multi-hop, bounded reachability,
shortest path (with the identical Neptune limitation), and aggregation — run
against a real graph via the compact `REAL_QUERIES` set. Everything else (the
one Bolt client, the batched-UNWIND load, the timing harness, the compatibility
handling) is unchanged, so the fairness properties above still hold.

**Why it's optional / not the default.** Stack Exchange now gates some dumps
behind site login, and sizes range from a few MB (small sites) to ~90 GB (Stack
Overflow), so availability and size aren't guaranteed. The synthetic generator
guarantees the bench always runs and is a cleaner match for the banking
scenario; the real dataset demonstrates the harness works on external data too.
`quant.stackexchange.com` is suggested for a quantitative-finance flavour.

```powershell
python -m bench.real_dataset --url "<archive.org .7z URL>" --out data_real
$env:BENCH_DATA_DIR = "data_real"
python run_benchmark.py load  --db neo4j --real
python run_benchmark.py bench --db neo4j --real
```
