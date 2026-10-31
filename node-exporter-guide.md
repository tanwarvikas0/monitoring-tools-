# Node Exporter on Amazon Linux 2023 with Prometheus

A practical guide to installing Node Exporter, running it as a systemd service, and connecting it to Prometheus on AWS EC2.

## Table of Contents

1. [What is Node Exporter](#1-what-is-node-exporter)
2. [Installation on Amazon Linux 2023](#2-installation-on-amazon-linux-2023)
3. [Create and Start the systemd Service](#3-create-and-start-the-systemd-service)
4. [Check Port 9100 and Metrics](#4-check-port-9100-and-metrics)
5. [Connect Node Exporter to Prometheus](#5-connect-node-exporter-to-prometheus)
6. [AWS Security Group Configuration](#6-aws-security-group-configuration)
7. [Useful Prometheus Queries](#7-useful-prometheus-queries)
8. [Troubleshooting a DOWN Target](#8-troubleshooting-a-down-target)
9. [Simple Interview Answers](#9-simple-interview-answers)

---

## 1. What is Node Exporter

Node Exporter is an official Prometheus exporter that exposes **hardware and OS-level metrics** of a Linux server, such as:

- CPU usage
- Memory (RAM) and swap
- Disk space and disk I/O
- Network traffic
- System load and uptime

It runs as a small background service and publishes metrics over HTTP on **port 9100** at the `/metrics` path. Prometheus **scrapes** (pulls) this endpoint at a fixed interval and stores the data for querying and alerting.

```
[EC2 server: Node Exporter :9100]  <--- scrape (pull) ---  [Prometheus :9090]  --->  Grafana / Alertmanager
```

---

## 2. Installation on Amazon Linux 2023

> Check the latest release at https://github.com/prometheus/node_exporter/releases and change `VERSION` if needed.

### Step 1: Update the system

```bash
sudo dnf update -y
```

### Step 2: Download and extract

```bash
cd /tmp
VERSION="1.8.2"
wget https://github.com/prometheus/node_exporter/releases/download/v${VERSION}/node_exporter-${VERSION}.linux-amd64.tar.gz
tar -xzf node_exporter-${VERSION}.linux-amd64.tar.gz
```

> For Graviton (ARM) instances, use `linux-arm64` instead of `linux-amd64`.

### Step 3: Move the binary

```bash
sudo cp node_exporter-${VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/node_exporter
```

### Step 4: Create a dedicated user

```bash
sudo useradd --no-create-home --shell /sbin/nologin node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### Step 5: Verify

```bash
node_exporter --version
```

---

## 3. Create and Start the systemd Service

### Create the service file

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Reload, enable, and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

### Check the status

```bash
sudo systemctl status node_exporter
```

You should see `active (running)`.

### View logs

```bash
sudo journalctl -u node_exporter -f
```

---

## 4. Check Port 9100 and Metrics

### Is the port listening?

```bash
sudo ss -tlnp | grep 9100
```

Expected output is similar to:

```
LISTEN 0 4096 *:9100 *:* users:(("node_exporter",pid=1234,fd=3))
```

### Fetch metrics locally

```bash
curl http://localhost:9100/metrics | head -20
```

### Search for a specific metric

```bash
curl -s http://localhost:9100/metrics | grep node_memory_MemAvailable_bytes
```

### Test from another server (e.g., the Prometheus server)

```bash
curl http://<NODE_EXPORTER_PRIVATE_IP>:9100/metrics | head
```

If this works from Prometheus but not from your laptop, that is normal when the Security Group only allows the Prometheus server.

---

## 5. Connect Node Exporter to Prometheus

Edit the Prometheus config on the Prometheus server:

```bash
sudo vi /etc/prometheus/prometheus.yml
```

Add a scrape job:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets:
          - "10.0.1.10:9100"   # server 1 (private IP)
          - "10.0.1.11:9100"   # server 2 (private IP)
        labels:
          env: "dev"
```

> Use **private IPs** when Prometheus and the targets are in the same VPC. It is faster, free of data transfer charges, and more secure.

### Validate the config (optional but recommended)

```bash
promtool check config /etc/prometheus/prometheus.yml
```

### Restart or reload Prometheus

```bash
sudo systemctl restart prometheus
# or, if started with --web.enable-lifecycle:
curl -X POST http://localhost:9090/-/reload
```

### Confirm the target is UP

Open `http://<PROMETHEUS_IP>:9090/targets` in your browser. The `node_exporter` job should show **UP**.

---

## 6. AWS Security Group Configuration

Node Exporter listens on **TCP 9100**. The Security Group on the **Node Exporter instance** must allow inbound traffic from Prometheus.

### On the Node Exporter EC2 instance

| Type       | Protocol | Port | Source                                       |
|------------|----------|------|----------------------------------------------|
| Custom TCP | TCP      | 9100 | Prometheus server's Security Group (or its private IP/32) |
| SSH        | TCP      | 22   | Your IP (for admin access)                   |

### On the Prometheus EC2 instance

| Type       | Protocol | Port | Source                          |
|------------|----------|------|---------------------------------|
| Custom TCP | TCP      | 9090 | Your IP (to open the UI)        |
| SSH        | TCP      | 22   | Your IP                         |

### Best practices

- Reference the **Security Group ID** of Prometheus as the source instead of IP addresses. It keeps working if the IP changes.
- **Never** open 9100 to `0.0.0.0/0`. Metrics expose internal system details.
- Prometheus needs no inbound rule for scraping, because it initiates the connection. Outbound traffic is allowed by default.
- Also check the **Network ACL** and any OS firewall if the connection still fails.

---

## 7. Useful Prometheus Queries

Run these in the Prometheus UI (`http://<PROMETHEUS_IP>:9090/graph`).

### Target health

```promql
up
up{job="node_exporter"} == 0
```

`1` means UP, `0` means DOWN.

### CPU usage (%)

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory usage (%)

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Available memory (GB)

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

### Disk usage (%) for the root filesystem

```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="tmpfs"} / node_filesystem_size_bytes{mountpoint="/",fstype!="tmpfs"}) * 100)
```

### Disk I/O (bytes per second)

```promql
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])
```

### Network traffic (bytes per second)

```promql
rate(node_network_receive_bytes_total{device!="lo"}[5m])
rate(node_network_transmit_bytes_total{device!="lo"}[5m])
```

### System load average

```promql
node_load1
node_load5
node_load15
```

### Uptime (hours)

```promql
(time() - node_boot_time_seconds) / 3600
```

### Example alert rule: instance down

```yaml
groups:
  - name: node-alerts
    rules:
      - alert: InstanceDown
        expr: up{job="node_exporter"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
```

---

## 8. Troubleshooting a DOWN Target

Work through these checks in order.

### Step 1: Read the error on the Targets page

Go to `http://<PROMETHEUS_IP>:9090/targets` and read the **Error** column. Common messages:

| Error message                         | Likely cause                                   |
|---------------------------------------|------------------------------------------------|
| `context deadline exceeded`           | Security Group / firewall blocking, or wrong IP |
| `connection refused`                  | Node Exporter not running or wrong port         |
| `no route to host`                    | Wrong IP, different VPC, or routing issue       |
| `server returned HTTP status 404`     | Wrong `metrics_path`                            |

### Step 2: Is Node Exporter running?

On the target server:

```bash
sudo systemctl status node_exporter
sudo journalctl -u node_exporter --no-pager | tail -30
```

If stopped:

```bash
sudo systemctl restart node_exporter
```

### Step 3: Is port 9100 listening?

```bash
sudo ss -tlnp | grep 9100
curl http://localhost:9100/metrics | head
```

### Step 4: Can Prometheus reach it?

On the **Prometheus server**:

```bash
curl -v http://<TARGET_PRIVATE_IP>:9100/metrics
```

If this hangs or times out, the problem is the network path:

- Security Group on the target does not allow 9100 from Prometheus
- Network ACL is blocking traffic
- OS firewall is blocking (`sudo iptables -L -n`)
- Instances are in different VPCs without peering

### Step 5: Check the Prometheus config

- Correct IP and port (`:9100`) in `prometheus.yml`
- No YAML indentation mistakes. Run `promtool check config /etc/prometheus/prometheus.yml`
- Prometheus was restarted or reloaded after the change

### Step 6: Check for IP changes

If the target instance was stopped and started, its **public IP** may have changed. Use a private IP or an Elastic IP.

### Quick checklist

- [ ] `node_exporter` service is active
- [ ] Port 9100 is listening
- [ ] `curl localhost:9100/metrics` works on the target
- [ ] Security Group allows 9100 from Prometheus
- [ ] `curl <target-ip>:9100/metrics` works from Prometheus
- [ ] Target IP in `prometheus.yml` is correct
- [ ] Prometheus was reloaded

---

## 9. Simple Interview Answers

**Q: What is Node Exporter?**
It is a Prometheus exporter that exposes Linux server metrics like CPU, memory, disk, and network on port 9100 so Prometheus can scrape them.

**Q: Does Prometheus push or pull metrics?**
Prometheus uses a **pull model**. It scrapes the `/metrics` endpoint of each target at a set interval.

**Q: Which port does Node Exporter use?**
Port **9100**. Prometheus itself uses 9090.

**Q: How do you run Node Exporter in production?**
As a **systemd service** under a dedicated non-login user, enabled to start on boot and set to restart on failure.

**Q: How do you add a new server to monitoring?**
Install Node Exporter on it, open port 9100 to Prometheus in the Security Group, add its `IP:9100` under `scrape_configs` in `prometheus.yml`, and reload Prometheus.

**Q: What does `up == 0` mean?**
Prometheus failed to scrape that target. The target is DOWN.

**Q: A target is DOWN. What do you check first?**
I check the error on the Targets page, then confirm the service is running, port 9100 is listening, and the Security Group allows traffic from Prometheus.

**Q: Which Security Group rule is needed?**
An inbound rule on the target for TCP 9100 with the Prometheus server's Security Group (or private IP) as the source. Never `0.0.0.0/0`.

**Q: Why use private IPs for targets?**
It is more secure, avoids data transfer costs, and keeps traffic inside the VPC.

**Q: How do you calculate CPU usage in PromQL?**
`100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

**Q: What is the difference between Prometheus and Grafana?**
Prometheus collects and stores metrics and runs queries and alerts. Grafana visualizes that data in dashboards.

**Q: What if a short-lived job needs to send metrics?**
Use the **Pushgateway**, since short-lived jobs may finish before Prometheus scrapes them.

---

## Quick Command Reference

```bash
# Service control
sudo systemctl start|stop|restart|status node_exporter
sudo journalctl -u node_exporter -f

# Port and metrics check
sudo ss -tlnp | grep 9100
curl http://localhost:9100/metrics | head

# Prometheus
promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```
