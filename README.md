# graph-db-benchmark

# Neo4j vs Amazon Neptune — Data-Lineage Benchmark

A self-contained test bench that compares **Neo4j** and **Amazon Neptune** as
the store for a **data-lineage knowledge graph** covering an investment bank's
capital-markets processes. It measures the three things you asked for:

1. **Performance** — query latency (cold, warm mean, p50/p95/p99) and load throughput.
2. **Cost** — an editable 3-year TCO model with verified 2026 AWS pricing.
3. **Ability to write complex queries** — a catalog of real lineage questions,
   each authored in Neo4j Cypher *and* Neptune-safe openCypher, so query-language
   gaps (e.g. Neptune has no `shortestPath()` or APOC) show up as results.

Everything runs on a **Windows laptop**. Neptune is optional — if you don't set
up AWS, you still get a complete Neo4j run plus the cost model.

---

## What's in the box

```
graph-bench/
├── run_benchmark.py         # single entry point (generate/load/bench/cost/report/all)
├── requirements.txt
├── .env.example             # all configuration, copy values into your shell
├── bench/
│   ├── config.py            # every tunable setting (env-overridable)
│   ├── data_generator.py    # deterministic capital-markets lineage graph -> CSV
│   ├── real_dataset.py      # OPTIONAL: build the graph from a real open dataset
│   ├── loader.py            # batched UNWIND load (identical path for both DBs)
│   ├── clients.py           # one Bolt client that talks to Neo4j AND Neptune
│   ├── queries.py           # the query catalog (Cypher + openCypher variants)
│   ├── runner.py            # timing + percentile statistics
│   ├── cost_model.py        # 3-year TCO (verified 2026 pricing, editable)
│   └── report.py            # combines results into one HTML report
└── docs/
    ├── SETUP_WINDOWS.md     # step-by-step Windows + Neo4j
    ├── AWS_NEPTUNE_SETUP.md # step-by-step AWS account + Neptune + tunnel
    └── METHODOLOGY.md       # what is measured and why it's fair
```

---

## The graph model

A purpose-built lineage DAG that mirrors how a bank tracks data from source
feeds to regulatory reports:

```
(:SourceSystem)-[:PRODUCES]->(:Dataset)-[:HAS_FIELD]->(:Field)
(:Dataset)-[:OWNED_BY]->(:BusinessUnit)
(:Field)-[:DERIVES_FROM]->(:Field)          # field-level lineage (the DAG)
(:Transformation)-[:CONSUMES]->(:Field)
(:Transformation)-[:OUTPUTS]->(:Field)
(:Field)-[:FEEDS]->(:Report)                # e.g. FRTB, LCR, CCAR
(:Control)-[:VALIDATES]->(:Field)           # data-quality controls
```

Default size ≈ **24k nodes / 71k edges** (generates in <1s, loads in a few
minutes). Fully seeded, so every run is identical and comparable. Change the
size in `.env` (`BENCH_N_FIELDS`, etc.).

The queries answer questions a bank actually asks, e.g.:
- *"If this source field is wrong, which regulatory reports are affected?"* (impact/blast-radius)
- *"Show the full upstream lineage of this reported figure."* (recursive traversal)
- *"Which report fields have no data-quality control on their immediate lineage?"* (governance gap)
- *"What's the shortest derivation path from input A to output B?"* (shortest path — the Neptune gap)

---

## Quick start (Neo4j only — free, ~15 minutes)

You need **Python 3.10+** and **Docker Desktop**. Full details in
[`docs/SETUP_WINDOWS.md`](docs/SETUP_WINDOWS.md).

**PowerShell:**

```powershell
# 1. Get the code into a folder and open it
cd C:\graph-bench

# 2. Create and activate a virtual environment
python -m venv venv
venv\Scripts\Activate.ps1

# 3. Install dependencies
pip install -r requirements.txt

# 3b. Sanity-check the layout and imports (no database needed)
python verify_setup.py

# 4. Start Neo4j in Docker (password must match .env)
docker run -d --name neo4j-bench `
  -p 7474:7474 -p 7687:7687 `
  -e NEO4J_AUTH=neo4j/testpassword `
  neo4j:5.20-community

# 5. Tell the tools the password (once per shell)
$env:NEO4J_PASSWORD = "testpassword"

# 6. Run the whole thing against Neo4j
python run_benchmark.py all --db neo4j
```

`all` runs generate → load → bench → cost → report. Open the result:

```powershell
start results\report.html
```

You can also run the steps individually:

```powershell
python run_benchmark.py generate
python run_benchmark.py load  --db neo4j
python run_benchmark.py bench --db neo4j
python run_benchmark.py cost
python run_benchmark.py report
```

---

## Adding Neptune (optional — needs an AWS account)

Neptune lives inside a private AWS network (VPC) and **cannot be reached
directly from your laptop**. The realistic options are covered fully in
[`docs/AWS_NEPTUNE_SETUP.md`](docs/AWS_NEPTUNE_SETUP.md):

- create a Neptune cluster (there is an AWS **Free Tier**: 750 hrs/month of
  `db.t3.medium` / `db.t4g.medium`),
- reach it from your laptop through an **SSH tunnel** via a tiny bastion EC2,
- point the bench at it and run the same commands with `--db neptune`.

Once the tunnel is up and your `.env` has `NEPTUNE_ENDPOINT` set:

```powershell
python run_benchmark.py load  --db neptune
python run_benchmark.py bench --db neptune
python run_benchmark.py report      # report now shows BOTH databases side by side
```

The same `neo4j` Python driver talks to both databases over Bolt, so the client
code path is identical and the latency comparison is fair.

---

## Using a real open dataset (optional)

The default graph is synthetic-but-realistic (and a precise match for the
banking scenario). If you specifically want a **real, openly-available large
dataset**, the bench can build its graph from a **Stack Exchange data dump**
(CC-BY-SA, on the Internet Archive). `quant.stackexchange.com` gives it a
quantitative-finance flavour. See [`docs/METHODOLOGY.md`](docs/METHODOLOGY.md)
for the mapping and this command:

```powershell
# download + convert a real dump to CSVs in .\data_real
python -m bench.real_dataset --url "<archive.org .7z URL>" --out data_real

# point the loader at it and run the real-graph query set
$env:BENCH_DATA_DIR = "data_real"
python run_benchmark.py load  --db neo4j --real
python run_benchmark.py bench --db neo4j --real
```

---

## What you get

- `results/report.html` — one page comparing both databases: latency table,
  compatibility column, load throughput, and the 3-year cost table.
- `results/bench_neo4j.json`, `results/bench_neptune.json` — every query's cold,
  warm-mean, p50/p95/p99, stddev, row count, and (if it failed) the error.
- `results/cost.json` — the full cost breakdown with all assumptions.

---

## Cleaning up (so you're not billed)

```powershell
docker stop neo4j-bench; docker rm neo4j-bench      # local Neo4j
```

For Neptune, **delete the cluster and the bastion EC2** when done — see the
teardown section in `docs/AWS_NEPTUNE_SETUP.md`. Neptune has no always-free
tier beyond the initial 750-hour window; a running cluster accrues charges.

---

## Honest notes on fairness

- Both databases are loaded with the **same batched-UNWIND Cypher** over Bolt,
  not their native bulk loaders, so "load throughput" compares like with like.
  (For a real multi-million-node load you'd use Neptune's S3 bulk loader and
  Neo4j's `admin import` — noted in the methodology.)
- Neptune run from your laptop pays a **network round-trip** that a local Neo4j
  container does not. To compare compute rather than network, run the bench
  from an EC2 instance in the same region (see methodology). The bench records
  both so you can reason about it.
- The cost model's inputs (instance types, I/O volume, ops overhead) are
  **assumptions you can and should edit** in `bench/cost_model.py`.
