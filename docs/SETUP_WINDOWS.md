# Windows Setup (Neo4j)

This gets the benchmark running against a local Neo4j on a Windows 10/11 laptop.
No AWS needed. Budget ~15 minutes. When you're done you'll have a
`results/report.html` you can open in a browser.

The Neptune half is separate and optional — see
[`AWS_NEPTUNE_SETUP.md`](AWS_NEPTUNE_SETUP.md).

---

## 1. Install Python 3.10+

1. Download from <https://www.python.org/downloads/windows/> (3.10, 3.11 or 3.12).
2. Run the installer and **tick "Add python.exe to PATH"** on the first screen.
3. Verify in a new PowerShell window:
   ```powershell
   python --version
   ```
   You should see `Python 3.10.x` or newer. If PowerShell says python isn't
   found, close and reopen the window (PATH changes need a fresh shell).

---

## 2. Install Docker Desktop

Neo4j runs in a Docker container so you don't have to install a Java runtime or
configure the database by hand.

1. Download Docker Desktop from <https://www.docker.com/products/docker-desktop/>.
2. Install it, reboot if asked, and launch it. Wait until the whale icon in the
   system tray says **Docker Desktop is running**.
3. Verify:
   ```powershell
   docker --version
   ```

> On Windows Home, Docker Desktop uses the WSL 2 backend. If the installer
> prompts you to install/update WSL 2, accept it and reboot.

---

## 3. Put the project on your machine

Unzip the project (or copy the `graph-bench` folder) to somewhere simple like
`C:\graph-bench`, then open PowerShell there:

```powershell
cd C:\graph-bench
```

You should see `run_benchmark.py`, `requirements.txt`, and the `bench\` folder.

---

## 4. Create a virtual environment and install dependencies

```powershell
python -m venv venv
venv\Scripts\Activate.ps1
```

Your prompt should now start with `(venv)`.

> **If activation is blocked** with a red "running scripts is disabled" error,
> allow local scripts for your user once, then activate again:
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> venv\Scripts\Activate.ps1
> ```

Install the Python packages:

```powershell
pip install -r requirements.txt
```

---

## 5. Start Neo4j in Docker

```powershell
docker run -d --name neo4j-bench `
  -p 7474:7474 -p 7687:7687 `
  -e NEO4J_AUTH=neo4j/testpassword `
  neo4j:5.20-community
```

- `-d` runs it in the background.
- `7474` is the browser UI, `7687` is the Bolt port the benchmark uses.
- The password is set to `testpassword` to match the default in `.env.example`.

Give it ~20 seconds to start, then confirm it's healthy:

```powershell
docker logs neo4j-bench --tail 15
```

Look for a line like `Started.` / `Remote interface available at
http://localhost:7474/`. You can also open <http://localhost:7474> in a browser
and log in with `neo4j` / `testpassword` to see the empty database.

---

## 6. Tell the tools the password

Once per PowerShell session:

```powershell
$env:NEO4J_PASSWORD = "testpassword"
```

(If you changed the password in the `docker run` command, use that value here.)

---

## 7. Run the benchmark

The one-shot command runs everything — generate the graph, load it, time the
queries, compute the cost model, build the report:

```powershell
python run_benchmark.py all --db neo4j
```

You'll see the graph statistics, a load progress log, then a per-query table
like:

```
  [ok ] q01_count_nodes                4.20ms
  [ok ] q04_upstream_depth5            6.80ms
  ...
```

Open the report:

```powershell
start results\report.html
```

---

## Running the steps individually

`all` is just these in order — run any of them on their own while experimenting:

```powershell
python run_benchmark.py generate          # rebuild the CSV graph
python run_benchmark.py load  --db neo4j  # (re)load it into Neo4j
python run_benchmark.py bench --db neo4j  # time the queries -> results\bench_neo4j.json
python run_benchmark.py cost              # cost model      -> results\cost.json
python run_benchmark.py report            # (re)build       -> results\report.html
```

To make the graph bigger or smaller, set env vars before `generate` (see
`.env.example` for the full list), e.g. a larger graph:

```powershell
$env:BENCH_N_FIELDS = "40000"
python run_benchmark.py generate
python run_benchmark.py load --db neo4j
```

---

## Cleaning up

Stop and remove the Neo4j container when you're done (your results stay on disk):

```powershell
docker stop neo4j-bench
docker rm neo4j-bench
```

To start fresh later, re-run the `docker run` command from Step 5.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `could not connect to Neo4j` | Container not ready or not running. `docker ps` should list `neo4j-bench`; if not, re-run Step 5 and wait ~20s. |
| `AuthenticationRateLimit` / auth errors | `$env:NEO4J_PASSWORD` doesn't match the container. Set it to match your `docker run` value, or recreate the container. |
| Activation blocked in PowerShell | Run the `Set-ExecutionPolicy` line in Step 4. |
| Port 7687 already in use | Another Neo4j/instance is running. Stop it, or map a different host port (`-p 7688:7687`) and set `$env:NEO4J_URI="bolt://localhost:7688"`. |
| `docker: command not found` | Docker Desktop isn't running or isn't on PATH. Launch it and open a fresh PowerShell. |
| Load feels slow | Normal over Bolt for the default graph (a few minutes). Lower `BENCH_N_FIELDS` for quicker iterations. |
