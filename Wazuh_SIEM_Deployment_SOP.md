# Wazuh SIEM Deployment — Standard Operating Procedure (SOP)

**Document Owner:** Ogbolu Samuel Precious
**Role:** Cybersecurity Engineer / SOC Analyst
**Deployment Type:** Single-Server (All-in-One: Manager + Indexer + Dashboard)
**Version:** 1.0
**Last Updated:** 2026

---

## 1. Purpose

This SOP documents the standard procedure for deploying a single-server Wazuh SIEM instance, covering pre-deployment planning, installation, core configuration, service validation, and baseline hardening. It is intended for use as a repeatable reference for lab, demo, or small-environment production deployments.

---

## 2. Scope

This document covers:

- Architecture overview of a single-server Wazuh deployment
- Pre-installation requirements and sizing
- Installation of the Wazuh Indexer, Manager, and Dashboard components
- Core service configuration and certificate setup
- Post-install validation and health checks
- Baseline security hardening
- Backup and maintenance considerations

This document does **not** cover agent deployment to endpoints, custom detection rule/decoder development, or multi-node clustering — these are addressed in separate SOPs.

---

## 3. Architecture Overview

In a single-server deployment, all three Wazuh components run on one host:

| Component | Function | Default Port(s) |
|---|---|---|
| **Wazuh Indexer** | Stores and indexes alert/event data (OpenSearch-based) | 9200 |
| **Wazuh Manager** | Analyzes data, applies rules/decoders, manages agents | 1514 (agent comms), 1515 (enrollment), 55000 (API) |
| **Wazuh Dashboard** | Web UI for visualization and management (OpenSearch Dashboards-based) | 443 |

```
                ┌───────────────────────────────┐
                │         Single Server          │
                │                                 │
   Agents ───►  │  Wazuh Manager (1514/1515)      │
  (future)      │        │                        │
                │        ▼                        │
                │  Wazuh Indexer (9200)            │
                │        │                        │
                │        ▼                        │
                │  Wazuh Dashboard (443)           │
                │                                 │
                └───────────────────────────────┘
                          ▲
                          │ HTTPS
                       Analyst
```

---

## 4. Pre-Deployment Requirements

### 4.1 Minimum System Specifications

| Resource | Minimum (Lab/Small) | Recommended (Production-lite) |
|---|---|---|
| CPU | 4 vCPU | 8 vCPU |
| RAM | 8 GB | 16 GB |
| Storage | 50 GB | 100+ GB (SSD preferred, scales with log retention) |
| OS | Ubuntu 22.04/24.04 LTS, RHEL 8/9, or CentOS Stream | Same |

### 4.2 Network Requirements

- Static IP or resolvable hostname for the server
- Outbound internet access (or local repo mirror) for package installation
- Inbound firewall rules planned for: 443 (Dashboard), 1514/1515 (agents), 55000 (API) — restrict source IPs where possible

### 4.3 Pre-Install Checklist

- [ ] OS installed and fully patched (`apt update && apt upgrade` / `yum update`)
- [ ] Hostname and `/etc/hosts` correctly configured
- [ ] NTP/time sync configured (Chrony or systemd-timesyncd) — critical for log correlation accuracy
- [ ] Sufficient disk allocated under `/var/lib/wazuh-indexer`
- [ ] Root or sudo access confirmed
- [ ] Firewall (ufw/firewalld) status known and documented

---

## 5. Installation Procedure

### 5.1 Method Selection

Wazuh provides an **automated installation script** (recommended for single-server/all-in-one deployments) and a **manual/step-by-step method** (used when granular control over each component is required). This SOP follows the automated script method as the primary path, since it is the standard approach for all-in-one deployments.

### 5.2 Step 1 — Download the Installation Assistant

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
```

> Always verify the script version against the official Wazuh documentation for the release you intend to deploy, since installer URLs are version-pinned.

### 5.3 Step 2 — Run the All-in-One Installation

```bash
sudo bash wazuh-install.sh -a
```

This single command:
- Installs and configures the Wazuh Indexer
- Installs and configures the Wazuh Manager
- Installs and configures the Wazuh Dashboard
- Generates and distributes self-signed certificates between components
- Starts and enables all services

### 5.4 Step 3 — Capture Generated Credentials

The installer outputs admin credentials for the Dashboard and Indexer at the end of execution. **Record these immediately** — they are also stored in:

```bash
sudo tar -xvf wazuh-install-files.tar
```

This archive contains certificates and the `wazuh-passwords.txt` file. Store this archive securely off-host (e.g., encrypted vault, password manager) and remove the plaintext copy from the server once recorded.

### 5.5 Step 4 — Verify Service Status

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should report `active (running)`. If any service fails to start, check logs before proceeding:

```bash
sudo journalctl -u wazuh-manager -n 100 --no-pager
sudo journalctl -u wazuh-indexer -n 100 --no-pager
sudo journalctl -u wazuh-dashboard -n 100 --no-pager
```

---

## 6. Post-Installation Validation

### 6.1 Indexer Health Check

```bash
curl -k -u admin:<password> https://localhost:9200/_cluster/health?pretty
```

Expected `status` field: `green` or `yellow` (yellow is acceptable on a single-node deployment due to lack of replica shards).

### 6.2 Manager API Check

```bash
curl -k -u wazuh-wui:<password> -X GET "https://localhost:55000/" 
```

Confirm a valid JSON response with API version details.

### 6.3 Dashboard Access Check

- Navigate to `https://<server-ip>:443` in a browser
- Log in with the admin credentials captured in Step 5.4
- Confirm the dashboard loads and the **Agents** view shows the manager as reachable (zero agents enrolled is expected at this stage)

### 6.4 Validation Checklist

- [ ] All three services active and enabled on boot (`systemctl is-enabled <service>`)
- [ ] Indexer cluster health returns green/yellow
- [ ] Manager API reachable and authenticated
- [ ] Dashboard accessible via HTTPS and login successful
- [ ] Disk usage baseline recorded for future capacity planning

---

## 7. Baseline Hardening

| Area | Action |
|---|---|
| **Certificates** | Replace self-signed certs with CA-signed or internally-issued certs for production use |
| **Firewall** | Restrict port 443/55000 access to known analyst IP ranges; restrict 1514/1515 to expected agent subnets |
| **Default credentials** | Rotate all default passwords generated during install immediately |
| **API access** | Disable or restrict the Wazuh API to internal network only unless integration requires external access |
| **OS hardening** | Disable unused services, enable host-based firewall, apply CIS baseline where applicable |
| **Logging** | Forward Wazuh's own system logs to a separate audit location if this server is in scope for compliance auditing |
| **Updates** | Establish a patch cadence for both OS and Wazuh components (check release notes before upgrading — indexer/manager/dashboard versions must stay aligned) |

---

## 8. Backup & Maintenance Considerations

- **Configuration backup:** `/var/ossec/etc/` (manager config, rules, decoders) should be backed up on a regular schedule
- **Certificate backup:** Retain the `wazuh-install-files.tar` archive in secure offline storage
- **Indexer data:** Plan a snapshot strategy (OpenSearch snapshot/restore) once retention requirements are defined
- **Capacity monitoring:** Track `/var/lib/wazuh-indexer` growth against expected event volume to project storage scaling needs

---

## 9. Known Limitations of This Deployment Model

- No high availability — a single-server deployment is a single point of failure
- Indexer runs without replica shards by default, so data resilience is limited
- Not horizontally scalable without migrating to a distributed/cluster architecture
- Suitable for lab, demo, proof-of-concept, or small fixed-scope environments — not recommended as-is for production environments with high event throughput or uptime SLAs

---

## 10. References

- Official Wazuh Documentation: https://documentation.wazuh.com
- Wazuh Installation Guide (Quickstart): https://documentation.wazuh.com/current/quickstart.html

---

## Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026 | Ogbolu Samuel Precious | Initial SOP creation — single-server deployment |
