### 一、下载部署包,更改其中两个包名称，放到/data下

```javascript
grafana-enterprise-8.4.4-1.x86_64.rpm
prometheus-2.34.0.tar.gz
node_exporter-1.3.1.linux-amd64.tar.gz
tar -xzvf prometheus-2.34.0.tar.gz
mv prometheus-2.34.0.linux-amd64 /data/prometheus
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


## kube-prometheus容器部署
```javascript
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus/manifests
#注意提前准备好pvc,如下是使用的ceph rbd提供的存储,需提前搭建,否则不能被挂载成功.
cat <<EOF | sudo tee grafana-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: grafana-pvc
    namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF


# 更改prometheus存储
cat <<EOF | sudo tee prometheus-prometheus.yaml 
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.32.1
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web

########################################
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: csi-rbd-sc
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
########################################

  enableFeatures: []
  externalLabels: {}
  image: quay.io/prometheus/prometheus:v2.32.1
  nodeSelector:
    kubernetes.io/os: linux
  podMetadata:
    labels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 2.32.1
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  probeNamespaceSelector: {}
  probeSelector: {}
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleNamespaceSelector: {}
  ruleSelector: {}
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: 2.32.1
EOF



#更改grafana pvc存储
cat <<EOF | sudo tee grafana-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: grafana
      app.kubernetes.io/name: grafana
      app.kubernetes.io/part-of: kube-prometheus
  template:
    metadata:
      annotations:
        checksum/grafana-config: 8a40383dc6577c8b30c5bf006ba9ab7e
        checksum/grafana-dashboardproviders: cf4ac6c4d98eb91172b3307d9127acc5
        checksum/grafana-datasources: 2e669e49f44117d62bc96ea62c2d39d3
      labels:
        app.kubernetes.io/component: grafana
        app.kubernetes.io/name: grafana
        app.kubernetes.io/part-of: kube-prometheus
        app.kubernetes.io/version: 8.3.3
    spec:
      containers:
      - env: []
        image: grafana/grafana:8.3.3
        name: grafana
        ports:
        - containerPort: 3000
          name: http
        readinessProbe:
          httpGet:
            path: /api/health
            port: http
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana-storage
          readOnly: false
        - mountPath: /etc/grafana/provisioning/datasources
          name: grafana-datasources
          readOnly: false
        - mountPath: /etc/grafana/provisioning/dashboards
          name: grafana-dashboards
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/alertmanager-overview
          name: grafana-dashboard-alertmanager-overview
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/apiserver
          name: grafana-dashboard-apiserver
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/cluster-total
          name: grafana-dashboard-cluster-total
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/controller-manager
          name: grafana-dashboard-controller-manager
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/k8s-resources-cluster
          name: grafana-dashboard-k8s-resources-cluster
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/k8s-resources-namespace
          name: grafana-dashboard-k8s-resources-namespace
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/k8s-resources-node
          name: grafana-dashboard-k8s-resources-node
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/k8s-resources-pod
          name: grafana-dashboard-k8s-resources-pod
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/k8s-resources-workload
          name: grafana-dashboard-k8s-resources-workload
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/k8s-resources-workloads-namespace
          name: grafana-dashboard-k8s-resources-workloads-namespace
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/kubelet
          name: grafana-dashboard-kubelet
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/namespace-by-pod
          name: grafana-dashboard-namespace-by-pod
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/namespace-by-workload
          name: grafana-dashboard-namespace-by-workload
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/node-cluster-rsrc-use
          name: grafana-dashboard-node-cluster-rsrc-use
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/node-rsrc-use
          name: grafana-dashboard-node-rsrc-use
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/nodes
          name: grafana-dashboard-nodes
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/persistentvolumesusage
          name: grafana-dashboard-persistentvolumesusage
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/pod-total
          name: grafana-dashboard-pod-total
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/prometheus-remote-write
          name: grafana-dashboard-prometheus-remote-write
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/prometheus
          name: grafana-dashboard-prometheus
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/proxy
          name: grafana-dashboard-proxy
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/scheduler
          name: grafana-dashboard-scheduler
          readOnly: false
        - mountPath: /grafana-dashboard-definitions/0/workload-total
          name: grafana-dashboard-workload-total
          readOnly: false
        - mountPath: /etc/grafana
          name: grafana-config
          readOnly: false
      nodeSelector:
        kubernetes.io/os: linux
      securityContext:
        fsGroup: 65534
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: grafana
      volumes:
###################添加storageclass pvc################################
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
######################################################################
      - name: grafana-datasources
        secret:
          secretName: grafana-datasources
      - configMap:
          name: grafana-dashboards
        name: grafana-dashboards
      - configMap:
          name: grafana-dashboard-alertmanager-overview
        name: grafana-dashboard-alertmanager-overview
      - configMap:
          name: grafana-dashboard-apiserver
        name: grafana-dashboard-apiserver
      - configMap:
          name: grafana-dashboard-cluster-total
        name: grafana-dashboard-cluster-total
      - configMap:
          name: grafana-dashboard-controller-manager
        name: grafana-dashboard-controller-manager
      - configMap:
          name: grafana-dashboard-k8s-resources-cluster
        name: grafana-dashboard-k8s-resources-cluster
      - configMap:
          name: grafana-dashboard-k8s-resources-namespace
        name: grafana-dashboard-k8s-resources-namespace
      - configMap:
          name: grafana-dashboard-k8s-resources-node
        name: grafana-dashboard-k8s-resources-node
      - configMap:
          name: grafana-dashboard-k8s-resources-pod
        name: grafana-dashboard-k8s-resources-pod
      - configMap:
          name: grafana-dashboard-k8s-resources-workload
        name: grafana-dashboard-k8s-resources-workload
      - configMap:
          name: grafana-dashboard-k8s-resources-workloads-namespace
        name: grafana-dashboard-k8s-resources-workloads-namespace
      - configMap:
          name: grafana-dashboard-kubelet
        name: grafana-dashboard-kubelet
      - configMap:
          name: grafana-dashboard-namespace-by-pod
        name: grafana-dashboard-namespace-by-pod
      - configMap:
          name: grafana-dashboard-namespace-by-workload
        name: grafana-dashboard-namespace-by-workload
      - configMap:
          name: grafana-dashboard-node-cluster-rsrc-use
        name: grafana-dashboard-node-cluster-rsrc-use
      - configMap:
          name: grafana-dashboard-node-rsrc-use
        name: grafana-dashboard-node-rsrc-use
      - configMap:
          name: grafana-dashboard-nodes
        name: grafana-dashboard-nodes
      - configMap:
          name: grafana-dashboard-persistentvolumesusage
        name: grafana-dashboard-persistentvolumesusage
      - configMap:
          name: grafana-dashboard-pod-total
        name: grafana-dashboard-pod-total
      - configMap:
          name: grafana-dashboard-prometheus-remote-write
        name: grafana-dashboard-prometheus-remote-write
      - configMap:
          name: grafana-dashboard-prometheus
        name: grafana-dashboard-prometheus
      - configMap:
          name: grafana-dashboard-proxy
        name: grafana-dashboard-proxy
      - configMap:
          name: grafana-dashboard-scheduler
        name: grafana-dashboard-scheduler
      - configMap:
          name: grafana-dashboard-workload-total
        name: grafana-dashboard-workload-total
      - name: grafana-config
        secret:
          secretName: grafana-config
EOF
```



### 四、企业微信报警

```javascript
[root@node1 ]# cat alertmanager-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.23.0
  name: alertmanager-main
  namespace: monitoring
stringData:
  alertmanager.yaml: |-
    "global":
      "resolve_timeout": "5m"
      wechat_api_corp_id: 'wwxxxxxxxxxxxxxxxxx'
      wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
    "inhibit_rules":
    - "equal":
      - "namespace"
      - "alertname"
      "source_matchers":
      - "severity = critical"
      "target_matchers":
      - "severity =~ warning|info"
    - "equal":
      - "namespace"
      - "alertname"
      "source_matchers":
      - "severity = warning"
      "target_matchers":
      - "severity = info"
    "receivers":
    - name: "wechat"
      wechat_configs:
      - send_resolved: true
        to_party: 1
        to_user: "@all"
        agent_id: 1xxxxxxx
        api_secret: "tTZCPhgSEGRGmaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa0"
        headers:
          subject: "{{ .CommonLabels.subject }}"
        html: '{{ template "email.html" . }}'
    "route":
      "group_by":
      - "namespace"
      - "alertname"
      - "job"
      "group_interval": "5m"
      "group_wait": "30s"
      "receiver": "wechat"
      "repeat_interval": "12h"
      "routes":
      - "matchers":
        - "alertname = Watchdog"
        "receiver": "wechat"
    templates:
    - /etc/alertmanager/config/*.tmpl
  email.tmpl: |-           # 告警模板
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
type: Opaque
```
### 五、邮箱报警
```
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.23.0
  name: alertmanager-main
  namespace: monitoring
stringData:
  alertmanager.yaml: |-
    global:
      resolve_timeout: 5m
    inhibit_rules:
    - equal:
      - namespace
      - alertname
      source_matchers:
      - severity = critical
      target_matchers:
      - severity =~ warning|info
    - equal:
      - namespace
      - alertname
      source_matchers:
      - severity = warning
      target_matchers:
      - severity = info
    receivers:
    - name: email
      email_configs:
      - send_resolved: true
        smarthost: 'smtp.qq.com:465'
        from: '@qq.com'
        auth_username: '@qq.com'
        auth_password: ''
        require_tls: false
        to: "@qq.com"
        headers:
          subject: "{{ .CommonLabels.subject }}"
        html: '{{ template "email.html" . }}'
    route:
      group_by:
      - namespace
      - alertname
      - job
      group_interval: 5m
      group_wait: 30s
      receiver: email
      repeat_interval: 12h
      routes:
      - matchers:
        - alertname = Watchdog
        receiver: email
    templates:
    - /etc/alertmanager/config/*.tmpl
  email.tmpl: |- # 告警模板
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
type: Opaque
```
