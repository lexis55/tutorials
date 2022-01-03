# How to Add IAM User and IAM Role to AWS EKS Cluster?

 You can find tutorial [here](https://antonputra.com/monitoring/Install-prometheus-and-grafana-on-ubuntu/).

## Install Prometheus on Ubuntu 20.04

ssh -i ~/Downloads/devops.pem ubuntu@107.23.226.82

sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus

https://prometheus.io/download/

wget https://github.com/prometheus/prometheus/releases/download/v2.32.1/prometheus-2.32.1.linux-amd64.tar.gz

tar -xvf prometheus-2.32.1.linux-amd64.tar.gz

sudo mkdir -p /data /etc/prometheus

cd prometheus-2.32.1.linux-amd64
ls -ltr
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
cd
rm -rf prometheus*

prometheus --version

prometheus --help


sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

sudo vim /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
 
StartLimitIntervalSec=500
StartLimitBurst=5
 
[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
 
[Install]
WantedBy=multi-user.target



sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus

journalctl -u prometheus -f


http://107.23.226.82:9090


## Install Node Exporter on Ubuntu 20.04

sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter

https://prometheus.io/download/

wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz

tar -xvf node_exporter-1.3.1.linux-amd64.tar.gz

sudo mv node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/bin/

rm -rf node_exporter*

node_exporter --version

node_exporter --help

sudo vim /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
 
StartLimitIntervalSec=500
StartLimitBurst=5
 
[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind
 
[Install]
WantedBy=multi-user.target



sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter

journalctl -u node_exporter -f --no-pager


http://107.23.226.82:9100

add node_exporter to the target list

sudo vim /etc/prometheus/prometheus.yml

  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]

promtool check config /etc/prometheus/prometheus.yml

(hot reload)
curl -X POST http://localhost:9090/-/reload
sudo systemctl restart prometheus

http://107.23.226.82:9090

## Install Grafana on Ubuntu 20.04
https://grafana.com/docs/grafana/latest/installation/debian/

sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install grafana

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server

http://107.23.226.82:3000

admin/admin
admin/devops123

add datasource - http://localhost:9090

sudo vim /etc/grafana/provisioning/datasources/datasources.yaml

apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: http://localhost:9090
    isDefault: true

sudo systemctl restart grafana-server

create first graph manually

find in Prometheus "scrape_duration_seconds"
http://107.23.226.82:9090

Call it - Scrape Duration

{{ job }}

import node exporter dashboard

https://grafana.com/grafana/dashboards/1860

1860

## Install Pushgateway Prometheus on Ubuntu 20.04

sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false pushgateway

wget https://github.com/prometheus/pushgateway/releases/download/v1.4.2/pushgateway-1.4.2.linux-amd64.tar.gz

tar -xvf pushgateway-1.4.2.linux-amd64.tar.gz

sudo mv pushgateway-1.4.2.linux-amd64/pushgateway /usr/local/bin/

rm -rf pushgateway*

pushgateway --version

pushgateway --help

sudo vim /etc/systemd/system/pushgateway.service

[Unit]
Description=Pushgateway
Wants=network-online.target
After=network-online.target
 
StartLimitIntervalSec=500
StartLimitBurst=5
 
[Service]
User=pushgateway
Group=pushgateway
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/pushgateway
 
[Install]
WantedBy=multi-user.target

sudo systemctl enable pushgateway
sudo systemctl start pushgateway
sudo systemctl status pushgateway

http://107.23.226.82:9091

sudo vim /etc/prometheus/prometheus.yml

  - job_name: pushgateway
    honor_labels: true
    static_configs:
      - targets: ["localhost:9091"]

promtool check config /etc/prometheus/prometheus.yml

(hot reload)
curl -X POST http://localhost:9090/-/reload
sudo systemctl restart prometheus

http://107.23.226.82:9090

echo "jenkins_job_duration_seconds 15.98" | curl --data-binary @- http://localhost:9091/metrics/job/backup


## Securing Prometheus with Basic Auth
https://prometheus.io/docs/guides/basic-auth/

sudo apt-get install python3-bcrypt

vim generate_password.py

import getpass
import bcrypt

password = getpass.getpass("password: ")
hashed_password = bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt())
print(hashed_password.decode())

python3 generate_password.py

$2b$12$0.NW9plpbBiJyJx4JQVs2uRkhe1QHqSe1xauE3VuKr2822xIlmDfa

sudo vim /etc/prometheus/web.yml

basic_auth_users:
    admin: $2b$12$0.NW9plpbBiJyJx4JQVs2uRkhe1QHqSe1xauE3VuKr2822xIlmDfa

promtool check web-config /etc/prometheus/web.yml

sudo vim /etc/systemd/system/prometheus.service

--web.config.file=/etc/prometheus/web.yml

promtool check config /etc/prometheus/prometheus.yml

sudo systemctl daemon-reload
sudo systemctl restart prometheus
sudo systemctl status prometheus

http://107.23.226.82:9090

update datasource

## Install Alertmanager on Ubuntu 20.04

sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false alertmanager

wget https://github.com/prometheus/alertmanager/releases/download/v0.23.0/alertmanager-0.23.0.linux-amd64.tar.gz

tar -xvf alertmanager-0.23.0.linux-amd64.tar.gz

sudo mkdir -p /alertmanager-data /etc/alertmanager

sudo mv alertmanager-0.23.0.linux-amd64/alertmanager.yml /etc/alertmanager/
sudo mv alertmanager-0.23.0.linux-amd64/alertmanager /usr/local/bin/

    DATA
    The storage is mandatory (it defaults to "data/") and is used to store Alertmanager's notification states and silences. It is not currently used for storing alerts themselves (those are continuously being resent by Prometheus servers anyways, as long as they are still firing).

    Without this state (or if you wipe it), Alertmanager would not know across restarts what silences were created, or what notifications were already sent.

rm -rf alertmanager*

alertmanager --version

alertmanager --help

sudo vim /etc/systemd/system/alertmanager.service

[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target
 
StartLimitIntervalSec=500
StartLimitBurst=5
 
[Service]
User=alertmanager
Group=alertmanager
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/alertmanager \
  --storage.path=/alertmanager-data \
  --config.file=/etc/alertmanager/alertmanager.yml
 
[Install]
WantedBy=multi-user.target

sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager

http://107.23.226.82:9093

sudo vim /etc/prometheus/dead-mans-snitch-rule.yml

groups:
- name: dead-mans-snitch
  rules:
  - alert: DeadMansSnitch
    annotations:
      message: This alert is integrated with DeadMansSnitch.
    expr: vector(1)

sudo vim /etc/prometheus/prometheus.yml

          - localhost:9093


promtool check config /etc/prometheus/prometheus.yml

sudo systemctl restart prometheus
sudo systemctl status prometheus

https://deadmanssnitch.com/


## Alertmanager Slack Channel Integration

sudo vim /etc/alertmanager/alertmanager.yml

  routes:
  - receiver: slack-notifications
    match:
      severity: warning

- name: slack-notifications
  slack_configs:
  - channel: "#alerts"
    send_resolved: true
    api_url: "https://hooks.slack.com/services/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"
    title: "{{ .GroupLabels.alertname }}"
    text: "{{ range .Alerts }}{{ .Annotations.message }}\n{{ end }}"

sudo systemctl restart alertmanager
sudo systemctl status alertmanager

sudo vim /etc/prometheus/batch-job-rules.yml

groups:
- name: batch-job-rules
  rules:
  - alert: JenkinsJobExceededThreshold
    annotations:
      message: Jenkins job exceeded threshold of 30 seconds
    expr: jenkins_job_duration_seconds{job="backup"} > 30
    for: 1m
    labels:
      severity: warning

sudo vim /etc/prometheus/prometheus.yml

promtool check config /etc/prometheus/prometheus.yml

sudo systemctl restart prometheus
sudo systemctl status prometheus

echo "jenkins_job_duration_seconds 31.87" | curl --data-binary @- http://localhost:9091/metrics/job/backup

echo "jenkins_job_duration_seconds 11.87" | curl --data-binary @- http://localhost:9091/metrics/job/backup

create "alerts" slack channel
