# Современные возможности Kubernetes

В этой главе мы рассмотрим современные возможности Kubernetes v1.32+, которые делают кластер более безопасным, 
производительным и удобным в управлении.

## Pod Security Standards

### Обзор

Pod Security Standards (PSS) заменили устаревшие PodSecurityPolicies в Kubernetes v1.25+. 
Они обеспечивают более простой и эффективный способ управления безопасностью подов.

### Уровни безопасности

1. **Privileged**: Неограниченная политика, предоставляет максимальные привилегии
2. **Baseline**: Минимально ограничивающая политика, предотвращает известные повышения привилегий
3. **Restricted**: Наиболее ограничивающая политика, следует принципам наименьших привилегий

### Применение PSS

```bash
# Примените Pod Security Standards
kubectl apply -f deployments/pod-security-standards.yaml
```

### Проверка политик

```bash
# Проверьте политики для namespace
kubectl get namespace default -o yaml | grep pod-security

# Проверьте нарушение политик
kubectl run test-pod --image=nginx:alpine --privileged
# Должно быть отклонено
```

## Внешний Cloud Controller Manager

### Обзор

В Kubernetes v1.31+ все в-tree cloud providers были удалены. Теперь требуется внешний Cloud Controller Manager 
для интеграции с облачными провайдерами.

### Развертывание Yandex Cloud Controller Manager

```bash
# Создайте Service Account в Yandex Cloud
yc iam service-account create --name k8s-controller-manager

# Назначьте роли
yc resource-manager folder add-access-binding <folder-id> \
  --role compute.admin \
  --subject serviceAccount:<service-account-id>

# Создайте ключ
yc iam key create --service-account-name k8s-controller-manager --output key.json

# Закодируйте в base64
FOLDER_ID=$(yc config get folder-id | base64 -w 0)
SA_KEY=$(cat key.json | base64 -w 0)

# Обновите секрет
kubectl create secret generic yandex-cloud-controller-manager-secret \
  --from-literal=folder-id=$FOLDER_ID \
  --from-literal=sa-key-json=$SA_KEY \
  -n kube-system
```

### Применение Cloud Controller Manager

```bash
kubectl apply -f deployments/yandex-cloud-controller-manager.yaml
```

### Проверка работы

```bash
# Проверьте поды Cloud Controller Manager
kubectl get pods -n kube-system -l k8s-app=yandex-cloud-controller-manager

# Проверьте логи
kubectl logs -n kube-system -l k8s-app=yandex-cloud-controller-manager
```

## User Namespaces (Kubernetes v1.33+)

### Обзор

User Namespaces обеспечивают дополнительную изоляцию контейнеров, запуская их в отдельном пользовательском пространстве.

### Включение User Namespaces

```bash
# Создайте под с включенными user namespaces
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: user-namespace-test
spec:
  hostUsers: false  # Включает user namespaces
  containers:
  - name: test
    image: busybox:1.28
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
EOF
```

### Проверка изоляции

```bash
# Проверьте user namespace
kubectl exec user-namespace-test -- cat /proc/self/uid_map
```

## EndpointSlices

### Обзор

EndpointSlices заменили устаревший Endpoints API, обеспечивая лучшую производительность и масштабируемость.

### Проверка EndpointSlices

```bash
# Создайте тестовый сервис
kubectl run nginx --image=nginx:alpine --port=80
kubectl expose pod nginx --port=80 --target-port=80

# Проверьте EndpointSlices
kubectl get endpointslices
```

## nftables Backend для kube-proxy

### Обзор

В Kubernetes v1.32+ nftables backend стал стабильным и обеспечивает лучшую производительность по сравнению с iptables.

### Включение nftables

```bash
# Обновите конфигурацию kube-proxy
kubectl edit configmap kube-proxy -n kube-system
```

Добавьте в data.config.conf:
```yaml
mode: "nftables"
```

### Перезапуск kube-proxy

```bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

## Traffic Distribution (Kubernetes v1.32+)

### Обзор

Новая функция traffic distribution позволяет более точно контролировать распределение трафика между подами.

### Пример использования

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  type: LoadBalancer
  trafficDistribution: PreferClose  # Новая функция
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: example
```

## Мониторинг и метрики

### Включение метрик

```bash
# Проверьте метрики API Server
curl -k https://localhost:6443/metrics

# Проверьте метрики CoreDNS
kubectl port-forward -n kube-system svc/kube-dns 9153:9153
curl http://localhost:9153/metrics
```

### Базовый мониторинг

```bash
# Создайте простой мониторинг
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-config
  namespace: kube-system
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
EOF
```

## Проверка всех функций

### Комплексная проверка

```bash
# Создайте тестовое приложение
kubectl create namespace modern-features-test
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: modern-test
  namespace: modern-features-test
spec:
  hostUsers: false
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF

# Проверьте статус
kubectl get pod modern-test -n modern-features-test
```

## Заключение

Современные возможности Kubernetes v1.32+ обеспечивают:

- **Улучшенную безопасность**: Pod Security Standards, User Namespaces
- **Лучшую производительность**: nftables, EndpointSlices, Traffic Distribution
- **Облачную интеграцию**: Внешний Cloud Controller Manager
- **Упрощенное управление**: Централизованные политики безопасности

Эти функции делают кластер более готовым к продакшену и упрощают его управление.

Дальше: [Мониторинг и наблюдаемость кластера](16-monitoring-observability.md) 