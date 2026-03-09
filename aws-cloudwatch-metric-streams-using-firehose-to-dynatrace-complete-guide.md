# AWS CloudWatch Metric Streams → Dynatrace: The Complete Field Guide

> **What this guide covers that the official docs don't:**
> Architecture patterns for multi-account/multi-region environments · DDU cost modelling before you go live · Hybrid integration design (when to use both methods simultaneously) · DQL query patterns for Metric Streams data · Operational runbook for the silent failure modes nobody documents
>
> **Coming in the next iteration:** Full IaC deployment via CloudFormation & Terraform · Monaco Monitoring-as-Code for managing dashboards and alerts at scale

---

## Table of Contents

1. [Understanding the Two Integration Patterns — Architecture Decision](#1-understanding-the-two-integration-patterns--architecture-decision)
2. [How Data Actually Flows — Architecture Diagrams](#2-how-data-actually-flows--architecture-diagrams)
3. [The Hybrid Pattern — Running Both Simultaneously](#3-the-hybrid-pattern--running-both-simultaneously)
4. [Multi-Account Multi-Region Architecture](#4-multi-account-multi-region-architecture)
5. [Prerequisites and IAM Considerations](#5-prerequisites-and-iam-considerations)
6. [Deployment — AWS Console (Manual)](#6-deployment--aws-console-manual)
7. [DDU Cost Modelling — Before You Enable "All Metrics"](#7-ddu-cost-modelling--before-you-enable-all-metrics)
8. [Entity Enrichment — The AWS Entities Extension](#8-entity-enrichment--the-aws-entities-extension)
9. [Querying Metric Streams Data with DQL](#9-querying-metric-streams-data-with-dql)
10. [Operational Runbook — Silent Failures and How to Catch Them](#10-operational-runbook--silent-failures-and-how-to-catch-them)
11. [High-Signal Metric Reference by AWS Service](#11-high-signal-metric-reference-by-aws-service)
12. [Limitations You Must Know](#12-limitations-you-must-know)
13. [Coming Next — IaC & Monaco *(In Progress)*](#13-coming-next--iac--monaco-in-progress)

---

## 1. Understanding the Two Integration Patterns — Architecture Decision

Dynatrace offers two distinct paths for ingesting AWS CloudWatch metrics. Most guides treat this as a simple choice. In practice, it is an architecture decision with significant downstream implications for alerting, entity topology, DDU consumption, and operational overhead.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              AWS CLOUDWATCH METRICS → DYNATRACE                         │
│                                                                         │
│  PATH A: Default Integration (Pull)      PATH B: Metric Streams (Push) │
│                                                                         │
│  CloudWatch ──poll every 5min──► ActiveGate ──► Dynatrace              │
│                                                                         │
│  CloudWatch ──stream real-time──► Firehose ──► Dynatrace API           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Decision matrix

| Dimension | Default Integration | Metric Streams | Winner for... |
|---|---|---|---|
| **Latency** | ~5 min polling delay | <60 sec (buffer-dependent) | Real-time alerting |
| **ActiveGate dependency** | Required at scale | Not required | Lean architecture |
| **Metric coverage** | Curated set per service | All CloudWatch metrics including custom namespaces | Custom/CWAgent metrics |
| **Where you configure** | Dynatrace UI | AWS CloudWatch console | IaC preference |
| **Granularity of selection** | Per metric + per statistic | Namespace level only | Fine-grained filtering |
| **Metric key prefix** | `ext:cloud.aws.<service>` | `cloud.aws.<service>` | — |
| **Entity topology (Smartscape)** | Native | Via extension (see §10) | Default integration |
| **AWS Tags on entities** | Available | Not available | Default integration |
| **Predefined alerts** | Available | Not available | Default integration |
| **Multi-account streaming** | Per-account polling | Native (CloudWatch cross-account streams) | Large org |
| **DDU cost model** | Predictable (poll-based) | Can spike with cardinality | Cost control |
| **PrivateLink** | Not available | Not available | — |

**Rule of thumb from production experience:**

- If you are starting from scratch on a small-to-medium AWS footprint: use the **Default Integration** and extend with Metric Streams only for gaps (custom namespaces, CWAgent, FSx, Backup, etc.)
- If you are operating at scale (100+ accounts, 10+ regions, or need near-real-time alerting): **Metric Streams becomes primary**, supplemented by default integration for entity-linked alerting
- In most enterprise environments: **both run simultaneously** — see §3

---

## 2. How Data Actually Flows — Architecture Diagrams

### Single-region Metric Streams flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  AWS ACCOUNT / REGION                                                   │
│                                                                         │
│  ┌──────────────────┐    OTel 1.0     ┌──────────────────────────────┐ │
│  │  CloudWatch      │ ─────────────►  │  Amazon Data Firehose        │ │
│  │  Metric Stream   │                 │                              │ │
│  │                  │                 │  Buffer: 3 MiB / 60s         │ │
│  │  [namespaces     │                 │  Encoding: GZIP              │ │
│  │   selected here] │                 │  Retry: 900s                 │ │
│  └──────────────────┘                 └──────┬───────────────────────┘ │
│                                              │                         │
│                                    ┌─────────▼──────────┐             │
│                                    │  S3 Backup Bucket  │             │
│                                    │  (failed records)  │             │
│                                    └────────────────────┘             │
└──────────────────────────────────────────────────────────────────────┬─┘
                                                                       │
                                              HTTPS POST (GZIP)        │
                                              to aws.cloud.dynatrace.com│
                                                                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  DYNATRACE TENANT                                                       │
│                                                                         │
│  ┌─────────────────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │  Metrics Ingest API │──► │  Grail (storage)  │──► │  Data Explorer│  │
│  │  /api/v2/metrics/   │    │                  │    │  Dashboards   │  │
│  │  ingest             │    │  cloud.aws.*     │    │  DQL queries  │  │
│  └─────────────────────┘    └──────────────────┘    └───────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  AWS Entities for Metric Streaming Extension (optional)         │   │
│  │  Maps metric dimensions → Dynatrace Entity Model                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### What happens at the Firehose buffer

This is the part most guides skip, but it matters for operational understanding:

```
Metric datapoint produced in CloudWatch
         │
         ▼
CloudWatch Metric Stream serializes to OpenTelemetry 1.0 protobuf
         │
         ▼
Firehose receives record, holds in buffer
         │
         ├── Buffer size reaches 3 MiB? → flush immediately
         │
         └── Buffer interval reaches 60s? → flush immediately
                         │
                         ▼
               Firehose attempts HTTPS POST to Dynatrace
                         │
              ┌──────────┴───────────┐
         200 OK                 4xx/5xx
              │                     │
         ✓ Delivered           Retry for up to 900s
                                     │
                              Still failing after 900s?
                                     │
                                     ▼
                              Written to S3 backup bucket
                              (MONITOR THIS — silent data loss)
```

---

## 3. The Hybrid Pattern — Running Both Simultaneously

This is the most important section for anyone operating at enterprise scale, and it is entirely absent from the official documentation.

**The reality:** No single integration covers every requirement. In mature environments, both integrations run concurrently — and this is intentional architecture, not a misconfiguration.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HYBRID INTEGRATION DESIGN                        │
│                                                                         │
│  DEFAULT INTEGRATION (polling)           METRIC STREAMS (push)         │
│  ─────────────────────────────           ─────────────────────         │
│  • EC2, RDS, ELB (entity-linked)         • CWAgent / Windows metrics   │
│  • Predefined Dynatrace alerts           • Custom CloudWatch namespaces  │
│  • Tag-based entity filtering            • FSx, Backup, NLB extended   │
│  • Smartscape topology                   • Near-real-time alerting      │
│  • AWS tag enrichment                    • New AWS services (day-0)     │
│                                                                         │
│  Metric prefix: ext:cloud.aws.*          Metric prefix: cloud.aws.*    │
│                                                                         │
│  ⚠ IMPORTANT: These two metric prefixes are DIFFERENT.                 │
│  You cannot mix them in a single alert or dashboard tile.              │
│  Design your dashboards and alerts explicitly for each prefix.         │
└─────────────────────────────────────────────────────────────────────────┘
```

### When to assign each service to each integration

| AWS Service | Recommended method | Reason |
|---|---|---|
| EC2 (status checks, standard) | Default integration | Entity linking, predefined alerts |
| EC2 (disk/memory via CWAgent) | Metric Streams | CWAgent is custom namespace, not in default integration |
| RDS | Default integration | Entity linking, topology |
| ELB/ALB | Default integration | Entity linking |
| NLB (healthy/unhealthy hosts) | Metric Streams | Extended NLB metrics not in default |
| FSx | Metric Streams | Not in default integration |
| AWS Backup | Metric Streams | Not in default integration |
| DynamoDB | Either / both | Depends on whether you need entity context or raw throughput metrics |
| Custom namespaces | Metric Streams only | Default integration cannot consume custom namespaces |

---

## 4. Multi-Account Multi-Region Architecture

This is another gap in the official documentation. Real AWS environments span multiple accounts and regions. Here is how to design the integration correctly.

### Single-account, multi-region

Each region requires its own Firehose delivery stream and CloudWatch Metric Stream. The Dynatrace ingest endpoint is the same across all regions.

```
┌──────────────────────────────────────────────────────────┐
│  AWS ACCOUNT                                             │
│                                                          │
│  us-east-1:  CloudWatch Stream ──► Firehose ──────────► │
│  eu-west-1:  CloudWatch Stream ──► Firehose ──────────► │──► Dynatrace
│  ap-south-1: CloudWatch Stream ──► Firehose ──────────► │    (single tenant)
│                                                          │
│  Each region: separate Firehose stream                   │
│  Same Dynatrace API URL and token across all             │
└──────────────────────────────────────────────────────────┘
```

### Multi-account architecture

AWS CloudWatch Metric Streams supports cross-account streaming natively via the `IncludeLinkedAccountsMetrics` parameter. This allows a central account to stream metrics on behalf of linked accounts.

```
┌────────────────────────────────────────────────────────────┐
│  MANAGEMENT ACCOUNT                                        │
│  CloudWatch Metric Stream                                  │
│  (IncludeLinkedAccountsMetrics: true)                      │
│                     │                                      │
│                     ▼                                      │
│  Amazon Data Firehose ──────────────────────────► Dynatrace│
│                     ▲                                      │
│  LINKED ACCOUNT A ──┤                                      │
│  LINKED ACCOUNT B ──┤  Metrics tagged with                 │
│  LINKED ACCOUNT C ──┘  aws.account.id dimension           │
└────────────────────────────────────────────────────────────┘
```

In Dynatrace, all metrics carry the `aws.account.id` dimension, allowing you to filter and split dashboards and alerts by account — even though everything flows through a single Firehose stream.

**CloudFormation parameter to enable:**
```yaml
IncludeLinkedAccountsMetrics: true
```

---

## 5. Prerequisites and IAM Considerations

### Dynatrace API token

Create a token with **only** the permissions needed:

```
Required scope: metrics.ingest
```

Do not reuse a token with broader permissions for Firehose delivery. Firehose has its API token stored in plaintext in the delivery stream configuration — scope it minimally.

### AWS IAM

CloudFormation deployment creates these IAM roles automatically. If creating manually:

**Role for CloudWatch to write to Firehose:**
```json
{
  "Effect": "Allow",
  "Action": [
    "firehose:PutRecord",
    "firehose:PutRecordBatch"
  ],
  "Resource": "arn:aws:firehose:<region>:<account-id>:deliverystream/<stream-name>"
}
```

**Role for Firehose to write to S3 (backup):**
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:GetBucketLocation",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::<backup-bucket-name>",
    "arn:aws:s3:::<backup-bucket-name>/*"
  ]
}
```

### Dynatrace environment networking

Your Dynatrace tenant must accept inbound HTTPS traffic from AWS Firehose. For **Dynatrace Managed** environments behind a corporate firewall, you will need to:
- Open inbound HTTPS (443) from the AWS IP ranges used by Firehose in your region
- Or route traffic through an ActiveGate as an HTTPS proxy (the ActiveGate URL becomes your API URL)

---

## 6. Deployment — AWS Console (Manual)

Use this section to deploy via the AWS Console. For teams preferring Infrastructure-as-Code, full **CloudFormation** and **Terraform** deployment guides are in progress — see [§13 Coming Next](#13-coming-next--iac--monaco-in-progress).


Use this only for one-off testing or when IaC is not an option. For production environments, use CloudFormation (§6) or Terraform (§7).

### Step 1 — Create the Firehose delivery stream

1. Navigate to **Amazon Data Firehose → Create Firehose stream**
2. Set **Source:** `Direct PUT` | **Destination:** `Dynatrace`
3. Name the stream (save this name — you will need it for the Metric Stream)
4. Under **Transform records**: ensure **Data Transformation is OFF**
5. Under **Destination settings**:

   | Field | Value | Why |
   |---|---|---|
   | Ingestion type | `Metrics` | Not Logs — common mistake |
   | HTTP endpoint URL | `Dynatrace - Global` (or EU/US) | Match your tenant region |
   | API token | Your Dynatrace token | `metrics.ingest` scope only |
   | API URL | Your environment URL | See Prerequisites |
   | Content encoding | `GZIP` | Required — reduces payload size |
   | Retry duration | `900` | Max retry before S3 fallback |
   | Buffer size | `3 MiB` | Recommended by Dynatrace |
   | Buffer interval | `60 seconds` | Minimum delivery latency |

6. Under **Backup settings**: `Failed data only` + create/select an S3 bucket
7. Review and **Create delivery stream**

### Step 2 — Create the CloudWatch Metric Stream

1. Navigate to **CloudWatch → Metrics → Streams → Create metric stream**
2. Select **Custom setup with Firehose**
3. Select the Firehose stream created in Step 1
4. **Output format:** `OpenTelemetry 1.0` — mandatory, do not select JSON
5. **Metrics to stream:** Select metrics (not all) unless you have modelled DDU cost first (see §9)
6. Enter a **metric stream name** → **Create metric stream**

---

## 7. DDU Cost Modelling — Before You Enable "All Metrics"

This section exists nowhere in the official documentation. It is the most common operational mistake made when deploying Metric Streams.

**What is a DDU?** A Davis Data Unit is Dynatrace's consumption metric. Each unique metric time series (metric + dimension combination) consumes DDUs. Metric Streams data is DDU-billed.

### Why "All Metrics" can be expensive

Consider a DynamoDB table with 10 operations, 5 tables, 3 regions, 2 accounts. Each metric's full dimension set can create:
```
10 (operations) × 5 (tables) × 3 (regions) × 2 (accounts) = 300 time series
× number of DynamoDB metrics = hundreds or thousands of DDUs per hour
```

S3, DynamoDB, and Lambda are particularly high-cardinality namespaces.

### Pre-deployment DDU estimation approach

```
Step 1: In CloudWatch, for each namespace you plan to stream:
        Note the number of metrics × number of unique dimension combinations

Step 2: Estimate time series count:
        TS = unique_metrics × unique_dimension_combinations

Step 3: DDU burn rate (approximate):
        DDU/hour ≈ TS × 0.001 (check current Dynatrace DDU pricing)

Step 4: Compare against your DDU license entitlement
```

### Recommended namespace streaming strategy

| Category | Approach | Namespaces |
|---|---|---|
| **Always include** (low cardinality, high signal) | Stream all metrics | `AWS/EC2`, `AWS/RDS`, `AWS/Backup`, `AWS/FSx`, `AWS/NetworkELB` |
| **Include with metric filter** (medium cardinality) | Select specific metrics | `AWS/DynamoDB`, `AWS/Lambda`, `AWS/SQS`, `AWS/SNS` |
| **Include with caution** (high cardinality) | Model first | `AWS/S3`, `AWS/ApiGateway`, `AWS/CloudFront` |
| **Custom namespaces** | Always explicit filter | `CWAgent`, `Windows/Default`, `ECS/ContainerInsights` |

---

## 8. Entity Enrichment — The AWS Entities Extension

By default, Metric Streams data arrives in Dynatrace as **raw metrics with dimensions** — not as entities in the Smartscape topology. This is the key architectural gap versus the Default Integration.

### What "no entity" means operationally

```
Without AWS Entities Extension:
  cloud.aws.ec2.statusCheckFailed
    dimensions: { aws.account.id, aws.region, instanceid }
    → Just a metric. No entity card. No topology context. No entity-based alerting.

With AWS Entities Extension:
  cloud.aws.ec2.statusCheckFailed
    dimensions: { aws.account.id, aws.region, instanceid }
    → Mapped to entity: EC2 instance (cloud:aws:ec2:instance)
    → Appears in Smartscape, entity-based dashboards, entity filtering
```

### Install the extension

1. In Dynatrace: **Hub** → search **AWS Entities for Metric Streaming** → **Deploy**
2. Follow the [extension lifecycle deployment instructions](https://docs.dynatrace.com/docs/extend-dynatrace/extensions20/extension-lifecycle#deploy-extension-from-dynatrace-hub)

### What entity types the extension maps

Based on metric dimensions, the extension creates entity representations for services including: EC2, RDS, ELB, DynamoDB, Lambda, S3, ECS, EFS, FSx, NLB, Backup, API Gateway, SQS, SNS, CloudFront, and more.

### Remaining gaps even with the extension

Even with the extension installed, two key capabilities remain unavailable vs. Default Integration:

1. **AWS resource tags** — Tags applied to AWS resources are not propagated to Dynatrace entities via Metric Streams. If your alerting or dashboarding relies on tag-based filtering, maintain the Default Integration for those services.

2. **Predefined anomaly detection alerts** — Dynatrace's out-of-the-box alert baselines are tied to Default Integration entities. For Metric Streams, you define custom metric events against `cloud.aws.*` metrics.

---

## 9. Querying Metric Streams Data with DQL

The official documentation shows metric keys but has no DQL query examples. These patterns are directly usable in Dynatrace Notebooks, Dashboards, and AutomationEngine workflows.

### Basic: EC2 status check failures

```sql
timeseries failures = avg(cloud.aws.ec2.statusCheckFailed),
  by: { aws.account.id, aws.region, instanceid }
| filter iAny(failures[] > 0)
```

### EC2 Windows disk free space (CWAgent namespace)

```sql
timeseries freeSpace = min(cloud.aws.windows.default.logicalDiskFreeSpace),
  by: { instanceid, objectname, aws.region }
| filter objectname == "C:"
| fieldsAdd minFree = arrayMin(freeSpace)
| fields instanceid, aws.region, objectname, minFree
| sort minFree asc
```

### NLB unhealthy host count — grouped by load balancer

```sql
timeseries unhealthy = max(cloud.aws.networkelb.unHealthyHostCount),
  by: { loadbalancer, targetgroup, aws.region }
| fieldsAdd peakUnhealthy = arrayMax(unhealthy)
| filter peakUnhealthy > 0
| fields loadbalancer, targetgroup, aws.region, peakUnhealthy
| sort peakUnhealthy desc
```

### Backup job failures — last 24h, by vault

```sql
timeseries failures = sum(cloud.aws.backup.numberOfBackupJobsFailed),
  by: { backupvaultname, resourcetype, aws.region },
  from: now()-24h
| fieldsAdd total = arraySum(failures)
| filter total > 0
| fields backupvaultname, resourcetype, aws.region, total
| sort total desc
```

### Multi-account metric comparison

```sql
timeseries failures = avg(cloud.aws.ec2.statusCheckFailed),
  by: { aws.account.id, aws.region }
| fieldsAdd avgFailures = arrayAvg(failures)
| sort avgFailures desc
```

### FSx storage utilization — alert threshold query

```sql
timeseries utilization = avg(cloud.aws.fsx.storageCapacityUtilization),
  by: { filesystemid, aws.region }
| fieldsAdd avgUtil = arrayAvg(utilization)
| filter avgUtil > 80
| fields filesystemid, aws.region, avgUtil
| sort avgUtil desc
```

---

## 10. Operational Runbook — Silent Failures and How to Catch Them

The most dangerous aspect of Metric Streams is that failures are silent by default. Metrics simply stop arriving — there are no alerts in Dynatrace because there is nothing to alert on.

### Failure mode 1: Firehose failing to reach Dynatrace API

**Detection:**
```bash
# Check Firehose delivery success rate in CloudWatch
# Note: metric name uses mixed case "Http" not "HTTP"
aws cloudwatch get-metric-statistics \
  --namespace AWS/Firehose \
  --metric-name DeliveryToHttpEndpoint.Success \
  --dimensions Name=DeliveryStreamName,Value=<your-stream-name> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Sum
```

**What to look for:** `DeliveryToHttpEndpoint.Success` count should be non-zero if metrics are being produced. A drop to zero means Firehose cannot reach Dynatrace. Also monitor `DeliveryToHttpEndpoint.DataFreshness` — AWS recommends alarming when this exceeds the 15-minute buffering limit.

**Detection via S3 backup:**
```bash
# Objects appearing in the backup bucket = failed deliveries
aws s3 ls s3://<your-backup-bucket>/ --recursive | wc -l
```

Set a CloudWatch alarm on this — and additionally alarm on `DeliveryToHttpEndpoint.DataFreshness` which AWS officially recommends for Firehose HTTP endpoint delivery streams:
```bash
# Alarm on backup bucket filling up (indicates failed deliveries)
aws cloudwatch put-metric-alarm \
  --alarm-name "Firehose-Backup-Objects-Accumulating" \
  --metric-name NumberOfObjects \
  --namespace AWS/S3 \
  --dimensions Name=BucketName,Value=<backup-bucket> Name=StorageType,Value=AllStorageTypes \
  --period 3600 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --statistic Average \
  --alarm-actions <your-sns-topic-arn>
```

```bash
# Alarm on Firehose delivery freshness — AWS officially recommends this alarm
# Triggers when data has not been delivered for more than 15 minutes (900 seconds)
aws cloudwatch put-metric-alarm \
  --alarm-name "Firehose-Dynatrace-DeliveryFreshness" \
  --metric-name DeliveryToHttpEndpoint.DataFreshness \
  --namespace AWS/Firehose \
  --dimensions Name=DeliveryStreamName,Value=<your-stream-name> \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 900 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --statistic Maximum \
  --alarm-actions <your-sns-topic-arn>
```

### Failure mode 2: CloudWatch Metric Stream not running

```bash
# Check metric stream status
aws cloudwatch list-metric-streams \
  --query 'Entries[*].{Name:Name,State:State,LastUpdateDate:LastUpdateDate}'
```

State should be `running`. If `stopped`, restart:
```bash
aws cloudwatch start-metric-streams --names <stream-name>
```

### Failure mode 3: Metrics older than 2 hours

CloudWatch will not stream metrics with data points older than 2 hours. This most commonly affects:
- Metrics from resources that are stopped (EC2 instances that are off)
- Infrequent metrics that are only emitted when an event occurs
- Metrics with aggregation delays (some services aggregate hourly)

**Detection:** Graph the metric in CloudWatch console. If the most recent data point is >2 hours old, it will not be streamed regardless of stream configuration.

### Monitoring dashboard for stream health (DQL)

Create this as a Dynatrace Notebook to give yourself operational visibility into the streaming pipeline:

```sql
// Metric freshness check — when did each metric last arrive?
data metrics
| filter matchesPhrase(metric.key, "cloud.aws.")
| summarize lastWritten = max(timestamp), count = count(),
    by: { metric.key }
| sort lastWritten asc
| limit 50
// Metrics sorted oldest-first — any surprises here indicate a streaming gap
```

---

## 11. High-Signal Metric Reference by AWS Service

These are the metrics worth streaming based on operational value — covering gaps that the Default Integration does not provide.

### Amazon EC2 — Extended OS metrics (requires CWAgent)

| Metric | Namespace | Why it matters |
|---|---|---|
| `LogicalDisk % Free Space` | `Windows/Default` | Windows disk — not available via default EC2 integration |
| `Memory % Committed Bytes In Use` | `Windows/Default` | Windows memory — not available via default EC2 integration |
| `disk_used_percent` | `CWAgent` | Linux disk — not available via default EC2 integration |
| `mem_used_percent` | `CWAgent` | Linux memory — not available via default EC2 integration |

**DQL query — Windows disks with <20% free:**
```sql
timeseries min(cloud.aws.windows.default.logicalDiskFreeSpace),
  by: { instanceid, objectname }
| filter objectname != "_Total"
| filter min(cloud.aws.windows.default.logicalDiskFreeSpace) < 20
```

### Amazon EC2 — Status checks

| Metric key in Dynatrace | Alert threshold |
|---|---|
| `cloud.aws.ec2.statusCheckFailed` | > 0 |
| `cloud.aws.ec2.statusCheckFailed_Instance` | > 0 |
| `cloud.aws.ec2.statusCheckFailed_System` | > 0 |
| `cloud.aws.ec2.statusCheckFailed_AttachedEBS` | > 0 |

### Amazon FSx

| Metric key | Alert threshold |
|---|---|
| `cloud.aws.fsx.storageCapacityUtilization` | > 80% (warning), > 90% (critical) |

### AWS Backup

| Metric key | Alert threshold |
|---|---|
| `cloud.aws.backup.numberOfBackupJobsFailed` | > 0 |
| `cloud.aws.backup.numberOfBackupJobsAborted` | > 0 |

### Amazon Network Load Balancer

| Metric key | Alert threshold |
|---|---|
| `cloud.aws.networkelb.unHealthyHostCount` | > 0 (by targetgroup) |
| `cloud.aws.networkelb.healthyHostCount` | < 1 (by targetgroup) — complete outage |

---

## 12. Limitations You Must Know

### Platform constraints

- **2-hour staleness limit** — A metric point is not streamed if it is more than 2 hours old. No workaround. This is a CloudWatch Metric Streams platform constraint.
- **Namespace-level selection only** — You cannot filter at the individual statistic level in CloudWatch Metric Streams (unlike Default Integration which allows per-statistic selection).
- **No tag propagation** — AWS resource tags are not available on Metric Streams metrics in Dynatrace, even with the AWS Entities extension. Plan entity-based filtering around dimensions (account ID, region, resource ID), not tags.
- **No predefined anomaly baselines** — Dynatrace's AI-driven baselines (Davis) apply to entity-linked metrics. Metric Streams data requires manually configured metric events.

### Operational constraints

- **Silent delivery failures** — Firehose delivery failures do not generate alerts in Dynatrace automatically (there is nothing to alert on if metrics stop arriving). Implement the S3 backup monitoring described in §13.
- **DDU consumption at cardinality** — High-cardinality namespaces (S3, DynamoDB, Lambda) can consume significant DDUs at scale. Model before enabling. See §9.
- **Delivery lag** — The minimum delivery lag is the Firehose buffer interval (60 seconds recommended). This is not real-time in the strictest sense — it is near-real-time.
- **Per-region deployment** — Each AWS region requires its own Firehose stream and CloudWatch Metric Stream. A 10-region, 5-account environment needs up to 50 independent stream deployments (managed via IaC — see §6 and §7).

### References

- [Dynatrace: Amazon CloudWatch Metric Streams](https://docs.dynatrace.com/docs/shortlink/aws-metric-streams)
- [Dynatrace Hub: AWS Entities for Metric Streaming](https://www.dynatrace.com/hub/detail/aws-entities-for-metric-streaming/)
- [AWS CloudWatch Metric Streams documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Metric-Streams.html)
- [AWS CloudWatch Metric Streams troubleshooting](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-metric-streams-troubleshoot.html)
- [Dynatrace Monaco (Monitoring-as-Code)](https://www.dynatrace.com/support/help/manage/configuration-as-code)
- [Amazon Data Firehose documentation](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)

---

*Written from direct operational experience managing Dynatrace Metric Streams at scale across large enterprise AWS environments. Feedback and contributions welcome.*

*Connect on [LinkedIn](https://linkedin.com/in/niladri-sekhar-mondal-work)*

---

## 13. Coming Next — IaC & Monaco *(In Progress)*

> 🚧 **The following sections are currently being authored and will be published in the next iteration of this guide.**

### CloudFormation Deployment *(Coming Soon)*

A complete walkthrough for deploying the full Metric Streams pipeline — Firehose delivery stream, CloudWatch Metric Stream, IAM roles, and S3 backup bucket — using a single Dynatrace-provided CloudFormation template. Will cover single-region, multi-region, and multi-account (linked accounts) patterns.

### Terraform Deployment *(Coming Soon)*

A fully working, production-ready Terraform module (`aws_kinesis_firehose_delivery_stream` + `aws_cloudwatch_metric_stream` + IAM + S3 lifecycle) with verified resource structure, correct `common_attributes` configuration, and multi-region workspace patterns.

### Monaco — Monitoring-as-Code at Scale *(Coming Soon)*

How to manage Metric Streams dashboards and metric event alerts across tens or hundreds of Dynatrace environments using Monaco v2. Will include a full project structure, verified `manifest.yaml` with `environmentGroups`, and ready-to-deploy metric event JSON templates for EC2, NLB, FSx, and AWS Backup.

---
