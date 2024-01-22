### 一、添加源

```javascript
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
yum install nfs-utils -y


#如果测试环境没有存储,可以搭建个单机版的NFS
docker run -d --name nfs \
    --privileged \
    --restart always \
    -p 2049:2049 \
    -v /data/nfs-share:/nfs-share \
    -e SHARED_DIRECTORY=/nfs-share \
    itsthenetwork/nfs-server-alpine:latest


helm install nfs \
  --namespace=nfs-provisioner --create-namespace \
  --set storageClass.defaultClass=true \
  --set nfs.server=172.27.0.3 \
  --set nfs.path=/ \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner
```

### 二、准备挂载.

```javascript
#准备对象存储,buket存数据，进入9000端口，创建名字为thanos的buket.
docker run -d   --restart always   -p 9000:9000   --name minio   -v /data/minio/data:/data   -e "MINIO_ROOT_USER=admin"   -e "MINIO_ROOT_PASSWORD=fastadmin"   minio/minio server /data --console-address ":9090"

cat <<EOF>values.yaml 
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
prometheus:
  thanosService:
    enabled: true
  extraSecret:
    name: bucket-config
    data:
      objstore.yml: |
        type: S3
        config:
          bucket: "thanos"
          endpoint: "172.27.0.3:9000"
          access_key: "admin"
          secret_key: "fastadmin"
          insecure: true
          signature_version2: false
  thanosServiceMonitor:
    enabled: true
  prometheusSpec:
    disableCompaction: true
    thanos:
      objectStorageConfig:
        name: bucket-config
        key: objstore.yml
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
thanosRuler:
  enabled: true
  thanosRulerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-client
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
EOF
```

### 三、启动集群

```javascript
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
  --set kubeEtcd.endpoints[0]=172.27.0.6 \
  --set kubeEtcd.endpoints[1]=172.27.0.7 \
  --set kubeEtcd.endpoints[2]=172.27.0.8 \
  --set kubeEtcd.service.port=2381 \
  --set kubeEtcd.service.targetPort=2381 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=local-path \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set alertmanager.service.type=LoadBalancer \
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
user = 1145023603@qq.com
password = owklbhkobjnabcde
from_address = 1145023603@qq.com
```

### 五、部署thanos

```javascript
cat <<EOF>thanos.yaml 
global:
  storageClass: "nfs-client"
existingObjstoreSecret: "bucket-config"
query:
  enabled: true
  replicaLabel: [prometheus_replica]
  dnsDiscovery:
    enabled: true
    sidecarsService: "prometheus-kube-prometheus-thanos-discovery"
    sidecarsNamespace: "monitoring"
bucketweb:
  enabled: true
queryFrontend:
  enabled: true 
compactor:
  enabled: true
  persistence:
    enabled: true
storegateway:
  enabled: true 
  persistence:
    enabled: true
ruler:
  enabled: true
  replicaLabel: prometheus_replica
  alertmanagers:
  - http://prometheus-kube-prometheus-alertmanager:9093
  existingConfigmap: "prometheus-kube-prometheus-grafana-overview"
  persistence:
    enabled: true
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
