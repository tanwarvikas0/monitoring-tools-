# Install Prometheus on Amazon Linux 2023 (EC2)

## 1. Connect and prepare the server

``` bash
ssh -i your-key.pem ec2-user@YOUR_EC2_PUBLIC_IP
cat /etc/os-release
sudo dnf update -y
sudo dnf install wget tar -y
```

Check architecture:

``` bash
uname -m
```

The download commands below are for `x86_64`.

## 2. Create a Prometheus user and directories

``` bash
sudo useradd --no-create-home --shell /sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

## 3. Download and extract Prometheus

Check the [official Prometheus download
page](https://prometheus.io/download/) for the current stable version.
Replace `VERSION` below with that version number.

``` bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/vVERSION/prometheus-VERSION.linux-amd64.tar.gz
tar -xzf prometheus-VERSION.linux-amd64.tar.gz
cd prometheus-VERSION.linux-amd64
```

## 4. Install the binaries and supporting files

Run these commands from the extracted directory:

``` bash
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Verify:

``` bash
prometheus --version
promtool --version
```

## 5. Configure Prometheus

Create or edit the configuration:

``` bash
sudo vi /etc/prometheus/prometheus.yml
```

Add:

``` yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

Check the configuration:

``` bash
sudo promtool check config /etc/prometheus/prometheus.yml
```

Expected: `SUCCESS`.

## 6. Create a systemd service

``` bash
sudo vi /etc/systemd/system/prometheus.service
```

Paste:

``` ini
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
Restart=always

[Install]
WantedBy=multi-user.target
```

Start and enable:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

Expected: `active (running)`.

## 7. Verify Prometheus

``` bash
sudo ss -lntp | grep 9090
curl http://localhost:9090/-/healthy
curl http://localhost:9090/metrics
sudo journalctl -u prometheus -n 50 --no-pager
```

Open this URL in your browser:

``` text
http://YOUR_EC2_PUBLIC_IP:9090
```

In AWS, allow inbound TCP port `9090` in the EC2 security group,
preferably only from your own public IP (`YOUR_IP/32`). Do not expose
the Prometheus UI publicly.

In the Prometheus UI, open **Status → Targets**. The `prometheus` target
should show **UP**.

Try this query:

``` promql
up
```

A value of `1` means the target is reachable; `0` means it is down.

## 8. Useful troubleshooting commands

``` bash
sudo systemctl status prometheus
sudo systemctl restart prometheus
sudo journalctl -u prometheus -f
sudo promtool check config /etc/prometheus/prometheus.yml
curl -I http://localhost:9090
```

If the service fails, check the logs and configuration before changing
permissions or restarting repeatedly.

------------------------------------------------------------------------

# How to explain this in an interview (simple English)

**Question: Have you installed Prometheus? Explain the steps.**

"Yes. I installed Prometheus on an Amazon Linux EC2 server.

1.  First, I connected to the EC2 instance using SSH.
2.  I updated the server and installed the required utilities.
3.  I created a separate Linux user for Prometheus for security.
4.  I downloaded and extracted the Prometheus package.
5.  I copied the Prometheus binaries to `/usr/local/bin` and created the
    configuration under `/etc/prometheus`.
6.  In `prometheus.yml`, I configured the scrape interval and the target
    that Prometheus should monitor.
7.  I created a systemd service so Prometheus starts automatically after
    a server reboot.
8.  I started the service and checked that it was running.
9.  I opened port 9090 only for the required IP in the EC2 security
    group.
10. Finally, I opened the Prometheus web UI, checked **Status →
    Targets**, and ran the `up` query to verify monitoring."

**If the interviewer asks: What is the next step?**

"To monitor EC2 CPU, memory, disk and network metrics, I install Node
Exporter on the Linux server. It exposes host metrics on port 9100. I
add that endpoint as a target in Prometheus. Then I connect Grafana to
Prometheus to create dashboards and alerts."

## Important interview points

-   **Prometheus:** Collects and stores time-series metrics.
-   **Scrape interval:** How often Prometheus collects metrics.
-   **Prometheus port:** `9090`.
-   **Node Exporter port:** `9100`.
-   **Grafana port:** `3000` (default).
-   **`prometheus.yml`:** Main configuration file.
-   **systemd:** Runs Prometheus as a Linux service.
-   **`up = 1`:** Target is reachable; **`up = 0`:** Target is down.

**Remember:** Prometheus monitoring of itself does not automatically
provide complete EC2 host metrics. Node Exporter is needed for
host-level CPU, memory, disk and network metrics.
