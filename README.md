# authentication_prometheus
repository provide implementing authentication and encryption between Prometheus and its target.

## Target Setup

### 1. Generate the Certificate and the Key
# a-Create node_exporter directory in /etc
```sh
sudo mkdir /etc/node_exporter
```
# b-Generate a 2048-bit RSA private key
```sh
sudo openssl genrsa -out /etc/node_exporter/node_exporter.key 2048
```
# c-Generate a certificate
```sh
sudo openssl req -new -key /etc/node_exporter/node_exporter.key  -out /etc/node_exporter/node_exporter.csr
```
### create config file for node exporter
# a- vi /etc/node_exporter/tarconf.yml file for certificate and key
```yaml
tls_server_config:
          cert_file: node_exporter.crt
          key_file: node_exporter.key
```
# b- Set ownership for the node_exporter directory
```sh
sudo chown -R node_exporter:node_exporter /etc/node_exporter
```
### update Node Exporter Service File
# vi /etc/systemd/system/node_exporter.service
```yaml
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target
[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.config.file="/etc/node_exporter/tarconf.yml"
[Install]
WantedBy=multi-user.target
```
### Reload the daemon and restart service 
```sh
sudo systemctl daemon-reload
```
```sh
sudo systemctl restart node_exporter
```
### test
```sh
curl https://localhost:9100/metrics 
curl -k https://localhost:9100/metrics
```
### Copy the node_exporter.crt to prometheus server
```sh
scp /etc/node_exporter/node_exporter.crt root@192.168.1.20:/etc/prometheus
```
## server setup

### Change the ownership of node_expoter.crt file to prometheus
```sh
sudo chown prometheus:prometheus /etc/prometheus/node_exporter.crt
```
### Edit the Prometheus Configuration File
vi /etc/prometheus/prometheus.yaml
```yaml
- job_name: 'Linux Server'
  scheme: https
  tls_config:
     ca_file: /etc/prometheus/node_exporter.crt
     insecure_skip_verify: true
  static_configs:
     - targets: ['192.168.1.19:9100']
```

### Restart Prometheus 

```sh
sudo systemctl restart prometheus
```
_______________________________________________________________________________________________________________________________________
Authentication

### Generate hashed passwd using apache2-utils package
```sh
sudo apt-get update && sudo apt install apache2-utils -y
htpasswd -nBC 12 "" | tr -d ':\n'
```

### Update tarconf.yml 
```sh
basic_auth_users:
  prometheus:$2y$12$2jvORcsJ2jybF7NaiTUPsev.Bljd/O0PfCk6YA73wqQo0.zY5lNMq
```
### Restart node_exporter service
 ```sh
 sudo systemctl restart node_exporter
 ```
### Update prometheus.yml with username and password

 ```yaml
   - job_name: 'Linux Server'
     scheme: https
     basic_auth:
       username: prometheus
       password: 123
     tls_config:
       ca_file: /etc/prometheus/node_exporter.crt
       insecure_skip_verify: true
 ```
### Restart prometheus service
 ```sh
 sudo systemctl restart prometheus
 ```
