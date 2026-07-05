# Chapter 18 — Deployer: from `cargo run` to AWS

> Closing the gap between a local demo and a remote deployment.

## The problem

You have a working Commonware app. `cargo run` works on your laptop. The
simulated network in tests works. Now you need to actually deploy it to
some cloud, on real validators, with real EC2 instances, real S3 buckets,
real metrics, real logs.

`commonware-deployer` is a CLI tool + library for doing exactly this.
Currently it supports AWS (EC2, S3, security groups). Other providers are
extensible.

## The CLI commands

From `deployer/src/lib.rs:23-49`:

| Command | What it does |
|---|---|
| `aws create` | Deploy EC2 instances across multiple regions from a YAML config |
| `aws update` | Update binaries + config in-place on all running instances |
| `aws authorize` | Add the deployer's current IP to all security groups |
| `aws destroy` | Tear down all resources for a deployment |
| `aws clean` | Delete the shared S3 bucket |
| `aws profile` | Capture a CPU profile from a running instance using samply |

The flow:

```bash
# Install the CLI
cargo install commonware-deployer

# Write a config file
cat > mychain.yaml <<EOF
regions:
  - us-east-1
  - us-west-2
instance_type: c6i.xlarge
binary: ./target/release/mychain
participants: 4
EOF

# Deploy
deployer aws create --config mychain.yaml

# Monitor
deployer aws profile --instance <id> --duration 60

# Update without re-deploying
deployer aws update --config mychain.yaml

# Tear down
deployer aws destroy --config mychain.yaml
```

## What gets provisioned

`docs/blogs/commonware-deployer.html` and the deployer source:

- **EC2 instances** across one or more regions.
- **Security groups** with proper inbound rules (deployer's IP for SSH,
  peer-to-peer ports for consensus, metrics ports for scraping).
- **S3 bucket** for shared state (e.g., for the `archive` storage
  primitive from chapter 06, or for blob persistence).
- **IAM roles** for S3 access.
- **CloudWatch** for metrics.
- **VPC + subnet configuration** if needed.

The deployer takes care of all of this from a YAML config. You don't have
to click through the AWS console.

## Multi-region deployment

The `regions` field in the YAML config lets you specify multiple regions.
Commonware's P2P (chapter 05) connects peers across regions; consensus
keeps working as long as the network is reasonably healthy.

Multi-region deployment is **the realistic deployment topology** for a
BFT system because:
- A single region's outage doesn't kill the network.
- Validators in different regions see different network conditions
  (good for test realism in prod).
- Geographically distributed validators resist regulatory pressure.

## The profile command — production debugging

`deployer aws profile` runs `samply` (a sampling profiler) on a running
instance. You get a CPU profile of your consensus engine under real load.

This is the **right way to debug performance issues in production**. Not
by SSHing in and running `top`. By attaching a sampling profiler and
seeing exactly which functions consume CPU.

## The library mode

`deployer` is also a Rust library (`deployer/src/aws/`). You can write
custom deployment workflows that use the deployer as a building block.
Commonware uses this internally to manage test deployments.

## The "your p2p demo runs locally. now what?" blog

`docs/blogs/your-p2p-demo-runs-locally-now-what.html` (Patrick O'Grady,
March 2025) walks through the entire deployment journey:

> You've built a peer-to-peer (p2p) demo that hums along on your local
> machine. Peers connect, messages flow, and for a moment, you're basking
> in localhost bliss. Then comes the inevitable question: how do you
> migrate this from your laptop to the cloud?

The blog is the canonical "how to ship" guide. Read it.

## When to use Deployer

Use it when:
- You're deploying to AWS.
- You have a multi-region topology.
- You want to update binaries without re-deploying.

Don't use it when:
- You're running a single-instance demo (just SSH and run).
- You're using a different cloud provider (write your own).
- You're running a light client / non-validator (no need for EC2).

## Where to look in the code

- `deployer/src/lib.rs` — the entry point + CLI documentation.
- `deployer/src/main.rs` — the CLI binary.
- `deployer/src/aws/` — the AWS implementation.
- `docs/blogs/commonware-deployer.html` — the original announcement.
- `docs/blogs/your-p2p-demo-runs-locally-now-what.html` — the practical
  walkthrough.

## AWS infrastructure — the mental model

Before the deployer makes sense, the cloud primitives underneath it need to.
Picture the hierarchy top-down.

A **region** (`us-east-1`, `eu-west-1`, `ap-southeast-2`) is a
geographically isolated cluster of data centers, served by independent
power grids and network backbones. A region is the unit at which AWS
itself fails — `us-east-1` has had several high-profile outages where
every availability zone inside it degraded simultaneously. For a BFT
network, "one region down" is exactly the failure mode you want to
survive. Commonware's deployer is opinionated here: validators should
**not** all sit in the same region. The `regions:` list in the YAML
config is the simplest knob for that.

Inside a region sit **Availability Zones (AZs)** — typically three per
region. Each AZ is one or more physically separate data centers, with
its own power and networking, but close enough (~10ms RTT to siblings)
to feel "local." AZ failure is more common than region failure. EC2
instances within a region are placed into specific AZs by your config
(subnet selection); ideally you spread validators across AZs within a
region so a single AZ outage takes out at most a third of local
validators, which a `n/3` Byzantine-tolerant consensus can absorb.

A **VPC** (Virtual Private Cloud) is your isolated private network in
the cloud. It looks like a switch you'd rack in your own data center
except it spans every AZ in the region. Your EC2 instances, your
subnets, your security groups all live inside a VPC. Default deployments
use the region's default VPC; serious deployments should create one
explicitly so you control the IP space.

**Subnets** are slices of a VPC tied to a specific AZ. A subnet is a
CIDR range (`10.0.1.0/24`) that lives in one AZ. Public subnets have a
route to an **Internet Gateway** (so instances get public IPs); private
subnets only have a **NAT Gateway** outbound. For a BFT network,
validators can sit in private subnets if they only receive P2P from
peers in known places; for the public internet model Commonware
assumes, public subnets are simpler.

**Security groups** are the firewall rules — they attach to ENIs
(elastic network interfaces) on instances and govern inbound/outbound
traffic. They are stateful (return traffic is automatically allowed)
and operate at the instance level, not the subnet level.

```
Region us-east-1
├── Availability Zone us-east-1a
│   └── Subnet 10.0.1.0/24 (public)  -- instance validator-0
├── Availability Zone us-east-1b
│   └── Subnet 10.0.2.0/24 (public)  -- instance validator-1
└── Availability Zone us-east-1c
    └── Subnet 10.0.3.0/24 (public)  -- instance validator-2
        (cross-AZ RTT ~1-2ms)
```

Three boxes, three AZs, three machines. That's the production topology
the deployer builds, repeated across each region you list.

## The deployment workflow, step by step

The deployer's `aws` subcommand group runs a deterministic lifecycle.
Here's each transition in detail.

### `aws create` — from zero to running

The create command does the heavy lifting. It is not idempotent in the
narrow sense — if resources from a previous run exist, you must
`destroy` first or hit naming collisions. Internally, the create
command (`deployer/src/aws/create.rs`) sequences idempotent SDK calls:

1. **Authenticate.** Resolve AWS credentials via the default chain:
   environment variables (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`),
   shared config file (`~/.aws/config`), EC2 instance metadata service
   (if running on EC2 already), or the IAM role of the running machine
   (if the deployer is itself on AWS).
2. **Validate the config.** Open the YAML, parse it into
   `deployer::aws::Config`, sanity-check regions exist, instance type
   is well-formed, participants count >= 4 (BFT minimum). Bad
   configurations fail before any resource is touched.
3. **Provision the shared S3 bucket.** Create the bucket with the name
   derived from the deployment (`<config-name>-<random-suffix>`),
   enable versioning and a default-encryption SSE-S3 policy, configure
   a lifecycle rule for object expiry.
4. **Provision the security group.** One per region. The inbound rules
   are tight: SSH (22) from your current IP only, P2P port (3000) from
   the validator subnets, metrics port (9090) from the monitoring
   subnet. Outbound is open by default (the validator pulls from S3,
   uploads to CloudWatch).
5. **Provision the IAM role.** One per region. EC2 instances assume
   this role on launch to read from S3 and write metrics to
   CloudWatch. Trusts the EC2 service principal only.
6. **Launch EC2 instances.** One `RunInstances` call per region, with
   the user-data script baked in (see below).
7. **Wait for instances to be ready.** Poll `DescribeInstanceStatus`
   until `instance-status.ok == true` and `system-status.ok == true`,
   with a 5-minute default timeout.
8. **Upload binaries and configs to S3.** The deployer puts the release
   binary in `binaries/<deployment-name>/<version>/` and configs in
   `configs/<deployment-name>/`.
9. **Signal instances to start.** The user-data script on each
   instance pulls from S3 on startup, so the instances self-bootstrap.
   The deployer also has a follow-up "ping" step that opens an SSH
   session to confirm the consensus process is up.

The user-data script is the closest thing to "bootstrapping an AMI":

```bash
#!/usr/bin/env bash
# user-data.sh — runs on first boot of each validator
set -euo pipefail

# Install runtime deps (deb style; Amazon Linux 2 uses dnf)
apt-get update && apt-get install -y awscli

# Pull the binary from S3
mkdir -p /opt/mychain
aws s3 cp s3://mychain-deployment/binaries/mychain-v0.1.0/mychain /opt/mychain/
chmod +x /opt/mychain/mychain

# Pull the per-instance config
aws s3 cp s3://mychain-deployment/configs/mychain/node-0.toml /etc/mychain.toml

# Run as a systemd service
systemctl enable mychain.service
systemctl start mychain.service
```

### `aws update` — zero-downtime binary rotation

The update command (`deployer/src/aws/update.rs`) replaces the
binaries running on every instance without tearing down the network.

1. Upload the new binary to a versioned S3 path.
2. For each instance, send a graceful restart signal via `aws ssm`
   send-command (Systems Manager agent runs by default on Amazon
   Linux 2 AMIs). The signal handler triggers the application to drain
   in-flight work, write its state, and `exec` the new binary.
3. Wait for the new process to come up (healthcheck via metrics).
4. Verify cluster health (e.g., that finalization is still progressing
   by querying the metrics endpoint).

This is **the** zero-downtime deployment path. It works because
Commonware's consensus has crash-recovery semantics (`start_and_recover`
in chapter 16): a restarted validator rejoins the network and resumes
from its journal/WAL.

### `aws destroy` — clean teardown

The destroy command (`deployer/src/aws/destroy.rs`) tears down in the
reverse order: instances, then security groups, then IAM roles. The
S3 bucket is **preserved** by default — keeps the audit trail and
lets you recover. Use `aws clean` to nuke the bucket when you're done
for good.

```bash
deployer aws destroy --config mychain.yaml
# Optional: --purge-s3 to also delete the bucket
```

## S3 — the shared spine

The S3 bucket is **how the deployment stays consistent across regions
and restarts**. Think of it as both a binary registry and a shared
backing store for state.

```
s3://<deployment-name>-<suffix>/
├── binaries/
│   └── <deployment>/
│       ├── mychain-v0.1.0            # tagged releases
│       ├── mychain-v0.1.0-rc.1       # release candidates
│       └── mychain-<git-sha>         # debug builds
├── configs/
│   └── <deployment>/
│       ├── node-0.toml
│       ├── node-1.toml
│       └── ...
└── shared/                           # application-level shared state
    ├── notarizations/                # archived consensus certs
    ├── blocks/                       # cold-archived blocks
    └── checkpoints/                  # state checkpoints
```

The `binaries/` and `configs/` prefixes are managed by the deployer
itself. The `shared/` prefix is yours — Commonware's `storage::archive`
primitive (chapter 06) writes here if you opt in.

**Encryption**: every object is encrypted at rest with SSE-S3 (AES-256).
Bucket policy denies any `PutObject` request without
`x-amz-server-side-encryption: AES256` so you can't accidentally
push plaintext.

**Lifecycle** (in the bucket policy created by the deployer):

- binaries: keep last 10 versions, expire older after 30 days
- configs: never expire (small, useful for audit)
- shared/notarizations: transition to Glacier after 90 days, expire
  after 365 days (this is what makes archival economic)
- shared/blocks: same as notarizations

For production, point the lifecycle at **Glacier Instant Retrieval**
(`GLACIER_IR`) rather than Deep Archive — it's still cheap, but
retrieval is milliseconds, not hours.

**Versioning** is on. You can roll back to a previous binary by
re-pointing the user-data script and bouncing instances.

## The security perimeter

Two layers of access control: **network-level** (security groups) and
**identity-level** (IAM).

### Security group rules — least privilege

```hcl
# SG for BFT validators ("mychain-validators")
ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.deployer_ip_cidr]  # /32 — your workstation
}
ingress {
    from_port   = 3000  # P2P consensus port
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = var.validator_subnet_cidrs   # restrict to known peers
}
ingress {
    from_port   = 9090  # Prometheus metrics
    to_port     = 9090
    protocol    = "tcp"
    cidr_blocks = var.monitoring_subnet_cidrs  # Prometheus subnet only
}
egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"   # all outbound (S3, CloudWatch, NTP)
    cidr_blocks = ["0.0.0.0/0"]
}
```

SSH is `/32` (your IP only) by default. Use `deployer aws authorize`
to grant a teammate temporary access. P2P is restricted to the
**subnets** of your peers, not `0.0.0.0/0` — that turns "expose to the
internet" into "expose to known validator hosts," which dramatically
shrinks the attack surface (a metaplex-style DNS rebinding attack has
nowhere to land).

### IAM roles — the runtime boundary

The deployer needs broad powers (`ec2:RunInstances`, `iam:PassRole`,
etc.). **Validators do not.** The pattern is two distinct IAM
principals:

**Deployer IAM principal** (your laptop or CI):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:RunInstances", "ec2:TerminateInstances", "ec2:DescribeInstances",
      "ec2:CreateSecurityGroup", "ec2:AuthorizeSecurityGroupIngress",
      "ec2:CreateTags", "ec2:DeleteSecurityGroup",
      "s3:CreateBucket", "s3:PutObject", "s3:GetObject", "s3:DeleteBucket",
      "s3:PutLifecycleConfiguration", "s3:PutBucketEncryption",
      "iam:CreateRole", "iam:AttachRolePolicy", "iam:PassRole",
      "iam:DeleteRole", "iam:DetachRolePolicy",
      "cloudwatch:PutMetricData", "logs:CreateLogGroup",
      "logs:PutLogEvents", "logs:CreateLogStream",
      "ssm:SendCommand", "ssm:GetCommandInvocation"
    ],
    "Resource": "*"
  }]
}
```

**Validator instance role** (assumed by EC2 at launch):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::mychain-deployment-*",
        "arn:aws:s3:::mychain-deployment-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:log-group:mychain-*"
    }
  ]
}
```

The deployer can do anything; the validator can only read its own
bucket slice and emit metrics. **Compromise of a validator does not
yield cluster-admin powers.** This is the standard production
boundary; copy it.

## Observability — what's standard

A BFT system that's not observable is a system you cannot operate.
Commonware's runtime exports two streams, and the deployer wires both.

### Metrics (OTLP/Prometheus)

The runtime emits metrics in OTLP-formatted text or Prometheus format
(via the `metrics` + `metrics-exporter-prometheus` crates). Default
exposition is on a configurable port (default 9090). The deployer
opens this port to the monitoring subnet only.

Standard metrics for a BFT app:

```
# Consensus
commonware_simplex_views_total{view="..."}
commonware_simplex_finalizations_total
commonware_simplex_finalization_latency_seconds_bucket
commonware_simplex_certificates_emitted_total

# P2P
commonware_p2p_peers_connected
commonware_p2p_messages_sent_total{channel="votes"}
commonware_p2p_messages_received_total{channel="votes"}
commonware_p2p_message_latency_seconds_bucket

# Storage
commonware_journal_appends_total{partition="log"}
commonware_journal_sync_duration_seconds_bucket

# Runtime
commonware_runtime_tasks_spawned_total
commonware_runtime_tasks_running
commonware_runtime_cpu_usage_seconds_total
```

These get scraped by Prometheus (deployed separately or via
Grafana Cloud), visualized in Grafana dashboards. The
[Commonware benchmarks dashboard](https://commonware.xyz/benchmarks)
is itself a Grafana board fed by CI runs of `estimator`.

### Logs (structured tracing)

The runtime uses `tracing` with structured fields, exported as JSON
lines. The deployer ships them to **CloudWatch Logs**, with one log
group per deployment:

```
/aws/mychain/<region>/<instance-id>
```

Every event has a span context, so tracing through a request that
crosses actor boundaries (chapter 11, the Simplex pipeline) is just
"follow the trace ID." Stream to CloudWatch, ship from CloudWatch to
Loki or Datadog, and you have a full observability stack.

### The profile command — debug by the numbers

`deployer aws profile` is the production-debugging workhorse
(`deployer/src/aws/profile.rs`). It attaches a sampling profiler to a
running instance and pulls a flame graph back. The default profiler is
`samply`, which uses Linux's `perf_event_open` under the hood. It
samples the call stack 1k times per second by default; a 60-second
profile captures ~60k samples, which is plenty for any "where did the
CPU go?" question.

```bash
# Capture 60 seconds of CPU usage from a validator
deployer aws profile \
    --config mychain.yaml \
    --instance i-0123456789abcdef \
    --duration 60

# Output: profile.json (samply's own format) +
#         profile.svg (a flame graph, opened in browser)
```

What you do with the flame graph:

- A wide block at "verify_signature" tells you your voting traffic is
  signature-bound (rare with ed25519 but possible with BLS).
- A wide block at "context.sleep" or "tokio::park" tells you validators
  are spending time waiting on each other (network-bound).
- A wide block at "rocksdb::compaction" tells you storage is the
  bottleneck (read or write).
- A wide block at "json::serialize" tells you the codec is doing more
  than it should (chapter 03 — switch to the binary codec).

**Use this**, not guess-and-check. The deployer's `profile` command is
production-safe: samply uses sampled profiling, so overhead is well
under 5%, and the instance keeps producing blocks the whole time.

## Cost considerations — the bill, broken down

A typical production deployment runs to a predictable monthly bill.
Walk through the line items.

### EC2 — compute

Pricing models you can pick from:

| Model           | c6i.xlarge | Tradeoff                                              |
|-----------------|------------|-------------------------------------------------------|
| On-demand       | ~$0.192/hr | Most flexible, ~$140/mo per instance, no commitment   |
| Reserved (1y)   | ~$0.118/hr | ~38% discount, commit for 12 months                    |
| Reserved (3y)   | ~$0.072/hr | ~62% discount, commit for 36 months                    |
| Spot            | ~$0.058/hr | ~70% discount, can be reclaimed with 2-min notice     |
| Savings Plans   | varies     | Flexible across instance families                      |

For a **testnet**, spot is a great match. The 2-minute reclaim
warning is well within the recovery time the BFT layer handles
transparently. For a **mainnet**, reserved instances or savings plans
are the norm — you can't tolerate spot reclamation mid-finalization.

### S3 — storage

Storage classes matter more than people realize:

| Class                          | $/GB-month | Retrieval  | Use for                              |
|--------------------------------|------------|------------|--------------------------------------|
| S3 Standard                    | $0.023     | instant    | Hot binaries + configs               |
| S3 Standard-IA                 | $0.0125    | instant    | Cold configs, accessed quarterly     |
| S3 Glacier Instant Retrieval   | $0.004     | ms         | Archived notarizations, blocks       |
| S3 Glacier Flexible Retrieval  | $0.0036    | minutes    | Archived audit logs                  |
| S3 Glacier Deep Archive        | $0.00099   | hours      | 7-year audit retention if you must   |

The deployer's default lifecycle transitions notarizations to Glacier
Instant after 90 days. The monthly cost for 100 GB of notarization
data drops from $2.30 to $0.40.

### Data transfer — the silent killer

Data transfer **out** to the internet is billable per GB:

| From → To               | $/GB  |
|-------------------------|-------|
| EC2 → internet          | $0.09 |
| EC2 → CloudFront        | $0.00 |
| S3 → internet           | $0.09 |
| Region A → Region B     | $0.02 |

Two scenarios to watch:

1. **Validator gossiping across regions.** Each cross-region message
   costs $0.02/GB. Simplex emits ~10 KB/sec per validator of P2P
   traffic. With 50 validators split 25/25 across `us-east-1` and
   `us-west-2`, the cross-region bill is roughly $3-5/month. Small.

2. **Archival reads.** If a client in Tokyo pulls a year-old notarization
   archive from `us-east-1`, they pay internet egress on the response.
   This is the larger bill. The fix is CloudFront in front of S3, or
   region-pinned archives (an S3 replication rule from `us-east-1` →
   `ap-northeast-1` for that one client base).

### CloudWatch — metrics + logs

- `PutMetricData`: $0.01 per 1k metrics. Cheap.
- `Logs ingested`: $0.50/GB. **This is where the bill balloons** if
  you log at trace level in production. The fix: log at `info` by
  default, use `debug` only when reproducing.

### A worked example

A 100-validator mainnet, 4 regions × 25 validators, c6i.2xlarge
on-demand:

```
4 regions × 25 instances × $0.384/hr × 730 hr    = $28,032/mo  (EC2)
100 instances × 200 GB gp3 storage × $0.115/GB    = $2,300/mo   (EBS)
S3 (1 TB mixed lifecycle)                          =    $40/mo  (S3)
Data transfer (cross-region, ~500 GB)              =    $10/mo
CloudWatch (100 instances, info-level logs)        =   $350/mo
                                                  ---------
                                                  ~$30,700/mo
```

Reserve those instances for 3 years and the EC2 line drops to
~$11,000/mo. **The biggest lever is reserved-vs-on-demand; the
fastest way to lose money is on-demand everything.**

## Disaster recovery — what happens when a region dies

A single region going down is the canonical failure Commonware's
deployer is built against. The recovery playbook, from cold:

1. **Notice:** CloudWatch alarm on multi-region latency goes red,
   or AWS publishes a region-wide incident. The deployer doesn't
   need to act here — validators in surviving regions keep
   finalizing (at `n-25` honest, you're well above the `n/3`
   Byzantine minimum).

2. **Assess:** `deployer aws list --config mychain.yaml` shows which
   instances are gone. The remaining `us-west-2` + `eu-west-1` +
   `ap-southeast-1` validators keep going. You have a 75-validator
   network at this point — still a healthy BFT.

3. **Spawn a replacement region:** the deployer has a different config
   file pointing at `ca-central-1` (a region you haven't used yet),
   with the same binary and configs. `deployer aws create --config
   dr-recovery.yaml`. The new validators join the network using their
   canonical public keys (which they always know, via the deployment's
   shared `participants` config).

4. **Catch up:** the new validators use `marshal::standard` or
   `Start::Height` (chapter 12) to fetch recent blocks from the
   existing ones. The bridge across the dead region is gone, but the
   new region bridges via the surviving regions.

5. **Restore S3 data:** the S3 bucket has **cross-region replication**
   enabled (the deployer does this by default). New region writes
   replicate to a bucket in the replacement region. Historical data
   survives the outage.

6. **Resume normal operations:** once the dead region comes back,
   you have a 5-region network. Tear down the DR region with
   `deployer aws destroy --config dr-recovery.yaml`. Or leave it for
   redundancy — that's the whole point of multi-region.

### The WAL pattern — how validators recover

The validator's local write-ahead log (`storage::journal`) is the unit
of recovery for a single instance. The flow on restart:

```
find last finalized height in WAL
  ↓
if known: ask the network for state sync from that height to current
  ↓
if unknown (full replay from genesis): replay the entire WAL
  ↓
join Simplex at the latest view
```

The WAL pattern means losing an instance is non-fatal — it just
fetches its state from peers on restart. This is the same pattern as
Postgres or Kafka, but tuned for adversarial environments: the WAL is
authentically structured so a Byzantine peer cannot feed you bogus
state during sync (chapter 16).

## Library mode — programmatic deploys

The deployer is also a Rust crate (`deployer/src/aws/lib.rs`), and most
of the heavy lifting is exposed as a library API. Programmatic use
lets you build deployment workflows the CLI doesn't cover — typical
uses:

- **CI/CD orchestration.** Custom GitHub Actions runners that deploy,
  smoke-test, and tear down.
- **Multi-config fanout.** A single push deploys the same binary to
  dev/staging/prod with different YAML configs.
- **Benchmark harness.** Deploy `examples/estimator` to dozens of
  regions in parallel for cross-region latency characterization
  (Commonware does this internally for the `1500 daily benchmarks`
  figure on the dashboard).

```rust
use commonware_deployer::aws::{Deployer, Config};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let deployer = Deployer::new(AwsConfig::from_env().await?);
    let config = Config::from_file("ci-bench.yaml")?;

    deployer.create(&config).await?;
    deployer.wait_healthy(&config, Duration::from_secs(300)).await?;

    // Run the workload
    let results = run_benchmark(&deployer, &config).await?;

    // Always tear down, even on failure
    let _ = deployer.destroy(&config).await;
    Ok(results)
}
```

The library API is async — every operation returns a future. Use
`tokio` directly; the deployer is a runtime-owning crate by design
(chapter 02 isolation rule).

## CI integration — `just deploy`

A typical GitHub Actions workflow that uses the deployer
(`deployer/Cargo.toml` already exports `commonware-deployer` as a
binary):

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/GitHubActionsDeployer
          aws-region: us-east-1

      - name: Install just + Rust
        run: |
          curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash
          rustup default stable

      - name: Build release binary
        run: cargo build --release --bin mychain

      - name: Deploy
        run: deployer aws create --config .github/deploy/testnet.yaml

      - name: Smoke test
        run: |
          ./scripts/smoke-test.sh \
            --endpoint deployer-status-testnet-us-east-1

      - name: Update DNS
        if: success()
        run: ./scripts/update-route53.sh

      - name: Tear down
        if: always()
        run: deployer aws destroy --config .github/deploy/testnet.yaml
```

The deployer's `aws status` (in the `list` subcommand) returns the
public IPs of running instances, so the smoke test can talk to them
directly. The DNS step is where the production deployment differs
from a CI deployment — CI tears down, prod keeps the DNS pointing at
the new instances.

A `just deploy` recipe wraps the whole flow:

```just
# justfile
deploy config:
    cargo build --release
    deployer aws create --config {{config}}
    deployer aws status --config {{config}}

destroy config:
    deployer aws destroy --config {{config}}
    @echo "destroyed {{config}}"
```

## Common gotchas — what trips people up

A grab-bag of failure modes the deployer doesn't catch for you.

### Regional quotas

Default AWS account quotas are brutal:

- EC2 instances per region: **5** (default, can be raised to thousands)
- VPCs per region: 5
- EIPs per region: 5
- IAM roles: 1000 (plenty)
- S3 buckets: 100 (plenty)

The first time you launch 25 instances in `ca-central-1`, you'll hit
the `RunInstances` quota and the deploy will fail with a
`RequestLimitExceeded` error. The fix is to **request quota increases
ahead of time** — a one-time 30-minute support ticket raises it. The
deployer's create command does check quota and warn, but it can't
raise it for you.

### IAM permission drift

The deployer's IAM principal needs all of `ec2:RunInstances`,
`iam:PassRole`, `s3:*` on the bucket prefix. If your org uses
permission boundaries (an SCP that gates everything), the deployer
will fail with `AccessDenied` errors that look like bugs but are
actually your boundary. The fix is to make sure the deployer role
is **outside** the boundary (or grant explicit permissions inside).

### Security group + NACL mismatch

Network ACLs (NACLs) are stateless, subnet-level firewall rules,
distinct from security groups. They default to "allow all," which is
fine for the deployer's defaults, but if your org enforces strict
NACLs (denying P2P ports), the validators can launch but never talk
to each other. The fix is to align NACLs and SGs in the same config
audit.

### Cross-region latency surprises

Validators in `us-east-1` (Virginia) and `ap-southeast-2` (Sydney)
see ~200ms RTT. Consensus still finalizes, but block times stretch.
The deployer doesn't warn you about cross-continent deployments. The
fix is to pick regions with <100ms RTT for the majority of your
validators, or accept the latency cost and plan for it (longer
timeouts).

### S3 bucket name collisions

S3 bucket names are globally unique across **all** AWS accounts.
Deploying two clusters with the same name (one in production, one in
CI) will fail with `BucketAlreadyExists`. The deployer appends a
random suffix by default — but if you override `bucket_name` in the
YAML, you're on the hook for uniqueness.

### Forgetting to seed fresh regions

A validator in `eu-west-1` that has never been used has no AMI
cache, no IAM service-linked role, etc. First deploy is slow
(2-3 minutes) while AWS warms up. Subsequent deploys in the same
region are fast (~30 seconds). The fix is to do a one-time
"warmup deploy" before your actual deployment matters.

### Missing egress rules

The deployer's default security group opens all outbound. If your
org enforces "no internet egress" (PCI compliance), validators can't
pull binaries from S3. The fix is a VPC endpoint for S3
(`com.amazonaws.us-east-1.s3` Gateway endpoint) — free, and avoids
the NAT hop.

## Rust pattern — async deploy with progress and backoff

The deployer's underlying API is async with progress reporting and
retry. Here's the shape of a production-grade deploy loop
(heavily condensed from `deployer/src/aws/create.rs`):

```rust
async fn create_with_progress(
    deployer: &Deployer,
    config: &Config,
) -> Result<()> {
    let total_steps = 7;
    let pb = ProgressBar::new(total_steps);
    pb.set_style(/* "▸ {bar} {pos}/{len} {msg}" */);

    // Step 1 — S3 bucket (may need retry; S3 eventual consistency)
    pb.set_message("creating S3 bucket");
    let bucket = retry_with_backoff(|| async {
        deployer.s3.create_bucket(&config.bucket_name).await
    }, 5).await?;
    pb.inc();

    // Step 2 — IAM role
    pb.set_message("creating IAM role");
    deployer.iam.create_role(&config.role_name, &policy_doc).await?;
    pb.inc();

    // ... steps 3-6 similar ...

    // Step 7 — EC2 (the slow one; report per-instance progress)
    pb.set_message(format!("launching {} instances", config.total_instances()));
    let mut instances = Vec::with_capacity(config.total_instances());
    let mut stream = deployer.ec2.run_instances_streamed(&launch_cfg).await?;
    while let Some(ev) = stream.next().await {
        match ev? {
            RunEvent::Launched(inst) => {
                instances.push(inst);
                pb.set_message(format!(
                    "launched {}/{}",
                    instances.len(),
                    config.total_instances(),
                ));
            }
            RunEvent::Failed(id, err) => {
                tracing::error!(instance = %id, error = %err, "launch failed");
                // Don't bail; some launches always fail in a quota-tight env
            }
        }
    }
    pb.inc();

    // Wait for instances to pass status checks
    pb.set_message("waiting for instances to be ready");
    deployer.ec2.wait_until_running(&instances, Duration::from_secs(300))
        .await?;
    pb.inc();

    pb.finish_with_message("deploy complete");
    Ok(())
}

async fn retry_with_backoff<T, F, Fut>(
    mut op: F,
    max_attempts: u32,
) -> Result<T>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T>>,
{
    let mut delay = Duration::from_millis(100);
    for attempt in 0..max_attempts {
        match op().await {
            Ok(v) => return Ok(v),
            Err(e) if e.is_transient() => {
                let jittered = delay + Duration::from_millis(rand::random::<u64>() % 100);
                tokio::time::sleep(jittered).await;
                delay *= 2;  // exponential
            }
            Err(e) => return Err(e),
        }
    }
    Err(anyhow!("max attempts exceeded"))
}
```

The shape you'll see again and again: `indicatif` (or `tracing`) for
progress, exponential backoff with jitter for transient AWS failures,
streaming for the long-tail operations like EC2 launch. The deployer's
internal `create.rs` does exactly this — when you read it, look for
the `Stream` returned by `RunInstances` and the
`tokio::time::sleep` jitter in `retry_with_backoff`.

## AWS infrastructure in full — the layered topology

The mental model from earlier is correct but thin. Here's the full
topology, layer by layer, with the deployment-relevant details.

### The geographic hierarchy — regions

A **region** (`us-east-1`, `eu-west-1`, `ap-southeast-2`) is a geographic
cluster of one or more physically isolated data centers. AWS regions are
**independent failure domains** — a region has its own power grid, its
own network backbone, its own staff. They do not share fate.

The regions that matter for BFT:

| Region code     | Location                  | Commonware relevance                  |
|-----------------|---------------------------|---------------------------------------|
| `us-east-1`     | Virginia, USA             | Default; cheapest; sometimes crowded. |
| `us-east-2`     | Ohio, USA                 | Good pairing with us-east-1 (~15ms).  |
| `us-west-2`     | Oregon, USA               | Cool climate, cheap power.            |
| `eu-west-1`     | Ireland                   | Common EU choice.                     |
| `eu-central-1`  | Frankfurt                 | Strict data residency.                |
| `ap-southeast-1`| Singapore                 | Asia-Pacific presence.                |
| `ap-northeast-1`| Tokyo                     | Latency-sensitive Asia.               |

Each region has a separate API endpoint
(`ec2.us-east-1.amazonaws.com`). Credentials are global; resources are
regional. A bucket in `us-east-1` is not visible to a process in
`us-west-2` unless explicitly replicated.

For a BFT network: pick three to five regions, ideally on two
continents. Don't go below three. Multi-region is the **single most
important defense** against regional outages.

### Availability Zones — within a region

Each region has 2-6 **Availability Zones** (AZs), typically three. Each
AZ is one or more physically separate data centers, with independent
power and networking, but with sub-10ms RTT between AZs in the same
region.

The crucial property: **AZ failures are common; region failures are
rare.** A region has had several high-profile outages (the 2017
`us-east-1` S3 outage, the 2021 `us-east-1` Kinesis outage, the 2023
`us-east-1` AWS Control Tower issue). An AZ fails much more often
(network switch reboots, power maintenance, cooling failures).

Commonware's deployer is opinionated: **place validators in distinct
AZs within a region**, so an AZ failure takes out at most a third of
the local validators. With `f < n/3` Byzantine tolerance, losing a
third of validators is survivable.

### VPCs — the network container

A **Virtual Private Cloud (VPC)** is your isolated network in AWS. It
spans every AZ in a region. Your subnets, route tables, security groups,
and ENIs (elastic network interfaces) all live inside it.

The CIDR block: `10.0.0.0/16` is conventional, giving 65,536 addresses.
The VPC itself doesn't do much — it's a logical container. The work
happens at the subnet, route table, and security-group level.

The deployer's default behavior: **use the region default VPC**. This
is fine for getting started. For production, create an explicit VPC:

```hcl
resource "aws_vpc" "validator" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "validator-vpc-${var.deployment}"
  }
}
```

Why create your own? Three reasons:

1. **IP planning.** The default VPC has a `172.31.0.0/16` range that
   overlaps with many corporate networks. VPC peering or VPN tunnels
   become painful.
2. **Network ACLs.** Default VPCs come with a permissive NACL that
   you cannot edit. An explicit VPC lets you apply strict NACLs (see
   the networking section below).
3. **Compliance.** PCI-DSS, HIPAA, FedRAMP all require explicit
   network segmentation. Default VPCs fail those audits.

### Subnets — AZ-tied slices

A **subnet** is a CIDR range tied to one specific AZ. Subnets don't
span AZs. A `10.0.1.0/24` in `us-east-1a` cannot host resources in
`us-east-1b`.

Two flavors:

- **Public subnets** — have a route to an Internet Gateway. Instances
  get public IPs.
- **Private subnets** — only outbound via NAT Gateway. Instances have
  no public IPs.

For BFT validators, **public subnets are simpler** if the validators
need to be reachable from the public internet (the most common case).
**Private subnets are safer** if all peers are known (e.g., a
permissioned consortium): no direct attack surface.

```
VPC: 10.0.0.0/16
├── Public subnet 10.0.1.0/24 in us-east-1a   → validator-0
├── Public subnet 10.0.2.0/24 in us-east-1b   → validator-1
├── Public subnet 10.0.3.0/24 in us-east-1c   → validator-2
├── Private subnet 10.0.10.0/24 in us-east-1a → monitoring (Prometheus)
└── Private subnet 10.0.20.0/24 in us-east-1a → DB / future
```

For a BFT deployment, you want at least one public subnet per AZ where
validators live. Monitoring and DB subnets are typically private.

### Internet Gateways and NAT Gateways

An **Internet Gateway (IGW)** is a horizontally-scaled AWS-managed
gateway that provides a route to the public internet. Attach one to
your VPC, then add a route in your public subnet's route table:

```
Destination: 0.0.0.0/0   →   igw-id
```

A **NAT Gateway** is a managed network address translation service. It
lets private subnet instances reach the internet (for S3, CloudWatch,
NTP) without being reachable from the internet. Costs ~$0.045/hr + data
processing.

The deployer creates both as needed. For the typical BFT deployment
with public-subnet validators, you only need the IGW. NAT is for
private-subnet validators.

## EC2 in full — the compute layer

EC2 is AWS's virtual machine service. The deployer treats instances as
the unit of validator deployment — one EC2 instance per validator.

### Instance types — what each family is for

AWS offers dozens of instance types, grouped into families:

| Family        | Use case                          | Examples                  |
|---------------|-----------------------------------|---------------------------|
| `t3` / `t4g` | Burstable, dev/test               | t3.medium, t4g.large      |
| `m5` / `m6i` | General purpose, balanced          | m6i.xlarge, m6i.2xlarge   |
| `c5` / `c6i` | Compute-optimized                 | c6i.xlarge, c6i.4xlarge   |
| `r5` / `r6i` | Memory-optimized                  | r6i.2xlarge               |
| `x2`          | High-memory (databases)           | x2idn.16xlarge            |
| `i3` / `i4i` | Local NVMe storage                | i3en.2xlarge              |
| `d2`          | Dense HDD storage                 | d2.xlarge                 |
| `p4` / `g5`   | GPU                               | p4d.24xlarge, g5.12xlarge |

For a BFT validator, the relevant choices:

- **`c6i.xlarge`** (4 vCPU, 8 GB RAM, ~$0.192/hr on-demand) — the
  Commonware default. BLS signature verification is compute-bound;
  compute-optimized is the right family.
- **`c6i.2xlarge`** (8 vCPU, 16 GB RAM) — production default; handles
  higher block rates.
- **`r6i.xlarge`** (4 vCPU, 32 GB RAM) — when the consensus app is
  memory-heavy (large state caches, QMDB hot data).
- **`t3.medium`** (2 vCPU, 4 GB RAM, ~$0.04/hr) — testnet only. Burstable
  means sustained CPU is limited; BFT is not burst-friendly.

### Pricing models

| Model           | Discount    | Commitment   | Risk                       |
|-----------------|-------------|--------------|----------------------------|
| On-demand       | 0%          | None         | Highest cost, no commitment |
| Reserved (1y)   | ~38%        | 1 year       | Locked-in                  |
| Reserved (3y)   | ~62%        | 3 years      | Long lock-in               |
| Savings Plans   | ~30%        | 1-3 years    | Family-flexible            |
| Spot            | ~70%        | None         | Can be reclaimed (2-min notice) |
| Dedicated Host  | varies      | 3 years      | BYOL licensing             |

**For testnets:** Spot. The 2-minute reclaim warning is well within
BFT recovery time. A validator that gets reclaimed restarts from
its WAL (chapter 16), and consensus continues.

**For mainnet:** Reserved Instances or Savings Plans. You cannot
tolerate a spot reclamation mid-finalization. The 38-62% discount
on the largest cost line is decisive.

**For dev/staging:** On-demand. The flexibility to tear down at 5pm
Friday matters more than the discount.

### AMIs (Amazon Machine Images)

An AMI is a snapshot of a virtual machine's root filesystem plus launch
permissions. The deployer uses `Amazon Linux 2023` as its base AMI for
two reasons:

1. The SSM (Systems Manager) agent is pre-installed and pre-configured.
   The `update` command's "send a graceful restart signal" depends on
   SSM.
2. The AWS CLI is in the package manager. The user-data script that
   pulls binaries from S3 needs `aws s3 cp`.

Alternative AMIs the deployer supports:

- `Ubuntu 22.04 LTS` — familiar to most teams, slightly slower to
  launch (no SSM agent pre-installed).
- `Debian 12` — minimal; what Commonware uses internally for
  benchmarks.

To bake your own AMI (e.g., for binary pre-installation), use
[`aws-image-builder`](https://aws.amazon.com/image-builder/) or packer.
The deployer's `images.rs` module handles AMI selection; you override
via the YAML config:

```yaml
ami_id: ami-0123456789abcdef   # custom AMI for validator
ami_id: ""                     # auto-select latest Amazon Linux 2023
```

### User data — bootstrap at launch

User data is a shell script (or cloud-init directive) that EC2 runs
once on first boot. The deployer's user-data script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Mount additional EBS volumes (if any)
mkdir -p /data
mount /dev/nvme1n1 /data || true   # optional journal volume

# Install runtime dependencies
dnf update -y
dnf install -y awscli jq

# Pull the validator binary from S3
mkdir -p /opt/mychain
aws s3 cp s3://mychain-deployment-${DEPLOY_SUFFIX}/binaries/mychain /opt/mychain/
chmod +x /opt/mychain/mychain

# Pull the per-instance config
aws s3 cp s3://mychain-deployment-${DEPLOY_SUFFIX}/configs/node-${NODE_INDEX}.toml /etc/mychain.toml

# Set up systemd
cat > /etc/systemd/system/mychain.service <<'EOF'
[Unit]
Description=mychain validator
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/opt/mychain/mychain --config /etc/mychain.toml
Restart=on-failure
RestartSec=5s
User=mychain
Group=mychain
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable mychain.service
systemctl start mychain.service
```

The script is **idempotent** on retry (S3 cp overwrites; systemctl
restart is fine). It assumes the EC2 instance role has
`s3:GetObject` permission on the deployment bucket (which the deployer
configures).

### Instance Metadata Service (IMDS)

Every EC2 instance has access to a metadata service at
`http://169.254.169.254/latest/meta-data/`. The deployer uses IMDS to:

1. **Discover the instance's own region, AZ, and instance ID** for
   tagging.
2. **Read instance role credentials** (no need to bake AWS keys into
   user data).

The deployer hardcodes **IMDSv2 only** (token-based, no IMDSv1
fallback). IMDSv1 had a well-known SSRF vulnerability class
(Capital One breach, 2019); v2 mitigates it.

```rust
// deployer fetches instance metadata via IMDSv2
let token = client.put("http://169.254.169.254/latest/api/token")
    .header("X-aws-ec2-metadata-token-ttl-seconds", "21600")
    .send().await?
    .text().await?;
let instance_id = client.get("http://169.254.169.254/latest/meta-data/instance-id")
    .header("X-aws-ec2-metadata-token", token)
    .send().await?
    .text().await?;
```

## S3 in full — the object store

S3 is AWS's object storage. The deployer uses it as both a binary
registry (the equivalent of an "artifact server" in CI) and a shared
backing store for state that needs to survive instance restarts.

### Buckets — the namespace

A **bucket** is a flat namespace for objects. Bucket names are
**globally unique** across all AWS accounts (you cannot have
`mychain-deployment` if another account created it first). Names must:

- Be 3-63 characters.
- Use only lowercase letters, numbers, hyphens, and periods.
- Begin and end with a letter or number.
- Not be formatted as IP addresses.

The deployer appends a random 8-character suffix to deployment names
to avoid collisions: `mychain-deployment-x7k3pq9z`.

### Objects — keys and values

An **object** is a key + value + metadata. Keys are Unicode strings up
to 1024 bytes. Values can be 0 bytes to 5 TB.

The deployer's bucket layout:

```
s3://mychain-deployment-x7k3pq9z/
├── binaries/
│   └── mychain/
│       ├── mychain-v0.1.0              # tagged releases
│       ├── mychain-v0.1.0-rc.1
│       └── mychain-${GIT_SHA}          # debug builds
├── configs/
│   └── mychain/
│       ├── node-0.toml
│       ├── node-1.toml
│       └── ...
└── shared/
    ├── notarizations/
    ├── blocks/
    └── checkpoints/
```

The `binaries/` and `configs/` prefixes are deployer-managed. The
`shared/` prefix is yours — `storage::archive` (chapter 06) writes
here.

### Storage classes

| Class                              | $/GB-month | Retrieval latency | Use for                       |
|------------------------------------|------------|-------------------|-------------------------------|
| S3 Standard                        | $0.023     | Instant           | Hot binaries, configs         |
| S3 Intelligent-Tiering             | varies     | Instant           | Unknown access pattern        |
| S3 Standard-IA                     | $0.0125    | Instant           | Infrequently accessed        |
| S3 One Zone-IA                     | $0.01      | Instant           | Single-AZ, recreatable       |
| S3 Glacier Instant Retrieval       | $0.004     | Milliseconds      | Archived notarizations        |
| S3 Glacier Flexible Retrieval      | $0.0036    | Minutes-hours     | Cold backups                 |
| S3 Glacier Deep Archive            | $0.00099   | Hours             | Compliance retention         |

For BFT deployments, **Standard** for hot data (`binaries/`,
`configs/`), **Glacier Instant Retrieval** for archived notarizations
and blocks (accessed rarely but must be available quickly when a
client needs them), and **Glacier Deep Archive** only for
multi-year regulatory retention.

### Lifecycle policies

Lifecycle policies automate transitions between storage classes. The
deployer's default policy:

```json
{
  "Rules": [
    {
      "ID": "binaries-lifecycle",
      "Filter": { "Prefix": "binaries/" },
      "Status": "Enabled",
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 30, "StorageClass": "GLACIER_IR" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 }
    },
    {
      "ID": "shared-lifecycle",
      "Filter": { "Prefix": "shared/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 90, "StorageClass": "GLACIER_IR" }
      ],
      "Expiration": { "Days": 2555 }
    }
  ]
}
```

The `binaries/` rule: keep last 10 versions in Standard; older
versions go to Glacier IR after 30 days; expire after 90 days (so the
storage doesn't grow unbounded).

The `shared/` rule: transition to Glacier IR after 90 days; expire
after 7 years (a typical audit retention).

### Encryption

S3 encrypts all objects at rest. Two server-side options:

- **SSE-S3** — AES-256, keys managed by AWS. Default. Free.
- **SSE-KMS** — AES-256, keys managed by AWS KMS. Per-request cost.
  Use when you need access logging via CloudTrail.

The deployer uses **SSE-S3 by default** and enforces it via a bucket
policy that denies any `PutObject` without the
`x-amz-server-side-encryption` header:

```json
{
  "Sid": "DenyUnencryptedUploads",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::mychain-deployment-*/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "AES256"
    }
  }
}
```

This is defense-in-depth: even if a misconfigured client tries to push
plaintext, the bucket rejects it.

### Versioning

Versioning keeps multiple variants of an object. Enabled by default in
the deployer's bucket. Cost: each version is stored separately.

For `binaries/`, versioning is essential — you can roll back to a
previous version by repointing the user-data script.

For `configs/`, versioning is useful for audit trails.

For `shared/`, versioning is wasteful — notarizations are immutable
once written, so the "old version" is the same as the "current
version." Consider **disabling versioning** on `shared/` if you're
optimizing for storage cost.

### Replication

**Cross-Region Replication (CRR)** copies new objects to a bucket in
another region, automatically. The deployer enables CRR by default for
disaster recovery:

```json
{
  "Role": "arn:aws:iam::ACCOUNT:role/S3ReplicationRole",
  "Rules": [{
    "ID": "ReplicateAll",
    "Status": "Enabled",
    "Filter": { "Prefix": "" },
    "Destination": {
      "Bucket": "arn:aws:s3:::mychain-deployment-dr-us-west-2",
      "StorageClass": "STANDARD_IA"
    }
  }]
}
```

The DR bucket sits in a different region. If `us-east-1` has a
catastrophic failure, the DR bucket in `us-west-2` has your data.

Replication is **eventual** — typically within seconds, but not
instantaneous. For critical writes, the application must wait for
replication ack or accept eventual consistency.

## IAM in full — the identity boundary

IAM (Identity and Access Management) is AWS's authorization layer.
The deployer uses two distinct IAM principals to enforce the
**principle of least privilege**: one for the deployer itself
(broad powers), one for each validator (narrow powers).

### The JSON policy structure

Every IAM policy is a JSON document with this shape:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OptionalStatementId",
      "Effect": "Allow" | "Deny",
      "Action": ["service:Action", ...],
      "Resource": ["arn:aws:service:region:account:resource", ...],
      "Condition": { ... }
    }
  ]
}
```

The fields:

- **`Version`** — policy language version. `"2012-10-17"` is current.
- **`Statement`** — array of allow/deny rules.
- **`Sid`** — optional human-readable statement ID.
- **`Effect`** — `"Allow"` permits, `"Deny"` forbids. Explicit `Deny`
  always wins over `Allow`.
- **`Action`** — list of API actions. Supports wildcards:
  `"s3:Get*"` matches all Get operations.
- **`Resource`** — list of ARN patterns the action applies to. Use
  specific ARNs, not `*`.
- **`Condition`** — optional context-based restriction (IP, MFA,
  time-of-day, tags).

### The principle of least privilege

Every IAM principal should have **exactly the permissions it needs, no
more**. This is harder than it sounds: AWS APIs are fine-grained, and
over-permissioning is the norm.

The deployer's approach: **two principals, maximally separated**.

**Deployer principal** — your laptop or CI:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:CreateSecurityGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:AssociateAddress",
        "ec2:DisassociateAddress",
        "ec2:CreateNetworkInterface",
        "ec2:AttachNetworkInterface",
        "ec2:DetachNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "s3:CreateBucket",
        "s3:DeleteBucket",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning",
        "s3:GetBucketEncryption",
        "s3:PutBucketEncryption",
        "s3:PutLifecycleConfiguration",
        "s3:GetLifecycleConfiguration",
        "s3:PutBucketReplication",
        "s3:GetBucketReplication",
        "s3:PutBucketPolicy",
        "s3:GetBucketPolicy",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PassRole",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "cloudwatch:PutMetricData",
        "cloudwatch:GetMetricStatistics",
        "logs:CreateLogGroup",
        "logs:DeleteLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "ssm:SendCommand",
        "ssm:GetCommandInvocation",
        "ssm:DescribeInstanceInformation"
      ],
      "Resource": "*"
    }
  ]
}
```

**Validator instance role** — assumed by each EC2 validator:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadingDeploymentBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::mychain-deployment-*",
        "arn:aws:s3:::mychain-deployment-*/*"
      ]
    },
    {
      "Sid": "AllowEmittingMetrics",
      "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"],
      "Resource": "*"
    },
    {
      "Sid": "AllowWritingLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:mychain-*:*"
    }
  ]
}
```

The deployer can create, modify, and destroy anything. The validator
can read its own binary and emit metrics. **Compromise of a validator
yields no cluster-admin powers** — the worst the attacker can do is
read public binaries and emit metrics. They cannot spawn new
instances, access other buckets, or modify IAM roles.

### The trust relationship

For the validator instance role to be assumable by EC2, its trust
policy must allow `ec2.amazonaws.com`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

Without this, EC2 cannot pass the role to the instance, and the
instance gets no AWS credentials.

### The deployer-vs-validator split — why it matters

The split exists because the deployer and validator run in **different
trust contexts**:

- The deployer is interactive (a developer or CI job). It can make
  mistakes. It needs broad powers **but its session is short-lived**
  (a CI job runs for minutes; a deploy takes an hour at most).
- The validator is autonomous. It runs for weeks without human
  intervention. It must have **no privilege to cause harm** even if
  an attacker fully compromises the underlying EC2 instance.

If you collapse them — give the validator the same broad IAM powers
as the deployer — you create a **lateral movement path**. An attacker
who compromises one validator can spawn 100 more instances, exfiltrate
the entire S3 bucket, and revoke your security group rules. The split
isn't paranoia; it's the standard production boundary.

### Permission boundaries and SCPs

If your organization uses AWS Organizations with **Service Control
Policies (SCPs)**, the deployer's IAM principal must be **outside the
SCP boundary** or the SCP must explicitly allow the deployer's
actions. SCPs are deny-by-default for the OU; the deployer's broad
`ec2:*`, `s3:*`, `iam:*` actions will be denied unless carved out.

If your organization uses **permission boundaries** (per-principal
max-permission limits), the deployer principal's effective permissions
are the **intersection** of its identity policy and its boundary. A
boundary set too tight will silently break the deployer with cryptic
`AccessDenied` errors.

The deployer's `validate_config` step checks for obvious permission
errors before provisioning, but cannot catch all SCP/boundary
interactions. If you see `AccessDenied` on a permissions combination
that "should work," check your SCP and boundary first.

## Networking in depth — the firewall layers

Two layers of firewall in AWS: **security groups** (stateful,
instance-level) and **Network ACLs** (stateless, subnet-level). The
deployer uses both, but the heavy lifting is in security groups.

### Security groups — the primary defense

A **security group (SG)** is a set of inbound and outbound rules
attached to an Elastic Network Interface (ENI). SGs are **stateful**:
if you allow inbound TCP 3000 from a peer, the response traffic is
automatically allowed (no need for an explicit outbound rule for the
return path).

Default SG behavior:

- **No inbound rules** — everything is denied by default.
- **All outbound allowed** — instances can reach anywhere.

The deployer creates one SG per region with three inbound rules:

```hcl
resource "aws_security_group" "validator" {
  name        = "validator-${var.deployment}"
  description = "BFT validator security group"
  vpc_id      = var.vpc_id

  ingress {
    description = "SSH from deployer"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.deployer_ip]   # /32 — single IP
  }

  ingress {
    description = "P2P consensus"
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = var.validator_subnet_cidrs  # restrict to peer subnets
  }

  ingress {
    description = "Prometheus metrics"
    from_port   = 9090
    to_port     = 9090
    protocol    = "tcp"
    cidr_blocks = var.monitoring_subnet_cidrs  # restrict to monitoring
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

The three rules are **least-privilege**:

- SSH from your IP only. Use `deployer aws authorize` to grant
  teammates temporary access.
- P2P from the validator subnets, not from `0.0.0.0/0`. This shrinks
  the attack surface dramatically — DNS rebinding, port scanning,
  SYN floods all become much harder.
- Metrics from the monitoring subnet only. The world doesn't need to
  see your Prometheus endpoint.

### Network ACLs — the second layer

**Network ACLs (NACLs)** are subnet-level, stateless firewalls. Each
rule must explicitly allow both directions; return traffic is not
auto-allowed like with SGs.

Default NACL: allow all. This is fine for getting started. Strict
deployments use a deny-by-default NACL:

```hcl
resource "aws_network_acl" "validator" {
  vpc_id = var.vpc_id
  subnet_ids = var.validator_subnet_ids

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "10.0.0.0/16"   # within VPC
    from_port  = 3000
    to_port    = 3000
  }

  # ephemeral ports for return traffic
  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  egress {
    protocol   = "-1"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
  }
}
```

The NACL is **belt and suspenders**. SGs are sufficient for most
threat models. NACLs help when:

- Your compliance regime requires subnet-level controls (PCI-DSS).
- You want to block entire IP ranges (e.g., known malicious networks).
- You need defense-in-depth against SG misconfiguration.

### Route tables — the routing layer

A **route table** is a set of routes that determines where traffic
flows within a VPC. Each subnet is associated with exactly one route
table.

The deployer's default route tables:

- **Public subnets** → `0.0.0.0/0` via Internet Gateway.
- **Private subnets** → `0.0.0.0/0` via NAT Gateway.

The route table associations are static (you can't dynamically route
through different gateways). For more complex topologies (multi-NIC,
VPN, Direct Connect), use Transit Gateway or VPC peering.

### NAT Gateways — outbound for private subnets

A **NAT Gateway** is a managed, highly-available network address
translation service. It lets private subnet instances reach the
internet (S3, CloudWatch, NTP) without being reachable from it.

Cost: ~$0.045/hour per NAT Gateway, plus per-GB data processing.
For a 100-validator deployment with all-private subnets, NAT alone
costs ~$330/month per region × 4 regions = $1,320/month.

**Common optimization**: share a single NAT Gateway across multiple
private subnets within an AZ. NAT Gateway is AZ-bound; if you have
multiple AZs, you need one NAT per AZ. To save cost: consolidate
subnets into fewer AZs (at the cost of AZ-failure blast radius).

### Internet Gateways — the public route

An **Internet Gateway (IGW)** is a horizontally-scaled AWS-managed
gateway providing a route from the VPC to the public internet.
Attach one to your VPC; add a route in the public subnet's route
table pointing `0.0.0.0/0` to the IGW.

The IGW is **free** — no hourly cost, no per-GB cost. The only cost
is the bandwidth out (covered in the cost section).

### VPC endpoints — private connectivity to AWS services

For private subnets that need S3 access, use a **VPC endpoint for S3**
(`com.amazonaws.us-east-1.s3`). This is a Gateway endpoint — free,
and routes S3 traffic over the AWS backbone without going through the
NAT.

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = var.vpc_id
  service_name = "com.amazonaws.${var.region}.s3"
  route_table_ids = var.private_route_table_ids
}
```

The deployer enables VPC endpoints by default for private-subnet
deployments. For public-subnet deployments, the public internet
route is fine and the endpoint is unnecessary.

## Observability in full — metrics, logs, traces, profiles

Observability has four pillars. The deployer wires three of them;
you write the fourth.

### The four pillars

1. **Metrics** — numerical time series. "Finalization latency p99 =
   312ms." Aggregatable, cheap, alertable.
2. **Logs** — discrete events with structured fields. "Validator 7
   notarized block 12345." Searchable, high-cardinality.
3. **Traces** — causal chains of events across services. "Request
   flow X took 250ms; consensus was 180ms, propagation was 70ms."
   Useful for debugging latency spikes.
4. **Profiles** — sampled call stacks. "Validator 7 spent 40% of CPU
   in BLS verification." The deployer's `aws profile` command.

### Metrics — OTLP and Prometheus

The runtime emits metrics via the `metrics` crate facade, with two
exposition formats:

- **Prometheus text format** — `metric_name{label="value"} number`,
  served via HTTP on `/metrics` (default port 9090). Scraped by
  Prometheus.
- **OTLP** — OpenTelemetry Protocol, pushed to a collector. Used when
  you have an OTLP-capable backend (Datadog, Honeycomb, etc.).

The deployer configures **Prometheus format** because it's the most
common and the cheapest. The Prometheus scraper runs **inside the VPC**
(typically on a `t3.small` instance or a managed Grafana Cloud
endpoint), pulling from the validators' `/metrics` endpoints.

Common BFT metrics to expose:

```rust
use commonware_runtime::Metrics;

let finalization_latency = context.register(
    "finalization_latency_seconds",
    "wall-clock from proposal broadcast to finalization",
    metrics::Histogram::new([0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0]),
);
let finalizations_total = context.register(
    "finalizations_total",
    "blocks finalized",
    metrics::Counter::new(),
);
let peers_connected = context.register(
    "peers_connected",
    "P2P peers currently connected",
    metrics::Gauge::new(),
);
let votes_sent = context.register_with_labels(
    "votes_sent_total",
    "vote messages sent",
    metrics::CounterFamily::default(),
    ["view", "destination"],
);
```

The deployer opens port 9090 only to the monitoring subnet. The world
doesn't see your metrics.

### Logs — structured via `tracing`

The runtime uses `tracing` with structured fields. The deployer
configures a JSON-layer subscriber that ships logs to CloudWatch
Logs:

```
/aws/mychain/<region>/<instance-id>
    ├── 2026/07/05/[$LATEST]abcdef123...   # log streams
    └── ...
```

Each log line is one JSON object:

```json
{
  "timestamp": "2026-07-05T22:26:00.123Z",
  "level": "INFO",
  "target": "consensus::simplex::voter",
  "span": {"view": 42, "height": 12345},
  "fields": {"message": "voted notarize", "view": 42, "hash": "0xabc..."},
  "span_id": "...",
  "trace_id": "..."
}
```

CloudWatch Logs Insights lets you query these:

```
fields @timestamp, level, fields.message
| filter level = "ERROR"
| stats count() by fields.message
```

For production, ship CloudWatch Logs to **Loki** or **Datadog** for
long-term retention and richer query. The deployer supports a
configurable log forwarder:

```yaml
observability:
  log_forwarder: loki   # or: datadog, none
  log_forwarder_endpoint: https://logs-prod-eu-west-0.grafana.net
```

### Traces — OpenTelemetry

Commonware doesn't ship OTLP traces by default; the runtime's tracing
infrastructure is compatible but you'd add the OTLP exporter yourself.
For BFT debugging, **traces are often overkill** — the consensus is
in-process, so the trace IDs are not crossing service boundaries.

If you have a multi-component deployment (consensus + mempool +
sequencer + gossip), enable OTLP tracing across components to see the
end-to-end flow.

### Profiles — `samply` and flame graphs

The deployer's `aws profile` command runs `samply` on a running
instance and pulls back a flame graph. The default profiler is
`samply`; alternatives include `perf` (Linux's native profiler) and
`cargo flamegraph`.

`perf` (Linux kernel profiling) is more capable than `samply`:

```bash
# Sample at 1kHz for 60 seconds
perf record -F 1000 -p $(pidof mychain) -g -- sleep 60

# Generate a flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > profile.svg
```

The deployer abstracts this; you don't need to SSH and run `perf`
yourself. The command:

```bash
deployer aws profile --config mychain.yaml --instance i-0123 --duration 60
```

Output: `profile.json` (samply format) + `profile.svg` (flame graph).

**Reading a flame graph**:

- **Wide block at `verify_signature`** — voting traffic is signature-
  bound. Switch to BLS aggregation or buffered signatures.
- **Wide block at `tokio::park` / `context.sleep`** — validators
  waiting on each other. Network-bound.
- **Wide block at `rocksdb::compaction`** — storage bottleneck.
  Consider tiered storage or compaction tuning.
- **Wide block at `bincode::serialize` / `json::serialize`** —
  codec bottleneck. Switch to the binary codec (chapter 03).

### Alerting — what's worth a page

Not every metric deserves an alert. The deployer's recommended
alerts:

| Alert                              | Threshold                | Action                          |
|------------------------------------|--------------------------|---------------------------------|
| `finalization_lag > 5 blocks`      | Stale (no progress)      | Page on-call                    |
| `view_changes_per_minute > 10`     | Churn (leader failures)  | Investigate network             |
| `cpu_usage > 80% for 5min`         | Compute-bound            | Scale up or optimize            |
| `disk_usage > 85%`                 | Filling up               | Add EBS volume, run GC          |
| `peers_connected < 2f+1`           | Insufficient peers       | Investigate network partitions  |
| `error_log_rate > 1/min`           | Persistent errors        | Investigate immediately         |
| `finalization_latency_p99 > 2s`    | Slow finality            | Investigate leader              |

The deployer emits CloudWatch alarms for these on `aws create`. You
configure SNS topics for notification.

## CI/CD in full — the deployment pipeline

CI/CD is the automation layer that takes your code from "merged to
main" to "running on validators." The deployer is the **target** of
CI/CD, not a CI/CD system itself.

### GitHub Actions — the standard pipeline

GitHub Actions is the most common CI for Commonware projects. A
production deployment pipeline:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:           # manual trigger

env:
  AWS_REGION: us-east-1
  DEPLOY_SUFFIX: ${{ github.sha }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
      - name: Cache cargo
        uses: Swatinem/rust-cache@v2
      - name: Run tests
        run: cargo test --all-features
      - name: Run conformance
        run: just test-conformance

  build:
    needs: test
    runs-on: ubuntu-22.04          # newer for newer AWS CLI
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      - name: Install just + Rust
        run: |
          curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to /usr/local/bin
          rustup default stable
      - name: Build release binary
        run: cargo build --release --locked
      - name: Upload binary to S3
        run: |
          aws s3 cp target/release/mychain \
            s3://mychain-deployment-${{ env.DEPLOY_SUFFIX }}/binaries/mychain
      - name: Upload configs to S3
        run: |
          aws s3 cp configs/ \
            s3://mychain-deployment-${{ env.DEPLOY_SUFFIX }}/configs/ --recursive

  deploy-testnet:
    needs: build
    runs-on: ubuntu-latest
    environment: testnet           # requires approval
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      - name: Install deployer
        run: cargo install commonware-deployer --locked
      - name: Deploy to testnet
        run: deployer aws create --config .github/deploy/testnet.yaml
      - name: Smoke test
        run: ./scripts/smoke-test.sh testnet
      - name: Tear down (if test only)
        if: github.event_name == 'pull_request'
        run: deployer aws destroy --config .github/deploy/testnet.yaml

  deploy-prod:
    needs: build
    runs-on: ubuntu-latest
    environment: production        # requires approval
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      - name: Install deployer
        run: cargo install commonware-deployer --locked
      - name: Deploy to prod (rolling)
        run: |
          deployer aws update --config .github/deploy/prod.yaml \
            --binary-path ./target/release/mychain
      - name: Verify health
        run: ./scripts/health-check.sh prod
```

### Secrets management

GitHub Actions secrets are encrypted at rest. The deployer needs:

| Secret                  | What                                       |
|-------------------------|--------------------------------------------|
| `AWS_DEPLOY_ROLE_ARN`   | IAM role for the deployer to assume        |
| `AWS_PROD_DEPLOY_ROLE_ARN` | IAM role for prod (higher bar)           |
| `SSH_KEY_PRIVATE`       | (optional) SSH key for instance access     |
| `DEPLOYER_SIGNING_KEY`  | (optional) For signed deploy events       |

Use **OIDC-based AWS authentication** instead of long-lived AWS keys:

```yaml
- name: Configure AWS (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT:role/GitHubActionsDeployer
    aws-region: us-east-1
```

OIDC issues short-lived credentials (1 hour) tied to the GitHub
workflow. No static AWS keys in your CI secrets.

### The `just` recipes

Commonware uses `just` (a `make` alternative) for command
abbreviation. Canonical recipes:

```just
# justfile

# Default: build everything
_default:
    just --list

# Run unit tests
test:
    cargo test --workspace

# Run conformance tests
test-conformance:
    cargo test --workspace --features conformance

# Run benchmarks
bench:
    cargo bench --workspace

# Deploy to testnet
deploy-testnet:
    cargo build --release
    deployer aws create --config .github/deploy/testnet.yaml

# Update production (rolling binary replace)
deploy-prod:
    cargo build --release
    deployer aws update --config .github/deploy/prod.yaml

# Profile a running validator
profile instance_id:
    deployer aws profile --config .github/deploy/prod.yaml --instance {{instance_id}} --duration 60

# Tear down testnet (cleanup)
destroy-testnet:
    deployer aws destroy --config .github/deploy/testnet.yaml

# Stream logs from a validator
logs instance_id:
    aws logs tail /aws/mychain/prod/{{instance_id}} --follow

# Run the full integration test suite
integration:
    cargo test --workspace --features integration
```

`just` is installed via the official installer:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to /usr/local/bin
```

### Deployment strategies

Three common patterns:

**Rolling update** — replace validators one at a time. Validators
stay online; consensus never pauses. The deployer's default
`update` strategy.

**Blue-green** — deploy a parallel "green" set of validators, verify,
then route traffic. More expensive (double the instances during
transition); zero-downtime guarantee.

**Canary** — deploy to 1-2 validators first, monitor, then expand.
Commonware's recommended for production mainnet updates.

```bash
# Canary: deploy new binary to a single validator first
deployer aws update --config mainnet.yaml --canary-validator 7

# Monitor for 30 minutes
./scripts/monitor.sh

# If healthy, roll out to all
deployer aws update --config mainnet.yaml
```

### Rollback

The deployer stores the previous binary versioned in S3:

```
s3://mychain-deployment/
├── binaries/
│   └── mychain/
│       ├── mychain-v0.1.0          # current
│       └── mychain-v0.0.9          # previous (kept for rollback)
```

To roll back: update the user-data script on running instances to
point at the previous binary, or `deployer aws update --binary-path
./target/release/mychain-v0.0.9`. Rollback takes ~3 minutes
(restart cycle).

## Cost optimization in depth — TCO and levers

The cost section above gave the line items. Here are the levers, in
order of impact.

### Total cost of ownership — the full model

For a 100-validator mainnet across 4 regions:

```
EC2 (compute)                    $28,032 / month
  c6i.2xlarge × 100 instances × $0.384/hr × 730 hr

EBS (block storage)              $2,300 / month
  100 × 200 GB gp3 × $0.115/GB

S3 (object storage)              $40 / month
  1 TB mixed Standard + Glacier IR

Data transfer (cross-region)     $10 / month
  ~500 GB × $0.02/GB

NAT Gateway (if private subnets) $1,320 / month
  4 regions × $0.045/hr × 730 hr

CloudWatch (metrics + logs)       $350 / month
  100 instances, info-level logs

Route 53 (DNS)                   $5 / month
  Hosted zone + queries

Total on-demand                  ~$32,057 / month
Total with 3-year reserved       ~$15,000 / month
```

The biggest lever is reserved instances — a 53% cost reduction for
the same deployment. The second biggest is right-sizing instance
types (c6i.xlarge may be sufficient; benchmarks tell you).

### Optimization strategies

**1. Reserved Instances / Savings Plans**

| Commitment | Discount | Use when                              |
|------------|----------|---------------------------------------|
| 1-year RI  | ~38%     | You're sure of the workload           |
| 3-year RI  | ~62%     | Long-term commitment                  |
| 1-year SP  | ~27%     | Flexible instance family              |
| 3-year SP  | ~45%     | Flexible family, long-term            |

For a BFT mainnet, **3-year RIs** are the right call once you have
proven the deployment. Until then, **on-demand** for safety.

**2. Right-sizing**

Run `estimator` (chapter 19) to measure validator CPU usage at your
target block rate. If validators are at 30% CPU on `c6i.2xlarge`,
you're paying for 70% unused compute. Drop to `c6i.xlarge`.

**3. Spot for non-critical paths**

The aggregator network (chapter 13) doesn't need on-demand
instances. Spot is fine. The deployer supports mixed-instance
configurations:

```yaml
validators:
  instance_type: c6i.xlarge
  pricing: on-demand
aggregators:
  instance_type: c6i.large
  pricing: spot
```

**4. S3 lifecycle**

The default lifecycle transitions notarizations to Glacier IR after
90 days. If your retention period is shorter, expire them. If it's
longer, transition to Glacier Deep Archive after 1 year.

**5. CloudWatch Logs optimization**

The biggest hidden cost. CloudWatch Logs charges **$0.50/GB ingested**.
At trace level, a busy validator emits ~5 GB/day. At info level,
~50 MB/day. The difference is $75/validator/month.

**6. Data transfer**

Cross-region data transfer is $0.02/GB. Cross-AZ is $0.01/GB.
Internet egress is $0.09/GB. To minimize:

- Pin regions to the same continent (cross-continent is $0.09/GB).
- Use CloudFront in front of S3 for public reads.
- Compress gossip traffic (Borsh encoding, chapter 03, is
  ~30% smaller than JSON).

### When to use which pricing model

| Workload                      | Model             | Why                          |
|-------------------------------|-------------------|------------------------------|
| Testnet (always torn down)    | Spot              | 70% discount, recovery OK    |
| Dev/staging                   | On-demand         | Flexibility over discount    |
| Mainnet (proven)              | 3-year Reserved   | 62% discount, locked in      |
| Aggregator network            | Spot              | Non-critical                 |
| Read-replica / archive nodes  | Spot              | Non-critical                 |
| Disaster recovery region      | Spot (warm pool)  | Spun up rarely               |
| Performance benchmarks        | On-demand         | Predictable timing           |

## Disaster recovery in full — backup, failover, restore

The previous section covered the region-down scenario. Here's the
full DR playbook.

### Backup strategies

The deployer assumes **three layers of backup**:

1. **Local EBS snapshots** — taken nightly, kept for 7 days.
2. **S3 cross-region replication** — `binaries/`, `configs/`, and
   `shared/` replicate to a DR bucket in another region.
3. **Periodic state exports** — the application exports its state
   to S3 every N blocks (typically N=10000, ~30 minutes).

The first protects against instance failure. The second against
regional failure. The third against corruption or bugs.

### The WAL pattern — single-instance recovery

The validator's local write-ahead log (chapter 16) is the unit of
recovery for a single instance:

```
1. find last finalized height in WAL
2. if known: ask network for state sync from that height to current
3. if unknown (full replay from genesis): replay entire WAL
4. join Simplex at the latest view
```

The WAL pattern means losing an instance is **non-fatal**. The
restarted validator:

1. Reads its WAL to find the last finalized height.
2. Contacts peers via the resolver (chapter 08) to fetch blocks
   since that height.
3. Verifies each block against the quorum certificate.
4. Joins Simplex at the current view.

Total recovery time: 30 seconds to 5 minutes, depending on how far
behind the validator is.

### Multi-region failover

The region-down scenario in detail:

1. **T+0**: Region `us-east-1` becomes unreachable.
2. **T+0 to T+5min**: Validators in surviving regions detect the
   outage (TCP timeouts on `us-east-1` peers).
3. **T+5min**: Surviving validators at `n-25` honest are still
   finalizing (well above the `n/3` Byzantine minimum).
4. **T+5min to T+30min**: You decide whether to spin up replacement
   validators in another region.
5. **T+30min**: `deployer aws create --config dr-recovery.yaml`
   brings up 25 validators in `ca-central-1`.
6. **T+30min to T+1hr**: New validators sync from surviving peers
   via state sync (chapter 12).
7. **T+1hr**: Network is at full validator count again.
8. **T+24hr+**: When `us-east-1` recovers, decide whether to add
   it back or leave the 5-region topology.

The S3 bucket's cross-region replication means the new region's
validators can read all historical binaries and configs from their
local replica.

### State export — the safety net

Beyond S3, **periodic state exports** provide a third safety net:

```rust
// in the application's finalize hook
async fn on_finalize(&mut self, block: Block, cert: Certificate) {
    // ... apply block to state ...

    if block.height % 10_000 == 0 {
        // every 10000 blocks, export state to S3
        let snapshot = self.database.snapshot().await?;
        let bytes = snapshot.encode()?;
        let key = format!("shared/checkpoints/{}.bin", block.height);
        self.s3.put_object(&key, bytes).await?;
    }
}
```

Snapshots in S3 are **point-in-time recovery** for catastrophic
corruption. If a bug corrupts the state after block 100000, restore
the snapshot at block 100000 and replay blocks 100001-110000.

### The recovery runbook

A documented runbook for incident response:

```markdown
# Recovery runbook

## Single instance failure
1. Check `deployer aws list --config mychain.yaml`.
2. Confirm the instance is dead (StatusCheck failed).
3. Terminate the failed instance: `aws ec2 terminate-instances --instance-ids i-...`
4. Launch replacement: `deployer aws update --config mychain.yaml`.
5. Wait for the new instance to sync (5 minutes).
6. Verify finalization progress in metrics.

## AZ failure
1. Same as single instance, but for all instances in the AZ.
2. The new instances land in different AZs (round-robin).
3. Validators in other AZs continue finalizing.

## Region failure
1. Detect: CloudWatch alarm or PagerDuty from synthetic monitor.
2. Assess: `deployer aws list` across regions shows which instances
   are gone.
3. Spawn replacement region: `deployer aws create --config dr.yaml`.
4. Verify: metrics show finalization progress in new region.
5. Update DNS if needed (Route 53 failover routing).

## Corruption recovery
1. Stop all validators (one region at a time).
2. Restore state from the most recent S3 checkpoint.
3. Restart validators; they replay from the checkpoint height.
4. Verify state matches the checkpoint's known root.
5. If state diverges, restore from an earlier checkpoint.
```

## Exercises — practice

Five hands-on exercises to deepen your understanding.

### Exercise 1 — Deploy a 4-validator local network

Set up a single-region 4-validator deployment using the `log`
example:

1. `cargo build --release --bin commonware-log` in the monorepo.
2. Write a YAML config with `regions: [us-east-1]`, `instance_type:
   t3.medium`, `participants: 4`.
3. `deployer aws create --config mychain.yaml`.
4. SSH into one instance, watch `journalctl -u mychain` for
   finalization messages.
5. `deployer aws destroy --config mychain.yaml`.

Verify: 4 instances launched, security group has SSH/P2P/metrics
rules, S3 bucket created with `binaries/` and `configs/` populated.

### Exercise 2 — Multi-region deployment

Extend exercise 1 to two regions:

1. Add `eu-west-1` to the `regions` list.
2. Re-run `deployer aws create`.
3. Verify: 8 instances total (4 per region), 2 S3 buckets (one
   per region), 2 security groups.
4. Use `deployer aws profile --instance <id> --duration 30` on a
   validator in each region; compare flame graphs.

Look for: differences in CPU profile due to cross-region latency
validators see from peer traffic.

### Exercise 3 — Capture and read a flame graph

Reproduce a CPU spike and capture a flame graph:

1. Deploy a 4-validator network.
2. From your laptop, send 10,000 small transactions to the mempool
   in a tight loop.
3. While the spike is happening, run `deployer aws profile
   --duration 60`.
4. Open the resulting `profile.svg` in a browser.

Identify: which function consumes the most CPU. Is it signature
verification, codec work, or storage?

### Exercise 4 — Cost analysis for a 100-validator mainnet

Use the AWS pricing calculator (or a spreadsheet) to model:

1. EC2 cost: 100 × `c6i.2xlarge` × on-demand vs 3-year reserved.
2. S3 cost: assume 1 TB of notarizations, lifecycle to Glacier IR
   after 90 days.
3. CloudWatch cost: assume 100 validators emitting info-level logs,
   5 GB/month each.
4. Data transfer cost: assume 10 KB/sec/validator cross-region.

Compute the difference between on-demand and 3-year reserved. What's
the payback period for the reserved commitment?

### Exercise 5 — Disaster recovery drill

Run the multi-region DR drill:

1. Deploy to `us-east-1` and `eu-west-1`.
2. Use `aws ec2 stop-instances` to simulate an `us-east-1` failure
   (more graceful than terminate; instance can resume).
3. Wait 5 minutes; verify `eu-west-1` is still finalizing.
4. `deployer aws create --config dr.yaml` to spin up `us-west-2`.
5. Verify `us-west-2` validators sync from `eu-west-1`.
6. `deployer aws destroy --config dr.yaml` after 30 minutes.

Look for: how long state sync takes from cold, whether the new
validators agree on the chain tip, and any finalization gaps.

## Where to look in the code (expanded)

- `deployer/src/lib.rs` — the entry point + CLI documentation.
- `deployer/src/main.rs` — the CLI binary.
- `deployer/src/aws/mod.rs` — the AWS module entry point.
- `deployer/src/aws/create.rs` — the `aws create` command.
- `deployer/src/aws/update.rs` — the `aws update` command.
- `deployer/src/aws/destroy.rs` — the `aws destroy` command.
- `deployer/src/aws/list.rs` — the `aws list` and `aws status` commands.
- `deployer/src/aws/authorize.rs` — the `aws authorize` command.
- `deployer/src/aws/profile.rs` — the `aws profile` command.
- `deployer/src/aws/clean.rs` — the `aws clean` command.
- `deployer/src/aws/ec2.rs` — EC2 operations (run, terminate, status).
- `deployer/src/aws/s3.rs` — S3 operations (create, upload, lifecycle).
- `deployer/src/aws/images.rs` — AMI selection.
- `deployer/src/aws/services.rs` — service initialization (SSM, IAM).
- `deployer/src/aws/utils.rs` — shared helpers (tagging, retry).
- `deployer/src/aws/attach.rs` — resource attachment logic.
- `docs/blogs/commonware-deployer.html` — the original announcement.
- `docs/blogs/your-p2p-demo-runs-locally-now-what.html` — the practical
  walkthrough.

## Appendix A — The AWS deployment workflow, in detail

`deployer/src/aws/`. The deployer wraps the AWS SDK (via the
`aws-config` / `aws-sdk-ec2` / `aws-sdk-s3` crates).

### Step 1 — Configuration

The deployer reads a YAML config file. Required fields:

```yaml
# Required: regions to deploy across
regions:
  - us-east-1
  - us-west-2
  - eu-west-1

# Required: EC2 instance type
instance_type: c6i.xlarge

# Required: number of instances (one per region)
instances_per_region: 4

# Required: the binary to deploy
binary_path: ./target/release/mychain

# Required: configuration files for each instance
config_files:
  - ./configs/node-0.toml
  - ./configs/node-1.toml
  - ...

# Optional: SSH key pair name
ssh_key_name: my-keypair

# Optional: VPC / subnet configuration (uses defaults if omitted)
vpc_id: vpc-...
subnet_ids:
  - subnet-...

# Optional: storage
storage:
  type: gp3
  size_gb: 100

# Optional: metrics
metrics:
  enabled: true
  port: 9090
```

### Step 2 — `aws create`

The deployer:

1. **Authenticates with AWS** via the default credentials chain (env
   vars, `~/.aws/credentials`, IAM role).
2. **Creates the S3 bucket** if it doesn't exist. The bucket holds:
   - The deployment's binaries and configs.
   - Shared state (e.g., cloud-archived blocks if you use `storage::archive`).
3. **Creates the security group** with proper inbound rules:
   - SSH (port 22) from the deployer's IP only.
   - P2P consensus port (default 3000) from the validator subnet.
   - Metrics port (default 9090) from your monitoring subnet.
4. **Creates the IAM role** for the EC2 instances with S3 access.
5. **Launches EC2 instances** in each region:
   - User data script that installs the deployer's binary and runs it.
   - Tags for identification (`Name=mychain-validator-N`).
6. **Waits for instances to be ready** (via `wait_until_running`).
7. **Uploads the binaries and configs** to S3.
8. **Triggers instance update** to download and run the new binaries.

### Step 3 — Instance lifecycle

Once running, each instance:

1. **Downloads** the binary and config from S3 on startup.
2. **Runs the binary** with the right command-line args.
3. **Reports metrics** to CloudWatch (if enabled).
4. **Sends logs** to CloudWatch Logs (if enabled).

Instances run continuously until `destroy` or `update`.

### Step 4 — `aws update`

The deployer:

1. Uploads the new binaries to S3.
2. For each running instance, sends a signal (SIGHUP or SIGTERM+restart
   depending on config) to trigger a restart.
3. Waits for the new binary to come up.
4. Verifies health via metrics (e.g., finalization progress).

This is **zero-downtime deployment** if your protocol supports it
(typically: the running validators stay up while new ones come up, then
the old ones are gracefully shut down).

### Step 5 — `aws destroy`

The deployer:

1. **Terminates** all EC2 instances.
2. **Deletes** the security group.
3. **Deletes** the IAM role.
4. **Optionally** keeps the S3 bucket (so you can recover data).

## Appendix B — The S3 bucket structure

The deployer's S3 bucket has this structure:

```
s3://mychain-deployment/
├── binaries/
│   ├── mychain-2026-05-15-abc123    (timestamp + git hash)
│   ├── mychain-2026-05-16-def456
│   └── ...
├── configs/
│   ├── node-0.toml
│   ├── node-1.toml
│   └── ...
└── shared/                           (optional, for cloud-archived state)
    ├── notarizations/
    └── blocks/
```

When a new instance starts, it downloads the latest binary from
`binaries/`. When you update, the new binary is uploaded and instances
restart to use it.

## Appendix C — The security group rules

```hcl
# SSH from deployer's IP
ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = [var.deployer_ip]  # /32 only
}

# P2P consensus port from validator subnet
ingress {
    from_port = 3000
    to_port = 3000
    protocol = "tcp"
    cidr_blocks = var.validator_subnet_cidrs
}

# Metrics port from monitoring subnet
ingress {
    from_port = 9090
    to_port = 9090
    protocol = "tcp"
    cidr_blocks = var.monitoring_subnet_cidrs
}
```

Rules are **least-privilege**:
- SSH only from the deployer's IP.
- P2P only from the validator subnets (not the world).
- Metrics only from the monitoring subnet.

## Appendix D — The `aws authorize` command

Adds the current (or specified) IP to the security group for SSH access:

```bash
deployer aws authorize --deployment mychain
# Adds this machine's public IP to the security group's SSH rule

deployer aws authorize --deployment mychain --ip 1.2.3.4
# Adds the specified IP
```

Useful when:
- You're working from a different IP than usual.
- You need to give a teammate access.
- Your IP changed (ISP rotation).

## Appendix E — The `aws profile` command

Runs `samply` (a sampling profiler) on a running instance:

```bash
deployer aws profile --deployment mychain --instance i-abc123 --duration 60
# Profiles instance i-abc123 for 60 seconds, dumps the result to stdout
```

`samply` is a Linux sampling profiler. It attaches to the consensus
process, samples the call stack 1000 times per second, and produces a
flame graph of CPU usage.

The deployer:
1. SSHes into the instance.
2. Identifies the consensus process (`pgrep` or similar).
3. Attaches `samply` to the process for `--duration` seconds.
4. Saves the output to a local file.
5. Streams it back to your terminal.

**Use this** when:
- A validator is slow and you don't know why.
- CPU is spiking but no obvious cause.
- You want to optimize your application's hot path.

## Appendix F — The `aws clean` command

Deletes the shared S3 bucket (and all its contents):

```bash
deployer aws clean --deployment mychain --force
```

The `--force` flag is required because this is destructive. Use when
you're decommissioning a deployment for good.

## Appendix G — The library mode

`deployer/src/aws/lib.rs`. As a Rust library, you can write custom
deployment workflows:

```rust
use commonware_deployer::aws::{Deployer, Config};

let deployer = Deployer::new(AwsConfig::from_env().await?);
let config = Config::from_file("config.yaml")?;

deployer.create(&config).await?;
deployer.wait_healthy(&config, Duration::from_secs(300)).await?;

// Custom post-deployment: run integration tests
run_integration_tests(&deployer, &config).await?;
```

Used by Commonware's internal tools for managing test deployments
(e.g., continuous benchmarks across regions).

## Appendix H — The CI integration

A typical CI pipeline:

```yaml
- name: Deploy to testnet
  run: |
    cargo build --release
    deployer aws create --config testnet.yaml

- name: Smoke test
  run: |
    # Wait for healthy
    deployer aws status --config testnet.yaml
    # Send some transactions, verify finalization
    ./scripts/smoke-test.sh

- name: Tear down
  if: always()
  run: |
    deployer aws destroy --config testnet.yaml
```

You get a fresh deployment per CI run. No state pollution between runs.

## Appendix I — The multi-region specifics

Each region:

- Gets its own S3 bucket (data residency).
- Gets its own EC2 instances.
- Gets its own security group (with rules scoped to that region's
  subnets).
- Shares the **shared bucket** via cross-region replication (optional).

The P2P layer (chapter 05) handles cross-region connections:
- Instances in `us-east-1` connect to instances in `us-west-2` via
  public IPs (or via VPC peering for tighter security).
- Latency between regions: ~60-80ms. Consensus still works; you just
  get slower finalization.

## Appendix J — Cost considerations

A typical deployment:

- 4 EC2 instances × `c6i.xlarge` (~$0.20/hr each) = ~$0.80/hr
- 3 regions = ~$2.40/hr = ~$1750/month
- S3 storage (100 GB): ~$3/month
- CloudWatch metrics + logs: ~$50/month
- Data transfer between regions: ~$50/month
- **Total: ~$1850/month**

For a testnet, you can use `t3.medium` (~$0.04/hr) instead, dropping
costs 5x.

For dev/staging, you can run a single region with 4 instances, ~$200/
month.

## Appendix K — Common patterns

### Blue-green deployment

```bash
# Deploy new version to "green" instances
deployer aws create --config green.yaml

# Smoke test green
./scripts/smoke-test.sh green-endpoint

# If green is healthy, switch traffic
deployer aws update --config blue.yaml --new-binary ./target/release/mychain-v2

# Tear down old version
deployer aws destroy --config old.yaml
```

### Canary deployment

```bash
# Deploy new version to 1 validator (10% of network)
deployer aws create --config canary.yaml

# Monitor for 1 hour
./scripts/monitor.sh

# If healthy, deploy to rest
deployer aws update --config main.yaml --new-binary ./target/release/mychain-v2
```

### Disaster recovery

```bash
# Region us-east-1 goes down

# Bring up replacement in us-west-2
deployer aws create --config dr-recovery.yaml

# Verifiers see the same validator set (just relocated)
# No consensus disruption if you have sufficient geographic diversity
```

## Appendix L — Common gotchas

### Forgetting to set region quotas

AWS accounts have default quotas on EC2 instances per region. If you
try to launch 50 instances in a region you've never used, you might
hit the quota and the deploy fails.

**Fix**: request a quota increase ahead of time, or distribute across
regions.

### IAM permissions

The deployer needs:
- `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:DescribeInstances`
- `ec2:CreateSecurityGroup`, `ec2:AuthorizeSecurityGroupIngress`
- `s3:CreateBucket`, `s3:PutObject`, `s3:GetObject`, `s3:DeleteBucket`
- `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole`
- `cloudwatch:PutMetricData`, `logs:CreateLogGroup`, `logs:PutLogEvents`

Make sure your AWS credentials have all of these.

### Cross-region latency

A validator in `us-east-1` and one in `ap-southeast-1` will see ~200ms
RTT. Consensus still works but is slow.

For low-latency consensus, keep validators in regions with < 100ms RTT.

## Where to look in the code (expanded)

- `deployer/src/lib.rs` — the entry point + CLI docs.
- `deployer/src/main.rs` — the CLI binary.
- `deployer/src/aws/lib.rs` — the AWS implementation.
- `deployer/src/aws/config.rs` — config parsing.
- `deployer/src/aws/network.rs` — VPC, subnet, security group setup.
- `deployer/src/aws/compute.rs` — EC2 instance management.
- `deployer/src/aws/storage.rs` — S3 bucket management.
- `deployer/src/aws/monitoring.rs` — CloudWatch integration.
- `docs/blogs/commonware-deployer.html` — the original announcement.
- `docs/blogs/your-p2p-demo-runs-locally-now-what.html` — the practical
  walkthrough.

## If you only remember three things

1. **`deployer aws create/update/destroy` from a YAML config.** Multi-region, multi-instance BFT deployment in one command.
2. **`deployer aws profile` = production debugging.** Sampling profiler on a running instance. Don't guess where CPU goes.
3. **Multi-region is the production reality.** Single region = single point of failure. Commonware's design assumes geographically distributed validators.

→ Next: **Chapter 19 — Examples**. Read the actual examples in
`examples/` to see composition patterns in action.