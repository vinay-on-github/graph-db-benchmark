# AWS Neptune Setup (from a Windows laptop)

This guide gets you from "no AWS account" to "the benchmark runs against a real
Neptune cluster." It is honest about the one thing most tutorials skip:

> **Neptune has no public endpoint. It only accepts connections from inside its
> VPC.** You cannot point your laptop straight at it. We solve this with a tiny
> **bastion EC2 host** and an **SSH tunnel** — the standard, cheapest approach.

Time: ~45 minutes the first time. Cost: a few dollars if you tear down promptly
(Neptune Free Tier covers 750 hours/month of `db.t3.medium`/`db.t4g.medium`).

---

## 0. Costs and the Free Tier (read first)

- **Neptune Free Tier:** 750 hours/month of a `db.t3.medium` or `db.t4g.medium`
  instance for the first period after you first use Neptune. One always-on
  small cluster fits inside 750 hours (~730 hours in a month). Storage
  ($0.10/GB-month), I/O ($0.20 per million requests), and backups are billed
  separately but are pennies at this scale.
- **Bastion EC2:** a `t3.micro` (or `t4g.micro`) is Free-Tier-eligible for
  750 hours/month in your first year; otherwise ~$0.01/hour.
- **After the Free Tier**, a small provisioned Neptune runs ~$254/month if left
  on. **Delete it when you're done** (Section 8).

Set a billing alarm now: **Billing → Budgets → Create budget → $10 threshold.**

---

## 1. Create an AWS account

1. Go to <https://aws.amazon.com> → **Create an AWS Account**.
2. Provide email, password, contact details, and a payment card (required even
   for Free Tier).
3. Choose the **Basic (free) support plan**.
4. Sign in to the **AWS Management Console**.

Pick a region and **stick with it** for everything below (top-right of the
console). This guide assumes **US East (N. Virginia) = `us-east-1`**.

---

## 2. Create the Neptune cluster

1. Console → search **Neptune** → **Databases** → **Create database**.
2. **Engine version:** latest (1.4.x or newer — needed for current openCypher).
3. **Templates / DB instance class:** choose **Development and Testing** and the
   smallest instance, **`db.t3.medium`** (Free-Tier-eligible).
4. **DB cluster identifier:** `lineage-bench`.
5. **Connectivity:**
   - **VPC:** the default VPC is fine.
   - **Public access:** **No** (Neptune can't be public — that's expected).
   - Note the **VPC** and **Subnet** it lands in; the bastion must share the VPC.
6. **IAM DB authentication:** **Disabled** for this test (simplest). We rely on
   network isolation + the tunnel instead.
7. Create it. Wait until status is **Available** (5–10 min).
8. Open the cluster and copy the **Writer endpoint** — it looks like
   `lineage-bench.cluster-abcde.us-east-1.neptune.amazonaws.com`. You'll put
   this in `.env` as `NEPTUNE_ENDPOINT` (no `https://`, no port).

---

## 3. Launch a bastion EC2 in the same VPC

The bastion is a cheap Linux box that *is* allowed to talk to Neptune; your
laptop tunnels through it.

1. Console → **EC2** → **Launch instance**.
2. **Name:** `neptune-bastion`.
3. **AMI:** Amazon Linux 2023. **Type:** `t3.micro`.
4. **Key pair:** create one, e.g. `bastion-key`, and **download `bastion-key.pem`**.
   Save it somewhere like `C:\Users\<you>\.ssh\bastion-key.pem`.
5. **Network settings → Edit:**
   - **VPC:** the **same VPC** as the Neptune cluster (important).
   - **Auto-assign public IP:** **Enable**.
   - **Security group:** create one, `bastion-sg`, allow **SSH (port 22) from My IP**.
6. Launch. When it's running, copy its **Public IPv4 address**.

---

## 4. Let the bastion reach Neptune (security groups)

Neptune has its own security group; it must accept traffic from the bastion.

1. Console → **EC2 → Security Groups**.
2. Find the **Neptune cluster's** security group (shown on the cluster's
   *Connectivity & security* tab).
3. **Edit inbound rules → Add rule:**
   - **Type:** Custom TCP, **Port:** `8182`,
   - **Source:** the **bastion's security group** (`bastion-sg`).
4. Save. Now the bastion (and only the bastion) can reach Neptune on 8182.

---

## 5. Open the SSH tunnel from Windows

Windows 10/11 ships with OpenSSH. In PowerShell:

```powershell
# Lock down the key file permissions once (OpenSSH refuses world-readable keys)
icacls "C:\Users\<you>\.ssh\bastion-key.pem" /inheritance:r
icacls "C:\Users\<you>\.ssh\bastion-key.pem" /grant:r "$($env:USERNAME):(R)"

# Open the tunnel: local 8182 -> Neptune:8182, via the bastion.
# Replace the pem path, bastion IP, and Neptune endpoint.
ssh -i "C:\Users\<you>\.ssh\bastion-key.pem" `
    -L 8182:lineage-bench.cluster-abcde.us-east-1.neptune.amazonaws.com:8182 `
    -N ec2-user@<BASTION_PUBLIC_IP>
```

Leave this window open — the tunnel stays up while it runs. `-N` means "don't
run a remote shell, just forward the port."

Because you forwarded **localhost:8182 → Neptune:8182**, the bench connects to
`localhost` but reaches Neptune. There's one wrinkle: Neptune presents a TLS
certificate for its real hostname, not `localhost`. Handle it by adding a hosts
entry so the name resolves locally:

1. Open Notepad **as Administrator** → open
   `C:\Windows\System32\drivers\etc\hosts`.
2. Add this line:
   ```
   127.0.0.1   lineage-bench.cluster-abcde.us-east-1.neptune.amazonaws.com
   ```
3. Save. Now the real hostname points at your tunnel, and TLS validation passes.

> Prefer not to touch `hosts`? Use the HTTPS transport instead
> (`NEPTUNE_ACCESS_MODE=https`), which the AWS SDK signs and validates cleanly,
> or run the whole bench from the bastion itself (Section 7).

---

## 6. Configure and run the benchmark

In a **second** PowerShell window (keep the tunnel running in the first):

```powershell
cd C:\graph-bench
venv\Scripts\Activate.ps1

# Point the bench at Neptune (endpoint = the real hostname you added to hosts)
$env:NEPTUNE_ENDPOINT = "lineage-bench.cluster-abcde.us-east-1.neptune.amazonaws.com"
$env:NEPTUNE_PORT     = "8182"
$env:NEPTUNE_REGION   = "us-east-1"
$env:NEPTUNE_ACCESS_MODE = "bolt"

# You should already have generated the data for the Neo4j run; if not:
python run_benchmark.py generate

python run_benchmark.py load  --db neptune
python run_benchmark.py bench --db neptune
python run_benchmark.py report
```

`report` now renders **both** databases side by side (if you also ran Neo4j).

---

## 7. Alternative: run everything on the bastion (no tunnel, lowest latency)

Tunnelling adds your home-to-AWS round-trip to every query. To measure Neptune's
*compute* rather than your Wi-Fi, copy the project to the bastion and run it there
(it's already inside the VPC):

```bash
# on the bastion (ssh in without -N)
sudo yum install -y python3-pip git
git clone <your copy of this project> graph-bench && cd graph-bench
pip3 install -r requirements.txt
export NEPTUNE_ENDPOINT=lineage-bench.cluster-abcde.us-east-1.neptune.amazonaws.com
python3 run_benchmark.py generate
python3 run_benchmark.py load  --db neptune
python3 run_benchmark.py bench --db neptune
```

Then `scp` `results/*.json` back to your laptop and run
`python run_benchmark.py report` locally to combine with the Neo4j numbers.

---

## 8. Tear down (stop the billing)

Do this as soon as you're finished:

1. **Neptune:** Console → Neptune → Databases → select `lineage-bench` →
   **Actions → Delete**. Delete the writer instance and the cluster. Decline the
   final snapshot unless you want to keep the data.
2. **EC2 bastion:** Console → EC2 → Instances → select `neptune-bastion` →
   **Instance state → Terminate**.
3. **(Optional) Security groups** `bastion-sg` and the Neptune inbound rule you
   added — delete if you won't reuse them.
4. Check **Billing → Bills** the next day to confirm nothing is still running.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Unable to connect to localhost:8182` | Tunnel window closed or bastion SG doesn't allow your IP on 22. Reopen the `ssh -L` command. |
| TLS/certificate hostname mismatch | You connected to `localhost`. Add the `hosts` entry (Section 5) so the real hostname resolves to `127.0.0.1`, or use `NEPTUNE_ACCESS_MODE=https`. |
| `Tunnel connection failed / 503` | Bastion can't reach Neptune. Check the Neptune security group allows port 8182 **from the bastion's security group** (Section 4). |
| `AccessDenied` on `neptunedata` | Using the HTTPS transport with IAM auth on. Either disable IAM DB auth on the test cluster, or attach an IAM policy granting `neptune-db:*` to your credentials. |
| Query `q07_shortest_path` shows *not supported* | Correct and expected — Neptune has no `shortestPath()`. That's a benchmark finding, not a bug. |
| Cluster stuck "Creating" > 15 min | Normal for first create; if > 30 min, check the events tab. |
