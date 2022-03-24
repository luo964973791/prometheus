### 一、下载部署包,更改其中两个包名称，放到/data下

```javascript
grafana-enterprise-8.4.4-1.x86_64.rpm
prometheus-2.34.0.tar.gz
node_exporter-1.3.1.linux-amd64.tar.gz
tar -xzvf prometheus-2.34.0.tar.gz
mv prometheus-2.34.0.linux-amd64 /data/grafana
tar -xzvf node_exporter-1.3.1.linux-amd64.tar.gz
mv node_exporter-1.3.1.linux-amd64 /data/node_exporter
tar -xzvf alertmanager-0.23.0.linux-amd64.tar.gz
mv alertmanager-0.23.0.linux-amd64 /data/alertmanager
yum install ./grafana-enterprise-8.4.4-1.x86_64.rpm -y
```

### 二、添加各个服务的service

```javascript
# 普罗米修斯
cat <<EOF | sudo tee /lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus
After=network.target
 
[Service]
Type=simple
User=root
WorkingDirectory=/data/prometheus
ExecStart=/data/prometheus/prometheus --config.file=/data/prometheus/prometheus.yml
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF


# node_exporter
cat <<EOF | sudo tee /lib/systemd/system/node_exporter.service 
[Unit]
Description=Node Exporter
After=network.target
Wants=network-online.target
 
[Service]
Type=simple
User=root
ExecStart=/data/node_exporter/node_exporter
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF





# alertmanager
cat <<EOF | sudo tee /lib/systemd/system/alertmanager.service
[Unit]
Description=alertmanager
After=alertmanager.target
 
[Service]
ExecStart=/data/alertmanager/alertmanager --config.file=/data/alertmanager/alertmanager.yml
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
EOF
```

### 三、启动

```javascript
systemctl enable node_exporter
systemctl enable prometheus
systemctl enable grafana-server
systemctl enable alertmanager

systemctl start node_exporter
systemctl start prometheus
systemctl start grafana-server
/usr/share/grafana/bin/grafana-cli plugins install  grafana-piechart-panel
```

