# Install Prometheus, node exporter, alertmanager, blackbox

### Add users and group

```
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```

### Create configuration and data directories

```
sudo mkdir /var/lib/prometheus
for i in rules rules.d files_sd; do sudo mkdir -p /etc/prometheus/${i}; done
```

### Download and Install Prometheus

```
sudo apt-get -y install wget
mkdir -p /tmp/prometheus && cd /tmp/prometheus
curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest \
  | grep browser_download_url \
  | grep linux-amd64 \
  | cut -d '"' -f 4 \
  | wget -qi -
```

### Extract the file

```
tar xvf prometheus*.tar.gz
cd prometheus*/
```

### Move the prometheus binary files to /usr/local/bin/

```
sudo mv prometheus promtool /usr/local/bin/
sudo mv prometheus.yml  /etc/prometheus/prometheus.yml
sudo mv consoles/ console_libraries/ /etc/prometheus/
cd ~/
rm -rf /tmp/prometheus
```

### Create/Edit a Prometheus configuration file

```
sudo vim /etc/prometheus/prometheus.yml
```

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

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
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```

### Create a Prometheus systemd Service unit file

```
sudo tee /etc/systemd/system/prometheus.service<<EOF

[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### Change directory permissions

```
for i in rules rules.d files_sd; do sudo chown -R prometheus:prometheus /etc/prometheus/${i}; done
for i in rules rules.d files_sd; do sudo chmod -R 775 /etc/prometheus/${i}; done
sudo chown -R prometheus:prometheus /var/lib/prometheus/
```

### Reload systemd daemon and start the service

```
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

```
systemctl status prometheus
```

Access Prometheus web interface on URL http://[ip_hostname]:9090


##  Install node_exporter 

```
curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest \
| grep browser_download_url \
| grep linux-amd64 \
| cut -d '"' -f 4 \
| wget -qi -

```

Extract downloaded file and move the binary file to /usr/local/bin

```
tar -xvf node_exporter*.tar.gz
cd  node_exporter*/
sudo cp node_exporter /usr/local/bin
```

Confirm installation.

```
node_exporter --version
```

### Create node_exporter service

```
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF
```

Reload systemd and start the service

```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
systemctl status node_exporter.service
```

### Add the node_exporter to the Prometheus server

```
sudo vim /etc/prometheus.yml
```

```
- job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

```
sudo systemctl restart prometheus
```


## Install TinyProxy

```
apt-get install tinyproxy -y
```

Create user

```
sudo useradd -rs /bin/false tinyproxy
```

```
nano /etc/tinyproxy/tinyproxy.conf
```


```
User tinyproxy
Group tinyproxy
Port 43210
#Allow 0.0.0.0/0      # Allow for all - NOT SECURELY
Allow 44.55.31.22    # Allow for specific ip address - SECURELY   
```

### Open port

```
ufw allow 43210/tcp
```

### Restart proxy

```
sudo systemctl restart tinyproxy
```

### Test

```
curl --proxy "http://127.0.0.1:43210" "http://httpbin.org/ip" -k
```



## Install Blackbox module

```
cd tmp && wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
```

```
tar xvzf blackbox_exporter-0.18.0.linux-amd64.tar.gz
```

```
cd blackbox_exporter-0.18.0.linux-amd64 && sudo mv blackbox_exporter /usr/local/bin/
```

### Create configuration folders for your blackbox exporter

```
sudo mkdir -p /etc/blackbox
sudo mv blackbox.yml /etc/blackbox
```

### Create a user account for the Blackbox exporter

```
sudo useradd -rs /bin/false blackbox
sudo chown blackbox:blackbox /usr/local/bin/blackbox_exporter
sudo chown -R blackbox:blackbox /etc/blackbox/*
```

### Create the Blackbox exporter service

```
cd /lib/systemd/system
sudo touch blackbox.service
sudo nano blackbox.service
```

```
[Unit]
Description=Blackbox Exporter Service
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=blackbox
Group=blackbox
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox/blackbox.yml \
  --web.listen-address=":9115"

Restart=always

[Install]
WantedBy=multi-user.target
```

### Daemon reload and service start

```
sudo systemctl daemon-reload
sudo systemctl enable blackbox.service
sudo systemctl start blackbox.service
```

Test metrics

```
curl http://localhost:9115/metrics
```

### Binding the Blackbox exporter with Prometheus

```
sudo nano /etc/prometheus/prometheus.yml

global:
  scrape_interval:     15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090', 'localhost:9115']
```

```
sudo systemctl restart prometheus
```

### Creating a Blackbox module AND module https_external_2xx with TinyProxy - proxy_url http://127.0.0.1:43210

```
modules:
  http_2xx:
    prober: http
  https_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      fail_if_ssl: false
      fail_if_not_ssl: true
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200]
      no_follow_redirects: false
      preferred_ip_protocol: "ip4"
  https_external_2xx:
    prober: http
    timeout: 5s
    http:
      method: GET
      fail_if_ssl: false
      fail_if_not_ssl: true
      valid_http_versions: ["HTTP/1.0", "HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200]
      no_follow_redirects: false
      proxy_url: "http://127.0.0.1:43210"
      preferred_ip_protocol: "ip4"
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
```
