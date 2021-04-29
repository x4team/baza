# Install Prometheus, node exporter, tinyproxy, blackbox, alertmanager, alertmanager-sns-forwarder, prometheus_bot(telegram)


## INSTALL PROMETHEUS

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


##  INSTALL node_exporter

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


## INSTALL TINYPROXY

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

### If need - add crontab job for restart tinyproxy every day

```
# Restart tinyproxy in 23:50
50 23 * * * systemctl restart tinyproxy
```

### Test

```
curl --proxy "http://127.0.0.1:43210" "http://httpbin.org/ip" -k
```



## INSTALL BLACKBOX MODULE

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
nano /etc/blackbox/blackbox.yml
```

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

### Check config

```
blackbox_exporter --config.check
```

### Restart blackbox

```
sudo systemctl restart blackbox.service
```

### Binding the Blackbox Exporter Module in Prometheus

```
scrape_configs:

...

    - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [https_external_2xx]   # USE BACKBOX module with TINYPROXY
    static_configs:
      - targets:
        - https://google.com    # Target to probe with https.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port
```

### Restart prometheus

```
sudo systemctl restart prometheus
```

### Check config 

```
https://localhost:1234/config
```

### To verify it, head over to the /graph endpoint, and issue the following PromQL request.

```
probe_success{instance="https://127.0.0.1:1234", job="blackbox"}
```


## INSTALL ALERTMANAGER

### Create user

```
sudo useradd --no-create-home --shell /bin/false alertmanager
```

### Creating Alert Rules

```
sudo touch /etc/prometheus/alert.rules.yml
sudo chown prometheus:prometheus /etc/prometheus/alert.rules.yml
```

### Add the rule_file to prometheus config

```
sudo nano /etc/prometheus/prometheus.yml


global:
  scrape_interval: 15s

rule_files:
  - alert.rules.yml

scrape_configs:
...
```

In order to make the alert rule, you’ll use Blackbox Exporter’s probe_success metric which returns 1 if the endpoint is up and 0 if it isn’t.

```
sudo nano /etc/prometheus/alert.rules.yml
```

```
groups:
- name: alert.rules
  rules:
  - alert: EndpointDown
    expr: probe_success == 0
    for: 10s
    labels:
      severity: "critical"
    annotations:
      summary: "Endpoint {{ $labels.instance }} down"
```

### Check alert.rule.yml

```
sudo promtool check rules /etc/prometheus/alert.rules.yml
```

### Restart Prometheus

```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

### Downloading Alertmanager

```
cd /tmp
curl -LO https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
tar xvf alertmanager-0.21.0.linux-amd64.tar.gz
```

```
sudo mv alertmanager-0.21.0.linux-amd64/alertmanager /usr/local/bin
sudo mv alertmanager-0.21.0.linux-amd64/amtool /usr/local/bin
```

### Fix rules

```
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool
```

### Remove files

```
rm -rf alertmanager-0.21.0.linux-amd64 alertmanager-0.21.0.linux-amd64.tar.gz
```

### Configuring Alertmanager To Send Alerts Over Email

```
sudo mkdir /etc/alertmanager
sudo chown alertmanager:alertmanager /etc/alertmanager
sudo nano /etc/alertmanager/alertmanager.yml
```

```
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@your_domain'
  smtp_require_tls: false
```

If you need specify host,port,auth,ssl etc 
https://lyz-code.github.io/blue-book/devops/prometheus/alertmanager/

### Ready config representation

```
global:
  resolve_timeout: 1m
  smtp_from: 'hellomail@mail.ru'
  smtp_smarthost: smtp.mail.ru:465
  smtp_auth_username: 'hellomail@mail.ru'
  smtp_auth_password: 'sajhdassabjdtqYE9SQHl4'
  smtp_require_tls: true
route:
  group_by: ['instance', 'severity']
  group_wait: 30s
  group_interval: 30s
  repeat_interval: 30s
  receiver: team-1

receivers:
  - name: 'team-1'
    email_configs:
      - to: 'tosend@mail.ru'
```

### Running Alertmanager

```
sudo nano /etc/systemd/system/alertmanager.service
```

```
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
WorkingDirectory=/etc/alertmanager/
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --web.external-url http://127.0.0.1:9093

[Install]
WantedBy=multi-user.target
```

``
sudo systemctl daemon-reload
sudo systemctl enable alertmanager.service
``

### Open the Prometheus configuration file:

```
...
rule_files:
  - alert.rules.yml

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - localhost:9093
...
```

### Service restart/start/status

```
sudo systemctl restart prometheus
sudo systemctl start alertmanager.service
sudo systemctl status prometheus
sudo systemctl status alertmanager.service
```

### Open Port

```
sudo ufw allow 9093/tcp
```



## INSTALL ALERTMANAGER-SNS-FORWARDER

0) First install docker https://docs.docker.com/engine/install/ubuntu/
1) Setup Amazon SNS and create arn:aws:sns:eu-west-1:xxxxxxxxxx:TopicName and Subscriptions: your_email@gmail.com

```
docker run -it -d --name alertmanager --env "AWS_REGION=eu-west-1" --env "SNS_FORWARDER_DEBUG=true" --env "SNS_FORWARDER_ARN_PREFIX=arn:aws:sns:eu-west-1:xxxxxxxx:" -p 9087:9087 datareply/alertmanager-sns-forwarder:0.2
```

### Alertmanager configuration file:

```
- name: 'sns-forwarder'
  webhook_configs:
  - send_resolved: True
    url: http://127.0.0.1:9087/alert/SNStopicNAME

```

### Ready alertmanager config file 

```
global:
  resolve_timeout: 1m
  smtp_from: 'hellomail@mail.ru'
  smtp_smarthost: smtp.mail.ru:465
  smtp_auth_username: 'hellomail@mail.ru'
  smtp_auth_password: 'sajhdassabjdtqYE9SQHl4'
  smtp_require_tls: true
route:
  group_by: ['instance', 'severity']
  group_wait: 30s
  group_interval: 30s
  repeat_interval: 30s
  receiver: team-1      #SET THE TEAM

receivers:
  - name: 'team-1'
    webhook_configs:
    - send_resolved: true
      url: http://127.0.0.1:9087/alert/ProxyAlertName
  - name: 'team-2'
    email_configs:
      - to: 'tosend@mail.ru'
```

### Restart service alertmanager

```
sudo systemctl start alertmanager.service
```



## INSTALL PROMETHEUS_BOT TELEGRAM

### Set chatid ENV in docker run

0) Create Telegram bot with BotFather, it will return your bot token

```
sudo docker run -it -d -p 9088:9087 --name prometheus_bot --env TELEGRAM_CHATID=12735163 ekeih/prometheus_bot:latest
```

## Create config file 

```
sudo nano config.yml
```

```
telegram_token: "token goes here"
# ONLY IF YOU USING DATA FORMATTING FUNCTION, NOTE for developer: important or test fail
#time_outdata: "02/01/2006 15:04:05" 
#template_path: "template.tmpl" # ONLY IF YOU USING TEMPLATE
#time_zone: "Europe/Rome" # ONLY IF YOU USING TEMPLATE
#split_msg_byte: 4000
send_only: true # use bot only to send messages.
```

### Copy config.yml to container and restart 

```
sudo docker cp config.yml prometheus_bot:/config.yml
sudo docker restart prometheus_bot
```

### Alert manager configuration file:

```
- name: 'admins'
  webhook_configs:
  - send_resolved: True
    url: http://127.0.0.1:9087/alert/-chat_id
```

### Ready alertmanager config file

```
global:
  resolve_timeout: 1m
  smtp_from: 'hellomail@mail.ru'
  smtp_smarthost: smtp.mail.ru:465
  smtp_auth_username: 'hellomail@mail.ru'
  smtp_auth_password: 'sajhdassabjdtqYE9SQHl4'
  smtp_require_tls: true
route:
  group_by: ['instance', 'severity']
  group_wait: 30s
  group_interval: 30s
  repeat_interval: 30s
  receiver: team-3      #SET THE TEAM, or ['team-1', 'team-2', 'team-3']

receivers:
  - name: 'team-1'
    webhook_configs:
    - send_resolved: true
      url: http://127.0.0.1:9087/alert/ProxyAlertName
  - name: 'team-2'
    email_configs:
      - to: 'tosend@mail.ru'
  - name: 'team-3'
    webhook_configs:
    - send_resolved: True
      url: http://127.0.0.1:9088/alert/-chat_id
```


### Restart service alertmanager

```
sudo systemctl start alertmanager.service
```



## Sources:

blackbox exporter

https://github.com/prometheus/blackbox_exporter

https://github.com/prometheus/blackbox_exporter/releases

https://lyz-code.github.io/blue-book/devops/prometheus/blackbox_exporter/#https-endpoint-through-an-http-proxy

Install blackbox https://devconnected.com/how-to-install-and-configure-blackbox-exporter-for-prometheus/

Additional:

https://github.com/prometheus/blackbox_exporter/issues/503


Install Prometheus and node_exporter

https://computingforgeeks.com/how-to-install-prometheus-and-node-exporter-on-debian/

https://prometheus.io/docs/guides/multi-target-exporter/


TinyProxy Install And Setup

https://techlist.top/install-tinyproxy-on-ubuntu-server/


AlertManager

https://www.digitalocean.com/community/tutorials/how-to-use-alertmanager-and-blackbox-exporter-to-monitor-your-web-server-on-ubuntu-16-04

https://medium.com/techno101/how-to-send-a-mail-using-prometheus-alertmanager-7e880a3676db

https://lyz-code.github.io/blue-book/devops/prometheus/alertmanager/

https://alibabacloud.com/blog/using-prometheus-and-blackbox-exporter-to-monitor-services-on-alibaba-cloud_596232

Amazon SNS

https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_SetupSNS.html



local resource:

AlertManager

http://127.0.0.1:9093/#/alerts

Prometheus Alerts

http://127.0.0.1:9090/alerts

Prometheus Targets

http://127.0.0.1:9090/targets

