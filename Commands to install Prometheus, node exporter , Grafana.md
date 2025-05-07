## 🚀 Step-by-Step: Install Prometheus on EC2 (Ubuntu)

---

### ✅ 1. Launch an EC2 Instance

* **OS:** Ubuntu 22.04
* **Instance Type:** t2.micro or larger
* **Security Group:** Open inbound port **9090** (Prometheus web UI) and **22** (SSH)
* **Storage:** 8GB or more
* Tag: `Name=prometheus-server`

---

### 🔑 2. Connect to EC2 via SSH

```bash
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

---

### 🛠️ 3. Install Prometheus

```bash
# Update and install wget
sudo apt update
sudo apt install -y wget

# Create Prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus

# Create directories
sudo mkdir /etc/prometheus /var/lib/prometheus

# Download latest Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz

# Extract and move files
tar xvf prometheus-2.52.0.linux-amd64.tar.gz
cd prometheus-2.52.0.linux-amd64/

sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/

# Set ownership
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

---

### ⚙️ 4. Create a Systemd Service

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Paste:

```ini
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

---

### ▶️ 5. Start Prometheus

```bash
# Reload systemd and start the service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus

# Check status
systemctl status prometheus
```

---

### 🌐 6. Access Prometheus UI

Open your browser:

```
http://<EC2-PUBLIC-IP>:9090
```

Make sure port **9090** is open in your EC2 Security Group.

---

## 📦 Optional: Add EC2 Metrics Exporter (node\_exporter)

If you want Prometheus to monitor the EC2 instance itself:

```bash
# Download and install node_exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo nano /etc/systemd/system/node_exporter.service
```

Paste:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
```

Start and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

Access node metrics:

```
http://<EC2-PUBLIC-IP>:9100/metrics
```

Then update `prometheus.yml` to scrape it:

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

---

## ✅ Done!

---

## 🎯 Final Stack on EC2

* ✅ **Prometheus** – Collects metrics from Node Exporter
* ✅ **Node Exporter** – Exposes system metrics to Prometheus
* ✅ **Alertmanager** – Sends alerts (email, Slack, etc.)
* ✅ **Grafana** – Visualizes metrics

---

## 📦 Step-by-Step: Install Grafana on EC2

### 🔹 1. Install Grafana (on same EC2 or a separate one)

```bash
# Add Grafana APT repo
sudo apt install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Install Grafana
sudo apt update
sudo apt install -y grafana

# Start Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### 🌐 2. Access Grafana

Open your browser:

```
http://<EC2-PUBLIC-IP>:3000
```

* Username: `admin`
* Password: `admin` (you’ll be prompted to change it)

---

### ➕ 3. Add Prometheus as a Data Source in Grafana

* Go to **Settings → Data Sources → Add data source**
* Choose **Prometheus**
* URL: `http://localhost:9090` (if on same EC2) or use the private IP if Prometheus is on another instance.
* Click **Save & Test**

Now you can start building dashboards! 🎨

---

## 🚨 Step-by-Step: Install Alertmanager

### 🔹 1. Download Alertmanager

```bash
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64

# Move files
sudo mv alertmanager amtool /usr/local/bin/
sudo mkdir /etc/alertmanager
sudo mv alertmanager.yml /etc/alertmanager/
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo chown -R alertmanager:alertmanager /etc/alertmanager
```

### 🔧 2. Create Alertmanager Systemd Service

```bash
sudo nano /etc/systemd/system/alertmanager.service
```

Paste:

```ini
[Unit]
Description=Alertmanager Service
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

[Install]
WantedBy=multi-user.target
```

```bash
sudo mkdir /var/lib/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager

sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
```

### 🌐 3. Access Alertmanager

Open your browser:

```
http://<EC2-PUBLIC-IP>:9093
```

---

## 🔗 4. Integrate Alertmanager with Prometheus

Update your `/etc/prometheus/prometheus.yml` on the Prometheus server:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]
```

Also, add `rule_files:` and create alert rules (e.g., `alerts.yml`):

```yaml
rule_files:
  - "alerts.yml"
```

Sample `alerts.yml`:

```yaml
groups:
- name: example
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

---

## ✅ Summary of Ports to Open in Security Group

| Component     | Port |
| ------------- | ---- |
| Prometheus    | 9090 |
| Node Exporter | 9100 |
| Grafana       | 3000 |
| Alertmanager  | 9093 |
| SSH           | 22   |

---

