# Blackbox Exporter in Prometheus

A simple guide to understand what Blackbox Exporter is, why we use it, and how to set it up.

---

## 📌 What is Blackbox Exporter?

**Blackbox Exporter** is a tool for Prometheus that checks if your websites, servers, and services are **working from the outside**.

It acts like a normal user. It tries to open your website, connect to your port, or ping your server, and then tells Prometheus:

- ✅ Is it up or down?
- ⏱️ How fast did it respond?
- 🔒 Is the SSL certificate about to expire?

It is called **"Blackbox"** because it does not look inside your application. It only checks what an outside user can see.

---

## 🤔 Blackbox vs Whitebox Monitoring

| | Blackbox Monitoring | Whitebox Monitoring |
|---|---|---|
| **What it checks** | Outside behavior (is it reachable?) | Inside details (CPU, memory, app metrics) |
| **Example** | "Can I open `https://example.com`?" | "How much memory is my app using?" |
| **Tool** | Blackbox Exporter | Node Exporter, app exporters |

👉 Best practice: use **both** together.

---

## ⚙️ How Does It Work?

```
┌────────────┐   1. scrape /probe    ┌───────────────────┐   2. check target   ┌──────────────┐
│ Prometheus │ ────────────────────► │ Blackbox Exporter │ ──────────────────► │ Website / IP │
│            │ ◄──────────────────── │                   │ ◄────────────────── │ / Port / DNS │
└────────────┘   4. send metrics     └───────────────────┘   3. get response   └──────────────┘
```

1. Prometheus asks Blackbox Exporter to check a target.
2. Blackbox Exporter tries to reach the target.
3. It records the result (success or failure, time taken, etc.).
4. Prometheus stores these metrics, and you can create dashboards and alerts.

---

## 🔍 What Can It Check? (Probes)

| Probe | What it does | Example use |
|---|---|---|
| **HTTP / HTTPS** | Opens a URL and checks the status code | Is my website up? |
| **TCP** | Connects to a port | Is my database port open? |
| **ICMP** | Sends a ping | Is my server reachable? |
| **DNS** | Checks DNS resolution | Is my domain resolving? |
| **gRPC** | Checks gRPC health | Is my gRPC service healthy? |

---

## 🚀 Installation

### Option 1: Docker (easiest)

```bash
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v $(pwd)/blackbox.yml:/config/blackbox.yml \
  prom/blackbox-exporter:latest \
  --config.file=/config/blackbox.yml
```

### Option 2: Binary

1. Download from the [official releases page](https://github.com/prometheus/blackbox_exporter/releases).
2. Extract and run:

```bash
tar xvfz blackbox_exporter-*.tar.gz
cd blackbox_exporter-*
./blackbox_exporter --config.file=blackbox.yml
```

It runs on port **9115** by default.

---

## 🛠️ Configuration

### Step 1: `blackbox.yml` (define modules)

A **module** is a set of rules for how to check a target.

```yaml
modules:
  # Check a website over HTTP/HTTPS
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []   # empty means any 2xx is OK
      method: GET

  # Check a TCP port
  tcp_connect:
    prober: tcp
    timeout: 5s

  # Ping a server
  icmp_check:
    prober: icmp
    timeout: 5s
```

### Step 2: `prometheus.yml` (tell Prometheus what to check)

```yaml
scrape_configs:
  - job_name: "blackbox-http"
    metrics_path: /probe
    params:
      module: [http_2xx]          # module name from blackbox.yml
    static_configs:
      - targets:
          - https://example.com
          - https://google.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115   # Blackbox Exporter address
```

> 💡 **Why relabeling?**
> Prometheus must talk to the Blackbox Exporter, not directly to your website.
> The relabeling passes your website URL as `?target=...` to the exporter.

Restart Prometheus after saving the changes.

---

## 🧪 Test It Manually

Open this in your browser or use `curl`:

```bash
curl "http://localhost:9115/probe?target=https://example.com&module=http_2xx"
```

You will see metrics like:

```
probe_success 1
probe_duration_seconds 0.25
probe_http_status_code 200
```

---

## 📊 Important Metrics

| Metric | Meaning |
|---|---|
| `probe_success` | `1` = check passed, `0` = check failed |
| `probe_duration_seconds` | Total time the probe took |
| `probe_http_status_code` | HTTP status code (200, 404, 500, etc.) |
| `probe_http_duration_seconds` | Time spent in each phase (DNS, connect, TLS, etc.) |
| `probe_ssl_earliest_cert_expiry` | When the SSL certificate expires (Unix timestamp) |
| `probe_dns_lookup_time_seconds` | Time taken to resolve DNS |

---

## 🚨 Example Alert Rules

```yaml
groups:
  - name: blackbox-alerts
    rules:

      # Alert when a website is down
      - alert: WebsiteDown
        expr: probe_success == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} is down"

      # Alert when the site is slow
      - alert: WebsiteSlow
        expr: probe_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} is responding slowly"

      # Alert when SSL certificate expires in less than 14 days
      - alert: SSLCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 14
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate for {{ $labels.instance }} expires in less than 14 days"
```

---

## ✅ Common Use Cases

- Website uptime monitoring
- API endpoint health checks
- SSL certificate expiry monitoring
- Checking if a server is reachable (ping)
- Checking if a database or service port is open
- DNS monitoring
- Monitoring from different locations or networks

---

## 💡 Tips and Best Practices

- Run Blackbox Exporter from a **different network** than your app to get a true outside view.
- Use **`for:`** in alerts (like `for: 1m`) to avoid false alarms from short glitches.
- Keep **timeouts** smaller than your scrape interval.
- Use **Grafana** to build dashboards (popular dashboard IDs: `7587`, `13659`).
- ICMP probes may need extra permissions (root or `NET_RAW` capability) on some systems.

---

## 🔗 Useful Links

- [Blackbox Exporter GitHub](https://github.com/prometheus/blackbox_exporter)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Example Configurations](https://github.com/prometheus/blackbox_exporter/blob/master/example.yml)

---

## 📝 Summary

> **Blackbox Exporter** = a tool that checks your websites and services **from outside**, like a real user, and sends the results to **Prometheus** so you can monitor, visualize, and get alerts.
