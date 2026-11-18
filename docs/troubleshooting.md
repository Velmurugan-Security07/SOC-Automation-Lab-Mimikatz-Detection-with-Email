# Troubleshooting Log

Real issues hit while building the lab and how each was handled.

| # | Symptom | Cause | Fix |
|---|---------|-------|-----|
| 1 | `curl ... -o /etc/apt/keyrings/apache-cassandra.asc` → *Permission denied*, curl error 23 | Writing to `/etc/apt/keyrings` without root | Re-run the command with `sudo` |
| 2 | `sudo sapoff -a` → *command not found* | Typo | Use `sudo swapoff -a` |
| 3 | `sudo systemctl - vm.max_map_count=262144` → *Unknown command verb* | Wrong tool | Use `sudo sysctl -w vm.max_map_count=262144` |
| 4 | `sysctl: cannot stat /proc/sys/-` | Stray `-` in the command | Remove the dash |
| 5 | `docker compose up -d` interrupted during the OpenSearch pull | Slow download, Ctrl+C | Re-run the command; the pull resumed and all containers started |
| 6 | `mkdir: shuffle-database: File exists` | Directory already in the cloned repo | Safe to ignore; run `chown` on it |
| 7 | Wazuh agent shows *Disconnected* in the overview | Agent service not running or manager IP wrong | Start the service on Target-pc and check the manager address (it later showed Active) |
| 8 | Shuffle TheHive node shows *Authentication needed* | No credentials saved for the app | Add the API key and instance URL under **Add Authentication** |
| 9 | Several Shuffle runs show error icons | Nodes not fully authenticated or parameters missing while wiring the workflow | Inspect each node in **Debug**, fix auth and parameters, re-run |
| 10 | Email body is raw JSON | Whole webhook payload used as the message | Reference only the needed fields with Shuffle variables |
| 11 | Alert shows `rule.groups: local, syslog, sshd` | Rule placed in the sshd example group | Moved to its own group in `local_rules.xml` |

## Useful checks

```bash
# Wazuh: validate and watch alerts
sudo /var/ossec/bin/wazuh-logtest
sudo tail -f /var/ossec/logs/alerts/alerts.json
sudo tail -f /var/ossec/logs/integrations.log

# Shuffle containers
sudo docker ps
sudo docker compose logs -f backend

# TheHive reachable?
curl -I http://192.168.56.151:9000
```
