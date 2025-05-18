<img src="./assets/banner.png" alt="Ika Tools"/>

# 🐙 **Ika Tools**

This repository provides a powerful monitoring stack for Ika Validator and Sui Fullnode deployments, combining Prometheus for metric collection, Alertmanager for multi-channel notifications **(Telegram, Discord, Slack, PagerDuty)**, and Grafana for visualization. It includes ready-made scrape and alerting configurations, systemd service units, and alert rules targeting consensus progress, sync checks, and node availability. Finally, it automates Grafana provisioning of the dashboard, with a configurable authority variable to filter panels to your own validator.

Contributions are highly welcomed! Feel free to open issues and submit pull requests to help us improve this monitoring stuck.

### **Our public dashboard is available here:** https://grafana.trusted-point.com/public-dashboards/bfabcced297e4ffab624b2576dd9f61c

## 🚀 Future Plans
- Add Docker support for easy, containerized deployments.
- Enhance server monitoring, including system-level metrics and log collection.
- Integrate additional notification channels.


## 📚 Table of Contents

* [Introduction]()
* [Docker Setup - TBA]()
* [Manual Installation](#manual-installation)
  * [1. Prometheus](#1-prometheus)
  * [2. Grafana](#2-grafana)
  * [3. Alertmanager](#3-alertmanager)

# **Manual Installation**

## 1. Prometheus

### 1.0 Define your variables

```bash
VERSION="3.3.1"
ARCH="linux-amd64"
PROMETHEUS_HOST="127.0.0.1"
PROMETHEUS_PORT="9090"
SUI_METRICS_TARGET="127.0.0.1:9174"
IKA_METRICS_TARGET="127.0.0.1:9184"
```
### 1.1 Create a prometheus user and directories

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus /var/lib/prometheus
```
### 1.2 Download & unpack the latest Prometheus

```bash
cd /etc/prometheus
sudo wget https://github.com/prometheus/prometheus/releases/download/v${VERSION}/prometheus-${VERSION}.${ARCH}.tar.gz
tar xvf prometheus-${VERSION}.${ARCH}.tar.gz --strip-components=1
./prometheus --version && ./promtool --version
sudo cp ./prometheus ./promtool /usr/local/bin/
```

### 1.3 Create config (Update targets if needed)

```bash
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

scrape_configs:
  - job_name: "Ika-Validator"
    static_configs:
      - targets: ["${IKA_METRICS_TARGET}"]
        labels:
          group: 'IkaValidator'
          network: 'testnet'

  - job_name: "Sui-Fullnode"
    static_configs:
      - targets: ["${SUI_METRICS_TARGET}"]
        labels:
          group: 'Fullnode'
          network: 'testnet'
EOF
```

```bash
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

### 1.4 Create systemd unit

```bash
sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address="${PROMETHEUS_HOST}:${PROMETHEUS_PORT}"

[Install]
WantedBy=multi-user.target
EOF
```

### 1.5 Start & enable

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable prometheus && \
sudo systemctl restart prometheus && \
sudo journalctl -u prometheus -f -o cat
```

## 2. Grafana


### 2.1 Install prerequisites and add Grafana APT repository

```bash
sudo apt-get update
sudo apt-get install -y software-properties-common wget apt-transport-https
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
```

### 2.2 Install Grafana

```bash
sudo apt-get update
sudo apt-get install -y grafana
```

### 2.3 Enable and start Grafana service

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable grafana-server && \
sudo systemctl restart grafana-server && \
sudo journalctl -u grafana-server -f -o cat
```

### 2.4 Verify Grafana is running by visiting `<IP>:3000`

* Enter `admin` for both username and password and update the default password

### 2.5 Add Prometheus as a data source

**Option A:** Create a provisioning YAML so Grafana will load the Prometheus data source on startup:

```bash
sudo tee /etc/grafana/provisioning/datasources/prometheus.yaml > /dev/null << EOF
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://${PROMETHEUS_HOST}:${PROMETHEUS_PORT}
    isDefault: true
    editable: true
EOF
```

```bash
sudo systemctl restart grafana-server && sudo journalctl -u grafana-server -f -o cat
```

**Option B:** Grafana supports adding data sources via the UI or entirely from the terminal (CLI).

* Navigate to Configuration → Data Sources → Add data source.
* Select Prometheus, set the URL to `http://127.0.0.1:<PROMETHEUS_PORT>`, and click Save & Test.

### 2.6 Provision Ika Dashboard

**Option A:** Grafana can automatically load dashboards from JSON files. To fetch your dashboard and make it appear in the UI, do the following:

```bash
sudo mkdir -p /var/lib/grafana/dashboards
sudo chown grafana:grafana /var/lib/grafana/dashboards
```

```bash
sudo tee /etc/grafana/provisioning/dashboards/dashboard.yaml > /dev/null << EOF
apiVersion: 1

providers:
  - name: 'ika-sui'
    orgId: 1
    folder: ''
    type: 'file'
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
EOF
```

```bash
sudo wget -O /var/lib/grafana/dashboards/Trusted-Point-Ika-Sui.json https://raw.githubusercontent.com/trusted-point/Ika-Tools/refs/heads/main/grafana/dashboards/Grafana-TrustedPoint-Ika-Sui.json
```

```bash
sudo systemctl restart grafana-server && sudo journalctl -u grafana-server -f -o cat
```

**Option B:** Manual UI Import

* Log in to Grafana at `http://<IP>:3000` with your credentials.
* In the left-hand menu, click the + icon and select Import.
* Under Import via file, click Upload JSON file and choose the Trusted-Point-Ika-Sui.json file (you may download it locally first).
* Click Import.

### 2.7 Configure the authority variable

**After importing or provisioning the dashboard, you need to set your own validator’s authority so the panels query the correct series.**

1. Open the dashboard you just imported (e.g. IKA x SUI → Your Ika Dashboard).
2. Click the gear icon (⚙️) in the top-right corner of the dashboard and select Variables.
3. Find the authority variable in the list and click Edit.
4. In Custom options, replace the example `TrustedPoint` with your own validator name(s) exactly as they appear in the authority label of your Ika metrics (e.g. MyValidatorNode).
5. Click Update to save the variable definition, then click Dashboard > Refresh (⟳) to re-run all queries with your new authority value.
6. The dashboard panels will now display metrics for the specified authority label.

## 3. Alertmanager

### 3.0 Set up variables

**NOTE:** If you don’t want to enable some of the provided notification channels, simply comment out its corresponding entry both under the `route:` section and in the `receivers:` list. Also, ensure that the final route (PagerDuty) does not include continue: true, so that alerts aren’t passed on after routing to PagerDuty.

```bash
VERSION="0.28.1"
ARCH="linux-amd64"
ALERTMANAGER_HOST="127.0.0.1"
ALERTMANAGER_PORT="9093"
ALERTMANAGER_WEBHOOK_PORT="9092"
TELEGRAM_BOT_TOKEN="123:AAA-AAA"
TELEGRAM_CHAT_ID="-123"
DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/123/AAA-BBB-CCC"
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/AAA/BBB/CCC"
PAGERDUTY_INTEGRATION_KEY="123abc"
```

### 3.1 Create an alertmanager user and directories

```bash
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo mkdir /etc/alertmanager /var/lib/alertmanager
```

### 3.2 Download & unpack the latest Alertmanager

```bash
cd /etc/alertmanager
sudo wget https://github.com/prometheus/alertmanager/releases/download/v${VERSION}/alertmanager-${VERSION}.${ARCH}.tar.gz
sudo tar xvf alertmanager-${VERSION}.${ARCH}.tar.gz --strip-components=1
./alertmanager --version && ./amtool --version
sudo cp ./alertmanager ./amtool /usr/local/bin/
```

### 3.4 Set up Alertmanager config

```bash
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null << EOF
global:
  resolve_timeout: 5m

route:
  receiver: "default"
  group_by: ["alertname"]
  group_wait: 20s
  group_interval: 5m
  repeat_interval: 1h

  routes:
    - receiver: "telegram"
      continue: true

    - receiver: "discord"
      continue: true

    - receiver: "slack"
      continue: true

    - receiver: "pagerduty"

receivers:
  - name: "default"
    webhook_configs:
      - url: "http://localhost:${ALERTMANAGER_WEBHOOK_PORT}"

  - name: "telegram"
    telegram_configs:
      - send_resolved: true
        bot_token: "${TELEGRAM_BOT_TOKEN}"
        chat_id: ${TELEGRAM_CHAT_ID}

  - name: "discord"
    discord_configs:
      - send_resolved: true
        webhook_url: "${DISCORD_WEBHOOK_URL}"

  - name: "slack"
    slack_configs:
      - send_resolved: true
        api_url: "${SLACK_WEBHOOK_URL}"

  - name: "pagerduty"
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_INTEGRATION_KEY}"
        send_resolved: true
EOF
```

### 3.5 Adjust directory ownership

```bash
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```

### 3.6 Create systemd unit

```bash
sudo tee /etc/systemd/system/alertmanager.service << EOF
[Unit]
Description=Prometheus Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address="${ALERTMANAGER_HOST}:${ALERTMANAGER_PORT}"

[Install]
WantedBy=multi-user.target
EOF
```

### 3.7 Start & enable

```bash
sudo systemctl daemon-reload && \
sudo systemctl enable alertmanager && \
sudo systemctl restart alertmanager && \
sudo journalctl -u alertmanager -f -o cat
```

### 🚨 Ika Validator Alerts Overview

* ⚠️ **IkaNoLeaderRoundProgress** — triggered when `consensus_last_committed_leader_round` is stuck.
* 🧊 **IkaNoSyncFetchedIndexProgress** — triggered when `consensus_commit_sync_fetched_index` is stuck.
* 🔴 **IkaNodeDown** — triggered when node not reachable.

### 🚨 Sui Fullnode Alerts Overview

* ⚠️ **SuiHighestSyncedCheckpointStuck** — triggered when `highest_synced_checkpoint` is stuck.
* 🧊 **SuiLastExecutedCheckpointStuck** — triggered when `last_executed_checkpoint` is stuck.
* 🔴 **SuiNodeDown** — triggered when node not reachable.


### 3.8 Set up alert rules for Ika Validator

```bash
sudo tee /etc/prometheus/ika_validator_alert_rules.yml > /dev/null << 'EOF'
groups:
- name: ika_validator_alerts
  rules:

  - alert: IkaNoLeaderRoundProgress
    expr: increase(consensus_last_committed_leader_round{network="testnet",group="IkaValidator"}[10m]) == 0
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "consensus_last_committed_leader_round has been stuck for 10m"
      description: "The metric `consensus_last_committed_leader_round` has not increased in the last 10 minutes."

  - alert: IkaNoSyncFetchedIndexProgress
    expr: increase(consensus_commit_sync_fetched_index{network="testnet",group="IkaValidator"}[10m]) == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "consensus_commit_sync_fetched_index has been stuck for 10m"
      description: "The metric `consensus_commit_sync_fetched_index` has not increased in the last 10 minutes."
    
  - alert: IkaNodeDown
    expr: up{job="Ika-Validator"} == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Ika node is down"
      description: "The Ika validator target is not reachable (up == 0) for over 10 minutes."
EOF
```

### 3.9 Set up alert rules for Sui Fullnode

```bash
sudo tee /etc/prometheus/sui_fullnode_alert_rules.yml > /dev/null << 'EOF'
groups:
- name: sui_fullnode_alerts
  rules:

  - alert: SuiHighestSyncedCheckpointStuck
    expr: increase(highest_synced_checkpoint{job="Sui-Fullnode"}[10m]) == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Sui highest_synced_checkpoint has been stuck for 10m"
      description: "The metric `highest_synced_checkpoint` has not increased in the last 10 minutes."

  - alert: SuiLastExecutedCheckpointStuck
    expr: increase(last_executed_checkpoint{job="Sui-Fullnode"}[10m]) == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Sui last_executed_checkpoint has been stuck for 10m"
      description: "The metric `last_executed_checkpoint` has not increased in the last 10 minutes."

  - alert: SuiNodeDown
    expr: up{job="Sui-Fullnode"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Sui Fullnode is down"
      description: "The Sui fullnode target is not reachable (up == 0) for over 10 minutes."
EOF
```

### 3.10 Edit prometheus.yml adn restart prometheus unit to grab alerts rules
```bash
sudo sed -i '/^scrape_configs:/i \
rule_files:\
  - "sui_fullnode_alert_rules.yml"\
  - "ika_validator_alert_rules.yml"\
\
alerting:\
  alertmanagers:\
    - static_configs:\
        - targets:\
            - "alertmanager:'"${ALERTMANAGER_PORT}"'"\
' /etc/prometheus/prometheus.yml
```

```bash
sudo systemctl restart prometheus && \
sudo journalctl -u prometheus -f -o cat
```

### 3.11 Send a test alert to Alertmanager. Ensure you receive the message in the configured channels

```bash
curl -X POST http://127.0.0.1:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "critical"
      },
      "annotations": {
        "summary": "🔥 Test alert",
        "description": "This is just a test of all notification channels."
      }
    }
  ]'
```

### 3.12 Resolve the test alert in Alertmanager or wait a few minutes so Alertmanager resolves the alert itself

```bash
curl -X POST http://127.0.0.1:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[
    {
      "labels": {
        "alertname": "TestAlert",
        "severity": "critical"
      },
      "annotations": {
        "summary": "🔔 Test alert (resolving)",
        "description": "This marks the TestAlert as RESOLVED."
      }
    }
  ]'
```