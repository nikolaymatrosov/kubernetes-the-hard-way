# Мониторинг и наблюдаемость кластера

В этой главе мы настроим базовый мониторинг и наблюдаемость для вашего Kubernetes кластера. Это поможет вам понимать состояние кластера и диагностировать проблемы.

## Базовые метрики

### Проверка метрик API Server

Kubernetes API Server предоставляет метрики в формате Prometheus:

```bash
# Проверьте метрики API Server
curl -k https://localhost:6443/metrics

# Или через kubectl proxy
kubectl proxy &
curl http://localhost:8001/metrics
```

### Метрики kubelet

Каждый узел предоставляет метрики через kubelet:

```bash
# Получите список узлов
kubectl get nodes

# Проверьте метрики kubelet на конкретном узле
kubectl proxy &
curl http://localhost:8001/api/v1/nodes/node-0/proxy/metrics
```

## Metrics Server

Metrics Server собирает метрики ресурсов (CPU, память) со всех узлов кластера.

### Установка Metrics Server

```bash
# Скачайте манифест Metrics Server
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Примените манифест
kubectl apply -f components.yaml
```

### Проверка работы

```bash
# Проверьте поды Metrics Server
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Проверьте метрики узлов
kubectl top nodes

# Проверьте метрики подов
kubectl top pods --all-namespaces
```

## Логирование

### Настройка централизованного логирования

Создайте простую систему логирования с использованием Fluentd:

```bash
# Создайте namespace для логирования
kubectl create namespace logging

# Создайте ConfigMap для Fluentd
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    
    <match kubernetes.**>
      @type stdout
    </match>
EOF
```

### Развертывание Fluentd

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
          - name: FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.logging"
          - name: FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging
EOF
```

## Простой мониторинг с Prometheus

### Установка Prometheus

```bash
# Создайте namespace для мониторинга
kubectl create namespace monitoring

# Установите Prometheus с помощью Helm (если установлен)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

### Альтернатива: Простой Prometheus

Если Helm не установлен, создайте простой Prometheus:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention.time=200h"
        - "--web.console.libraries=/etc/prometheus/console_libraries"
        - "--web.console.templates=/etc/prometheus/consoles"
        - "--web.enable-lifecycle"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus
        - name: prometheus-storage
          mountPath: /prometheus
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
      - name: prometheus-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
  type: ClusterIP
EOF
```

### Доступ к Prometheus

```bash
# Пробросьте порт Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Откройте в браузере: http://localhost:9090
```

## Алерты и уведомления

### Создание простых алертов

```bash
# Создайте ConfigMap с правилами алертов
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  alert.rules: |
    groups:
    - name: kubernetes
      rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ \$labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes"
      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ \$labels.instance }}"
          description: "Memory usage is above 80% for 5 minutes"
EOF
```

## Визуализация с Grafana

### Установка Grafana

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
  type: ClusterIP
EOF
```

### Доступ к Grafana

```bash
# Пробросьте порт Grafana
kubectl port-forward -n monitoring svc/grafana 3000:3000

# Откройте в браузере: http://localhost:3000
# Логин: admin, Пароль: admin
```

## Проверка мониторинга

### Проверьте все компоненты

```bash
# Проверьте поды мониторинга
kubectl get pods -n monitoring

# Проверьте сервисы
kubectl get svc -n monitoring

# Проверьте логи Prometheus
kubectl logs -n monitoring deployment/prometheus

# Проверьте логи Grafana
kubectl logs -n monitoring deployment/grafana
```

## Заключение

Теперь у вас есть базовая система мониторинга и наблюдаемости:

- **Metrics Server**: Метрики ресурсов (CPU, память)
- **Prometheus**: Сбор и хранение метрик
- **Grafana**: Визуализация метрик
- **Fluentd**: Централизованное логирование
- **Алерты**: Уведомления о проблемах

Это поможет вам:
- Отслеживать производительность кластера
- Диагностировать проблемы
- Планировать масштабирование
- Обеспечивать надежность

Дальше: [Очистка ресурсов](17-cleanup.md) 