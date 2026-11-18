# 🤖 SOC Automation Lab – Mimikatz Detection with Email Alerts (Wazuh + Shuffle + TheHive)

A hands-on Security Operations (SOC) automation lab (**SOC-Automation-Lab-Mimikatz-Detection-with-Email**) that detects **Mimikatz credential dumping** on a Windows endpoint and automatically escalates it: Wazuh raises a custom level-15 alert, Shuffle (SOAR) receives it over a webhook, creates the case in TheHive, and emails the analyst — with the alert arriving as a **push notification on a phone**. This project follows the MyDFIR SOC automation methodology and extends it with a custom detection rule, a documented troubleshooting log and evidence for every stage.

[![Architecture Diagram](images/01-architecture-diagram.png)](images/01-architecture-diagram.png)

> 💡 **Click any screenshot in this README to open it full-size.** All 34 screenshots are in [`images/`](images).

* * *

## 📌 Project Objectives

- Build a SIEM + SOAR + case-management pipeline entirely in a local lab
- Detect a real adversary technique (MITRE ATT&CK **T1003 – OS Credential Dumping**) with a custom Wazuh rule
- Forward Sysmon telemetry from a Windows 10 endpoint to the Wazuh manager
- Automate alert hand-off to Shuffle, TheHive and email with no manual steps
- Deliver the alert to an analyst's mobile device
- Document every step so the lab is fully reproducible

* * *

## 🧱 Lab Architecture

| Host | Role | IP Address |
|------|------|------------|
| **Wazuh Server** (`wazuh-server`) | SIEM / XDR manager + dashboard | `192.168.56.104` |
| **Target-pc** (Windows 10) | Monitored endpoint – Wazuh agent + Sysmon | `192.168.56.150` |
| **Ubuntu Server** (`ubnwbs`) | Shuffle (Docker) + TheHive 5 | `192.168.56.151` |
| **Analyst** | Gmail on desktop and phone | — |

**Network:** `192.168.56.0/24` (isolated host-only VirtualBox network)

### Data flow

```
Windows 10 (Sysmon) ──▶ Wazuh Agent ──▶ Wazuh Manager (rule 100002, level 15)
                                              │  <integration> webhook (JSON)
                                              ▼
                                      Shuffle workflow
                                       ├─▶ TheHive (case / alert)
                                       └─▶ Email (SMTP) ──▶ Gmail ──▶ 📱 phone notification
```

1. `mimikatz.exe` starts on Target-pc → Sysmon **Event ID 1** (Process Create)
2. Wazuh agent ships the event to the manager
3. Custom rule **100002** matches and fires at **level 15**, tagged **T1003**
4. The `<integration>` block posts the alert JSON to the Shuffle webhook
5. The Shuffle workflow calls TheHive and sends an email alert
6. The analyst receives the alert in Gmail and as a push notification

* * *

## 🧰 Tools & Technologies

| Category | Tool |
|----------|------|
| Virtualization | VirtualBox |
| SIEM / XDR | Wazuh 4.x (manager, indexer, dashboard) |
| Endpoint telemetry | Sysmon + Wazuh agent (Windows 10) |
| SOAR | Shuffle (open source, Docker Compose 2.40.3) |
| Case management | TheHive 5.8 |
| TheHive dependencies | OpenJDK 11, Apache Cassandra 5.0, Elasticsearch 7.17.29 |
| Notification | SMTP (Gmail) |
| Adversary tool (lab only) | Mimikatz 2.2.0 x64 |
| Server OS | Ubuntu Server |
| Target OS | Windows 10 (build 19045) |

* * *

## 📊 Project Status

| Stage | Status | Evidence |
|-------|--------|----------|
| Wazuh server + Windows agent | ✅ Working | [02](images/02-wazuh-dashboard-overview.png), [03](images/03-wazuh-deploy-windows-agent.png) |
| Custom Mimikatz rule (T1003) | ✅ Working | [04](images/04-custom-rule-local-rules-xml.png), [07](images/07-rule-100002-events.png), [10](images/10-event-details-rule-and-mitre-mapping.png) |
| Wazuh → Shuffle webhook integration | ✅ Working | [29](images/29-wazuh-ossec-conf-shuffle-integration.png), [33](images/33-gmail-mimikatz-alert-email.png) |
| Email alert via SMTP | ✅ Working | [30](images/30-shuffle-email-smtp-configuration.png), [33](images/33-gmail-mimikatz-alert-email.png) |
| Mobile push notification | ✅ Working | [34](images/34-mobile-push-notification.png) |
| TheHive node in Shuffle | ⚙️ Configured (API auth set) – case creation being verified | [27](images/27-shuffle-thehive-authentication.png), [28](images/28-shuffle-thehive-authentication-saved.png) |
| IOC enrichment (VirusTotal) | 🔜 Planned | — |
| Active response (isolate / kill process) | 🔜 Planned | — |

* * *

## 🔧 Build Walkthrough

### 1. Wazuh server and Windows agent

The Wazuh dashboard showed no active agents until the Windows endpoint was enrolled with the generated MSI deployment command, pointing at the manager `192.168.56.104`.

[![Wazuh overview](images/02-wazuh-dashboard-overview.png)](images/02-wazuh-dashboard-overview.png)
[![Deploy new agent](images/03-wazuh-deploy-windows-agent.png)](images/03-wazuh-deploy-windows-agent.png)

* * *

### 2. Custom detection rule – Mimikatz (MITRE ATT&CK [T1003](https://attack.mitre.org/techniques/T1003/))

Rule `100002` extends the Sysmon Event ID 1 group and matches the process image with a case-insensitive PCRE2 expression. Full file: [`configs/local_rules.xml`](configs/local_rules.xml).

```xml
<rule id="100002" level="15">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.image" type="pcre2">(?i)mimikatz\.exe</field>
  <description>mimikatz execution detected - credential dumping tool</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```

[![local_rules.xml](images/04-custom-rule-local-rules-xml.png)](images/04-custom-rule-local-rules-xml.png)

* * *

### 3. Detection validated in Wazuh

Running Mimikatz on Target-pc produced **2 level-15 alerts** for rule `100002`, mapped to *Credential Access → OS Credential Dumping (T1003)*. The Sysmon fields (image path, command line, hashes, parent process, integrity level) are preserved in the alert.

[![Threat Hunting dashboard](images/05-wazuh-threat-hunting-dashboard.png)](images/05-wazuh-threat-hunting-dashboard.png)
[![Rule 100002 filter](images/06-rule-100002-dashboard-filter.png)](images/06-rule-100002-dashboard-filter.png)
[![Rule 100002 events](images/07-rule-100002-events.png)](images/07-rule-100002-events.png)
[![Event details 1](images/08-event-details-sysmon-process-fields.png)](images/08-event-details-sysmon-process-fields.png)
[![Event details 2](images/09-event-details-parent-process-and-system.png)](images/09-event-details-parent-process-and-system.png)
[![Event details 3](images/10-event-details-rule-and-mitre-mapping.png)](images/10-event-details-rule-and-mitre-mapping.png)
[![Overview with critical alerts](images/11-wazuh-overview-critical-alerts.png)](images/11-wazuh-overview-critical-alerts.png)

* * *

### 4. TheHive 5 installation (Java, Cassandra, Elasticsearch)

TheHive 5 needs three backing components on the Ubuntu server. Note the first Cassandra key download failed with *Permission denied* until it was re-run with `sudo` — see [troubleshooting](docs/troubleshooting.md).

[![OpenJDK 11](images/12-java-install.png)](images/12-java-install.png)
[![Cassandra install](images/13-cassandra-install.png)](images/13-cassandra-install.png)
[![Elasticsearch install](images/14-elasticsearch-install.png)](images/14-elasticsearch-install.png)

* * *

### 5. TheHive setup – organisation and analyst user

Created a dedicated `SOC-Lab` organisation ("Soc automation lab") and an analyst user.

[![TheHive login](images/15-thehive-login.png)](images/15-thehive-login.png)
[![TheHive organisation and user](images/16-thehive-organisation-and-analyst-user.png)](images/16-thehive-organisation-and-analyst-user.png)

* * *

### 6. Shuffle installation (Docker Compose)

```bash
sudo apt install docker-compose-v2
git clone https://github.com/Shuffle/Shuffle
cd Shuffle
mkdir shuffle-database
sudo chown -R 1000:1000 shuffle-database
sudo swapoff -a
sudo sysctl -w vm.max_map_count=262144
sudo docker compose up -d
```

OpenSearch (used by Shuffle) requires `vm.max_map_count=262144`. After a re-run following an interrupted image pull, all containers (`orborus`, `backend`, `opensearch`, `frontend`, `security`) started.

[![Docker Compose](images/17-docker-compose-install.png)](images/17-docker-compose-install.png)
[![Shuffle git clone](images/18-shuffle-git-clone.png)](images/18-shuffle-git-clone.png)
[![docker compose up](images/19-shuffle-docker-compose-up.png)](images/19-shuffle-docker-compose-up.png)
[![Containers started](images/20-shuffle-containers-started.png)](images/20-shuffle-containers-started.png)
[![Shuffle login](images/21-shuffle-login.png)](images/21-shuffle-login.png)
[![Shuffle onboarding](images/22-shuffle-onboarding.png)](images/22-shuffle-onboarding.png)

* * *

### 7. Shuffle workflow – `mimikatz-detection-workflow`

The workflow starts with a **Webhook** trigger, then the **TheHive** app, then the **Email** app. TheHive is authenticated with an API key and the instance URL (`http://<thehive-ip>:9000`).

[![New workflow](images/23-shuffle-new-workflow.png)](images/23-shuffle-new-workflow.png)
[![Workflow canvas](images/24-shuffle-workflow-canvas-thehive-and-webhook.png)](images/24-shuffle-workflow-canvas-thehive-and-webhook.png)
[![Webhook running](images/25-shuffle-webhook-trigger-running.png)](images/25-shuffle-webhook-trigger-running.png)
[![TheHive API key page](images/26-thehive-api-key-page.png)](images/26-thehive-api-key-page.png)
[![TheHive authentication](images/27-shuffle-thehive-authentication.png)](images/27-shuffle-thehive-authentication.png)
[![TheHive authentication saved](images/28-shuffle-thehive-authentication-saved.png)](images/28-shuffle-thehive-authentication-saved.png)

> 🔒 API keys, webhook URLs and e-mail addresses have been redacted from the screenshots in this repository.

* * *

### 8. Wazuh → Shuffle integration

The Wazuh manager forwards alerts to the workflow webhook with an `<integration>` block in `/var/ossec/etc/ossec.conf`. Sanitised config: [`configs/ossec-integration.xml`](configs/ossec-integration.xml).

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://SHUFFLE_IP:3001/api/v1/hooks/webhook_REPLACE_ME</hook_url>
  <level>3</level>
  <alert_format>json</alert_format>
</integration>
```

[![ossec.conf integration](images/29-wazuh-ossec-conf-shuffle-integration.png)](images/29-wazuh-ossec-conf-shuffle-integration.png)

* * *

### 9. Email notification (SMTP)

The Shuffle **Email → Send email smtp** action uses `smtp.gmail.com:587` with a Gmail App Password.

[![SMTP configuration](images/30-shuffle-email-smtp-configuration.png)](images/30-shuffle-email-smtp-configuration.png)
[![Workflow run history](images/31-shuffle-workflow-run-history.png)](images/31-shuffle-workflow-run-history.png)

* * *

## ⚔️ Attack Simulation & End-to-End Result

> ⚠️ Run offensive tools only on systems you own, inside an isolated lab.

**Attack:** Mimikatz 2.2.0 executed from `C:\Users\vel\Downloads\mimikatz_trunk\x64` on Target-pc.

[![Mimikatz on Target-pc](images/32-mimikatz-executed-on-target-pc.png)](images/32-mimikatz-executed-on-target-pc.png)

**Result:** within about two minutes the analyst received the *"Mimikatz Alert Detected"* email containing the Sysmon event details (image path, command line, user, parent process `cmd.exe`, integrity level and file hashes), and a push notification on the phone.

[![Gmail alert](images/33-gmail-mimikatz-alert-email.png)](images/33-gmail-mimikatz-alert-email.png)

<p align="center">
  <a href="images/34-mobile-push-notification.png">
    <img src="images/34-mobile-push-notification.png" alt="Mobile push notification" width="320">
  </a>
</p>

| Timeline (observed, one run) | |
|---|---|
| Sysmon process-create (`UtcTime`) | 2026-11-17 16:59:09 UTC (22:29 IST) |
| Email delivered to Gmail | 22:31 IST |
| Phone notification | 22:32 IST |

* * *

## 🗺️ Attack Chain Summary

| Stage | MITRE ATT&CK | Action | Detected / Handled Via |
|-------|--------------|--------|------------------------|
| 1 | T1003 – OS Credential Dumping | `mimikatz.exe` executed on Windows 10 | Sysmon Event ID 1 → Wazuh rule 100002 (level 15) |
| 2 | — | Alert forwarded | Wazuh `<integration>` → Shuffle webhook |
| 3 | — | Notification | Shuffle → SMTP → Gmail → mobile push |
| 4 | — | Case management | Shuffle → TheHive (configured; verification in progress) |

* * *

## 🧠 Skills Demonstrated

- **SIEM engineering:** deployed Wazuh, enrolled a Windows agent and ingested Sysmon telemetry
- **Detection engineering:** wrote a custom rule with PCRE2 field matching and MITRE ATT&CK mapping (T1003)
- **SOAR automation:** built a Shuffle workflow with webhook trigger, TheHive and SMTP actions
- **Integration:** configured the Wazuh `<integration>` hook and TheHive API authentication
- **Linux & containers:** Docker Compose, kernel parameter tuning for OpenSearch, apt repository and key management
- **Troubleshooting:** diagnosed permission, package and workflow errors (see [docs/troubleshooting.md](docs/troubleshooting.md))
- **Security hygiene:** redacted secrets from public evidence and documented credential rotation
- **Documentation:** reproducible, evidence-backed write-up in SOC-report style

* * *

## ⚠️ Known Issues & Lessons Learned

- **Rule group:** rule 100002 was originally placed inside the example `sshd` group, so alerts show `rule.groups: local, syslog, sshd`. The shipped [`local_rules.xml`](configs/local_rules.xml) moves it to `sysmon,credential_access`.
- **Filename-only detection:** renaming the binary evades the rule. Rule `100003` also matches `originalFileName`; LSASS access (Sysmon Event ID 10) is the next detection to add.
- **Integration noise:** `<level>3</level>` forwards almost every alert. Add `<rule_id>100002</rule_id>` to forward only the Mimikatz rule.
- **Workflow run errors:** most runs in the history show error indicators while the webhook, TheHive and Email nodes were being wired and authenticated.
- **Raw JSON email body:** the email dumps the full event. A cleaner message should use Shuffle variables for the key fields only.
- **Persistence:** `vm.max_map_count` is set with `sysctl -w` and is lost on reboot; add it to `/etc/sysctl.conf`.
- **TheHive trial licence:** the instance showed a 7-day Platinum trial banner; the Community licence remains available after registration.

* * *

## 📂 Repository Structure

```
.
├── README.md
├── LICENSE
├── .gitignore
├── configs/
│   ├── local_rules.xml
│   └── ossec-integration.xml
├── docs/
│   ├── setup-guide.md
│   └── troubleshooting.md
└── images/            # 34 screenshots (secrets redacted)
    ├── 01-architecture-diagram.png
    ├── ...
    └── 34-mobile-push-notification.png
```

* * *

## 🚀 Next Steps / Future Work

- Confirm TheHive case/alert creation end to end and capture the case screenshot
- Add VirusTotal enrichment of the file SHA256 before case creation
- Restrict the integration to rule `100002` and format the email body
- Add a Wazuh **active-response** step (kill process / isolate host) after analyst approval
- Add detections for LSASS access (Sysmon ID 10) and RDP brute force (Event ID 4625)
- Export the Shuffle workflow JSON into this repo
- Send richer alerts to Telegram or Slack

* * *

## 📖 Credit

Lab methodology based on the [MyDFIR](https://www.youtube.com/@MyDFIR) SOC automation project, extended with an original detection rule, mobile alerting, troubleshooting notes and documentation by the repository author.

Related project: [Active Directory Home Lab](https://github.com/Velmurugan-Security07/Active-Directory-Home-Lab)

## 👤 Author

**Velmurugan A.** – Aspiring SOC Analyst · [GitHub](https://github.com/Velmurugan-Security07) · [LinkedIn](https://www.linkedin.com/in/velmurugan-anand)

## 📄 License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
