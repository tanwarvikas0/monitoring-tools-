# Quick Setup: Elasticsearch & Kibana on CentOS / RHEL

A concise copy-paste installation cheatsheet for RPM-based Linux servers.

---

## 1. Add Elastic Repository

```bash
# 1. Import GPG Key
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# 2. Add Repo
sudo tee /etc/yum.repos.d/elasticsearch.repo <<EOF
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

---

## 2. Install & Start Elasticsearch

```bash
# Install package
sudo dnf install -y elasticsearch

# Start & enable on boot
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
```

> **Note:** Save the generated `elastic` superuser password displayed in your terminal output.

---

## 3. Install & Start Kibana

```bash
# Install package
sudo dnf install -y kibana

# Allow external access (optional)
sudo sed -i 's/#server.host: "localhost"/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml

# Start & enable on boot
sudo systemctl enable --now kibana
```

---

## 4. Connect Kibana to Elasticsearch

1. **Generate enrollment token:**
   ```bash
   sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
   ```
2. Open `http://<SERVER_IP>:5601` in your browser and paste the token.
3. **Generate verification code:**
   ```bash
   sudo /usr/share/kibana/bin/kibana-verification-code
   ```
4. Enter the code in the browser and log in with user `elastic`.

---

## 5. Firewall Configuration (firewalld)

```bash
sudo firewall-cmd --add-port={9200,5601}/tcp --permanent
sudo firewall-cmd --reload
```