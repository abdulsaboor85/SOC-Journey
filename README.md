<div align="center">

# 🛡️ SOC Journey

### Cybersecurity portfolio: hands on labs built from scratch, broken, debugged, and confirmed working

*BS Computer Science, Final Year*
*Capital University of Science and Technology (CUST), Islamabad*

---

![GitHub last commit](https://img.shields.io/github/last-commit/abdulsaboor85/SOC-Journey?style=for-the-badge&color=navy)
![GitHub repo size](https://img.shields.io/github/repo-size/abdulsaboor85/SOC-Journey?style=for-the-badge&color=navy)
![Focus](https://img.shields.io/badge/Focus-Cybersecurity-teal?style=for-the-badge)

</div>

---

## ⭐ Featured project: Honeypot SOC on AWS

<p align="center">
<a href="./Honeypot_SOC/"><img src="./Honeypot_SOC/diagrams/fig01_overview.png" width="800" alt="Honeypot SOC on AWS: Honeypot, Wazuh, n8n, Email and TheHive"></a>
</p>

I left a fake server open on the internet for one week. **152 connections from 83 IP addresses** found it. Six of them logged in with weak passwords, and one uploaded a file that **46 of 64 security vendors** flag as a miner. My pipeline (Cowrie, Wazuh, n8n and TheHive) recorded it, scored it and documented it as a case.

[📄 Read the project](./Honeypot_SOC/) · [🛠 Setup guide](./Honeypot_SOC/SETUP.md) · [▶ Demo video](./Honeypot_SOC/video/honeypot_demo.mp4)

---

## 📁 Projects

| # | Project | What It Covers | Status |
|---|---|---|---|
| 1 | [Enterprise IAM Lab](./IAM/) | Active Directory, SSO, RBAC, VPN, 2FA | ✅ Complete |
| 2 | [Snort IDS Lab](./Snort/) | Intrusion Detection, Network Signatures | ✅ Complete |
| 3 | [Wazuh SIEM Lab](./Wazuh_Report/) | Log Analysis, Security Monitoring | ✅ Complete |
| 4 | [Wireshark Analysis](./wireshark/) | Packet Capture, Traffic Analysis | ✅ Complete |
| 5 | [Mini SOAR on AWS](./Mini-SOAR/) | Automated detection, threat enrichment, case management, and response using Wazuh, TheHive, n8n, and VirusTotal | ✅ Complete |
| 6 | [Honeypot SOC on AWS](./Honeypot_SOC/) ⭐ | A Cowrie honeypot on the internet, Wazuh rules, n8n enrichment with AbuseIPDB and VirusTotal, and TheHive cases. Real traffic recorded and documented. | ✅ Complete |

---

## 🔬 About This Repository

I built every lab in this repository myself, inside a virtualized environment, using the same tools real security teams use. My focus throughout has been understanding why these security controls exist, not just clicking through a setup guide.

Each project includes:
- A full explanation of what I built and why I built it that way
- The real problems I ran into while building it
- How I actually debugged those problems, using logs and evidence rather than guessing
- Screenshots and recordings showing the finished system actually working

---

## 🧰 Tools and Technologies Used Across All Labs

![Active Directory](https://img.shields.io/badge/Active_Directory-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=for-the-badge&logo=wireshark&logoColor=white)
![pfSense](https://img.shields.io/badge/pfSense-212121?style=for-the-badge&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)
![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-3F51B5?style=for-the-badge)
![TheHive](https://img.shields.io/badge/TheHive-F5A623?style=for-the-badge)
![Cowrie](https://img.shields.io/badge/Cowrie-Honeypot-2E7D32?style=for-the-badge)
![Snort](https://img.shields.io/badge/Snort-EF3E36?style=for-the-badge)

---

## 🤝 Connect

I am a final year student looking for IT Support and SOC analyst roles.
[GitHub](https://github.com/abdulsaboor85) · [LinkedIn](https://www.linkedin.com/in/abdul-saboor-082866290/)

---

<div align="center">

*Built with patience, broken with curiosity, fixed with logs.*

</div>

---

## Disclaimer

Everything in this repository, every test, every demonstration, was done in an authorized lab or virtual environment purely for learning purposes. The honeypot only receives connections. I never attacked anyone.

Please do **not** use any technique, command, or tool shown here against a computer, network, or system that you do not own or do not have clear permission to test.
