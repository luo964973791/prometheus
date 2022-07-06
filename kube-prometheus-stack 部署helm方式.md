### 一、添加源

```javascript
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update


#如果测试环境没有存储,可以搭建个单机版的NFS
docker run -d --name nfs \
    --privileged \
    --restart always \
    -p 2049:2049 \
    -v /nfs-share:/nfs-share \
    -e SHARED_DIRECTORY=/nfs-share \
    itsthenetwork/nfs-server-alpine:latest


helm install nfs-subdir-external-provisioner \
  --namespace=nfs-provisioner --create-namespace \
  --set storageClass.defaultClass=true \
  --set nfs.server=172.27.0.6 \
  --set nfs.path=/ \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner
```

### 二、准备挂载.

```javascript
cat <<EOF>values.yaml 
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: csi-rbd-sc
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: csi-rbd-sc
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
EOF
```

### 三、启动集群

```javascript
helm install prometheus  \
  --namespace monitoring --create-namespace \
  --set grafana.service.type=NodePort \
  --set prometheus.service.type=NodePort \
  --set alertmanager.service.type=NodePort \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=csi-rbd-sc \
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

### 五、查看告警是否正常

```javascript
http://x.x.x.x:30090/targets?search=
#点击Unhealthy修复不正常的组件，例如
kubectl edit cm/kube-proxy -n kube-system
    metricsBindAddress: 0.0.0.0:10249
kubectl delete pod -l k8s-app=kube-proxy -n kube-system
```

### 六、创建应用测试监控效果

```javascript
cat <<EOF> test.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: fabxc/instrumented_app
        ports:
        - name: web
          containerPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
EOF

kubectl apply -f test.yaml
```

### 七、添加test应用监控

```javascript
cat <<EOF> monitoring-test.yaml 
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  labels:
    team: frontend
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
EOF

kubectl apply -f monitoring-test.yaml
```

### 八、查看test应用监控是否正常

```javascript
http://x.x.x.x:30090/targets?search=
```

