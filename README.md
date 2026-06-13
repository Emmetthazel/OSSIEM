# 🛡️ OSSIEM — Open Source SIEM & Identity Management for Cloud Security

<div align="center">

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.9.0-00A1F1?style=for-the-badge&logo=wazuh&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-IAM-4D4D4D?style=for-the-badge&logo=keycloak&logoColor=white)
![Graylog](https://img.shields.io/badge/Graylog-6.0.6-FF3633?style=for-the-badge&logo=graylog&logoColor=white)
![Velociraptor](https://img.shields.io/badge/Velociraptor-DFIR-1E8449?style=for-the-badge)
![Trivy](https://img.shields.io/badge/Trivy-Aquasec-1904DA?style=for-the-badge)

**A fully containerized, open-source SIEM platform integrating threat detection, identity management, digital forensics, and vulnerability analysis — built and validated as an academic final-year project (PFA) at EMSI Rabat.**

*by BOULAHBACH Abdelkhalek & BOULAHBACH Zineb — Cybersecurity Engineering, EMSI Rabat 2026*

</div>

---

## 📖 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Stack Components](#stack-components)
- [Pre-Deployment](#pre-deployment)
- [Deployment](#deployment)
- [Post-Deployment Configuration](#post-deployment-configuration)
  - [Keycloak — Identity & Access Management](#keycloak--identity--access-management)
  - [Wazuh — SIEM & EDR](#wazuh--siem--edr)
  - [Graylog — Log Management](#graylog--log-management)
  - [Velociraptor — DFIR](#velociraptor--dfir)
  - [Trivy — Vulnerability Scanning](#trivy--vulnerability-scanning)
- [Operational Validation — Attack Simulation](#operational-validation--attack-simulation)
- [Results](#results)
- [Credits](#credits)

---

## Overview

OSSIEM is a complete, production-inspired security operations platform deployed as **16 Docker containers** via a single `docker compose up -d` command. It was built to address the dual challenge of **cloud-native threat detection** and **centralized identity governance** in containerized environments.

The platform was validated through a real attack simulation: an aggressive Nmap reconnaissance scan launched from Kali Linux against the infrastructure, with real-time detection and log centralization in Graylog.

**Key capabilities:**

- 🔐 **Identity & Access Management** — Keycloak with a dedicated `OSSIEM-Cloud` realm, RBAC roles for SOC operators, OAuth2/OIDC authentication
- 🛡️ **SIEM / EDR** — Wazuh 4.9 with OpenSearch indexing, MITRE ATT&CK mapping, PCI-DSS / GDPR / HIPAA / NIST 800-53 compliance tagging
- 📊 **Log Management & Threat Hunting** — Graylog 6.0 with Fluent Bit ingestion, full-text search, stream-based alerting
- 🦖 **Digital Forensics (DFIR)** — Velociraptor with VQL queries and artifact collection
- 🔍 **Vulnerability Analysis** — Trivy/Aquasec container image scanning against the NVD
- ⚙️ **Supervised Application Layer** — CoPilot (SOC dashboard) integrating all components through verified connectors

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              Docker Host — 192.168.40.128                       │
│                                                                 │
│  ┌──────────────┐  ┌──────────────────────────────────────┐    │
│  │  IAM Layer   │  │         SIEM / Detection Layer        │    │
│  │  Keycloak    │  │  Wazuh Manager  │  Wazuh Indexer      │    │
│  │  :8080       │  │  :1514/:1515    │  :9200 (OpenSearch)  │    │
│  │  OSSIEM-Cloud│  │  :55000 (API)   │  Wazuh Dashboard     │    │
│  └──────────────┘  └──────────────────────────────────────┘    │
│                                                                 │
│  ┌──────────────┐  ┌──────────────────────────────────────┐    │
│  │  Log Mgmt    │  │           DFIR Layer                  │    │
│  │  Graylog     │  │  Velociraptor   │  MinIO               │    │
│  │  :9000 (UI)  │  │  :8000-8001     │  :9000 (artifacts)   │    │
│  │  :514 (Sysl) │  │  :8889 (API)    │                      │    │
│  │  :12201 (GELF│  └──────────────────────────────────────┘    │
│  └──────────────┘                                               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              CoPilot Application Layer (Supervised)      │   │
│  │  Frontend :80/:443 │ Backend :5000 │ MySQL :3306         │   │
│  │  MongoDB :27017    │ Grafana :3000 │ Nuclei │ MCP        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Networks: ossiem_default  │  ossiem_ossiem_default             │
└─────────────────────────────────────────────────────────────────┘
          ▲
          │  nmap -A -p 1514,1515,55000,8080
          │
   [Kali Linux — 192.168.40.1]  ←→  [SOC Analyst Browser]
```

---

## Stack Components

| Container | Image | Role | Ports |
|---|---|---|---|
| `copilot-keycloak` | `quay.io/keycloak/keycloak:latest` | IAM — Authentication & RBAC | 8080, 8443 |
| `wazuh.manager` | `ghcr.io/socfortress/wazuh-manager:4.9.0` | SIEM — Correlation & Detection | 1514, 1515, 55000 |
| `wazuh.indexer` | `wazuh/wazuh-indexer:4.9.0` | OpenSearch — Indexing | 9200 |
| `wazuh.dashboard` | `wazuh/wazuh-dashboard:4.9.0` | SOC Dashboard (Wazuh) | 443 |
| `graylog` | `graylog/graylog:6.0.6` | Log Management & Threat Hunting | 9000, 514, 12201 |
| `velociraptor` | `wlambert/velociraptor` | DFIR — Forensics & VQL | 8000–8001, 8889 |
| `copilot-frontend` | `ghcr.io/socfortress/copilot-frontend:latest` | SOC UI | 80, 443 |
| `copilot-backend` | `ghcr.io/socfortress/copilot-backend:latest` | SOC API | 5000 |
| `copilot-mysql` | `mysql:8.0.38-debian` | Relational DB | 3306 |
| `mongodb` | `mongo:6.0.14` | Document DB (Graylog + CoPilot) | 27017 |
| `copilot-minio` | `ghcr.io/socfortress/minio:release.2024-09-13` | Object Storage — Artifacts | 9000 |
| `grafana` | `grafana/grafana-enterprise` | Metrics Dashboards | 3000 |
| `copilot-nuclei-module` | `ghcr.io/socfortress/copilot-nuclei:latest` | Web Vulnerability Scanning | — |
| `copilot-mcp` | `ghcr.io/socfortress/copilot-mcp:latest` | Protocol Module | — |

---

## Pre-Deployment

### 1. System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| OS | Kali Linux / Ubuntu 22.04 | Kali Linux 2024.x |
| Docker Engine | 24.x | Latest |
| Docker Compose | v2.x (plugin) | Latest |
| RAM | 8 GB | 16 GB |
| Storage | 40 GB free | 80 GB SSD |
| CPU | 4 vCPU | 8 vCPU |

### 2. Build the Custom Wazuh Manager Image

```bash
cd wazuh/custom-wazuh-manager
# Follow the build instructions in the subdirectory README
```

### 3. Generate Wazuh SSL Certificates

```bash
# Use the provided cert generation script
docker compose -f wazuh/generate-indexer-certs.yml run --rm generator

# Place certs in the correct directory
# wazuh/config/wazuh_indexer_ssl_certs/

# Copy root CA to Graylog directory (required for Graylog-Wazuh TLS)
cp wazuh/config/wazuh_indexer_ssl_certs/root-ca.pem graylog/
```

### 4. Configure Environment Variables

```bash
# Copy and edit the environment file
cp .env.example .env
nano .env
```

Fill in all required variables: Wazuh credentials, Graylog secrets, Keycloak admin password, MinIO keys.

---

## Deployment

Once pre-deployment steps are complete, start the entire stack with:

```bash
docker compose up -d
```

Expected output — all 16 containers started:

```
✔ Network ossiem_ossiem_default    Created
✔ Network ossiem_default           Created
✔ Container velociraptor           Started
✔ Container copilot-keycloak       Started
✔ Container wazuh.indexer          Started
✔ Container copilot-minio          Started
✔ Container copilot-mysql          Started
✔ Container wazuh.manager          Started
✔ Container copilot-nuclei-module  Started
✔ Container grafana                Started
✔ Container mongodb                Started
✔ Container copilot-frontend       Started
✔ Container wazuh.dashboard        Started
✔ Container graylog                Started
✔ Container copilot-backend        Started
✔ Container copilot-mcp            Started
```

Verify all containers are running:

```bash
docker compose ps
```

All containers should show status `Up`. If any show `Exited`, check logs with `docker logs <container-name>`.

---

## Post-Deployment Configuration

### Keycloak — Identity & Access Management

Access the Keycloak admin console at `http://<HOST_IP>:8080`.

#### Create the OSSIEM-Cloud Realm

1. Log in as admin
2. Click **Create Realm**
3. Set the realm name to `OSSIEM-Cloud`
4. Save

#### Create SOC Roles

Navigate to **Realm Roles** → **Create role**:

| Role Name | Description | Access Level |
|---|---|---|
| `SecOps_Admin` | Full SOC platform access — configure rules, administer tools, trigger DFIR | Administrator |
| `SecOps_Analyst` | Read-only SOC access — view alerts, search logs, consult dashboards | Analyst |

#### Create SOC Users

Navigate to **Users** → **Add user**:

```
Username : soc_admin
First name: <Admin first name>
Role      : SecOps_Admin

Username : soc_analyst
First name: <Analyst first name>
Role      : SecOps_Analyst
```

Set passwords under the **Credentials** tab for each user. Enable **Email Verified** if not using email confirmation.

> ⚠️ **Security note:** In production, enable TOTP (Time-based One-Time Password) MFA for all SOC users under **Authentication** → **Required Actions**.

---

### Wazuh — SIEM & EDR

#### Install SOCFortress Custom Rules

After initial deployment, install the custom detection rules inside the Wazuh Manager container:

```bash
docker exec -it wazuh.manager /bin/bash
```

```bash
dnf install git -y
```

```bash
curl -so ~/wazuh_socfortress_rules.sh \
  https://raw.githubusercontent.com/socfortress/OSSIEM/main/wazuh_socfortress_rules.sh \
  && bash ~/wazuh_socfortress_rules.sh
```

#### Create Required Users and Roles

In the Wazuh Dashboard (`https://<HOST_IP>`), create the internal users needed for the Graylog integration before proceeding to the next step.

#### Verify Cluster Health

After full integration, the Wazuh Dashboard Overview should show:

- **OpenSearch cluster**: `GREEN`
- **Agent**: `1 ONLINE`
- **Active shards**: 11 primary shards
- **Disk**: indexer node operational

---

### Graylog — Log Management

#### Add Wazuh Root CA to Graylog Java Keystore

This step is required for Graylog to connect securely to the Wazuh Indexer over TLS.

```bash
docker exec -it graylog bash
```

```bash
cp /opt/java/openjdk/lib/security/cacerts /usr/share/graylog/data/config/
```

```bash
cd /usr/share/graylog/data/config/

keytool -importcert \
  -keystore cacerts \
  -storepass changeit \
  -alias wazuh_root_ca \
  -file root-ca.pem
```

Type `yes` when prompted to trust the certificate.

#### Configure Fluent Bit Input

In the Graylog UI (`http://<HOST_IP>:9000`):

1. Go to **System** → **Inputs**
2. Select **Raw/Plaintext TCP**
3. Click **Launch new input**
4. Configure:

```
Title           : WAZUH EVENTS FLUENT BIT
Port            : 5555
Bind address    : 0.0.0.0
Number of workers: 8
Max message size : 2097152
```

5. Click **Save** — the input should display `RUNNING`

This input receives Wazuh events forwarded by Fluent Bit from all supervised containers.

---

### Velociraptor — DFIR

#### Generate the API Configuration File

CoPilot uses a Velociraptor API config file to connect and issue VQL queries.

```bash
docker exec -it velociraptor /bin/bash
```

```bash
./velociraptor --config server.config.yaml \
  config api_client \
  --name admin \
  --role administrator,api \
  api.config.yaml
```

Copy the generated `api.config.yaml` to the path expected by CoPilot (defined in your `.env` file).

#### Verify Connectors in CoPilot

Once CoPilot is running, navigate to **Connectors** and verify the following show `Configured` + `Verified`:

- ✅ Wazuh-Indexer
- ✅ Wazuh-Manager
- ✅ Graylog
- ✅ Velociraptor
- ✅ Grafana

---

### Trivy — Vulnerability Scanning

Run a vulnerability scan on any image in the stack:

```bash
# Scan the copilot-frontend image
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image \
  ghcr.io/socfortress/copilot-frontend:latest \
  > trivy_report.txt

# View the summary
head -n 40 trivy_report.txt
```

Expected result structure:

```
Target: ghcr.io/socfortress/copilot-frontend:latest (alpine 3.23.4)
Total: 19 (UNKNOWN: 0, LOW: 4, MEDIUM: 13, HIGH: 2, CRITICAL: 0)
```

> 💡 Integrate Trivy into your CI/CD pipeline to automatically block builds with CRITICAL CVEs.

---

## Operational Validation — Attack Simulation

This project was validated through a Red Team / Blue Team simulation.

### Red Team — Offensive Reconnaissance (Nmap)

From a Kali Linux machine on the same network:

```bash
nmap -A -p 1514,1515,55000,8080 192.168.40.128
```

| Option | Purpose |
|---|---|
| `-A` | Aggressive mode: OS detection, version detection, NSE scripts, traceroute |
| `-p 1514,1515,55000` | Wazuh Manager ports (agent, registration, API) |
| `-p 8080` | Keycloak HTTP port |

**Result:** All 4 ports returned `filtered` — confirming iptables DROP rules are active on the Docker host. The scan completed in 6.11 seconds with no OS fingerprint available due to the filtering.

### Blue Team — Detection & Centralization (Graylog)

While the scan was running, Graylog received and indexed the security events generated by the infrastructure:

```
Query: gl2_source_input:6a25b7a51d140c028a65c993
Result: 2 events  |  Executed in 12ms
Sources: 172.18.0.10 (wazuh.manager), 172.18.0.3 (wazuh agent)
```

Each event carried structured JSON with compliance mappings:

```json
{
  "rule": {
    "level": 3,
    "description": "Wazuh server started.",
    "id": "502"
  },
  "agent": { "name": "wazuh.manager" },
  "compliance": {
    "pci_dss": ["10.6.1"],
    "gdpr": ["IV.35.7.d"],
    "hipaa": ["164.312.b"],
    "nist_800_53": ["AU.6"]
  }
}
```

### Retrieve CoPilot Admin Password (First Boot)

```bash
docker logs "$(docker ps \
  --filter ancestor=ghcr.io/socfortress/copilot-backend:latest \
  --format "{{.ID}}")" 2>&1 | grep "Admin user password"
```

---

## Results

| Metric | Value |
|---|---|
| Containers deployed | 16 / 16 ✅ |
| SOC connectors verified | 5 / 5 ✅ |
| Wazuh cluster status | GREEN ✅ |
| Agents online | 1 ✅ |
| CVEs identified (frontend image) | 19 (0 CRITICAL, 2 HIGH, 13 MEDIUM, 4 LOW) |
| Nmap scan detected by Graylog | ✅ Real-time |
| Compliance frameworks covered | PCI-DSS, GDPR, HIPAA, NIST 800-53 |
| IAM roles defined | SecOps_Admin, SecOps_Analyst |

---

## Credits

Built on top of the excellent work from [SOCFortress](https://github.com/socfortress) and the [Wazuh Team](https://github.com/wazuh).

---

<div align="center">
<b>BOULAHBACH Abdelkhalek & BOULAHBACH Zineb</b><br>
Cybersecurity Engineering — EMSI Rabat, 2026<br>
PFA : Mise en place d'un SIEM pour la sécurité Cloud avec gestion des identités
</div>
