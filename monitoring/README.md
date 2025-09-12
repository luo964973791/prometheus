
# 监控系统使用说明

## 概述
本项目包含完整的监控解决方案，包括 Prometheus、Grafana 和 Alertmanager。

## 快速开始

### 启动服务
```javascript
# 启动所有监控服务
mkdir prometheus/data
mkdir alertmanager/data
mkdir grafana/data
chmod -R 777 prometheus/data
chmod -R 777 alertmanager/data
chmod -R 777 grafana/data
docker-compose up -d

# 查看服务状态
docker-compose ps
```
## 服务访问信息

### Grafana 仪表板
- **URL**: http://localhost:3000
- **用户名**: admin
- **密码**: admin

### Kafka 监控面板
- **端口**: 21078
- **访问地址**: http://localhost:21078

### Prometheus
- **URL**: http://localhost:9090

### Alertmanager
- **URL**: http://localhost:9093

## 监控指标

### 支持的监控目标
- Elasticsearch
- Kafka
- MySQL MGR
- Nginx
- Redis

### 告警规则
告警规则配置文件位于 `prometheus/rules/` 目录下：
- `es.yml` - Elasticsearch 告警规则
- `kafka.yml` - Kafka 告警规则
- `MySQL-MGR.yml` - MySQL MGR 告警规则
- `nginx.yml` - Nginx 告警规则
- `redis.yml` - Redis 告警规则

## 配置说明

### 邮件告警配置
邮件模板和配置位于 `alertmanager/` 目录：
- `config.yml` - Alertmanager 主配置
- `email.tmpl` - 邮件模板


## 故障排除

### 常见问题
1. **服务无法启动**: 检查端口是否被占用
2. **Grafana 无法访问**: 确认容器状态和端口映射
3. **Kafka 连接失败**: 验证 Kafka 服务是否正常运行

### 日志查看
```javascript
# 查看所有服务日志
docker-compose logs

# 查看特定服务日志
docker-compose logs prometheus
docker-compose logs grafana
docker-compose logs alertmanager
```

## 开发指南

### 添加新的监控目标
1. 在 `prometheus/prometheus.yml` 中添加新的 scrape 配置
2. 创建对应的告警规则文件
3. 重启 Prometheus 服务

### 自定义仪表板
1. 登录 Grafana
2. 创建新的仪表板
3. 添加 Prometheus 数据源
4. 配置查询和可视化


### 监控新node节点
```javascript
docker run -d \
  --name node-exporter \
  --restart unless-stopped \
  -p 9100:9100 \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/host:ro \
  quay.io/prometheus/node-exporter:v1.8.1 \
  --path.rootfs=/host
```
