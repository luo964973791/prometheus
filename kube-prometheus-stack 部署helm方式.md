### 一、添加源

```javascript
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/kube-prometheus-stack
kubectl create ns monitoring
tar zxvf kube-prometheus-stack-x.x.tgz
cd kube-prometheus-stack
vi values.yaml
prometheus:
  service:
    type: LoadBalancer
  prometheusSpec:
    replicas: 2
    retention: 12h
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources:
            requests:
              storage: 2Gi
    thanos:
      objectStorageConfig:
        existingSecret:
          name: thanos-objstore
          key: objstore.yml
    externalLabels:
      cluster: cluster-A
  thanosService:
    enabled: true
    type: LoadBalancer
    clusterIP: ""
  thanosServiceMonitor:
    enabled: true
  extraSecret:
    name: thanos-objstore
    data:
      objstore.yml: |
        type: S3
        config:
          bucket: "thanos"
          endpoint: "172.27.0.3:9000"
          access_key: "minioadmin"
          secret_key: "minioadmin"
          insecure: true
kubeEtcd:
  enabled: true
  endpoints:
    - 172.27.0.3
  service:
    port: 2381
    targetPort: 2381

alertmanager:
  enabled: true
  service:
    type: LoadBalancer
  config:
    global:
      resolve_timeout: 5m
      smtp_from: '456@qq.com'
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_auth_username: '456@qq.com'
      smtp_auth_password: 'kl'
      smtp_require_tls: false
    templates:
      - '/etc/alertmanager/config/*.tmpl'
    route:
      receiver: 'Default'
      group_by: ['alertname', 'cluster']
      group_wait: 5s
      group_interval: 1m
      repeat_interval: 1h
      routes:
        - matchers:
            - alertname = "Watchdog"
          receiver: null
    receivers:
      - name: 'Default'
        email_configs:
          - to: '123@qq.com'
            send_resolved: true
            headers:
              subject: "{{ .CommonLabels.subject }}"
            html: '{{ template "email.html" . }}'
        
  tplConfig: false  # 保持为 false，不让 Helm 模板引擎处理 Alertmanager 模板语法
  templateFiles:
    email.tmpl: |-
      {{ define "email.html" }}
      <html>
        <body>
          {{- range $index, $alert := .Alerts.Firing -}}
            <p>========= ERROR ==========</p>
            <h3 style="color:red;">告警名称: {{ $alert.Labels.alertname }}</h3>
            <p>告警级别: {{ $alert.Labels.severity }}</p>
            <p>告警机器: {{ $alert.Labels.instance }} {{ $alert.Labels.device }}</p>
            <p>告警详情: {{ $alert.Annotations.summary }}</p>
            <p>告警时间: {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}</p>
            <p>========= END ==========</p>
          {{- end }}
        </body>
      </html>
      {{- end }}
  alertmanagerSpec:
    replicas: 1
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources:
            requests:
              storage: 2Gi

grafana:
  service:
    type: LoadBalancer
  persistence:
    enabled: true
    storageClassName: local-path
  defaultDashboardsTimezone: cst
  enabled: true
  sidecar:
    dashboards:
      multicluster:
        global:
          enabled: true
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Thanos
        type: prometheus
        url: http://thanos-query-frontend.monitoring:9090
        access: proxy

#有高级规则依次添加
additionalPrometheusRules:
#--------------------------------------------------------------------------nginx-----------------------------------------------------------------------------------
  - name: 172-27-0-9-nginx-rules
    groups:
      - name: 172-27-0-9-nginx rules
        rules:
          - alert: NginxIsDown
            expr: nginx_up{job="nginx"} == 0
            for: 5m
            labels:
              severity: page
            annotations:
              summary: NGINX service is down

          - alert: NginxHighErrorRate
            expr: sum(rate(nginx_http_requests_total{status=~"5.*"}[5m])) / sum(rate(nginx_http_requests_total[5m])) * 100 > 5
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High Nginx 5xx Error Rate (instance {{ $labels.instance }})"
              description: "The Nginx instance {{ $labels.instance }} has a high 5xx error rate."

          - alert: NginxSlowResponseTime
            expr: avg(nginx_http_request_duration_seconds_bucket{le="1"} / sum(nginx_http_request_duration_seconds_count)) > 0.05
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High Nginx Slow Response Time (instance {{ $labels.instance }})"
              description: "The Nginx instance {{ $labels.instance }} has a high average response time."

          - alert: NginxHighConnections
            expr: nginx_connections{state="active"} > 100
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High Nginx Connections (instance {{ $labels.instance }})"
              description: "The Nginx instance {{ $labels.instance }} has a high number of active connections."

          - alert: NginxLongQueue
            expr: nginx_http_queue_length > 10
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Long Nginx Request Queue (instance {{ $labels.instance }})"
              description: "The Nginx instance {{ $labels.instance }} has a long request queue."
#--------------------------------------------------------------------------nginx-----------------------------------------------------------------------------------

#--------------------------------------------------------------------------httpd-----------------------------------------------------------------------------------
  - name: 172-27-0-4-httpd-rules
    groups:
    - name: 172-27-0-4-httpd rules
      rules:    
        - alert: ApacheDown
          expr: 'apache_up == 0'
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: Apache down (instance {{ $labels.instance }})
            description: "Apache down\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
    
        - alert: ApacheWorkersLoad
          expr: '(sum by (instance) (apache_workers{state="busy"}) / sum by (instance) (apache_scoreboard) ) * 100 > 80'
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Apache workers load (instance {{ $labels.instance }})
            description: "Apache workers in busy state approach the max workers count 80% workers busy on {{ $labels.instance }}\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
    
        - alert: ApacheRestart
          expr: 'apache_uptime_seconds_total / 60 < 1'
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: Apache restart (instance {{ $labels.instance }})
            description: "Apache has just been restarted.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
#--------------------------------------------------------------------------httpd-----------------------------------------------------------------------------------



    additionalScrapeConfigs:
      - job_name: "172-27-0-3-nginx-exporter"
        scrape_interval: 15s
        static_configs:
          - targets: [ '172.27.0.3:9113' ]
      - job_name: "172-27-0-4-apache-exporter"
        scrape_interval: 15s
        static_configs:
          - targets: [ '172.27.0.4:9117' ]

####################################################################################
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring -f ./values.yaml
helm upgrade prometheus prometheus-community/kube-prometheus-stack -n monitoring -f ./values.yaml


####################################################################################

#检查kube-proxy
kubectl edit cm/kube-proxy -n kube-system
## Change from
    metricsBindAddress: 127.0.0.1:10249 ### <--- Too secure
## Change to
    metricsBindAddress: 0.0.0.0:10249
kubectl delete pod -l k8s-app=kube-proxy -n kube-system
kubectl get prometheusrule -n monitoring #查看告警规则
kubectl edit prometheusrule -n monitoring prometheus-kube-prometheus-general.rules #删除watchdog策略.

#检查etcd
cat /etc/etcd.env | grep METRICS_URLS
ETCD_LISTEN_METRICS_URLS=http://172.27.0.6:2381,http://127.0.0.1:2381
curl http://172.27.0.6:2381/metrics

#更改告警时间
kubectl get prometheusrules -A -o yaml | sed 's/for: 15m/for: 30s/g' | kubectl apply -f -
kubectl get prometheusrules -A -o yaml | sed 's/for: 1h/for: 30s/g' | kubectl apply -f -
kubectl get prometheusrules -A -o yaml | sed 's/for: 10m/for: 30s/g' | kubectl apply -f -
kubectl get prometheusrules -A -o yaml | sed 's/for: 5m/for: 30s/g' | kubectl apply -f -


#部署thanos.
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
cat > values.yaml <<EOF
objstoreConfig: |-
  type: s3
  config:
    bucket: thanos
    endpoint: 172.27.0.3:9000
    access_key: minioadmin
    secret_key: minioadmin
    insecure: true

query:
  enabled: true
  replicaCount: 3
  replicaLabel: prometheus_replica
  stores:
    - "prometheus-kube-prometheus-thanos-discovery.monitoring:10901"

queryFrontend:
  enabled: true
  service:
    type: LoadBalancer

bucketweb:
  enabled: true
  service:
    type: LoadBalancer

compactor:
  enabled: true
  persistence:
    enabled: true

storegateway:
  enabled: true
  persistence:
    enabled: true

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

receive:
  enabled: false

ruler:
  enabled: true
  replicaLabel: prometheus_replica
  serviceMonitor:
    enabled: true
  alertmanagers:
    - http://alertmanager-operated.monitoring:9093
  config: |-
    groups:
      - name: "metamonitoring"
        rules:
          - alert: "PrometheusDown"
            expr: absent(up{prometheus="monitoring/prometheus-kube-prometheus-prometheus"})
    persistence:
        enabled: true
  persistence:
    enabled: true

minio:
  enabled: true
  auth:
    rootUser: minioadmin
    rootPassword: "minioadmin"
  defaultBuckets: "thanos"
  service:
    type: LoadBalancer
EOF

helm install thanos bitnami/thanos --namespace monitoring -f values.yaml
helm upgrade thanos bitnami/thanos --namespace monitoring -f values.yaml
#thanos每两个小时上报一次数据到S3对象存储，才部署好对象存储没数据正常，下面的命令手动触发是否上传.
curl -X POST http://$(kubectl get svc -n monitoring | grep prometheus-kube-prometheus-prometheus | awk '{print $3}'):9090/api/v1/admin/tsdb/snapshot
kubectl logs -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c thanos-sidecar
```

#监控nginx
helm install nginx-export -n monitoring \
  --set nginxServer="http://nginx-service.nginx.svc.cluster.local/stub_status" \
  --set serviceMonitor.enabled=true \
  prometheus-community/prometheus-nginx-exporter
```


### 四、访问grafana

```javascript
kubectl -n monitoring get svc | grep grafana
user: admin
pass: prom-operator
import > 13105

[root@node1 ~]# kubectl edit cm -n  monitoring prometheus-grafana   #配置邮箱告警.
grafana.ini: [analytics]
[smtp]
enabled = true
host = smtp.qq.com:465
user = @qq.com
password = 
from_address = @qq.com
```
