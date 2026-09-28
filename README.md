# AI-Augmented-SOC-Alert-Triage
AWS-hosted Wazuh SIEM with a Python + Claude API pipeline that triages alerts in real time, with MITRE ATT&amp;CK context, severity scoring, and recommended actions.

An AI-assisted first-pass triage pipeline for Wazuh security alerts. A Python service watches Wazuh's alert stream in real time, sends qualifying alerts to Claude via the Anthropic API, and writes back a severity rating, likely cause, and recommended response action.
<img width="1896" height="858" alt="16" src="https://github.com/user-attachments/assets/a25f9307-2aa0-4cc2-9702-11c419273144" />

## Why This Project
 
Tier 1 SOC analysts spend much of their time on repetitive first-pass triage: reading an alert, judging severity, and deciding what to do next. This project automates that first pass so an analyst starts with context instead of a raw log line, while keeping the human as the final decision-maker.

## Architecture
<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/312fe7b5-0b2b-4b86-a6f1-e26706ea33d3" />

## Tech Stack
 
| Component | Purpose |
|---|---|
| AWS EC2 (Ubuntu 22.04, us-east-1) | Hosts the full lab |
| Wazuh 4.x (all-in-one) | SIEM: manager, indexer, and dashboard |
| Python 3 + venv | Runs the triage service |
| Anthropic Python SDK | Calls Claude for alert analysis |
| systemd | Keeps the triage service running in the background |

## How It Works
 
1. **Detection:** Wazuh monitors the host (SSH auth, PAM sessions, sudo, package changes, file integrity) and writes every alert as a JSON line to `/var/ossec/logs/alerts/alerts.json`.
2. **Ingestion:** `triage.py` tails that file like `tail -f`, reading only new alerts.
3. **Filtering:** Only alerts at or above a configurable Wazuh level (`MIN_SEVERITY_LEVEL = 5`) are sent for triage, which keeps API cost and noise down.
4. **Enrichment:** The prompt includes the rule description, level, rule ID, MITRE ATT&CK technique, agent name, source IP, and full log line.
5. **Analysis:** Claude returns a severity (Low / Medium / High / Critical), a likely cause, and a recommended action.
6. **Output:** Results are appended to `triage_report.log` with the alert timestamp and description.

### 1. Generate the alert
 
A loop of SSH logins as a non-existent user simulates password-guessing activity.
 
```bash
for i in 1 2 3 4 5; do ssh wronguser@localhost; done
```
terminal showing the failed SSH loop 
<img width="1012" height="325" alt="7" src="https://github.com/user-attachments/assets/a0299d97-8b8d-43ad-87f8-ca758097639f" />

### 2. Wazuh detects it
 
Wazuh fires rule **5710** ("sshd: Attempt to login using a non-existent user"), level 5, mapped to MITRE ATT&CK **T1110.001 (Password Guessing)** and **T1021.004 (SSH)**.
Threat Hunting events table showing rule 5710 alerts 
<img width="1861" height="737" alt="13" src="https://github.com/user-attachments/assets/19817ba5-079a-435e-9b71-b0cd03f83b71" />

### 3. Before: the raw alert
alert detail, fields
<img width="1160" height="771" alt="17" src="https://github.com/user-attachments/assets/240b0bc3-836f-47f1-9612-e0407a6ce789" />
alert detail, MITRE and compliance mapping
<img width="732" height="590" alt="18" src="https://github.com/user-attachments/assets/db7b23ba-f033-4d49-a9d4-5326d67cac25" />

```
rule.id:          5710
rule.level:       5
rule.description: sshd: Attempt to login using a non-existent user
rule.mitre.id:    T1110.001, T1021.004
data.srcip:       127.0.0.1
data.srcuser:     wronguser
full_log:         sshd[86745]: Invalid user wronguser from 127.0.0.1 port 39694
```

### 4. The triage service picks it up
triage.py running and printing "Triaged:" lines 
<img width="1657" height="286" alt="15" src="https://github.com/user-attachments/assets/7a51e945-e680-451e-ac67-df325cca34fb" />

### 5. After: the AI triage assessment
triage_report.log output
<img width="1895" height="502" alt="14" src="https://github.com/user-attachments/assets/21956e5e-1481-408b-8d4d-3118688e612c" />
> **Severity:** Low
>
> **Likely cause:** The source IP is 127.0.0.1 (localhost) and the username is "wronguser," which points to a manual test or validation check rather than an external brute-force attack.
>
> **Recommended action:** Verify whether this was an authorized test. If it was unexpected, review local user activity and cron jobs for scripts making local SSH connections. No external action is needed since the source is localhost.
 
The model correctly used the source IP to downgrade severity. A rule-only system would treat this the same as an attack from an external IP.

## Setup
 
### Prerequisites
- AWS account with an EC2 instance running Ubuntu 22.04
- Wazuh all-in-one installed ([official docs](https://documentation.wazuh.com))
- An Anthropic API key
### Install
 
```bash
sudo apt update && sudo apt install -y python3-pip python3-venv
mkdir ~/ai-triage && cd ~/ai-triage
python3 -m venv venv
source venv/bin/activate
pip install anthropic
export ANTHROPIC_API_KEY="your-key-here"   # never commit this
```
 
Copy `triage.py` into `~/ai-triage/`, then run it:
 
```bash
sudo venv/bin/python3 triage.py
```
 
`sudo` is required because Wazuh restricts read access to `alerts.json`.
 
### Run as a background service
 
Create `/etc/systemd/system/ai-triage.service` (see `ai-triage.service` in this repo), then run:
 
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ai-triage
```
## Challenges & Lessons Learned
 
**Networking.** The instance's route table had a blackhole route to the internet gateway, so SSH and EC2 Instance Connect both failed. Fixing it meant re-associating the subnet with a route table that has a valid `0.0.0.0/0 → igw` route.

EC2 Instance Connect "blackhole route" warning -->
<img width="1813" height="570" alt="4" src="https://github.com/user-attachments/assets/0301e27a-0b38-4028-9051-7d3ead37febc" />

**Wazuh installer URL.** The generic `4.x` installer URL returned an XML error page instead of a script, so bash failed with a syntax error. The fix was to use the versioned URL from the current Wazuh documentation.
<img width="1037" height="117" alt="5" src="https://github.com/user-attachments/assets/8f05a389-28f0-4980-ae50-867f208facfc" />

**Instance sizing.** Wazuh's indexer needs far more RAM and disk than free-tier instances provide. The instance was upsized and the EBS volume expanded. After the disk filled up, the OpenSearch read-only index block also had to be cleared.
 
**Agent on the manager host.** Installing `wazuh-agent` on the same host as the manager conflicts on Ubuntu. The manager already monitors itself as built-in agent `000`, so no separate agent was needed.
 
**API response handling.** The first version read only `response.content[0].text`, which can break if a response contains more than one content block. The fix joins every text block, and `max_tokens` was raised to 1024 to avoid cut-off responses.
 
---








