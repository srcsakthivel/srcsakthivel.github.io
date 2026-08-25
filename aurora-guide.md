---
layout: page
title: "Amazon Aurora — From Cluster Basics to DR"
permalink: /aurora-guide/
---

# Amazon Aurora — From Cluster Basics to DR
{: .no_toc }

A comprehensive guide covering Aurora architecture, endpoints, sizing, storage, security, monitoring, high availability, backups, and disaster recovery.

## Table of Contents
{: .no_toc }

- TOC
{:toc}

---

## 1. Cluster Architecture

**The foundation: separation of compute and storage**

Unlike traditional databases where each instance manages its own storage, Aurora **separates compute from storage** entirely.

An Aurora DB cluster consists of two key components:
- **DB Instances (Compute):** 1 primary (read/write) + 0–15 Aurora Replicas (read-only)
- **Cluster Volume (Storage):** A shared, distributed storage volume spanning 3 Availability Zones

> **Key Insight:** All instances read from the *same* shared storage. No replication lag at the storage level — replicas see new data within milliseconds.

### Traditional RDS vs Aurora

| Feature | Traditional RDS | Aurora |
|---------|----------------|--------|
| Storage | Each instance owns EBS | All share one distributed volume |
| Replication | Logical (WAL replay) | Shared storage (no replay) |
| Read Replicas | Max 5 | Up to 15 |
| Storage Scaling | Manual resize | Auto-scales (10 GB → 256 TiB) |
| Replica Lag | Seconds to minutes | Typically < 20ms |

---

## 2. Endpoints & Connectivity

**How your application connects to the right instance**

Aurora provides **4 endpoint types**:

### Cluster (Writer) Endpoint

```
mydb.cluster-xyz123.us-east-1.rds.amazonaws.com
```

- Always points to the **current primary instance**
- Auto-updates during failover
- Use for: All writes (DML/DDL) and read-after-write consistency

### Reader Endpoint

```
mydb.cluster-ro-xyz123.us-east-1.rds.amazonaws.com
```

- **Load-balances** connections across all replicas
- If no replicas exist, routes to primary
- Use for: Read-heavy workloads, reporting

### Instance Endpoint

```
mydb-instance-1.xyz123.us-east-1.rds.amazonaws.com
```

- Direct connection to a **specific instance**
- Does NOT auto-redirect during failover
- Use for: Debugging, specific instance access only

### Custom Endpoint

```
my-analytics.cluster-custom-xyz123.us-east-1.rds.amazonaws.com
```

- You pick which instances belong
- Type = READER (replicas only) or ANY (writer + replicas)
- Use for: Isolating analytics workloads to specific high-memory replicas

### Failover Behavior

- Aurora DNS endpoints use a **5-second TTL**
- During failover: DNS auto-updates → cluster endpoint points to new primary
- Total failover time with replicas: **~30 seconds**

> ⚠️ **Common Mistake:** Don't use instance endpoints for production apps — they won't auto-redirect during failover.

---

## 3. Instance Classes & Sizing

**Choosing the right compute for your workload**

### Memory-Optimized (R-series) — MOST COMMON

- **Families:** db.r6g, db.r7g, db.r8g, db.x2g
- **Memory:** 16 GB – 1,024 GB
- **Processor:** Graviton2/3/4 or Intel
- **Best for:** Production workloads

### Burstable (T-series) — DEV/TEST ONLY

- **Families:** db.t4g, db.t3
- **Memory:** 4 GB – 32 GB
- **Mode:** Unlimited burst
- **Best for:** Development and testing

### Serverless v2 — PAY-PER-USE

- **Range:** 0.5 – 256 ACUs
- **1 ACU ≈** 2 GB RAM + proportional CPU
- **Billing:** Per-second
- **Best for:** Variable/unpredictable workloads

### Sizing Decision Guide

| Workload | Recommended | Why |
|----------|-------------|-----|
| Steady OLTP, 100+ connections | db.r7g.2xlarge (64 GB) | Stable memory, Graviton3 |
| Small app, < 50 connections | db.r7g.large (16 GB) | Right-sized |
| Dev/Test | db.t4g.medium (4 GB) | Minimal cost |
| Variable traffic | Serverless v2 (2–128 ACUs) | Auto-scales |

> **ACU Conversion:** Instance memory (GB) ÷ 2 = equivalent Serverless v2 max ACUs.

---

## 4. Storage & I/O

**Auto-scaling, 6 copies across 3 AZs, and pricing models**

### Storage Architecture

- **6 copies** of your data across **3 Availability Zones** (2 per AZ)
- **Quorum Writes:** 4/6 must ACK → write confirmed
- **Quorum Reads:** 3/6 → read confirmed (recovery only)
- **Self-Healing:** Lost copies auto-repaired in background
- **Fault tolerance:** Lose entire AZ + 1 more copy → still serves reads
- **Auto-scales:** 10 GB → 256 TiB, no provisioning needed

### I/O-Optimized vs Standard

| | Aurora Standard | Aurora I/O-Optimized |
|---|---|---|
| Storage cost | $0.10/GB-month | $0.225/GB-month |
| I/O charges | $0.20 per million requests | $0 (zero!) |
| Best when | I/O < 25% of total spend | I/O ≥ 25% of total spend |
| Switch | To I/O-Optimized any time | Back to Standard any time |

> **Decision Rule:** If I/O is ≥ 25% of your total Aurora cost → I/O-Optimized saves money. Switch without downtime.

---

## 5. Parameter Groups & Configuration

**Two-tier configuration: cluster-level and instance-level**

### DB Cluster Parameter Group

- **Scope:** ALL instances in the cluster
- **Controls:** Storage layout, replication, time zone, character sets, audit logging, SSL enforcement

### DB Instance Parameter Group

- **Scope:** ONE specific instance
- **Controls:** Memory buffers, max_connections, work_mem, temp tables

### Precedence

```
Cluster Param Group → overridden by → DB Param Group → Final Value
```

### Key Parameters

| Parameter | Level | Guidance |
|-----------|-------|----------|
| `max_connections` | Instance | Based on memory. Use RDS Proxy instead of raising. |
| `shared_buffers` | Instance | Aurora manages differently. Leave at default. |
| `require_secure_transport` | Cluster | Set ON for production (enforce TLS). |
| `server_audit_logging` | Cluster | Enable for compliance. |
| `performance_schema` | Instance | Enable for Performance Insights (~1-2% overhead). |

> **Best Practice:** Always create a custom parameter group at cluster creation — the default cannot be modified later.

---

## 6. Security

**VPC, authentication, encryption, and credential management**

### Defense-in-Depth Layers

1. **Network Isolation (VPC)** — Private subnets, DB Subnet Groups (2+ AZs), Security Groups
2. **Authentication** — Password / IAM DB Auth (token-based) / Kerberos (AD)
3. **Encryption at Rest (KMS)** — AES-256 for storage, backups, snapshots. All new clusters encrypted by default.
4. **Encryption in Transit (TLS)** — Enforce with `require_secure_transport=ON` or `rds.force_ssl=1`
5. **Secrets Manager** — Auto-rotation of master credentials. Native RDS integration.

### Security Checklist

- ✅ DB Subnet Group in private subnets across 2+ AZs
- ✅ Security Group: allow DB port from app SG only
- ✅ Disable public accessibility
- ✅ Encryption at rest with customer-managed CMK
- ✅ Enforce TLS via parameter group
- ✅ IAM DB Auth or Secrets Manager (no hardcoded passwords)
- ✅ Enable audit logging for compliance

---

## 7. Monitoring & Performance

**CloudWatch, Performance Insights, Enhanced Monitoring, Database Insights**

### Four Complementary Tools

| Tool | What It Shows | Granularity |
|------|--------------|-------------|
| CloudWatch Metrics | CPU, connections, IOPS, latency, memory | 1-minute (automatic) |
| Performance Insights | DB load (AAS), wait events, top SQL | Per-second |
| Enhanced Monitoring | OS-level: per-process CPU, memory, swap | 1-second |
| Database Insights | Slow SQL, execution plans, correlation | Unified dashboard |

### Key Alerting Thresholds

| Metric | Threshold | Action |
|--------|-----------|--------|
| CPUUtilization | > 80% sustained | Scale up or add replicas |
| FreeableMemory | < 500 MB | Scale up instance class |
| DatabaseConnections | > 80% of max | Use RDS Proxy |
| AuroraReplicaLag | > 100ms | Check replica sizing |
| DiskQueueDepth | > 10 sustained | Consider I/O-Optimized |

### Troubleshooting Workflow

1. **CloudWatch:** Check CPUUtilization and DatabaseConnections
2. **Performance Insights:** DB Load (AAS) vs vCPU line
3. **Wait Events:** I/O → storage, Lock → contention, CPU → bad queries
4. **Top SQL:** Identify specific queries causing load
5. **Enhanced Monitoring:** OS-level breakdown
6. **Slow Query Log:** Queries exceeding threshold

---

## 8. High Availability

**Failover mechanics, replica promotion, and RDS Proxy**

### Automatic Failover Timeline

| Time | Event |
|------|-------|
| 0s | Failure detected |
| ~5s | DNS TTL expires |
| ~15s | Replica promoted |
| ~30s | Connections resume |

### Priority Tiers

- Each replica has tier 0–15. Lower = promoted first.
- Tie-breaker: largest instance wins.
- Set designated failover target to tier 0.

### HA Patterns

| Configuration | Failover Time | Connection Handling |
|---|---|---|
| Single instance (no HA) | Minutes | Full reconnect required |
| Multi-AZ replicas | ~30 seconds | Reconnect on DNS update |
| Multi-AZ + RDS Proxy | ~1-2 seconds (app view) | Proxy holds & redirects |

> **RDS Proxy:** Sits between app and Aurora. Maintains connection pool. During failover, holds connections and redirects to new primary — app sees brief pause, not a disconnect. Best for Lambda, microservices, short-lived connections.

---

## 9. Backup & Restore

**Automated backups, snapshots, PITR, Backtrack, cloning**

### Recovery Options

| Method | Retention | Speed | Result | Engine |
|--------|-----------|-------|--------|--------|
| Automated Continuous | 1–35 days | Fast | Per-second PITR | Both |
| Manual Snapshot | Forever | Medium | New cluster | Both |
| PITR | Within retention | Minutes | New cluster | Both |
| Backtrack | Up to 72 hours | Seconds | In-place rewind | MySQL only |
| Clone | N/A | Instant | Linked copy (CoW) | Both |

### When to Use Each

| Scenario | Best Method |
|----------|-------------|
| Accidental DELETE (< 72h) | Backtrack (MySQL only) |
| Exact timestamp recovery | PITR |
| Before risky deployment | Manual Snapshot |
| Cross-region DR copy | Snapshot Copy |
| Test with prod data | Clone (copy-on-write) |

> **Aurora Cloning:** Creates full copy via copy-on-write. Near-instant, no extra cost initially. Perfect for testing with production data.

---

## 10. Disaster Recovery

**Global Database, cross-region replication, RPO/RTO**

### Aurora Global Database

| Metric | Value |
|--------|-------|
| **RPO** | ~1 second |
| **RTO** | < 1 minute |
| **Secondary Regions** | Up to 5 |
| **Replication** | Storage-level, asynchronous |

### Switchover vs Failover

| | Managed Switchover | Unplanned Failover |
|---|---|---|
| When | Planned migration | Region outage |
| Data loss | Zero | ~1-5 seconds |
| Downtime | ~1-2 min | < 1 min |
| Reversible | Yes (auto) | Manual rebuild |

### Global Database vs Snapshot Copy

| Feature | Global Database | Snapshot Copy |
|---------|----------------|--------------|
| Replication | Continuous | Manual/scheduled |
| RPO | ~1 second | Hours |
| RTO | < 1 minute | 30-60 minutes |
| Serve reads in DR | Yes | No |
| Cost | Full cluster in secondary | Snapshot storage only |

> **Headless Clusters:** Secondary region doesn't need compute until failover. Storage replication runs without instances — add compute only during promotion. Saves significant DR cost.

---

## Summary

| # | Topic | Key Takeaway |
|---|-------|-------------|
| 1 | Architecture | Compute/storage separation, primary + up to 15 replicas |
| 2 | Endpoints | 4 types, DNS-based failover routing (5s TTL) |
| 3 | Sizing | R-series (prod) / T-series (dev) / Serverless v2 (variable) |
| 4 | Storage | 6 copies, 3 AZs, I/O-Optimized vs Standard |
| 5 | Parameters | Cluster vs instance level, always create custom groups |
| 6 | Security | VPC + IAM auth + KMS + TLS + Secrets Manager |
| 7 | Monitoring | CloudWatch + Performance Insights + Enhanced Monitoring |
| 8 | HA | Multi-AZ ~30s failover; +RDS Proxy = ~1-2s |
| 9 | Backup | Continuous PITR, snapshots, Backtrack, cloning |
| 10 | DR | Global Database: RPO ~1s, RTO < 1 min |

---

*Sources: [AWS Aurora Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/), [AWS Database Blog](https://aws.amazon.com/blogs/database/)*
