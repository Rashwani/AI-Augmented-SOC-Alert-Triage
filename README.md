# AI-Augmented SOC Alert Triage
 
A lab pipeline that automates the **first pass of SOC alert triage** using the Wazuh SIEM and the Claude API. Wazuh detects suspicious activity and writes alerts to disk; a Python service watches that log, sends each qualifying alert to Claude, and gets back a structured triage verdict — severity, likely cause, MITRE ATT&CK technique, and a recommended action — which it saves to a JSONL report.
 
Built as a portfolio project to explore where an LLM can reduce manual effort in a Tier 1 analyst workflow, and — just as importantly — where it can't.
 
---
 
## Architecture
 
```
<img width="1408" height="768" alt="Gemini_Generated_Image_h7x7r5h7x7r5h7x7" src="https://github.com/user-attachments/assets/a09ddf1f-3662-4978-aaa2-e144642a4da9" />

```
 
## Tech stack
 
- **Cloud:** AWS EC2 (Ubuntu 24.04 LTS)
- **SIEM:** Wazuh 4.14 all-in-one (manager, indexer, dashboard)
- **AI:** Python 3 + Anthropic SDK, Claude API
- **Detections used:** SSH brute force, authentication failures, File Integrity Monitoring (FIM), user/group creation, privilege escalation, package changes
## What the AI layer adds
 
A standard Wazuh homelab stops at detection. The differentiator here is the triage layer: instead of an analyst reading each raw alert, Claude produces a consistent, structured verdict for every alert above a severity threshold. Each result is written as one JSON line so the output is measurable, not just readable.
 
Design decisions worth noting:
- **Prompt-injection aware.** Log fields can contain attacker-controlled text, so the alert is wrapped and explicitly marked as untrusted data in the system prompt, with an instruction never to follow embedded commands.
- **Structured output.** The model is constrained to return a fixed JSON schema (`severity`, `mitre_technique`, `likely_cause`, `recommended_action`, `confidence`), making the results easy to aggregate and audit.
- **Robust log tailing.** The watcher handles Wazuh's log rotation and partially written lines so it keeps running unattended.
## Sample: before and after
 
**Raw Wazuh alert (input):**
```json
{"rule":{"level":10,"description":"sshd: brute force trying to get access to the system.","id":"5712",
"mitre":{"id":["T1110"],"tactic":["Credential Access"]}},"agent":{"name":"ip-172-31-108-235"},
"data":{"srcip":"187.153.255.190"},"full_log":"... Failed password for invalid user ..."}
```
 
**AI triage verdict (output):**
```json
{"severity":"Medium","mitre_technique":"T1110","likely_cause":"Repeated failed SSH logins from a
single source indicate an automated brute-force attempt against non-existent accounts.",
"recommended_action":"Block the source IP at the firewall and confirm no account lockouts or
successful logins followed from it.","confidence":"High"}
```
 
Raw alerts as Wazuh writes them to `alerts.json`:
 
![Raw Wazuh alert JSON](docs/raw-alert.png)
 
The pipeline triaging live alerts, assigning a different severity to each based on context:
 
![Triage pipeline running](docs/pipeline-running.png)
 
## Results
 
Across a test run that generated **86 Wazuh alerts** from six simulated activity types, **73 met the severity threshold and were triaged by Claude**, with the following distribution:
 
| AI severity | Count |
|-------------|-------|
| High        | 24    |
| Medium      | 21    |
| Low         | 28    |
 
![Severity distribution](docs/severity-distribution.png)
 
The model consistently rated multi-event correlation alerts (e.g. the level-10 brute-force rule, new user/group creation) higher than isolated single events, and downgraded benign FIM changes in a test directory — behaviour you'd want from a first-pass filter.
 
The same activity in the Wazuh Threat Hunting dashboard — 86 alerts across the run:
 
![Wazuh Threat Hunting overview](docs/dashboard-overview.png)
 
![Varied alert types in the dashboard](docs/dashboard-events.png)
 
## Setup
 
Full step-by-step build notes — AWS account through systemd service — are in **[SETUP.md](SETUP.md)**. In short: an Ubuntu EC2 host runs Wazuh 4.14 all-in-one and a Python virtualenv with the Anthropic SDK; the API key lives in a `.env` file; `triage.py` watches the alerts log and writes verdicts to `triage_report.jsonl`.
 
## Running it
 
```bash
cd ~/ai-triage && source venv/bin/activate
set -a; . ./.env; set +a
python3 triage.py
```
 
In a second terminal, generate test alerts (e.g. a simulated SSH brute force):
```bash
for i in $(seq 1 10); do ssh -o BatchMode=yes -o StrictHostKeyChecking=no wronguser@localhost; done
```
 
Check the output:
```bash
cat ~/ai-triage/triage_report.jsonl
```
