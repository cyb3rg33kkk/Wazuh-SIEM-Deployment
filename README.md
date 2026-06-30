[Wazuh_Agent_Enrollment_SOP.md](https://github.com/user-attachments/files/29499347/Wazuh_Agent_Enrollment_SOP.md)
# Wazuh Agent Enrollment — Standard Operating Procedure (SOP)

**Document Owner:** Ogbolu Samuel Precious
**Role:** Cybersecurity Engineer / SOC Analyst
**Scope:** Windows and Linux endpoint enrollment to a single-server Wazuh Manager
**Version:** 1.0
**Last Updated:** 2026

---

## 1. Purpose

This SOP documents the standard procedure for enrolling Windows and Linux endpoints as Wazuh agents against an existing Wazuh Manager. It assumes the Manager, Indexer, and Dashboard are already deployed and operational (see companion document: `README.md` — Wazuh SIEM Deployment SOP).

---

## 2. Scope

This document covers:

- Pre-enrollment prerequisites (manager-side and endpoint-side)
- Agent installation and enrollment on Windows endpoints
- Agent installation and enrollment on Linux endpoints (Debian/Ubuntu and RHEL/CentOS families)
- Post-enrollment verification
- Common enrollment failure scenarios and troubleshooting
- Agent lifecycle management basics (restart, remove, re-enroll)

This document does **not** cover detection rule/decoder development, log source configuration beyond default agent telemetry, or fleet automation/deployment-at-scale tooling (e.g., GPO, Ansible) — those are addressed in separate SOPs.

---

## 3. Architecture Context

```
        ┌─────────────────────┐
        │   Wazuh Manager      │
        │   (1514 / 1515)       │
        └──────────┬──────────┘
                    │
        ┌───────────┼───────────┐
        │                       │
┌───────▼────────┐     ┌────────▼────────┐
│  Windows Agent   │     │   Linux Agent    │
│  (ossec-agent)   │     │   (wazuh-agent)  │
└─────────────────┘     └─────────────────┘
```

- **Port 1514/TCP** — Agent-to-manager event communication
- **Port 1515/TCP** — Agent enrollment (registration) — can be disabled after enrollment if static keys are used instead

---

## 4. Pre-Enrollment Prerequisites

### 4.1 Manager-Side Checklist

- [ ] Wazuh Manager service is running and reachable (`systemctl status wazuh-manager`)
- [ ] Manager IP/hostname is resolvable from the endpoint network
- [ ] Firewall allows inbound 1514 and 1515 from the target endpoint subnet(s)
- [ ] Decision made: **auto-enrollment** (agent registers itself using `agent-auth`) vs **manual key generation** (analyst pre-generates keys via `manage_agents` or the Dashboard)

### 4.2 Endpoint-Side Checklist

- [ ] Endpoint has outbound network access to the Manager on 1514/1515
- [ ] Administrative/root privileges available for install
- [ ] Endpoint hostname is unique and meaningful (avoid generic names like `DESKTOP-1` in shared environments — naming convention should support SOC triage)
- [ ] Time sync (NTP) confirmed on endpoint to avoid event timestamp drift

### 4.3 Naming Convention Recommendation

Adopt a consistent agent naming standard before mass enrollment, e.g.:

```
<site>-<ostype>-<hostname>
LAG-WIN-FIN01
LAG-LNX-WEB02
```

This avoids ambiguity later when triaging alerts across multiple agents.

---

## 5. Windows Agent Enrollment

### 5.1 Step 1 — Download the Agent Package

On the Windows endpoint (PowerShell, run as Administrator):

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi -OutFile $env:tmp\wazuh-agent.msi
```

> Replace `4.x.x` with the version matching your Manager's version. Agent and Manager major.minor versions should align.

### 5.2 Step 2 — Install with Enrollment Parameters

```powershell
msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER='<manager-ip-or-hostname>' `
  WAZUH_AGENT_NAME='LAG-WIN-FIN01' `
  WAZUH_REGISTRATION_SERVER='<manager-ip-or-hostname>'
```

This installs the agent and performs auto-enrollment against the manager in a single step.

### 5.3 Step 3 — Start the Service

```powershell
NET START WazuhSvc
```

### 5.4 Step 4 — Verify Local Agent Status

```powershell
Get-Service -Name WazuhSvc
```

Expected status: `Running`.

---

## 6. Linux Agent Enrollment

### 6.1 Debian/Ubuntu Family

**Step 1 — Add the Wazuh repository and import the GPG key:**

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee -a /etc/apt/sources.list.d/wazuh.list
apt-get update
```

**Step 2 — Install with enrollment variables set inline:**

```bash
WAZUH_MANAGER='<manager-ip-or-hostname>' \
WAZUH_AGENT_NAME='LAG-LNX-WEB02' \
apt-get install wazuh-agent -y
```

**Step 3 — Enable and start the service:**

```bash
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### 6.2 RHEL/CentOS Family

**Step 1 — Import the GPG key and add the repo:**

```bash
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

cat > /etc/yum.repos.d/wazuh.repo << EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF
```

**Step 2 — Install with enrollment variables set inline:**

```bash
WAZUH_MANAGER='<manager-ip-or-hostname>' \
WAZUH_AGENT_NAME='LAG-LNX-WEB02' \
yum install wazuh-agent -y
```

**Step 3 — Enable and start the service:**

```bash
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### 6.3 Manual Enrollment Alternative (Both Distros)

If not enrolling at install time, manually run `agent-auth` against the manager and configure `/var/ossec/etc/ossec.conf` with the `<server>` block pointing to the manager IP, then restart the service.

---

## 7. Post-Enrollment Verification

### 7.1 Manager-Side Verification

On the Wazuh Manager:

```bash
sudo /var/ossec/bin/agent_control -l
```

This lists all enrolled agents and their connection status (`Active`, `Disconnected`, `Never connected`).

Alternatively, check via the Manager API:

```bash
curl -k -u wazuh-wui:<password> "https://localhost:55000/agents?status=active" 
```

### 7.2 Dashboard Verification

- Navigate to **Agents** in the Wazuh Dashboard
- Confirm the new agent appears with status **Active**
- Confirm telemetry is flowing by checking the **Security Events** view filtered to the new agent

### 7.3 Verification Checklist

- [ ] Agent shows `Active` in `agent_control -l` output
- [ ] Agent visible and `Active` in the Dashboard Agents view
- [ ] Events from the agent visible in Security Events within a few minutes of enrollment
- [ ] Agent hostname matches the naming convention defined in Section 4.3

---

## 8. Common Enrollment Failures & Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Agent stuck at `Never connected` | Firewall blocking 1514/1515, or wrong manager IP | Verify connectivity with `telnet <manager-ip> 1514`; check firewall rules on both ends |
| Agent shows `Disconnected` after initial connect | Time drift between agent and manager | Sync NTP on both hosts |
| Enrollment fails with key mismatch | Agent was previously enrolled and re-installed without removing old keys | On manager, remove the stale agent entry (`manage_agents -r <id>`), then re-enroll |
| Windows service won't start | MSI install ran without admin rights, or `ossec.conf` malformed | Re-run install as Administrator; validate XML syntax in `ossec.conf` |
| Linux service fails silently | SELinux/AppArmor blocking agent process | Check `audit.log`/`dmesg`; adjust policy or add exception |
| Duplicate agent name rejected | Name collision with existing enrolled agent | Confirm naming convention compliance before install; remove orphaned entries on manager |

---

## 9. Agent Lifecycle Management Basics

### 9.1 Restart an Agent

```bash
# Linux
systemctl restart wazuh-agent

# Windows (PowerShell, admin)
Restart-Service -Name WazuhSvc
```

### 9.2 Remove an Agent (Manager-Side)

```bash
sudo /var/ossec/bin/manage_agents -r <agent-id>
```

Always remove decommissioned agents from the manager to keep the Dashboard's agent inventory accurate and avoid false "disconnected" alerts.

### 9.3 Re-Enroll an Agent

If an endpoint is rebuilt or its keys are invalidated, remove the stale entry on the manager first (Section 9.2), then repeat the relevant installation/enrollment steps from Section 5 or 6.

---

## 10. References

- Official Wazuh Agent Installation Guide: https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html
- Official Wazuh Agent Enrollment Documentation: https://documentation.wazuh.com/current/user-manual/agent-enrollment/index.html

---

## Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026 | Ogbolu Samuel Precious | Initial SOP creation — Windows and Linux agent enrollment |
