# Setup Guide

Reproduction steps for the SOC Automation Lab. Evidence for each step is in the numbered screenshots in [`../images`](../images).

## 1. Network

Create a VirtualBox host-only network `192.168.56.0/24` and attach all VMs.

| Host | IP |
|------|----|
| Wazuh server | 192.168.56.104 |
| Target-pc (Windows 10) | 192.168.56.150 |
| Ubuntu server (Shuffle + TheHive) | 192.168.56.151 |

## 2. Wazuh and the Windows agent

1. Install the Wazuh server and dashboard on the Wazuh VM.
2. In the dashboard open **Endpoints → Deploy new agent**, choose **Windows (MSI)**, enter the manager IP and run the generated command in an elevated PowerShell on Target-pc.
3. Install **Sysmon** with a process-creation config on Target-pc.
4. Add the Sysmon channel to the agent `ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

5. Restart the Wazuh agent service.

## 3. Custom rule

Copy [`../configs/local_rules.xml`](../configs/local_rules.xml) to `/var/ossec/etc/rules/local_rules.xml` on the Wazuh server and restart the manager.

## 4. TheHive 5 (Ubuntu server)

```bash
sudo apt install openjdk-11-jdk -y && java -version

# Apache Cassandra
sudo mkdir -p /etc/apt/keyrings
sudo curl -o /etc/apt/keyrings/apache-cassandra.asc https://downloads.apache.org/cassandra/KEYS
echo "deb [signed-by=/etc/apt/keyrings/apache-cassandra.asc] https://debian.cassandra.apache.org 50x main" | sudo tee /etc/apt/sources.list.d/cassandra.sources.list
sudo apt update && sudo apt install -y cassandra

# Elasticsearch 7.x
sudo apt install elasticsearch -y
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```

Then install TheHive 5 from the StrangeBee packages, configure it for Cassandra and Elasticsearch, and browse to `http://192.168.56.151:9000`. Create the `SOC-Lab` organisation and an analyst user, then generate an API key under **Account → API Key**.

## 5. Shuffle (Ubuntu server)

```bash
sudo apt install docker-compose-v2 -y
git clone https://github.com/Shuffle/Shuffle
cd Shuffle
mkdir shuffle-database
sudo chown -R 1000:1000 shuffle-database
sudo swapoff -a
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf   # persist across reboots
sudo docker compose up -d
```

Open `http://192.168.56.151:3001` and create the admin account.

## 6. Workflow

1. Create `mimikatz-detection-workflow`.
2. Add a **Webhook** trigger and click **Start**; copy the Webhook URI.
3. Add the **TheHive** app, authenticate with the API key and `http://192.168.56.151:9000`.
4. Add the **Email** app (`smtp.gmail.com`, port `587`, Gmail App Password) with subject *Mimikatz Alert Detected*.
5. Connect Webhook → TheHive → Email and save.

## 7. Wazuh integration

Add [`../configs/ossec-integration.xml`](../configs/ossec-integration.xml) to `/var/ossec/etc/ossec.conf` using your webhook URL, then restart:

```bash
sudo systemctl restart wazuh-manager
```

## 8. Test

On Target-pc, as an authorised lab exercise only, run Mimikatz from its folder. Confirm the alert in **Threat Hunting** with `rule.id: 100002`, then check the workflow run history in Shuffle and your inbox.
