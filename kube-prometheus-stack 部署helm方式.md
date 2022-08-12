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
helm install prometheus  \
  --namespace monitoring --create-namespace \
  --set grafana.service.type=NodePort \
  --set prometheus.service.type=NodePort \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=nfs-client \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set alertmanager.service.type=NodePort \
  --set alertmanager.alertmanagerSpec.replicas=3 \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=nfs-client \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=2Gi \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.persistence.enabled=true \
  --set grafana.defaultDashboardsTimezone=cst \
  --set grafana.persistence.storageClassName=nfs-client \
  -f values.yaml \
  prometheus-community/kube-prometheus-stack
```

### 四、访问grafana

```javascript
kubectl -n monitoring get svc | grep grafana
user: admin
pass: prom-operator
import > 13105
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


