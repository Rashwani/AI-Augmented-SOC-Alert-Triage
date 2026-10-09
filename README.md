# AI-Augmented SOC Alert Triage
 
A lab pipeline that automates the **first pass of SOC alert triage** using the Wazuh SIEM and the Claude API. Wazuh detects suspicious activity and writes alerts to disk; a Python service watches that log, sends each qualifying alert to Claude, and gets back a structured triage verdict — severity, likely cause, MITRE ATT&CK technique, and a recommended action — which it saves to a JSONL report.
 
Built as a portfolio project to explore where an LLM can reduce manual effort in a Tier 1 analyst workflow, and — just as importantly — where it can't.
 
---
 
## Architecture
 
```
┌──────────────┐     writes      ┌────────────────────┐    reads new    ┌──────────────┐
│    Wazuh     │  ───────────▶   │  alerts.json (log) │  ───────────▶   │  triage.py   │
│  (manager +  │   JSON alerts   │  /var/ossec/logs   │   tail -f       │  (watcher)   │
│  indexer +   │                 └────────────────────┘                 └──────┬───────┘
│  dashboard)  │                                                               │ alert JSON
└──────────────┘                                                               ▼
       ▲                                                              ┌──────────────────┐
       │ monitors (agent 000)                                         │   Claude API     │
       │                                                              │  severity /cause │
┌──────────────┐                                                      │  MITRE / action  │
│  Ubuntu host │                                                      └────────┬─────────┘
│  (EC2)       │                                                               │ verdict
└──────────────┘                                                               ▼
                                                                    triage_report.jsonl
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
 
## Results
 
Across a test run that generated **86 Wazuh alerts** from six simulated activity types, **73 met the severity threshold and were triaged by Claude**, with the following distribution:
 
| AI severity | Count |
|-------------|-------|
| High        | 24    |
| Medium      | 21    |
| Low         | 28    |
 
The model consistently rated multi-event correlation alerts (e.g. the level-10 brute-force rule, new user/group creation) higher than isolated single events, and downgraded benign FIM changes in a test directory — behaviour you'd want from a first-pass filter.
 
## Screenshots
 
| | |
|---|---|
| `docs/raw-alert.png` | Raw JSON alert in `alerts.json` (input) |
| `docs/pipeline-running.png` | `triage.py` triaging live alerts with mixed severities |
| `docs/severity-distribution.png` | Severity breakdown across the run |
| `docs/dashboard-overview.png` | Wazuh Threat Hunting view (86 alerts) |
| `docs/dashboard-events.png` | Varied alert types in the dashboard |
 
## Setup
 
> Full build notes are in [`SETUP.md`](SETUP.md). High level:
 
1. Launch an Ubuntu EC2 instance (t3.large recommended; the all-in-one stack needs ~8 GB RAM and 50 GB disk).
2. Install Wazuh: `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a`
3. Enable realtime FIM on a test directory in `/var/ossec/etc/ossec.conf`.
4. Add your user to the `wazuh` group so the script can read the alerts log without root.
5. Create a virtualenv and `pip install anthropic`.
6. Store your key in `.env` as `ANTHROPIC_API_KEY=...` and run `python3 triage.py`.
7. (Optional) Run it as a systemd service to keep it running after disconnect.
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
 
## Limitations and honest scope
 
This is a learning lab, and it's worth being clear about what it does **not** replicate:
 
- **Benign, self-generated activity.** All "attacks" are harmless commands run on the host itself, so this tests that detection and triage *fire correctly*, not real adversary behaviour.
- **No separate attacker.** The host monitors itself (agent 000), so source IPs and patterns aren't realistic. A dedicated attacker instance would be the highest-value next step.
- **No false-positive tuning or ticketing.** Real SOCs deal with alert volume, noise suppression, and case management that this lab doesn't cover.
- **LLM triage is advisory.** The model's verdict is a first-pass aid for a human analyst, not an autonomous decision. It can be wrong, and attacker-controlled log content is a genuine prompt-injection risk that this project mitigates but does not eliminate.
