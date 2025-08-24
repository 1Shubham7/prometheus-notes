```
wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
```

```
tar -xvzf prometheus-3.5.0.linux-amd64.tar.gz
```
```
sudo mv prometheus promtool /usr/local/bin/
sudo mkdir /etc/prometheus
sudo mv prometheus.yml /etc/prometheus/
```

in new promethues, console and console_lib is part of the binary itself so no need to do this - 
`sudo mv consoles/ console_libraries/ /etc/prometheus/`

```
sudo mkdir /var/lib/prometheus
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

```
vim nano /etc/systemd/system/prometheus.service
```
```
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

```
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

```
sudo systemctl status prometheus
```

```
curl http://localhost:9090
```

in target install node_exporter 

```
sudo mv node_exporter /usr/local/bin/
```
```
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```
```
sudo vim /etc/systemd/system/node_exporter.service
```
```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```
```
sudo systemctl status node_exporter
```
```
curl http://localhost:9100/metrics
```

in that `/etc/prometheus/prometheus.yml`

```
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["node01:9100", "node02:9100"]
```

```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

---

## Setting up encryption and authentication:

in the target servers

```
sudo apt-get update
sudo apt install -y apache2-utils
```

```
mkdir -p /etc/node_exporter
touch /etc/node_exporter/config.yml
chown -R node_exporter:node_exporter /etc/node_exporter
```

```
vim /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file=/etc/node_exporter/config.yml

[Install]
WantedBy=multi-user.target
```

```
 htpasswd -nBC 10 "" | tr -d ':\n'; echo
```
```
vim  /etc/node_exporter/config.yml

basic_auth_users:
  prometheus: $2y$10$LXJynYLJAwTBAVESqlWYf.rC7vEGEmCoZDDhU0tiqO3GDZ64AsXtu
```

```
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
sudo systemctl status node_exporter
```

In Prometheus server:

```
vim /etc/prometheus/prometheus.yml

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "nodes"
    static_configs:
      - targets: ["node01:9100", "node02:9100"]
    basic_auth:   
      username: "prometheus"
      password: "secret-password"
```

```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```
