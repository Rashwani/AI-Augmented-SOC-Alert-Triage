# Setup Guide

Full build steps for the AI-Augmented SOC Alert Triage lab, from a fresh AWS account to a running pipeline. Commands assume an Ubuntu EC2 host and a Windows local machine (PowerShell).

---

## 1. AWS account and billing guard

If you don't have an account, create one at [aws.amazon.com](https://aws.amazon.com). Then set a safety net so you're never surprised by a charge:

- Console → search **Billing** → **Budgets** → **Create budget** → **Zero spend budget** → enter your email.

## 2. Launch the EC2 instance

Console → **EC2** → **Launch instance**.

- **Name:** `wazuh-soc-lab`
- **AMI:** Ubuntu Server 24.04 LTS (22.04 also fine)
- **Instance type:** `t3.large` (2 vCPU, 8 GB RAM). The all-in-one Wazuh stack will not run on a micro instance. Stop the instance when you're done each day to conserve credits.
- **Key pair:** Create new, name it `wazuh-soc-lab-key`. The `.pem` file downloads once — keep it safe.
- **Network / security group:** allow **SSH (22)** and **HTTPS (443)**, both with source **My IP**.
- **Storage:** **50 GB gp3**. The 8 GB default fills up and breaks the dashboard install.

Launch, wait ~2 minutes, and copy the **Public IPv4 address**.

## 3. Connect from Windows

Open PowerShell in the folder containing your `.pem` file:

```powershell
icacls .\wazuh-soc-lab-key.pem /inheritance:r /grant:r "$($env:USERNAME):(R)"
ssh -i .\wazuh-soc-lab-key.pem ubuntu@YOUR_PUBLIC_IP
```

`icacls` locks the key file down so SSH will accept it (the Windows equivalent of `chmod 400`). Type `yes` when prompted. You're connected when the prompt reads `ubuntu@ip-...`.

## 4. Install Wazuh

```bash
sudo apt update && sudo apt -y upgrade
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

This installs the manager, indexer, and dashboard on one host (10–15 minutes). **Copy the admin password it prints at the end — it is shown only once.** If you miss it:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt | grep -A1 "'admin'"
```

Stop apt from upgrading Wazuh unexpectedly:

```bash
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
```

Open `https://YOUR_PUBLIC_IP` in your browser, click through the certificate warning, and log in as **admin**.

## 5. Agent

Nothing to do. The manager already monitors itself as **agent 000**. Installing `wazuh-agent` on the same host causes conflicts on Ubuntu.

## 6. Enable FIM and generate test alerts

Enable realtime File Integrity Monitoring on a test directory:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Inside the `<syscheck>` block, set `<frequency>300</frequency>` and add:

```xml
<directories realtime="yes" check_all="yes">/home/ubuntu/fim-test</directories>
```

Create the folder **before** restarting (realtime won't attach to a non-existent folder), then restart:

```bash
mkdir -p ~/fim-test
sudo systemctl restart wazuh-manager
```

Generate a spread of alerts:

```bash
# File integrity
echo hello > ~/fim-test/a.txt

# SSH brute force
for i in $(seq 1 10); do ssh -o BatchMode=yes -o StrictHostKeyChecking=no wronguser@localhost; done

# New user / group (persistence)
sudo useradd -m hacker1 && sudo userdel -r hacker1
sudo groupadd backdoor && sudo groupdel backdoor

# Failed privilege escalation
for i in 1 2 3; do sudo -k; echo "wrongpassword" | sudo -S whoami 2>/dev/null; done

# Package change
sudo apt install -y cowsay && sudo apt remove -y cowsay
```

Check the Wazuh dashboard's **Threat Hunting** section — alerts appear within a minute or two.

## 7. Allow the script to read alerts without root

Wazuh restricts the alerts log. Add yourself to the `wazuh` group instead of running the script as root:

```bash
sudo usermod -aG wazuh ubuntu
exit        # you MUST reconnect for the group change to take effect
```

Reconnect, then confirm:

```bash
tail -n 3 /var/ossec/logs/alerts/alerts.json
```

It should print without a "Permission denied" error.

## 8. Python environment

```bash
sudo apt install -y python3-pip python3-venv
mkdir -p ~/ai-triage && cd ~/ai-triage
python3 -m venv venv && source venv/bin/activate
pip install anthropic
```

The `(venv)` prefix in your prompt means it's active. Re-run `source venv/bin/activate` each time you reconnect.

## 9. Anthropic API key

On your own computer, go to [console.anthropic.com](https://console.anthropic.com):

- **Billing** → add a few dollars of credit and set a monthly spend limit.
- **API Keys** → **Create Key** → name it `soc-triage-lab` → copy it (shown only once).

On the server, save it to a `.env` file (keep the prefix, no quotes):

```bash
nano ~/ai-triage/.env
```

```
ANTHROPIC_API_KEY=sk-ant-your-full-key-here
```

```bash
chmod 600 ~/ai-triage/.env
```

## 10. Add the triage script

Copy [`triage.py`](triage.py) into `~/ai-triage/`, then verify it parsed cleanly:

```bash
cd ~/ai-triage && source venv/bin/activate
python3 -m py_compile triage.py && echo OK
```

## 11. Run the pipeline

**First terminal** — load the key and start the watcher:

```bash
cd ~/ai-triage && source venv/bin/activate
set -a; . ./.env; set +a
python3 triage.py
```

It prints `Watching for new Wazuh alerts...` and waits. That's correct.

**Second terminal** — open a new PowerShell window, SSH in again, and fire some alerts:

```bash
for i in $(seq 1 10); do ssh -o BatchMode=yes -o StrictHostKeyChecking=no wronguser@localhost; done
```

The first terminal prints a severity for each alert. Read the saved report:

```bash
cat ~/ai-triage/triage_report.jsonl
```

## 12. (Optional) Run as a background service

Stop the script (Ctrl+C), then:

```bash
sudo nano /etc/systemd/system/ai-triage.service
```

```ini
[Unit]
Description=AI SOC Alert Triage
After=network-online.target wazuh-manager.service

[Service]
User=ubuntu
SupplementaryGroups=wazuh
WorkingDirectory=/home/ubuntu/ai-triage
EnvironmentFile=/home/ubuntu/ai-triage/.env
ExecStart=/home/ubuntu/ai-triage/venv/bin/python3 /home/ubuntu/ai-triage/triage.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now ai-triage
journalctl -u ai-triage -f
```

The key stays in `.env` rather than hardcoded in the service file.

## Cost & safety checklist

- Keep the zero-spend budget alert active.
- **Stop** (not terminate) the EC2 instance when you're done for the day to conserve free-tier hours — software and data are preserved.
- Set a monthly spend limit on your Anthropic account.
- Keep security group rules restricted to **My IP**, never `0.0.0.0/0`.
- Never commit your `.pem` file or API key — the included `.gitignore` covers both.
