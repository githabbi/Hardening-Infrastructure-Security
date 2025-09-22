# Hardening Infrastructure Security via Monitoring & Incident Detection

> Engineering Internship • SUP’COM (University of Carthage)\
> Internship host: DOT’COM • Period: June 23 – August 30, 2025

A practical, end‑to‑end lab that combines monitoring, SIEM, and automation to strengthen infrastructure security. The project deploys **Zabbix** for observability, **Wazuh** for threat detection and vulnerability management, and **Ansible** to automate agent rollout—validated through targeted tests (e.g., SSH brute‑force simulation, EICAR malware sample) and real dashboards.

> 📄 Full report: [\`\`](./final_report.pdf)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [Results (Highlights)](#results-highlights)
- [Reproduce the Lab](#reproduce-the-lab)
- [Project Structure (suggested)](#project-structure-suggested)
- [What I Learned](#what-i-learned)
- [Challenges & Next Steps](#challenges--next-steps)
- [Screenshots / Dashboards](#screenshots--dashboards)
- [Acknowledgments](#acknowledgments)
- [License](#license)

---

## Overview

**Goal:** *Hardening infrastructure security through monitoring and incident detection.*\
This project integrates:

- **Zabbix** → infrastructure & service monitoring (Linux hosts, Nginx, PostgreSQL, Docker, Zabbix server health)
- **Wazuh (SIEM)** → host‑based intrusion detection, vulnerability scanning, file integrity, command auditing, malware detection with VirusTotal enrichment
- **Ansible** → automated Wazuh agent deployment at scale

The environment mirrors a production‑like SaaS stack ("Aurora") using a safe, isolated clone for hands‑on implementation and measurement.

---

## Architecture

**Core nodes** (VMs within the same LAN):

- **Zabbix Server VM** — collects metrics from agents; visual dashboards, alerting
- **Wazuh All‑in‑One VM** — Manager + Indexer + Dashboard on a single node (for lab scale)
- **Ansible Controller VM** — automates agent rollout and configuration
- **Monitored Hosts** — Nginx server, PostgreSQL (in Docker), Docker host, Zabbix server itself

**Agent strategy:**

- Zabbix Agent (v1) for PostgreSQL template coverage
- Zabbix Agent **2** with Docker plugin for container insights
- Wazuh Agent on monitored hosts for SIEM telemetry

---

## Key Features

- **Observability (Zabbix)**

  - System metrics: CPU, memory, disk, network
  - Service metrics: Nginx requests/connections; PostgreSQL connections, transactions, ping, uptime
  - Docker metrics: per‑container CPU/MEM, traffic, images/containers/goroutines
  - Alerting via triggers (e.g., memory > 90% on PG VM)

- **Detection & Response (Wazuh)**

  - Vulnerability detection with CVE correlation (NVD feeds)
  - File Integrity Monitoring (FIM) for sensitive paths (e.g., `/etc`, `/usr/bin`, lab paths like `/home/test`)
  - Authentication attack detection (SSH brute‑force / invalid users / failed logins)
  - Command & privilege auditing via `auditd` on the manager
  - Malware detection with **VirusTotal** hash lookups (validated using the EICAR test file)

- **Automation (Ansible)**

  - Role‑based, idempotent Wazuh agent installation & registration
  - Inventory‑driven scaling for multiple hosts

---

## Results (Highlights)

- **Zabbix dashboards** confirm healthy data collection across hosts and services (server health, system performance, Nginx, PostgreSQL, Docker).
- **Proactive alerting** (e.g., PG VM memory > 90%) demonstrates actionable thresholds.
- **Wazuh vulnerability findings** include critical CVEs on third‑party packages (e.g., `zabbix-agent`), showing patching priorities.
- **SSH brute‑force simulation** triggers Wazuh alerts with attacker IP, target host, MITRE mapping.
- **FIM** detects file create/modify/delete and permission changes in real time—including compliance mappings.
- **EICAR malware test** detected and enriched with VirusTotal results and permalink.
- **Ansible playbooks** reliably deploy agents and bring them online with the manager.

> See `final_report.pdf` for screenshots and detailed analysis of each result.

---

## Configuration (not committed)

Because this repo is public, sensitive configuration is **not** committed. Use the examples below to recreate your own config safely with placeholders or Ansible Vault.

### 1) Ansible inventory — `inventory/hosts`

> Define the target hosts and SSH settings. Replace placeholders with your values.

```ini
[wazuh_agents]
"name_agent" ansible_host="host_ip" ansible_user=ansible

```

### 2) Group variables — `inventory/group_vars/wazuh_agents.yml`

> Central place for variables applied to all hosts in the `wazuh_agents` group. **Do not hardcode secrets**; use environment variables or Ansible Vault.

```yaml
wazuh_manager_ip: "manager_ip"
```

### 3) Tasks (role) — what each file does

- `tasks/repo.yml` — add Wazuh package repository & GPG
- `tasks/install.yml` — install/upgrade the agent package
- `tasks/register.yml` — enroll with the manager via `agent-auth -m {{ wazuh_manager_address }} -P {{ wazuh_registration_password }} -A {{ inventory_hostname }} -G {{ wazuh_agent_group }}`
- `tasks/config.yml` — render `ossec.conf.j2` and drop labels/tags if used
- `tasks/service.yml` — enable + start the agent service
- `tasks/validate.yml` — check in with the manager/API to confirm the agent is active

---

## Reproduce the Lab

> **Disclaimer:** This is a **training environment**. Do not test offensive techniques on systems you don’t own or without permission.

### Prerequisites

- 3–5 Linux VMs (e.g., Ubuntu Server) on the same private network
- Admin access (SSH) and Internet egress for package installs

### High‑Level Steps

1. **Zabbix Server**

   - Install Zabbix Server + Frontend + DB (PostgreSQL chosen in this lab)
   - Add hosts and apply templates: *Linux by Zabbix Agent*, *Nginx by Zabbix Agent*, *PostgreSQL by Zabbix Agent*, *Docker by Zabbix Agent 2*
   - Configure triggers (e.g., memory thresholds)

2. **Zabbix Agents**

   - Deploy Agent (v1) where PG metrics are needed; Agent **2** for Docker plugin
   - Use active/passive mode based on network/firewall constraints

3. **Wazuh (All‑in‑One)**

   - Single‑node install (Manager, Indexer, Dashboard)
   - Register agents; enable modules: Vulnerabilities, FIM, SSH auth checks, Command auditing (`auditd`), Malware; connect VirusTotal

4. **Ansible Automation**

   - Create an Ansible role (e.g., `wazuh_agent`) with tasks for repo, package install, templated `ossec.conf`, registration, service mgmt
   - Maintain `hosts` inventory and `group_vars`; run playbooks to roll out at scale

5. **Validation**

   - Generate benign load (Nginx traffic; PG queries)
   - Trigger safe tests (e.g., invalid SSH attempts from a lab box; EICAR file) to see detections end‑to‑end

> Command‑level details, macros, and configuration snippets are documented in `final_report.pdf`.

---

## Repository Layout

This is the actual layout of the repo you can publish (configs generated locally by readers using the section above):

```
wazuh-agent-automation/
├─ inventory/
│  ├─ group_vars/
│  │  └─ wazuh_agents.yml
│  └─ hosts
├─ playbooks/
│  ├─ deploy_agent.yml
│  └─ deploy-sshkey.yml
├─ roles/
│  └─ wazuh_agent/
│     ├─ defaults/
│     │  └─ main.yml
│     ├─ handlers/
│     │  └─ main.yml
│     ├─ tasks/
│     │  ├─ repo.yml
│     │  ├─ install.yml
│     │  ├─ register.yml
│     │  ├─ config.yml
│     │  ├─ service.yml
│     │  └─ validate.yml
│     ├─ templates/
│     │  └─ ossec.conf.j2
│     └─ vars/
│        └─ main.yml
├─ ansible.cfg
├─ final_report.pdf
└─ README.md
```

**What each top‑level item does**

- `inventory/` — static inventory + group variables for target hosts
- `playbooks/` — entry points for running the automation
- `roles/wazuh_agent/` — reusable role encapsulating install, register, config, validate
- `final_report.pdf` — full internship report with evidence and dashboards
- `README.md` — this document

---

## What I Learned

- **Monitoring Engineering:** building host/service dashboards; defining meaningful triggers; capacity & trend analysis
- **SIEM Operations:** onboarding endpoints; rule tuning; triaging auth anomalies; correlating with MITRE ATT&CK
- **Endpoint Security:** vulnerability management lifecycle; FIM baselining; safe malware testing; VirusTotal enrichment
- **Automation:** writing maintainable Ansible roles and inventories for repeatable deployments
- **Platform Thinking:** designing a small but realistic topology and validating end‑to‑end detections
- **Soft Skills:** scoping work, documenting findings, communicating results with actionable visuals

---

## Challenges & Next Steps

**Challenges**

- Limited VM disk space impacted initial installs
- Late access to the final environment reduced time for large‑scale tests

**Next Steps**

1. Roll out agents across all staging/production hosts for full coverage
2. Add HA for Zabbix and Wazuh; scale indexers/dashboards
3. Introduce SOAR‑style automation for high‑confidence alerts (IP blocklists, isolate host, credential rotation)
4. Enrich with additional threat intel feeds; continue false‑positive reduction and correlation rules
5. Run focused red‑team exercises; iterate on rules and thresholds

---

## Acknowledgments

- **DOT’COM** cybersecurity team for mentorship and environment access
- **Supervisor:** Mr. *Haythem Benna*
- **SUP’COM / University of Carthage** for academic guidance

---

> **Author:** Habib Chelbi\
> **Academic Year:** 2024–2025

