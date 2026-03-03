# GitHub Runners Optimizer

A unified monitoring, cost attribution, and optimization platform for self-hosted GitHub Actions runners across Kubernetes (ARC legacy, ARC scale sets) and EC2.

## Table of Contents

- [GitHub Runners Optimizer](#github-runners-optimizer)
  - [Table of Contents](#table-of-contents)
  - [Problem Statement](#problem-statement)
  - [Goals](#goals)
  - [Current Runner Infrastructure](#current-runner-infrastructure)
    - [Legacy ARC (Kubernetes)](#legacy-arc-kubernetes)
    - [New ARC (Kubernetes)](#new-arc-kubernetes)
    - [EC2 Runners](#ec2-runners)
    - [Authentication](#authentication)
  - [Features](#features)
    - [Tier 1: Unified Visibility](#tier-1-unified-visibility)
    - [Tier 2: Cost Attribution](#tier-2-cost-attribution)
    - [Tier 3: Optimization](#tier-3-optimization)
    - [Tier 4: Intelligence](#tier-4-intelligence)
  - [Architecture](#architecture)
    - [High-Level Design](#high-level-design)
  - [Data Sources and API Limits](#data-sources-and-api-limits)
    - [GitHub API](#github-api)
    - [Kubernetes API](#kubernetes-api)
    - [Prometheus](#prometheus)
    - [AWS CloudWatch](#aws-cloudwatch)
  - [Security Considerations](#security-considerations)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [Alternatives Considered](#alternatives-considered)

---

## Problem Statement

The organization operates **three independent self-hosted runner systems**:

| System | Technology | Scope |
|---|---|---|
| Legacy ARC | `actions-runner-controller` v0.23.7 (summerwind) on EKS | ~40 RunnerSets across 15+ teams |
| New ARC | `gha-runner-scale-set-controller` v0.13.1 (GitHub-official) on EKS | 1 scale set (SDLC team) |
| EC2 Runners | `terraform-aws-github-runner` v7.3.0 with Lambda + SQS | 8 runner types (Podman/Docker, x64/ARM64) |

Monitoring is fragmented:

- **Legacy ARC**: Prometheus ServiceMonitors, but no dedicated dashboard.
- **New ARC**: Metrics endpoints exposed but not scraped.
- **EC2**: CloudWatch Logs and Lambda metrics, no unified view.
- **GitHub**: Runner status and workflow data visible only through GitHub UI.

There is no single view that answers:

- How much runner compute does each team consume?
- Are runner sizes right-sized for the workloads they run?
- How long do jobs wait in queue before a runner picks them up?
- Which runner system is the most cost-effective for a given workload type?
- Are there idle or underutilized runner definitions that could be consolidated?

---

## Goals

1. **Unified visibility** across all three runner systems in a single dashboard.
2. **Cost attribution** per team, runner type, and runner system.
3. **Right-sizing recommendations** based on actual resource usage vs. allocated resources.
4. **Queue time monitoring** with alerting when jobs wait too long.
5. **Optimization suggestions** to reduce cost and improve developer experience.

---

## Current Runner Infrastructure

### Legacy ARC (Kubernetes)

- **Namespace**: `github-actions-runners`
- **Controller**: `actions-runner-controller` v0.23.7 with webhook server at `actions.shared.empathy.co`
- **CRDs**: `RunnerSet` + `HorizontalRunnerAutoscaler` per team-size combination
- **Runner definitions**: ~40 entries in `actions-runners-platform/values.yaml` (play, search, ether, data, index, security, platform, edocs, ui, motive, kroger, openprivacy, x, ltr, academy, experienceai, it, motivemarket, common, mcpplatform)
- **Sizes**: small (500m CPU / 2Gi), medium (1 CPU / 4Gi), big (1.8-4 CPU / 8-16Gi), titan (4 CPU / 8Gi)
- **Scaling**: Webhook-based via `HorizontalRunnerAutoscaler`, scale 0 → max on `workflowJob` events, scale down after 300s
- **Container runtime**: Docker-in-Docker sidecar
- **Images**: Custom ECR image (`platform/custom-github-runner:v2.329.0-ubuntu-22.04`) or summerwind base

### New ARC (Kubernetes)

- **Controller namespace**: `arc-controller`
- **Runner namespace**: `gha-runners`
- **Controller**: `gha-runner-scale-set-controller` v0.13.1
- **Scale set**: `gha-runner-scale-set` (single definition: `gha-runners-play-small`)
- **Scaling**: Long-poll from controller to GitHub Actions service (no webhook needed), 0-10 runners
- **Container runtime**: Docker-in-Docker via init container
- **Resources**: 4 CPU / 16Gi per runner

### EC2 Runners

- **Module**: `github-aws-runners/terraform-aws-github-runner` v7.3.0 (multi-runner mode)
- **VPC**: `shared-services`, subnet `shared-services-private-eu-west-1a`
- **8 runner types**: rootful-x64, rootful-arm, rootful-x64-platform-small, rootful-arm64-platform-small, rootful-x64-platform-big, rootful-arm64-platform-big, docker-x64-platform-small, docker-arm64-platform-small
- **Instance types**: Spot (t3, t4g, m5, m6g, c5, c6g — .large to .2xlarge)
- **AMIs**: Built by AWS Image Builder (Podman rootful and Docker variants)
- **Scaling**: GitHub webhook → API Gateway → Lambda → SQS (FIFO) → scale-up Lambda → EC2, scale-down every 5 min
- **Max per type**: 5 instances

### Authentication

All three systems authenticate with GitHub via the same GitHub App, with credentials stored in AWS Secrets Manager at path `shared/actions-runner-controller`.

---

## Features

### Tier 1: Unified Visibility

**1.1 — Real-Time Runner Dashboard**

A single-pane view showing every runner across all systems:

| Column | Source |
|---|---|
| Runner name | GitHub API / K8s API |
| System (Legacy ARC / New ARC / EC2) | Derived from runner name prefix or K8s namespace |
| Status (idle / active / provisioning / offline) | GitHub API |
| Team | Parsed from runner name (e.g., `play-small` → `play`) |
| Size | Parsed from runner name (e.g., `play-small` → `small`) |
| Architecture | K8s node selector / EC2 instance type |
| Current job (if active) | GitHub API / webhook event |
| Uptime | K8s pod age / EC2 instance launch time |

**1.2 — Job Queue Monitor**

- Current queue depth: jobs in `queued` state with no runner assigned.
- Queue time per job: duration between `workflow_job.queued` and `workflow_job.in_progress` timestamps.
- Queue time percentiles: p50, p90, p99 over configurable time windows.
- Breakdown by team, runner label, and system.

**1.3 — Webhook Health Tracker**

- Track webhook delivery success/failure rates (via GitHub App delivery logs or by detecting orphaned jobs).
- Latency: time from webhook delivery to runner becoming ready.
- Alert on: delivery failures, prolonged queue times, scaling failures.

**1.4 — Alerting**

| Alert | Condition | Severity |
|---|---|---|
| Job stuck in queue | Queue time > 5 minutes | Warning |
| Job stuck in queue | Queue time > 15 minutes | Critical |
| Runner system unhealthy | 0 runners registered for a team | Critical |
| Scaling failure | Scale-up event with no corresponding runner within 3 min | Warning |
| Webhook delivery failure | Consecutive webhook delivery failures > 3 | Critical |
| EC2 spot interruption | Spot instance terminated during active job | Warning |

---

### Tier 2: Cost Attribution

**2.1 — Per-Team Cost Breakdown**

Calculate cost for each team by aggregating:

- **K8s runners**: `cpu_requests × duration` and `memory_requests × duration`, priced at the cluster's node cost per resource-unit-hour (derived from EC2 instance type pricing for the EKS node group).
- **EC2 runners**: EC2 instance cost (spot price × duration) per runner type.
- Attribute to team by parsing the runner name.

Output: monthly cost report per team, with trend lines.

**2.2 — Cost per Workflow / Repository**

For each workflow run, correlate:

- Runner used (from `workflow_job` event `runner_name` field).
- Duration (from `workflow_job` `started_at` / `completed_at`).
- Compute cost (from runner type → instance type → price).

Identify the most expensive workflows and repositories.

**2.3 — System Cost Comparison**

Compare cost-per-job-minute across the three systems:

- K8s Legacy ARC: shared node cost amortized across runner pods.
- K8s New ARC: same model.
- EC2: direct instance cost (spot).

This helps decide where to route workloads for cost efficiency.

---

### Tier 3: Optimization

**3.1 — Right-Sizing Recommendations**

Compare resource **requests** vs actual **usage** for K8s runners:

- Pull `container_cpu_usage_seconds_total` and `container_memory_working_set_bytes` from Prometheus.
- Aggregate by runner name over a rolling 7-day window.
- Flag runners where p95 usage < 50% of requests.

Example output:

```
RECOMMENDATION: Downsize search-big
  Current: 4 CPU / 16Gi memory
  Observed p95: 1.8 CPU / 9.2Gi memory
  Suggested: search-medium (1 CPU / 8Gi) or custom (2 CPU / 10Gi)
  Estimated savings: ~$X/month
```

**3.2 — Runner Pool Consolidation**

Identify underused runner definitions:

- Runner types with < N jobs/month.
- Runner types whose workloads could be served by a differently-named runner with the same resource profile.

Example:

```
CONSOLIDATION CANDIDATE: academy-small
  Jobs last 30 days: 3
  Resource profile identical to: common-small
  Suggestion: Migrate academy workflows to runs-on: common-small
```

**3.3 — Idle Time Analysis**

- K8s: Time between runner pod becoming ready and picking up a job (wasted resource-seconds).
- K8s: Time after job completion before pod termination (`scaleDownDelaySecondsAfterScaleOut: 300` = 5 min idle after each job).
- EC2: Time between instance launch and job pickup, and `minimum_running_time_in_minutes: 5` waste.

**3.4 — Docker Layer Cache Efficiency (K8s)**

For legacy ARC runners with `volumeClaimTemplates` for `docker-cache`:

- Track cache hit rates by monitoring Docker pull times.
- Identify runners where the PVC is not providing value (ephemeral runners rarely reuse PVCs).

---

### Tier 4: Intelligence

**4.1 — Predictive Scaling**

- Analyze historical job patterns to identify recurring spikes (e.g., daily CI bursts at 10:00 CET).
- Generate recommended `idle_config` or `minRunners` adjustments to pre-warm capacity.
- Output as actionable configuration diffs.

**4.2 — Failure Correlation**

- Correlate job failures (`conclusion: failure`) with infrastructure events:
  - OOM kills (K8s pod events).
  - Spot interruptions (EC2 instance state changes).
  - Docker daemon crashes (DinD sidecar restarts).
  - Node pressure (K8s node conditions).
- Surface: "5 failures on `ether-big` this week — 4 were OOM killed. Increase memory limit from 16Gi to 20Gi."

**4.3 — Workflow Routing Suggestions**

- Analyze job resource consumption and suggest the cheapest runner label that satisfies the workload.
- Example: "Repository `empathyco/my-service` uses `runs-on: play-big` (8Gi) but p95 memory is 1.5Gi. Switch to `play-small` to save ~$X/month."

**4.4 — Migration Impact Simulator**

- For planning the Legacy ARC → New ARC migration:
  - Simulate scale set behavior for each existing RunnerSet based on historical job patterns.
  - Identify workflows using multi-label `runs-on` patterns that would need updating.
  - Estimate Docker cache performance impact.

---

## Architecture

### High-Level Design

```
┌───────────────────────────────────────────────────────────────┐
│                   GitHub Runners Optimizer                      │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                      Web UI (React)                      │  │
│  │  Dashboard │ Cost Reports │ Recommendations │ Alerts     │  │
│  └──────────────────────────┬──────────────────────────────┘  │
│                              │                                 │
│  ┌──────────────────────────▼──────────────────────────────┐  │
│  │                    REST / GraphQL API                     │  │
│  │                      (Go / Python)                       │  │
│  └──────────────────────────┬──────────────────────────────┘  │
│                              │                                 │
│  ┌──────────────────────────▼──────────────────────────────┐  │
│  │                     Core Engine                           │  │
│  │                                                           │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │  │
│  │  │ Job Tracker  │  │ Cost Engine  │  │ Right-Sizer   │  │  │
│  │  └──────────────┘  └──────────────┘  └───────────────┘  │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │  │
│  │  │ Queue Monitor│  │ Consolidator │  │ Alert Engine  │  │  │
│  │  └──────────────┘  └──────────────┘  └───────────────┘  │  │
│  │  ┌──────────────┐  ┌──────────────┐                     │  │
│  │  │ Predictor    │  │ Failure      │                     │  │
│  │  │              │  │ Correlator   │                     │  │
│  │  └──────────────┘  └──────────────┘                     │  │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                  Data Ingestion Layer                      │ │
│  │                                                           │ │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────────────┐ │ │
│  │  │ Webhook  │  │ K8s      │  │ Prom   │  │ CloudWatch │ │ │
│  │  │ Receiver │  │ Informer │  │ Client │  │ Client     │ │ │
│  │  └──────────┘  └──────────┘  └────────┘  └────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                   PostgreSQL + TimescaleDB                 │ │
│  └───────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘

External Data Sources:
  ← GitHub (workflow_job webhooks, REST API for runner inventory)
  ← Kubernetes API (RunnerSet, EphemeralRunner, Pod watches)
  ← Prometheus (container resource usage metrics)
  ← CloudWatch (Lambda metrics, EC2 instance logs, custom metrics)
```

---

## Data Sources and API Limits

### GitHub API

| Auth Method | Rate Limit | Notes |
|---|---|---|
| GitHub App installation token | 15,000 requests/hour | Recommended. Rolling window. |
| Secondary limits | 900 points/min, 100 concurrent | Built-in backoff handles this. |
| GraphQL | 12,500 points/hour | Point-based, complex queries cost more. |

**Estimated usage with webhook-based architecture:**

| Operation | Calls/hour | Purpose |
|---|---|---|
| Runner inventory poll | 60 | 1 call/min |
| Runner groups poll | 12 | 1 call/5 min |
| Backfill missed events | ~50 | On-demand |
| **Total** | **~122** | **< 1% of 15,000 limit** |

Webhook events (workflow_job) are pushed by GitHub and consume zero API calls.

**Important**: GitHub retains workflow run data for **90 days** via the API. Start collecting data immediately; historical data beyond 90 days is only available if stored externally.

### Kubernetes API

- No formal rate limits. API priority and fairness may throttle low-priority clients under cluster load.
- Use **informers** (watch-based) instead of polling. Near-zero overhead.
- Required RBAC permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: github-runners-optimizer
rules:
  - apiGroups: ["actions.summerwind.dev"]
    resources: ["runnersets", "horizontalrunnerautoscalers"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["actions.github.com"]
    resources: ["autoscalingrunnersets", "ephemeralrunners"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list"]
```

### Prometheus

- No external rate limits (self-hosted).
- Query performance: use recording rules for expensive aggregations.
- Typical retention: 15-30 days. The optimizer should pull and store data before it ages out.
- Key metrics to query:

```promql
# CPU usage per runner pod
rate(container_cpu_usage_seconds_total{namespace="github-actions-runners", container="runner"}[5m])

# Memory usage per runner pod
container_memory_working_set_bytes{namespace="github-actions-runners", container="runner"}

# Pod count by runner set
count by (label_app) (kube_pod_labels{namespace="github-actions-runners"})

# ARC controller queue depth
workqueue_depth{namespace="github-actions-runners"}
```

### AWS CloudWatch

| API | Limit | Notes |
|---|---|---|
| `GetMetricData` | 50 TPS | More than sufficient. |
| `GetLogEvents` | 25 TPS per log group | Sufficient for periodic log analysis. |
| `DescribeInstances` | 100/s | For EC2 runner inventory. |

CloudWatch query costs are negligible for this use case (~$0.01 per 1,000 metrics queried).

---


## Security Considerations

| Concern | Mitigation |
|---|---|
| GitHub App private key | Store in AWS Secrets Manager, inject via ExternalSecret (same pattern as runner credentials) |
| Webhook signature validation | Verify `X-Hub-Signature-256` header on every incoming webhook using the shared secret |
| Database credentials | Kubernetes Secret, rotated via ExternalSecret |
| RBAC | ClusterRole scoped to read-only access on runner CRDs and pods |
| Network exposure | Only the webhook endpoint is externally reachable; dashboard is internal-only or behind SSO |
| Data sensitivity | Job names and repo names are visible; no source code or secrets are collected |
| AWS credentials | IRSA (IAM Roles for Service Accounts) for CloudWatch access, no static keys |

---

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| GitHub changes webhook payload format | Data ingestion breaks | Low | Schema validation with graceful degradation; pin to API version |
| Prometheus retention too short for trend analysis | Missing historical data | Medium | Pull and store resource usage data every 60s; don't rely on Prometheus for history |
| Legacy ARC deprecated/removed before migration | Monitoring code becomes dead weight | Medium | Design data model to be system-agnostic; runner system is a tag, not a structural dependency |
| Webhook endpoint goes down | Missing job events | Medium | Implement a reconciliation job that backfills from GitHub API on startup and periodically |
| Cost estimates inaccurate | Misleading recommendations | Medium | Allow manual override of cost rates; show confidence intervals; validate against AWS billing |
| Spot price volatility (EC2) | Cost attribution varies | Low | Use average spot price over the time window, not list price |
| Scope creep | Delays delivery | High | Strict phased approach; each phase delivers standalone value |

---

## Alternatives Considered

| Alternative | Pros | Cons | Verdict |
|---|---|---|---|
| **Grafana dashboards only** | No custom code, uses existing infra | No cost attribution, no recommendations, can't correlate across systems, no webhook ingestion | Insufficient — only covers visibility for K8s |
| **GitHub's built-in insights** | Zero effort | No cost data, no cross-system view, no optimization | Insufficient — too limited |
| **Datadog CI Visibility** | Managed, rich features | Expensive, vendor lock-in, doesn't cover K8s pod-level metrics or EC2 specifics | Overkill and incomplete |
| **RunsOn / Actuated** | Managed runner solutions | Replaces infrastructure rather than monitoring it, doesn't support ARC | Wrong tool — these are runner replacements, not monitors |
| **Custom Grafana + recording rules + CloudWatch datasource** | Uses existing tools | Fragile, no persistent storage, no optimization logic, hard to maintain complex dashboards | Partial — good for Tier 1 visibility only, but doesn't scale to Tiers 2-4 |
| **Build custom app (this proposal)** | Full control, tailored to our exact multi-system setup, enables optimization | Engineering effort, maintenance burden | Recommended — the only option that covers all requirements |

A pragmatic hybrid approach is also viable: use **Grafana for Tier 1 dashboards** (fastest time-to-value) and build the custom app for **Tiers 2-4** (cost, optimization, intelligence), feeding data into the same PostgreSQL backend.
