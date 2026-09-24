# Mini-SOAR on AWS — Setup Guide

This document walks through building a Mini Security Orchestration, Automation and Response (SOAR) pipeline on AWS, from a blank account to a working detect → enrich → case → approve → respond loop.

**Stack:** Wazuh (detection) · VirusTotal (enrichment) · TheHive (case management) · n8n (automation/orchestration)

![Architecture Diagram](screenshots/architecture-diagram.jpg)

---

## 1. AWS Account Setup

Created a new AWS account on the Free Plan ($100 signup credit, up to $200 over 6 months).

![AWS Account Credits](screenshots/aws-account.png)

Region set to a location close to me (not the AWS default) for lower latency.

**Note:** the Free Plan restricts EC2 instance types to a small set (t3.micro, t3.small, t4g.micro, t4g.small, c7i-flex.large, m7i-flex.large). Bigger instances need the Paid Plan.

---

## 2. Launching the Servers

Two EC2 instances:

| Server | Instance Type | Specs | Role |
|---|---|---|---|
| SOC-Automation | m7i-flex.large | 2 vCPU, 8GB RAM | Wazuh + TheHive + n8n |
| Windows-Target | c7i-flex.large | 2 vCPU, 4GB RAM | Attack target (Windows Server 2022) |

- OS: Ubuntu 24.04 LTS (SOC-Automation), Windows Server 2022 Base (Windows-Target)
- Storage: 60GB on SOC-Automation (resized up after running out of space mid-build)
- Key pairs: RSA + .pem for both

![EC2 Instances Running](screenshots/ec2-instances.png)

Security group inbound rules opened across the setup (SSH, HTTPS, custom ports for TheHive/n8n/Wazuh API):

![Security Groups](screenshots/security-groups.png)

**Connecting:**
- SOC-Automation → EC2 Instance Connect (browser SSH)
- Windows-Target → AWS Fleet Manager Remote Desktop (browser RDP) — the local Windows Remote Desktop app wasn't available on my host machine, so this was the fallback that worked

---

## 3. Installing Wazuh

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
cd wazuh-docker/single-node
sudo sysctl -w vm.max_map_count=262144
docker compose -f generate-indexer-certs.yml run --rm generator
sudo usermod -aG docker $USER
docker compose up -d
```

Opened ports in the EC2 security group: **443** (dashboard), **1514/1515** (agent traffic), **55000** (API).

![Wazuh Dashboard](screenshots/wazuh-dashboard.png)

---

## 4. Installing the Wazuh Agent + Sysmon on Windows-Target

Agent installed via the Wazuh dashboard's "Deploy new agent" wizard (generates the exact PowerShell install command, enrolled against SOC-Automation's private IP).

Sysmon installed using the SwiftOnSecurity config for solid default detection rules:

```powershell
mkdir C:\Sysmon
cd C:\Sysmon
# Download Sysmon + sysmonconfig-export.xml
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

![Sysmon Install](screenshots/sysmon-install.png)

Added a group config in Wazuh so it reads Sysmon's Windows event channel:

```xml
<agent_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

![Wazuh Agent Active](screenshots/wazuh-agent-active.png)

---

## 5. Installing TheHive (+ Cortex)

Used StrangeBee's official `prod1-thehive` Docker profile (not the "testing" folder) with its `init.sh` script, which auto-generates certs, passwords, and `.env`:

```bash
git clone https://github.com/StrangeBeeCorp/docker.git thehive-docker
cd thehive-docker/prod1-thehive
bash ./scripts/init.sh
docker compose up -d
```

**RAM/CPU tuning** was required since the default profile assumes far more resources than our 8GB/2-vCPU box:
- Lowered Cassandra/Elasticsearch/TheHive heap sizes
- Lowered per-service CPU limits (defaults exceeded the 2 vCPUs available)
- Moved TheHive's Nginx to port **8443** (443 was already used by Wazuh)

Fixed an SSL config issue where TheHive's Elasticsearch connector defaulted to HTTPS while Elasticsearch itself ran plain HTTP:
```
elasticsearch.ssl.enabled = false
```
(added to `thehive/config/index.conf`)

![TheHive Dashboard](screenshots/thehive-login.png)

**Important:** TheHive's default "admin" organisation cannot manage cases (platform-admin only). Created a separate **"SOC"** organisation with a Service-type user (`n8n-automation`) holding `manageCase/create`, `manageCase/update`, `manageAlert/create` permissions, and generated its API key for n8n to use.

---

## 6. Installing n8n

```bash
docker run -d --name n8n --restart unless-stopped -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -e WEBHOOK_URL=http://<public-ip>:5678/ \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Chose n8n over Shuffle for lighter RAM usage (Shuffle's internal OpenSearch database needs 1GB+ just to run).

**Note:** `WEBHOOK_URL` must be updated (and the container recreated) every time the EC2 instance restarts and gets a new public IP — otherwise email approval links break.

---

## 7. Building the n8n Workflow

Full node chain:

**Receive Wazuh Alert** (Webhook, POST) → **VirusTotal - Check IP Reputation** → **TheHive - Create Case** → **Email - Request Approval** → **Check Approval Decision** (IF) →
- **true:** **Wazuh - Authenticate** → **Wazuh - Block IP** → **TheHive - Add Comment (Approved)**
- **false:** **TheHive - Add Comment (Declined)**

![Full n8n Workflow](screenshots/n8n-workflow-full.png)

### 7a. Wazuh → n8n integration

Added to Wazuh's `ossec.conf` (Settings → Edit configuration in the dashboard):
```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://<n8n-ip>:5678/webhook/<webhook-id></hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```
Used Wazuh's built-in `shuffle` integration type, since it's just a standard JSON webhook POST — works with n8n too.

![Wazuh Integration Config](screenshots/wazuh-integration-config.png)

### 7b. VirusTotal enrichment

GET request to VirusTotal's IP reputation endpoint, using an `x-apikey` header. Result fed into the case description.

### 7c. TheHive case creation

POST to `/api/v1/case` with a Header Auth (`Bearer <API key>`) credential from the `n8n-automation` user.

### 7d. Email approval

Used n8n's built-in **"Send message and wait for response"** email operation (Approval type, double-button Approve/Disapprove) — Gmail SMTP with an App Password. This pauses the workflow until the person clicks a button in the email.

### 7e. Wazuh active response (IP block)

Two-step: authenticate against Wazuh's REST API to get a JWT token, then PUT to `/active-response` using Wazuh's built-in `win_route-null` Windows blocking command, sourcing the attacker's IP from the original alert data.

### 7f. Case comment (closing the loop)

Final step posts a comment back to the TheHive case confirming what action was taken (or that it was declined) — so the case record is the full, self-contained proof of the incident lifecycle.

---

## 8. Testing

Triggered controlled failed logins on Windows-Target:
```powershell
net use \\localhost\ipc$ /user:Administrator WrongPassword123!
```

![Approval Email](screenshots/approval-email.png)
![Case Comment](screenshots/case-auto-comment.png)
![Case Closed](screenshots/case-closed-summary.png)

---

## Known Limitations

- Test alerts were generated locally (RDP/PowerShell into Windows-Target itself), so the "attacker" IP shows as loopback (`::1`) rather than a real external address. The extraction logic is confirmed correct — a real external attack would populate a real IP.
- File-based Sysmon alerts (e.g. "Executable dropped in folder") carry no network/IP data — this is expected, not a bug.
