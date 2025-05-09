To configure **Alertmanager** to send alerts via **personal email** and then **Slack**, follow these detailed steps:

---

## 🔧 Prerequisites

1. Prometheus and Alertmanager installed and running.
2. Internet access from the EC2 instance (for SMTP/Slack webhooks).
3. A Gmail (or other SMTP-enabled) email account.
4. Slack workspace + access to create a webhook.

---

## 🔁 Workflow Overview

1. Configure **Prometheus** to send alerts to **Alertmanager**.
2. Configure **Alertmanager** with:

   * **Gmail SMTP** (or other SMTP) for personal email.
   * **Slack webhook URL** for Slack alerts.
3. Define **alert rules** in Prometheus.

---

## 📁 Folder Structure Suggestion

```
/etc/
├── prometheus/
│   ├── prometheus.yml
│   └── alert.rules.yml
└── alertmanager/
    └── alertmanager.yml
```

---

## 🔧 Step 1: Configure Alertmanager for Email

### 📄 `/etc/alertmanager/alertmanager.yml`

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your.email@gmail.com'
  smtp_auth_username: 'your.email@gmail.com'
  smtp_auth_password: 'your_app_password'
  smtp_require_tls: true

route:
  receiver: 'email-notifications'
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 1h

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'your.email@gmail.com'
        send_resolved: true
```

> ⚠️ **Important:** Gmail no longer allows basic password auth. You must [generate an App Password](https://support.google.com/accounts/answer/185833) and use that in `smtp_auth_password`.

---

## 🔧 Step 2: Add Slack Notifications to Alertmanager

### 1. **Create Slack Webhook**:

* Go to your Slack workspace: `https://<your-workspace>.slack.com/apps`
* Search for **Incoming Webhooks**
* Click **Add to Slack**
* Choose channel → Click **Add Incoming Webhook**
* Copy the **Webhook URL**

### 2. **Modify `/etc/alertmanager/alertmanager.yml`**:

```yaml
global:
  # (same as before)

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 1h
  receiver: 'email-notifications'

  routes:
    - match:
        severity: slack
      receiver: 'slack-notifications'

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'your.email@gmail.com'
        send_resolved: true

  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXXXX/XXXXX/XXXXXXXX'
        channel: '#alerts'
        send_resolved: true
```

---

## 🧠 Step 3: Define Alert Rules in Prometheus

### 📄 `/etc/prometheus/alert.rules.yml`

```yaml
groups:
  - name: instance-alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
```

---

## 🧾 Step 4: Point Prometheus to Alertmanager & Load Rules

### 📄 `/etc/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']  # default Alertmanager port

rule_files:
  - "alert.rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

---

## 🔄 Step 5: Reload and Restart Services

```bash
# Check Prometheus config
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/alert.rules.yml

# Restart Alertmanager
sudo systemctl restart alertmanager

# Restart Prometheus
sudo systemctl restart prometheus

# Check status
sudo systemctl status alertmanager
sudo systemctl status prometheus
```

---

## ✅ Step 6: Test Alert

* Stop Prometheus temporarily:

  ```bash
  sudo systemctl stop prometheus
  ```
* Wait a minute to trigger the alert.
* Check:

  * Slack channel
  * Email inbox
* Restart Prometheus:

  ```bash
  sudo systemctl start prometheus
  ```

---

## ✅ Tips

* Use different `severity` labels to route to email, Slack, PagerDuty, etc.
* Use `send_resolved: true` for resolution notifications.
* Check Alertmanager UI: `http://<your-server-ip>:9093` for alert status.

---


Here's a **complete and detailed guide** to configure **Alertmanager** with both:

1. **Gmail SMTP (or any SMTP provider)** to send alerts to your **personal email**
2. **Slack Webhook** to send alerts to a **Slack channel**

---

## ✅ Final Goal

Prometheus → Alertmanager → Email & Slack Notifications

---

## 📁 File: `/etc/alertmanager/alertmanager.yml`

```yaml
global:
  # EMAIL (Gmail SMTP)
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your.email@gmail.com'
  smtp_auth_username: 'your.email@gmail.com'
  smtp_auth_password: 'your_app_password'  # NOT your Gmail password
  smtp_require_tls: true

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 30s
  repeat_interval: 1h
  receiver: 'email-notifications'  # default receiver

  routes:
    - match:
        severity: 'slack'
      receiver: 'slack-notifications'

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'your.email@gmail.com'
        send_resolved: true

  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts'
        send_resolved: true
```

---

## ✉️ Step-by-Step: Gmail SMTP Configuration

### 🔐 Get a Gmail App Password (don't use your real password!)

1. Go to: [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. Generate an app password (select **Mail** and **Other**, name it “Alertmanager”)
3. Use this app password in `smtp_auth_password`

> 💡 If you don’t see *App Passwords*, enable 2-Step Verification first.

---

## 💬 Step-by-Step: Slack Webhook Configuration

1. Go to: [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)
2. Click **Create a Webhook**
3. Choose your channel (e.g. `#alerts`)
4. Copy the generated **Webhook URL**
5. Paste it in `api_url` under `slack_configs`

---

## 🔍 Example Alert Rule (Prometheus Side)

### 📄 `/etc/prometheus/alert.rules.yml`

```yaml
groups:
  - name: example-alert
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} is not responding."
```

### Then include it in Prometheus config:

📄 `/etc/prometheus/prometheus.yml`

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "alert.rules.yml"
```

---

## 🔄 Reload/Restart Services

```bash
sudo systemctl restart alertmanager
sudo systemctl restart prometheus
```

---

## 🔎 Test the Setup

You can test:

* **Email**: Temporarily stop Prometheus → wait for 1 minute → check inbox.
* **Slack**: Same test, check your Slack channel.

---

## 🛠 Troubleshooting

| Issue             | Solution                                                     |
| ----------------- | ------------------------------------------------------------ |
| No email received | Ensure App Password is correct, port 587 is open, check logs |
| No Slack alert    | Double-check webhook URL and match `severity: slack` in rule |
| Invalid YAML      | Use `yamllint` or `promtool check config`                    |
| Alert not firing  | Ensure alert rule condition (like `up == 0`) is actually met |

---

Would you like a downloadable `.yml` bundle or GitHub-ready directory structure?


Absolutely! Here's a complete **beginner-friendly, step-by-step guide** to configure Slack for receiving Alertmanager alerts — from creating a Slack account to integrating Slack with Prometheus Alertmanager using webhooks.

---

## ✅ Part 1: Create a Slack Account and Workspace

### 🔹 Step 1: Sign Up

1. Go to [https://slack.com/get-started](https://slack.com/get-started)
2. Click **"Create a new workspace"**
3. Enter your **email address**, verify the code sent to your email
4. Give your **workspace a name** (e.g., `DevOps Alerts`)
5. Give your **project or team name** (e.g., `Monitoring`)
6. Create a **Slack channel** (e.g., `#alerts`)
7. Invite teammates (you can skip this if it’s just you)

Now you're inside your **Slack workspace**!

---

## ✅ Part 2: Create a Slack Webhook URL

### 🔹 Step 2: Enable Incoming Webhooks

1. Visit: [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **“Create New App”**
3. Choose **“From scratch”**
4. App Name: `Alertmanager` (or any name you like)
5. Select your **workspace**, then click **Create App**

---

### 🔹 Step 3: Activate Webhooks Feature

1. In the left sidebar, click **“Incoming Webhooks”**
2. Toggle the switch **“Activate Incoming Webhooks”** to ON

---

### 🔹 Step 4: Add a Webhook to a Channel

1. Scroll down and click **“Add New Webhook to Workspace”**
2. Select the **channel** (e.g., `#alerts`) you created earlier
3. Click **“Allow”**
4. You’ll now get a **Webhook URL**, e.g.:

   ```
   https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
   ```
5. **Copy this URL** — you'll need it in Alertmanager config

---

## ✅ Part 3: Configure Alertmanager for Slack

### 🔹 Step 5: Add to `alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: slack-notifications

receivers:
  - name: slack-notifications
    slack_configs:
      - send_resolved: true
        username: 'Alertmanager'
        channel: '#alerts'
        api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
```

> 🔒 **Replace the `api_url` with your real Slack Webhook URL**.

---

### 🔹 Step 6: Restart Alertmanager

```bash
# If running directly
./alertmanager --config.file=alertmanager.yml

# Or, if running via systemd
sudo systemctl restart alertmanager
```

---

### 🔹 Step 7: Test It

1. Trigger an alert from Prometheus (e.g., fake a CPU alert)
2. Watch Slack for messages in `#alerts`

---

Would you like help testing this setup with a sample Prometheus alert rule to confirm everything works?
