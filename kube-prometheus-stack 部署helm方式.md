### 一、添加源

```javascript
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm pull prometheus-community/kube-prometheus-stack
tar zxvf kube-prometheus-stack-x.x.tgz
cd kube-prometheus-stack
vi values.yaml  #更改里面的配置,这里已邮箱报警为例.
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      smtp_from: 'from@qq.com'
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_auth_username: 'from@qq.com'
      smtp_auth_password: '12345'
      smtp_require_tls: false
    templates:
      - '/etc/alertmanager/config/*.tmpl'
    route:
      receiver: Default
      group_by: ['alertname', 'cluster']
      continue: false
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
    receivers:
    - name: Default
      email_configs:
      - to: 'to@qq.com'
        send_resolved: true
        headers:
          subject: "{{ .CommonLabels.subject }}"
        html: '{{ template "email.html" . }}'
  tplConfig: false
  templateFiles:
    email.tmpl: |-
      {{ define "email.html" }}
      <html>
        <body>
          {{- if gt (len .Alerts.Firing) 0 -}}
          {{- range $index, $alert := .Alerts -}}
            <p>========= ERROR ==========</p>
            <h3 style="color:red;">告警名称: {{ .Labels.alertname }}</h3>
            <p>告警级别: {{ .Labels.severity }}</p>
            <p>告警机器: {{ .Labels.instance }} {{ .Labels.device }}</p>
            <p>告警详情: {{ .Annotations.summary }}</p>
            <p>告警时间: {{ .StartsAt.Format "2006-01-02 15:04:05" }}</p>
            <p>========= END ==========</p>
          {{- end }}
          {{- end }}
          {{- if gt (len .Alerts.Resolved) 0 -}}
          {{- range $index, $alert := .Alerts -}}
            <p>========= INFO ==========</p>
            <h3 style="color:green;">告警名称: {{ .Labels.alertname }}</h3>
            <p>告警级别: {{ .Labels.severity }}</p>
            <p>告警机器: {{ .Labels.instance }}</p>
            <p>告警详情: {{ .Annotations.summary }}</p>
            <p>告警时间: {{ .StartsAt.Format "2006-01-02 15:04:05" }}</p>
            <p>恢复时间: {{ .EndsAt.Format "2006-01-02 15:04:05" }}</p>
            <p>========= END ==========</p>
          {{- end }}
          {{- end }}
        </body>
      </html>
      {{- end }}



#启动
helm install prometheus -n monitoring -f ./values.yaml .
helm upgrade prometheus -n monitoring -f ./values.yaml . \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.thanosService.enabled=true \
  --set prometheus.service.type=LoadBalancer \
  --set kubeEtcd.enabled=true \
  --set prometheus.prometheusSpec.retention=365d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=local-path \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set alertmanager.service.type=LoadBalancer \
  --set alertmanager.tplConfig=false \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=local-path \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.persistence.enabled=true \
  --set grafana.defaultDashboardsTimezone=cst \
  --set grafana.persistence.storageClassName=local-path



#检查kube-proxy
kubectl edit cm/kube-proxy -n kube-system
## Change from
    metricsBindAddress: 127.0.0.1:10249 ### <--- Too secure
## Change to
    metricsBindAddress: 0.0.0.0:10249
kubectl delete pod -l k8s-app=kube-proxy -n kube-system

#检查etcd
cat /etc/etcd.env | grep METRICS_URLS
ETCD_LISTEN_METRICS_URLS=http://172.27.0.6:2381,http://127.0.0.1:2381
curl http://172.27.0.6:2381/metrics



helm install prometheus  \
  --namespace monitoring --create-namespace \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.thanosService.enabled=true \
  --set prometheus.service.type=LoadBalancer \
  --set kubeEtcd.enabled=true \
  --set prometheus.prometheusSpec.retention=365d \
  --set kubeEtcd.endpoints[0]=172.27.0.6 \
  --set kubeEtcd.endpoints[1]=172.27.0.7 \
  --set kubeEtcd.endpoints[2]=172.27.0.8 \
  --set kubeEtcd.service.port=2381 \
  --set kubeEtcd.service.targetPort=2381 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=local-path \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set alertmanager.service.type=LoadBalancer \
  --set alertmanager.tplConfig=false \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=local-path \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.persistence.enabled=true \
  --set grafana.defaultDashboardsTimezone=cst \
  --set grafana.persistence.storageClassName=local-path \
  prometheus-community/kube-prometheus-stack


#监控nginx
helm install nginx-export -n monitoring \
  --set nginxServer="http://172.27.0.14/stub_status" \
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

### 五、部署thanos

```javascript
cat <<EOF>values.yaml 
additionalPrometheusRulesMap:
  - groups:
    - name: nginx_rules
      rules:
        - record: nginx_http_requests_total
          expr: sum(nginx_http_requests_total) by (instance)
        - record: nginx_http_status_5xx
          expr: sum(nginx_http_responses_total{status="5xx"}) by (instance)
        - record: nginx_up
          expr: up == 1
        - alert: HighErrorRate
          expr: rate(nginx_http_status_5xx[5m]) > 0.5
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on NGINX"
            description: "Error rate is too high on NGINX for the last 5 minutes."
fullnameOverride: prometheus
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    configReloaders: true
    general: true
    k8s: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeScheduler: true
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true
grafana:
  adminPassword: admin-password
  enabled: true
  defaultDashboardsTimezone: cst
  serviceMonitor:
    enabled: true
  service:
    type: LoadBalancer
  persistence:
    enabled: true
  persistence:
    storageClassName: local-path	
prometheus:
  thanosService:
    enabled: true
  service:
    type: LoadBalancer
  prometheusSpec:
    retention: 365d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources:
            requests:
              storage: 2Gi
    additionalScrapeConfigs:
      - job_name: "nginx-exporter"
        scrape_interval: 15s
        static_configs:
          - targets: [ '172.27.0.15:9113' ]
kubeScheduler:
  enabled: true
kubeProxy:
  enabled: true
kubeStateMetrics:
  enabled: true
kube-state-metrics:
  selfMonitor:
    enabled: true
kubeEtcd:
  enabled: true
  endpoints:
    - 172.27.0.6
    - 172.27.0.7
    - 172.27.0.8
  service:
    port: 2381
    targetPort: 2381
alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'email_router'
      group_by: ['namespace']
      routes:
        - receiver: 'email'
          matchers:
            - alertname =~ "InfoInhibitor|Watchdog"
    receivers:
      - name: 'email_router'
        email_configs:
          - to: '@qq.com'
            from: '@qq.com'
            smarthost: smtp.qq.com:465
            auth_username: '@qq.com'
            auth_password: ''
            send_resolved: true
            require_tls: false
    templates:
    - '/etc/alertmanager/config/*.tmpl'
  templateFiles:
    template_1.tmpl: |-
        {{ define "email.from" }}@qq.com{{ end }}
        {{ define "email.to" }}@qq.com{{ end }}
        {{ define "email.to.html" }}
        {{ range .Alerts }}
        =========start==========<br>
        告警程序: prometheus_alert <br>
        告警级别: {{ .Labels.severity }} 级 <br>
        告警类型: {{ .Labels.alertname }} <br>
        故障主机: {{ .Labels.instance }} <br>
        告警主题: {{ .Annotations.summary }} <br>
        告警详情: {{ .Annotations.description }} <br>
        触发时间: {{ .StartsAt.Format "2019-08-04 16:58:15" }} <br>
        =========end==========<br>
        {{ end }}
        {{ end }}
  service:
    type: LoadBalancer
  tplConfig: true
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          resources:
            requests:
              storage: 2Gi
EOF
```

### 六、启动thaons

```javascript
helm install thanos -f ./thanos.yaml bitnami/thanos -n monitoring
```

### 七、页面设置添加thanos新地址

```javascript
http://thanos-query-frontend.monitoring:9090/
```

### 八、部署loki

```javascript
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack --set grafana.enabled=true,prometheus.enabled=true,prometheus.alertmanager.persistentVolume.enabled=true,prometheus.server.persistentVolume.enabled=true,loki.persistence.enabled=true,loki.persistence.storageClassName=local-path,loki.persistence.size=5Gi -n monitoring
kubectl get secret --namespace monitoring loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
