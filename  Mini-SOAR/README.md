# 🛡️ Mini-SOAR on AWS

An automated Security Orchestration, Automation and Response (SOAR) pipeline — built from scratch on AWS, broken, debugged, and confirmed working end to end.

## What It Does

Detects suspicious activity on a Windows server, checks the attacker's IP against threat intelligence, creates a documented case, asks a human to approve a response, then automatically blocks the IP and logs the outcome — the same detect → investigate → approve → respond loop used in real Security Operations Centers.

**Flow:** Wazuh detects → VirusTotal enriches → TheHive creates a case → n8n emails an approval request → human approves/declines → Wazuh blocks the IP → TheHive case is auto-updated with the outcome.

![Architecture Diagram](screenshots/architecture-diagram.jpg)

## Stack

| Tool | Role |
|---|---|
| Wazuh | Detection (SIEM + EDR agent, Sysmon on the Windows target) |
| VirusTotal | IP reputation enrichment |
| TheHive | Case management / incident record |
| n8n | Workflow automation & orchestration |
| AWS EC2 | Hosting (2 instances: SOC-Automation, Windows-Target) |

## Status: ✅ Complete

Full pipeline tested and confirmed working end to end, including:
- Real-time detection (failed logins, RDP anomalies, Sysmon events)
- Automated threat intel enrichment
- Human-in-the-loop approval via email
- Automated response action (IP block)
- Full audit trail — every action logged back to the case

## Documentation

- [`SETUP.md`](SETUP.md) — full step-by-step build guide, including every issue hit and how it was fixed
- [`REPORT.md`](REPORT.md) — findings, challenges, and lessons learned

## Key Challenges Solved

- TheHive's RAM/CPU defaults assumed far more resources than a small cloud instance — had to manually tune every service
- Self-signed SSL certificates rejected by both n8n and Wazuh's integration scripts
- AWS assigns a new public IP on every instance restart — had to rebuild environment-variable and hardcoded-URL handling around this
- Wazuh's alert JSON structure varies by alert type — had to trace the exact field path for source IPs directly from raw logs

## Disclaimer

Built and tested entirely in an isolated AWS lab environment for educational purposes. Do not use these techniques against systems you don't own or have explicit permission to test.