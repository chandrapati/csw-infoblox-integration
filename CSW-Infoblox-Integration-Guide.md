# Cisco Secure Workload → Infoblox Integration Guide

A step-by-step guide to enriching Cisco Secure Workload (CSW) inventory with **Infoblox IPAM/DNS metadata** by configuring an **Infoblox External Orchestrator**. Imported Infoblox *extensible attributes* become Secure Workload **labels** (prefixed `orchestrator_`) that you use to build scopes and enforcement policies.

> **⚠ Disclaimer:** This is a **community reference guide** prepared by Cisco Solutions Engineering — not an official Cisco product document. Always refer to the [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/support/security/tetration/series.html) and the [Cisco Secure Workload Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) for authoritative, up-to-date guidance.

---

## 1. Overview

Cisco Secure Workload builds scopes and micro-segmentation policies from **labels** attached to inventory (IP addresses / workloads). Getting rich, accurate labels is the single biggest driver of good policy. If your organization already tracks metadata in **Infoblox** — DDI (DNS, DHCP, IPAM) with *extensible attributes* such as `Department`, `Environment`, `Owner`, `Location`, or `Application` — you can pull that context directly into CSW instead of re-labeling by hand.

The **Infoblox External Orchestrator** connects Secure Workload to an Infoblox Grid over the Infoblox **WAPI** (REST API) and imports:

| Infoblox object | Imported into CSW inventory? | Notes |
|---|---|---|
| **Subnets / networks** | Metadata only | Not directly searchable as inventory (CSW inventory is IP-based); their extensible attributes can be inherited by hosts |
| **Hosts** (`record:host`) | ✅ Yes | Become inventory items with labels |
| **A / AAAA records** | ✅ Yes | Become inventory items with labels |

> **Key rule:** Only Infoblox objects that have **extensible attributes attached** are imported. Objects with no extensible attributes are skipped.

### What you get

- Infoblox **extensible attributes** appear in CSW as labels prefixed with `orchestrator_` (e.g. `orchestrator_Department`).
- Use those labels in **inventory search**, **scope definitions**, and **policy** — no manual CMDB export required.
- One consistent source of truth: update the attribute in Infoblox, and CSW picks it up on the next snapshot.

---

## 2. Architecture

![CSW and Infoblox Integration Architecture](csw-infoblox-architecture.png)

*Secure Workload's Infoblox external orchestrator polls the Infoblox Grid over HTTPS/WAPI, imports hosts and A/AAAA records that carry extensible attributes, and publishes them as `orchestrator_*` labels into the inventory database — where they drive scopes and enforcement policies.*

**Data flow:**

1. The **external orchestrator** (type `infoblox`) in CSW opens an **HTTPS** connection to an Infoblox Grid member's **WAPI** endpoint.
2. CSW authenticates with an Infoblox account that has **REST API read** privileges.
3. On each **full snapshot** interval, CSW pulls subnets, hosts, and A/AAAA records **that have extensible attributes**.
4. Extensible attributes are converted to CSW labels with the `orchestrator_` prefix and merged onto the matching inventory IPs.
5. Those labels are now available for **inventory search → scopes → policies**.

> **Connectivity:** The orchestrator connects from the CSW cluster appliance servers (on-prem), from the cloud (SaaS/TaaS), or from the **Secure Connector** VM. If Infoblox is not directly reachable from CSW — which is the typical case for **SaaS** — you must set up a **Secure Connector tunnel** (see §4).

---

## 3. Prerequisites

### On the Infoblox side

- An Infoblox appliance/Grid with **IPAM and/or DNS** enabled.
- **WAPI (REST API) enabled** and reachable over **HTTPS (TCP/443)**.
- Supported WAPI versions per the [CSW Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html): **2.6, 2.6.1, 2.7, 2.7.1** (2.7.1 recommended).
- A **dedicated service account** with **read** privileges to send REST API requests (see §3.1 — least privilege).
- **Extensible attributes** defined and attached to the hosts/records/subnets you want to label (e.g. `Department`, `Environment`, `Owner`).

### On the Secure Workload side

- Secure Workload **4.x** (on-prem or SaaS). Infoblox is configured under **Manage → External Orchestrators**.
- A user with **Site Admin / Root Scope Owner** rights (or an API key with the `external_integration` capability if using the API — see §7).
- For **SaaS / non-routable Infoblox**: a **Secure Connector** tunnel deployed and healthy (see §4).

### Network / firewall

- Allow **HTTPS (TCP/443)** from the CSW source (cluster appliances, SaaS egress, or Secure Connector VM) **→ Infoblox Grid member(s)**.

### 3.1 Create a least-privilege Infoblox service account (recommended)

Do **not** reuse `admin`. Create a dedicated, read-only API account:

1. In Infoblox Grid Manager: **Administration → Administrators → Admins → Add**.
2. Create a group (e.g. `csw-readonly`) with an **admin role** granting **read-only** access to Networks, Hosts, and DNS records, plus **API access**.
3. Create a user (e.g. `svc-csw`) in that group with a strong, unique password.
4. Verify the account can reach WAPI (see §5, the reachability test).

> **Security:** Store this credential in your secrets manager. Never commit it to source control. CSW stores it encrypted; you must **re-enter the password every time you edit** the orchestrator config.

---

## 4. (SaaS only) Deploy the Secure Connector tunnel

Skip this section if your CSW cluster is **on-prem** and can reach Infoblox directly.

For **Secure Workload SaaS/TaaS**, or when Infoblox sits in a network the CSW cluster cannot route to, the orchestrator's HTTPS traffic must be tunneled through the **Secure Connector**:

1. In CSW: **Manage → Secure Connector** → follow the wizard to download and register the Secure Connector client on a Linux VM that **can** reach Infoblox.
2. Deploy the VM per the [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) sizing (2 vCPU / 4 GB RAM; RHEL/Rocky/Alma supported).
3. Confirm the Secure Connector shows **Active/Connected** in the CSW UI.
4. When you create the orchestrator (§5), check **Secure Connector Tunnel**.

---

## 5. Configure the Infoblox external orchestrator (UI)

**Path:** `Manage → External Orchestrators → Create New Configuration`

### Step 1 — Basic tab

Select **Type = Infoblox**, then fill the common fields:

| Field | Required | Value / guidance |
|---|---|---|
| **Name** | ✅ | Unique per tenant, e.g. `infoblox-grid-prod` |
| **Type** | ✅ | `Infoblox` (not editable after creation) |
| **Description** | ❌ | e.g. "Prod Infoblox Grid — IPAM/DNS label enrichment" |
| **Username** | ✅ | The Infoblox service account (e.g. `svc-csw`) |
| **Password** | ✅ | The service account password (re-enter on every edit) |
| **Full Snapshot Interval (s)** | ✅ | How often to re-poll Infoblox. Start with `3600` (1 hour). Large grids can take up to an hour for the first snapshot |
| **Accept Self-signed Cert** | ❌ | Check only if the Infoblox WAPI presents a self-signed cert (lab). Prefer a CA-signed cert in production |
| **Secure Connector Tunnel** | ❌ | Check for **SaaS / non-routable** Infoblox (see §4) |

> **Note:** `Delta interval` and `Verbose TSDB metrics` are **not applicable** to Infoblox — a full snapshot poll is used.

### Step 2 — Hosts List tab

Click **Hosts List** and add one row per Infoblox Grid member that exposes WAPI:

| Column | Value |
|---|---|
| **Host Name / IP** | Infoblox Grid member DNS name, IPv4, or IPv6 |
| **Port** | `443` |

- Add **multiple members of the same Grid** for **high availability** — the orchestrator fails over to the next entry on connection errors.
- To import from a **different Grid**, create a **separate orchestrator**.

> **Note:** IPv4 and IPv6 (dual-stack) are supported; dual-stack is a **BETA** feature.

### Step 3 — Record type selectors (Infoblox Config)

Choose which record types to import. All are **enabled by default**; deselect any you don't need to reduce volume:

| Selector | Default | Imports |
|---|---|---|
| Network records | On | Subnets/networks (for attribute inheritance) |
| Host records | On | `record:host` objects |
| A records | On | IPv4 A records |
| AAAA records | On | IPv6 AAAA records |

### Step 4 — (Optional) Alerts tab

- Check **Alert enabled** to be notified if the orchestrator disconnects.
- Set **Severity** (LOW → IMMEDIATE ACTION) and **Disconnect Duration (m)**.
- Also enable **Connector Alerts** under **Manage → Workloads → Alert Configs**.

### Step 5 — Create

Click **Create**. Because the first snapshot is asynchronous, allow **~1 minute** for the connection status to update, and **up to 1 hour** for a large first full snapshot to appear in inventory.

---

## 6. Verify the import

### 6.1 Check connection status

Open the orchestrator row: the connection status should be healthy. If `authentication_failure = true`, read `authentication_failure_error` for the reason (bad credentials, unreachable host, cert error).

### 6.2 Confirm system labels

Every imported object gets these system labels:

| Key | Value |
|---|---|
| `orchestrator_system/orch_type` | `infoblox` |
| `orchestrator_system/cluster_id` | *(CSW cluster id)* |
| `orchestrator_system/machine_id` | *(source machine id)* |
| `orchestrator_system/machine_name` | *(source machine name)* |

### 6.3 Confirm extensible-attribute labels

Every Infoblox **extensible attribute** is imported as `orchestrator_<AttributeName>`.

**Path:** `Investigate → Inventory Search` (a.k.a. Inventory / Search)

Example queries:

```
orchestrator_system/orch_type = infoblox
```

```
orchestrator_Department = Finance
```

```
orchestrator_Environment = Production and orchestrator_system/orch_type = infoblox
```

> A host with an Infoblox extensible attribute `Department = Finance` is found in CSW as `orchestrator_Department = Finance`. **Always add the `orchestrator_` prefix.**

### 6.4 Use the labels in scopes & policies

1. **Manage → Scopes and Inventory** → create/edit a scope using a query like `orchestrator_Environment = Production`.
2. Build workspace policies against those scopes.
3. Treat `orchestrator_*` labels like any other CSW label in policy discovery and enforcement.

---

## 7. (Alternative) Configure via the OpenAPI

Prefer automation? Use the Secure Workload **OpenAPI**. The API key needs the `external_integration` capability.

`POST /openapi/v1/orchestrator/{rootScopeName}`

**Example payload (Python, using the CSW `restclient`):**

```python
import json

req_payload = {
    "name": "infoblox-grid-prod",
    "type": "infoblox",
    "description": "Prod Infoblox Grid - IPAM/DNS label enrichment",
    "hosts_list": [
        {"host_name": "infoblox-gm1.corp.example.com", "port_number": 443},
        {"host_name": "infoblox-gm2.corp.example.com", "port_number": 443}
    ],
    "username": "svc-csw",
    # Pull the secret from env / a secrets manager - never hardcode credentials
    "password": os.environ["INFOBLOX_CSW_PASSWORD"],
    "full_snapshot_interval": 3600,
    "insecure": False,                 # True only to accept self-signed certs (lab)
    "use_secureconnector_tunnel": False,  # True for SaaS / non-routable Infoblox
    "infoblox_config": {
        "enable_network_record": True,
        "enable_host_record": True,
        "enable_a_record": True,
        "enable_aaaa_record": True
    }
}

resp = restclient.post(
    "/orchestrator/Default",
    json_body=json.dumps(req_payload)
)
print(resp.status_code, resp.text)
```

> **Security note (applied per repo policy):** the sample reads the password from an environment variable (`os.environ`) rather than embedding it. Never commit credentials, WAPI passwords, or API keys to source control; store them in a secrets manager and inject at runtime.

Useful endpoints:

| Action | Method + path |
|---|---|
| List orchestrators | `GET /openapi/v1/orchestrator/{scope}` |
| Create orchestrator | `POST /openapi/v1/orchestrator/{scope}` |
| Get one orchestrator | `GET /openapi/v1/orchestrator/{scope}/{id}` |
| Update orchestrator | `PUT /openapi/v1/orchestrator/{scope}/{id}` |
| Delete orchestrator | `DELETE /openapi/v1/orchestrator/{scope}/{id}` |

`infoblox_config` fields: `enable_network_record`, `enable_host_record`, `enable_a_record`, `enable_aaaa_record` (all default `True`).

---

## 8. Caveats & limits

| Limit | Value |
|---|---|
| Max **subnets** imported | **50,000** |
| Max **hosts + A/AAAA records** imported (combined) | **400,000** |
| Objects **without** extensible attributes | **Not imported** |
| Delta polling | **Not supported** for Infoblox — full snapshot only |
| Subnets in inventory search | **Not searchable** (CSW inventory is IP-based); subnet extensible attributes are only searchable via hosts *if marked inherited* in Infoblox |
| Dual-stack (IPv4 + IPv6) | Supported (**BETA**) |
| First full snapshot | May take **up to 1 hour** on large grids |

---

## 9. Troubleshooting

| Symptom | Likely cause & fix |
|---|---|
| **Connection / auth failure** | Firewall not permitting HTTPS from the CSW source (cluster / SaaS egress / Secure Connector VM) to Infoblox; wrong credentials; account lacks REST API privilege. Check `authentication_failure_error`. |
| **Certificate error** | WAPI presents a self-signed cert. Install a CA-signed cert (preferred) or check **Accept Self-signed Cert** / set `insecure: True` (lab only). |
| **Not all expected objects imported** | Only objects **with extensible attributes** are imported, and per-type limits apply (§8). Attach extensible attributes in Infoblox to the objects you need. |
| **Can't find a subnet in inventory** | By design — CSW inventory holds IP addresses (hosts / A/AAAA), not subnets. |
| **Can't find a host / A record** | Remember the **`orchestrator_` prefix** in search. Subnet attributes not marked **inherited** in Infoblox won't appear on hosts. |
| **Labels stale after Infoblox change** | CSW refreshes on the **full snapshot interval**. Lower the interval or wait for the next poll. |
| **SaaS can't reach Infoblox** | Deploy / repair the **Secure Connector** tunnel (§4) and enable it on the orchestrator. |

---

## 10. Video walkthrough

The official Cisco Secure Workload channel demonstrates this integration end to end:

▶ **[Watch: Cisco Secure Workload — Infoblox Integration](https://www.youtube.com/watch?v=gdhMWviAZig)**

| Stage | What to watch for |
|---|---|
| Overview | Why Infoblox extensible attributes make great CSW labels |
| External orchestrator setup | Type = Infoblox, hosts list, credentials, snapshot interval |
| Record type selection | Choosing networks / hosts / A / AAAA |
| Verification | Inventory search with `orchestrator_*` labels |
| Using labels | Building scopes and policies from imported attributes |

---

## 11. References

- [Cisco Secure Workload — External Orchestrators (On-Prem 4.0)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-external-orchestrators-in-secure-workload.html)
- [Cisco Secure Workload — External Orchestrators (SaaS 4.0)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40/m-external-orchestrators.html)
- [Cisco Secure Workload — OpenAPI (Orchestrators)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
- [Cisco Secure Workload Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)
- [Infoblox WAPI documentation](https://www.infoblox.com/) (see your Grid's `/wapidoc` endpoint for version-specific schema)
