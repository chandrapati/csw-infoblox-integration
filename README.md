# Cisco Secure Workload — Infoblox Integration Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/secure-workload/index.html) for authoritative guidance.

Import **Infoblox IPAM/DNS extensible attributes** into Cisco Secure Workload as **labels** for scopes, inventory filters, and segmentation policy.

---

## Contents

| File | Description |
|------|-------------|
| [`CSW-Infoblox-Integration-Guide.md`](./CSW-Infoblox-Integration-Guide.md) | Full integration guide (Markdown) |
| [`CSW-Infoblox-Integration-Guide.html`](./CSW-Infoblox-Integration-Guide.html) | Standalone HTML (run `./build.sh`) |
| [`docs/CUSTOMER-HANDOFF.md`](./docs/CUSTOMER-HANDOFF.md) | **Share with customer** — Infoblox team checklist & access request |
| [`docs/00-official-references.md`](./docs/00-official-references.md) | Canonical Cisco doc links |
| [`build.sh`](./build.sh) | Rebuild HTML from Markdown |

---

## At a glance

| Topic | Answer |
|-------|--------|
| **Integration type** | External orchestrator (label import only) |
| **Infoblox access** | **Read-only WAPI** — CSW does not write to Infoblox |
| **What gets imported** | Subnets, hosts, A/AAAA records **with extensible attributes** |
| **CSW labels** | `orchestrator_<attribute>=<value>` |
| **Enforcement on Infoblox** | **No** — unlike F5/NetScaler |
| **SaaS / private grid** | Often requires [Secure Connector](https://github.com/chandrapati/csw-secure-connector) |

---

## Quick start

1. Share [`docs/CUSTOMER-HANDOFF.md`](./docs/CUSTOMER-HANDOFF.md) with the customer's **Infoblox / IPAM team**.
2. Confirm WAPI reachability (direct or via Secure Connector).
3. CSW UI: **Manage → Workloads → External Orchestrators → Create** → Type **Infoblox**.
4. Validate **Connection Status: Success** and labels in inventory search.

---

## Related resources

| Repository | Description |
|------------|-------------|
| [csw-infoblox-integration](https://github.com/chandrapati/csw-infoblox-integration) | **This repo** |
| [csw-secure-connector](https://github.com/chandrapati/csw-secure-connector) | Tunnel when CSW cannot reach WAPI directly |
| [CSW-User-Education](https://github.com/chandrapati/CSW-User-Education) | CSW learning path + video library |
| [CSW-External-Orchestrators-Guide](https://github.com/chandrapati/CSW-User-Education/blob/main/docs/user-education/CSW-External-Orchestrators-Guide.md) | All external orchestrator types |
| [csw-servicenow-integration](https://github.com/chandrapati/csw-servicenow-integration) | Often paired with Infoblox for CMDB + IPAM labels |
