# Infoblox integration — customer handoff checklist

**Purpose:** Give this page to your customer's **Infoblox / IPAM / DNS** team when scoping Cisco Secure Workload (CSW) label import from Infoblox.

**Integration summary:** CSW reads subnet, host, and DNS record metadata from Infoblox via **WAPI (HTTPS)** and maps **extensible attributes** to workload labels. CSW **does not create, modify, or delete** Infoblox objects.

---

## What we need from the Infoblox team

### 1. Access model — read-only only

| Required | Not required |
|----------|----------------|
| **Read-only WAPI** access to import objects | Read-write or grid admin |
| API access enabled on the service account | Interactive admin login for CSW |
| HTTPS to grid member WAPI endpoint | Inbound connections from CSW to arbitrary ports |

> Cisco Secure Workload uses Infoblox as a **label source only**. There is **no policy enforcement** back to Infoblox (unlike F5 or NetScaler integrations). A dedicated **read-only** WAPI account is sufficient and recommended.

### 2. Service account

Please provision:

| Item | Detail |
|------|--------|
| **Account name** | e.g. `svc_csw_infoblox` (dedicated service account) |
| **WAPI version** | **2.7.1** recommended (supported: 2.6, 2.6.1, 2.7, 2.7.1) |
| **Permissions** | **READ** on networks/subnets, `record:host`, `record:a`, `record:aaaa`, and extensible attributes for objects in scope |
| **API access** | WAPI/API access enabled on the admin profile |
| **Scope** | Limit to network views / zones that should appear in CSW (principle of least privilege) |

### 3. Connectivity

CSW must reach the Infoblox **WAPI HTTPS endpoint** from:

- The Secure Workload cluster (on-prem), **or**
- Cisco Secure Workload **SaaS** via a **[Secure Connector](https://github.com/chandrapati/csw-secure-connector)** VM in your network, **or**
- Your approved corporate egress path

| Direction | Source | Destination | Port |
|-----------|--------|-------------|------|
| Outbound | CSW or Secure Connector VM | Infoblox grid member WAPI | **443/TCP** (HTTPS) |
| Outbound | Secure Connector VM (if used) | CSW SaaS FQDN | **443/TCP** |

**No inbound firewall rules** to the Secure Connector VM are required.

### 4. Grid endpoint(s)

Provide one or more **grid member** FQDNs or IPs for the **Hosts List** in CSW:

```
Example:
  infoblox-grid1.corp.example.com:443
  infoblox-grid2.corp.example.com:443   (optional HA member)
```

- One external orchestrator in CSW = **one Infoblox grid**
- Multiple members in the hosts list = **failover** within that grid
- A second grid requires a **second** orchestrator configuration

### 5. Extensible attributes (critical)

CSW imports **only** objects that have **extensible attributes** attached.

| Action for Infoblox team | Why |
|--------------------------|-----|
| Confirm which **EAs** drive segmentation (e.g. `Environment`, `Application`, `Owner`, `Department`) | These become CSW labels |
| Ensure target subnets/hosts/records have EAs populated | Objects **without** EAs are **skipped** |
| Use **inherited** EAs on subnets where host records should inherit searchable labels | Non-inherited subnet EAs may not appear on host inventory |

### 6. Record types to import

CSW can import (select during setup):

- [ ] **Subnets** (network objects)
- [ ] **Hosts** (`record:host`)
- [ ] **A / AAAA** DNS records

Deselect types not needed to reduce sync time and object counts.

---

## Scale limits (Cisco documented)

| Object type | Maximum |
|-------------|---------|
| Subnets | **50,000** |
| Hosts + A/AAAA records (combined) | **400,000** |

First full snapshot may take **up to ~1 hour** on large grids.

---

## What CSW will produce

| Infoblox source | CSW label pattern | Example |
|-----------------|-------------------|---------|
| Extensible attribute `Department` | `orchestrator_Department` | `orchestrator_Department=Finance` |
| System metadata | `orchestrator_system/orch_type` | `infoblox` |

Use these labels in **inventory filters**, **scopes**, and **policy** (e.g. segment by `orchestrator_Environment=Production`).

**Note:** Subnets are used for label context but **cannot be found via inventory search** (CSW inventory is IP/host oriented). Search hosts and A/AAAA records using `orchestrator_<attribute>=<value>`.

---

## Security & compliance talking points

- **Read-only** — no write path to Infoblox from CSW
- **Outbound-only** from Secure Connector (when used)
- **Mutual TLS** on Secure Connector tunnel (SaaS designs)
- **Dedicated service account** — auditable, rotatable credentials
- **No customer data** leaves Infoblox except metadata CSW already needs for labeling (per your data-handling review)

---

## Validation (joint test)

| # | Test | Owner | Pass |
|---|------|-------|------|
| 1 | WAPI login with service account from Secure Connector VM or CSW network | Infoblox + CSW team | ☐ |
| 2 | CSW external orchestrator **Connection Status: Success** | CSW admin | ☐ |
| 3 | Sample host with EA visible in CSW inventory search | CSW admin | ☐ |
| 4 | Scope built on `orchestrator_<EA>` matches expected IPs | Security architect | ☐ |

---

## References

- Full guide: [CSW-Infoblox-Integration-Guide.md](../CSW-Infoblox-Integration-Guide.md)
- Cisco official: [External Orchestrators — Infoblox](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-external-orchestrators.html)
- Secure Connector (if needed): [github.com/chandrapati/csw-secure-connector](https://github.com/chandrapati/csw-secure-connector)

---

*Template version 2026-06-26 — generic; no customer-specific names. Customize contact names and endpoints locally before sending.*
