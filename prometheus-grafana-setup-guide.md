# Prometheus + Grafana Monitoring Setup Guide (AWS EC2 / Amazon Linux 2023)

This guide sets up:
- **Server A** — runs Prometheus + Grafana (the monitoring server)
- **Server B** — the server you want to monitor (runs Node Exporter)

You need **two EC2 instances** (Amazon Linux 2023), each with a public IP and SSH access.

Replace these placeholders throughout:
- `MONITOR_SERVER_IP` → public IP of Server A (Prometheus + Grafana)
- `TARGET_SERVER_IP` → public IP of Server B (the server being monitored)

---

## Part 1 — Set up Server A (Prometheus + Grafana)

SSH into Server A and become root:
```bash
sudo su -
```

### 1.1 Update the system and install dependencies
```bash
sudo dnf update -y
sudo dnf install wget tar -y
```

### 1.2 (Recommended) Add swap space
Small instances (e.g. t2.micro / t3.micro, ~1GB RAM) can run out of memory installing Grafana. Add 2GB of swap to be safe:
```bash
dd if=/dev/zero of=/swapfile bs=1M count=2048
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile swap swap defaults 0 0' >> /etc/fstab
free -h   # confirm swap is active
```

### 1.3 Install Prometheus
```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.13.2/prometheus-3.13.2.linux-amd64.tar.gz
tar -xvf prometheus-3.13.2.linux-amd64.tar.gz
cd prometheus-3.13.2.linux-amd64
```

Set up a dedicated user and standard directories (so Prometheus survives reboots and isn't stuck in `/tmp`):
```bash
useradd --no-create-home --shell /bin/false prometheus
mkdir -p /etc/prometheus /var/lib/prometheus

cp prometheus /usr/local/bin/
cp promtool /usr/local/bin/
cp -r consoles /etc/prometheus/
cp -r console_libraries /etc/prometheus/
```

### 1.4 Create the Prometheus config
```bash
cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          app: "prometheus"

  - job_name: "node_exporter"
    static_configs:
      - targets: ["TARGET_SERVER_IP:9100"]
        labels:
          app: "node_exporter"
EOF
```
**Replace `TARGET_SERVER_IP` with Server B's public (or private) IP.**

```bash
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### 1.5 Run Prometheus as a systemd service
```bash
cat > /etc/systemd/system/prometheus.service << 'EOF'
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
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now prometheus
systemctl status prometheus
```
Confirm it's running and healthy:
```bash
curl http://localhost:9090/-/healthy
```

### 1.6 Install Grafana
```bash
cat > /etc/yum.repos.d/grafana.repo << 'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

dnf install grafana -y
systemctl enable --now grafana-server
systemctl status grafana-server
```
> If this install gets **"Killed"**, it's the OOM killer — make sure you did step 1.2 (swap) first, then retry.

### 1.7 Open required ports in Server A's Security Group
In AWS Console → EC2 → Server A's instance → **Security** tab → security group → **Edit inbound rules**, add:

| Type | Port | Source |
|---|---|---|
| Custom TCP | 9090 | Your IP (Prometheus UI) |
| Custom TCP | 3000 | Your IP (Grafana UI) |

---

## Part 2 — Set up Server B (the server to monitor)

SSH into Server B and become root:
```bash
sudo su -
```

### 2.1 Install Node Exporter
```bash
useradd --no-create-home --shell /bin/false node_exporter || true

cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64

cp node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### 2.2 Run Node Exporter as a systemd service
```bash
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
systemctl status node_exporter
```
Confirm it's serving metrics:
```bash
curl http://localhost:9100/metrics | head -5
```

### 2.3 Open port 9100 in Server B's Security Group
In AWS Console → EC2 → Server B's instance → **Security** tab → security group → **Edit inbound rules**, add:

| Type | Port | Source |
|---|---|---|
| Custom TCP | 9100 | `MONITOR_SERVER_IP/32` (Server A's IP only — **not** 0.0.0.0/0) |

> Scoping the source to Server A's IP specifically matters — node_exporter has no authentication and exposes system details, so it shouldn't be open to the whole internet.

---

## Part 3 — Connect Prometheus to Grafana

1. Open Grafana: `http://MONITOR_SERVER_IP:3000`
   Default login: `admin` / `admin` (you'll be asked to set a new password).

2. **Connections → Data sources → Add data source → Prometheus**
   - URL: `http://localhost:9090`
   - Click **Save & test** — should show a green success message.

3. **Dashboards → New → Import**
   - Enter dashboard ID: **`1860`** (Node Exporter Full)
   - Click **Load**
   - Select your Prometheus data source
   - Click **Import**

4. On the dashboard, use the **Instance** dropdown at the top to select `TARGET_SERVER_IP:9100` if it's not already selected.

You should now see live CPU, memory, disk, and network graphs for Server B.

---

## Part 4 — Verify everything

- Prometheus targets page: `http://MONITOR_SERVER_IP:9090/targets`
  Both `prometheus` and `node_exporter` jobs should show state **UP**.
- Grafana dashboard should show live-updating graphs (refreshes every ~15s–1min depending on dashboard settings).

---

## Troubleshooting quick reference

| Problem | Likely cause | Fix |
|---|---|---|
| `dnf install grafana` process gets "Killed" | Out of memory (OOM killer) | Add swap (step 1.2), retry |
| `node_exporter` target shows DOWN on Prometheus `/targets` | Security Group blocking port 9100 | Open port 9100 on Server B, source = Server A's IP |
| Can't reach Prometheus/Grafana UI in browser | Security Group blocking 9090/3000 | Open those ports on Server A for your IP |
| Config changes don't apply | Forgot to reload Prometheus | `systemctl reload prometheus` (or `kill -HUP <pid>` if run manually) |
| `curl: (7) Failed to connect` to localhost:9100 on Server A | Normal — node_exporter is only installed on Server B | Run the check on Server B instead |

---

## Adding more servers to monitor later

Repeat **Part 2** on each new server, then add another block to `/etc/prometheus/prometheus.yml` on Server A:
```yaml
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "TARGET_SERVER_IP:9100"
          - "ANOTHER_SERVER_IP:9100"
        labels:
          app: "node_exporter"
```
Then reload:
```bash
systemctl reload prometheus
```
