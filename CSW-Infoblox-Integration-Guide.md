---
title: "Cisco Secure Workload — Infoblox Integration Guide"
subtitle: "Import IPAM/DNS extensible attributes as workload labels"
author: "Cisco Secure Workload Solutions Engineering"
date: "2026-06-26"
---

# Cisco Secure Workload — Infoblox Integration Guide

Import **Infoblox IPAM and DNS metadata** into Cisco Secure Workload (CSW) as **labels** for inventory, scopes, and microsegmentation policy. This integration is **label-only** — CSW does not enforce policy on Infoblox and does not write back to the grid.

---

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Infoblox team preparation](#4-infoblox-team-preparation)
5. [Configure CSW external orchestrator](#5-configure-csw-external-orchestrator)
6. [Labels and inventory behavior](#6-labels-and-inventory-behavior)
7. [Using labels in scopes and policy](#7-using-labels-in-scopes-and-policy)
8. [Limits and caveats](#8-limits-and-caveats)
9. [Troubleshooting](#9-troubleshooting)
10. [POV validation checklist](#10-pov-validation-checklist)
11. [Official references](#11-official-references)

---

## 1. Overview

### What Infoblox integration does

| Capability | Supported |
|------------|-----------|
| Import subnets, hosts, A/AAAA records | Yes |
| Map **extensible attributes** to CSW labels | Yes |
| Create CSW inventory from DNS alone | Partial — labels attach to existing IP inventory |
| Enforce segmentation on Infoblox | **No** |
| Write to Infoblox WAPI | **No** |

### When to use Infoblox vs other orchestrators

| Use Infoblox when… | Use something else when… |
|--------------------|--------------------------|
| IPAM/DNS EAs define app, env, owner, zone | VMware tags → **vCenter** orchestrator |
| Macro-segmentation by subnet metadata | Cloud tags → **AWS/Azure/GCP connectors** |
| DNS names as labels on known IPs | CMDB ownership → **ServiceNow** orchestrator |
| Agentless flow + IPAM context | Flow-only visibility → **NetFlow** / **ERSPAN** |

Infoblox is commonly paired with **ServiceNow CMDB** (application ownership) and **NetFlow** (agentless traffic) in hybrid estates.

---

## 2. Architecture

```
┌─────────────────────┐         HTTPS WAPI (443)        ┌──────────────────────┐
│  Cisco Secure       │ ──────────────────────────────► │  Infoblox Grid       │
│  Workload           │         read-only import        │  (IPAM + DNS)        │
│  External Orch.     │ ◄────────────────────────────── │  Extensible attrs    │
└─────────┬───────────┘         labels / metadata       └──────────────────────┘
          │
          │ labels: orchestrator_<EA>=<value>
          ▼
┌─────────────────────┐
│  Inventory / Scopes │
│  / Policy           │
└─────────────────────┘
```

### SaaS with private Infoblox

When CSW runs as **SaaS** and Infoblox is on a private network, deploy a **[Secure Connector](https://github.com/chandrapati/csw-secure-connector)** VM:

```
Infoblox (private) ◄──443── Secure Connector VM ──mTLS──► CSW SaaS
```

The connector initiates **outbound-only** connections. No inbound firewall holes to the connector.

---

## 3. Prerequisites

### Infoblox

| Requirement | Detail |
|-------------|--------|
| **WAPI version** | 2.6, 2.6.1, 2.7, or **2.7.1** (recommended) |
| **Service account** | **Read-only** WAPI access (sufficient for standard integration) |
| **Extensible attributes** | Objects **without** EAs are **not imported** |
| **Reachability** | HTTPS from CSW cluster or Secure Connector to grid member WAPI |

### Cisco Secure Workload

| Requirement | Detail |
|-------------|--------|
| **Role** | Permission to create/edit external orchestrators |
| **Version** | CSW 3.x / 4.x per your deployment (SaaS or on-prem) |
| **Secure Connector** | Required when SaaS cannot reach Infoblox directly |

### Access level — read-only is sufficient

CSW Infoblox integration is **import-only**. Unlike F5 BIG-IP or Citrix NetScaler orchestrators, it does **not** deploy ACLs or policies to Infoblox.

| Access | Needed? |
|--------|---------|
| WAPI **read** | **Yes** |
| WAPI **write** / grid admin | **No** |

---

## 4. Infoblox team preparation

Share [`docs/CUSTOMER-HANDOFF.md`](./docs/CUSTOMER-HANDOFF.md) with the customer's Infoblox administrators. Minimum deliverables:

1. **Service account** with read-only WAPI and API access enabled
2. **Grid member hostname(s)** for the CSW hosts list
3. **Firewall** allow: CSW or Secure Connector → Infoblox **443/TCP**
4. **Extensible attribute** inventory — which EAs should drive segmentation
5. **Population** of EAs on subnets/hosts/records in scope

### Recommended extensible attributes

Examples (customer-specific):

| EA name | Typical use in CSW |
|---------|-------------------|
| `Environment` | Prod / Dev / Test scopes |
| `Application` | App-tier microsegments |
| `Department` | Business-unit macro segments |
| `Owner` | Accountability / change windows |
| `DataClassification` | Compliance-oriented policies |

Ensure EAs are **inherited** on subnets where host records should inherit searchable labels.

---

## 5. Configure CSW external orchestrator

### Step 1 — Verify connectivity

From the same network path CSW will use (or from the Secure Connector VM):

```bash
curl -k -u 'svc_csw:PASSWORD' \
  'https://infoblox-grid.example.com/wapi/v2.7.1/network?_max_results=1'
```

Expect HTTP 200 and JSON results (not 401/403/timeout).

### Step 2 — Create orchestrator in CSW UI

1. **Manage → Workloads → External Orchestrators**
2. **Create New Configuration**
3. Set fields:

| Field | Value |
|-------|-------|
| **Type** | `Infoblox` |
| **Name** | Unique name, e.g. `infoblox-prod-grid` |
| **Description** | Optional — owner, change ticket, environment |
| **Hosts List** | Grid member FQDN or IP (one grid per orchestrator; add members for HA) |
| **Username / Password** | Read-only WAPI service account |
| **Record types** | Select subnets, hosts, A/AAAA as needed |

4. **Create**

### Step 3 — Wait for first sync

- Initial **full snapshot** may take **up to ~1 hour** on large grids
- **Connection Status** should show **Success**
- Subsequent updates use delta sync per CSW schedule

### Step 4 — Validate labels

**Inventory → Search** using:

```
orchestrator_Environment=Production
```

(or your EA name). Confirm expected hosts appear.

---

## 6. Labels and inventory behavior

### Label naming

| Source | CSW label key | Example value |
|--------|---------------|---------------|
| Extensible attribute `Department` | `orchestrator_Department` | `Finance` |
| Orchestrator metadata | `orchestrator_system/orch_type` | `infoblox` |
| Orchestrator metadata | `orchestrator_system/name` | `infoblox-prod-grid` |
| Orchestrator metadata | `orchestrator_system/cluster_id` | UUID of config |

Prefix `orchestrator_` is added automatically to extensible attribute names.

### What appears in inventory

| Object imported | In inventory search? | Notes |
|-----------------|---------------------|-------|
| Host (`record:host`) | Yes | Labels from host + inherited subnet EAs |
| A / AAAA records | Yes | Labels from record EAs |
| Subnets | Labels applied | **Not** findable via inventory search — use host/A record search |

### What is skipped

- Objects **without** extensible attributes
- Objects outside the service account's permitted network views
- Record types not selected during configuration

---

## 7. Using labels in scopes and policy

### Example scope query

Segment all production workloads tagged in Infoblox:

```
orchestrator_Environment = Production
```

Combine with other label sources:

```
orchestrator_Environment = Production AND orchestrator_Application = Payments
```

### Common patterns

| Pattern | Query idea |
|---------|------------|
| Macro-segment by VLAN/subnet EA | Scope on `orchestrator_Zone=DMZ` |
| App + env | `orchestrator_Application=X AND orchestrator_Environment=Prod` |
| Infoblox + CMDB | `orchestrator_Application=X AND orchestrator_cmdb_ci_class=...` (ServiceNow) |
| Agentless + IPAM | NetFlow-discovered IPs + `orchestrator_*` from Infoblox |

### Policy workflow

1. Build **inventory filter** or **scope** using `orchestrator_*` labels
2. Run **policy discovery** (if agents present) or define **static allow lists**
3. **Simulate** in workspace before enforcement
4. Infoblox label changes propagate on next orchestrator sync — re-validate scopes after large EA changes

---

## 8. Limits and caveats

| Limit | Value |
|-------|-------|
| Max subnets | 50,000 |
| Max hosts + A/AAAA (combined) | 400,000 |
| WAPI versions | 2.6 – 2.7.1 |
| Enforcement on Infoblox | Not supported |
| Write-back to Infoblox | Not supported |

### Operational caveats

- **EA-only import** — plan a data-quality pass before go-live
- **One orchestrator per grid** — multiple grids = multiple configs
- **Subnet search** — search hosts/records, not subnet objects directly
- **Sync latency** — label changes in Infoblox appear after next delta/full cycle
- **DNS orchestrator is separate** — AXFR-based DNS orchestrator is different from Infoblox; do not duplicate unless intentional

---

## 9. Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Connection Status: Failed | Firewall, wrong host, bad creds | Test `curl` WAPI from connector VM; verify 443 |
| Success but no labels | Missing extensible attributes | Add EAs in Infoblox; confirm inheritance |
| Partial inventory | Record types deselected or RBAC scope | Re-check import options and service account permissions |
| Slow first sync | Large grid | Expected up to ~1 hour; monitor orchestrator status |
| Labels stale | Sync interval / orchestrator paused | Check orchestrator status; trigger reconnect if supported |
| SaaS timeout | No Secure Connector | Deploy connector per [csw-secure-connector](https://github.com/chandrapati/csw-secure-connector) |
| 401 / 403 from WAPI | Account lacks read on object types | Infoblox team: grant READ on network, host, A/AAAA |

### Disconnect alerts

CSW can alert when an external orchestrator disconnects. Use a controlled test (temporary firewall block) during POV to validate alerting if required.

---

## 10. POV validation checklist

| # | Test | Expected result | Pass |
|---|------|-----------------|------|
| 1 | WAPI reachability from CSW path | HTTP 200 on test query | ☐ |
| 2 | Orchestrator connection | Status **Success** | ☐ |
| 3 | Host with EA in inventory | `orchestrator_<EA>` visible | ☐ |
| 4 | Scope on EA | Scope matches expected IP set | ☐ |
| 5 | EA change in Infoblox | Label updates after sync window | ☐ |
| 6 | Secure Connector (if SaaS) | Tunnel healthy; no inbound rules needed | ☐ |
| 7 | Scale spot-check | Object counts within documented limits | ☐ |

---

## 11. Official references

| Document | Link |
|----------|------|
| CSW User Guide — External Orchestrators (Infoblox) | [Cisco doc](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-external-orchestrators.html) |
| CSW Secure Connector | [csw-secure-connector](https://github.com/chandrapati/csw-secure-connector) |
| CSW User Education — External Orchestrators | [CSW-User-Education](https://github.com/chandrapati/CSW-User-Education) |
| Infoblox WAPI documentation | [Infoblox WAPI](https://infoblox-docs.atlassian.net/wiki/spaces/NAGIOS/pages/79498627/WAPI) |

---

*Community reference guide — verify against current Cisco Secure Workload release notes for your deployment.*
