# Развертывание Kubernetes Control Plane

В этой лабораторной работе вы развернете Kubernetes Control Plane на одном инстансе (упрощенная архитектура) и настроите его
для обеспечения удаленного доступа. Вы также создадите внешний балансировщик нагрузки, который даст удаленным клиентам
доступ к API Kubernetes.
На узле будут установлены следующие компоненты: сервер API Kubernetes, Scheduler и Controller Manager.

## Важное

Команды из этой лабораторной выполняются на инстансе `server`. В упрощенной архитектуре мы используем один control plane
узел вместо трех для упрощения обучения.

### Параллельное выполнение команд через tmux

[tmux](https://github.com/tmux/tmux/wiki) может быть использован для параллельного выполнения команд на нескольких
инстансах. Смотри [Параллельное выполнение команд в tmux](01-prerequisites.md)

## Развертывание Kubernetes Control Plane

### Подключение к server

```bash
# Подключитесь к server
ssh yc-user@<server-external-ip>
```

### Распаковка конфигураций

```bash
# Распакуйте конфигурации
tar -xzf server-configs.tar.gz
```

Создайте директорию для конфигов Kubernetes:

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Скачайте и установите Kubernetes Controller

Скачайте официальный релиз Kubernetes:

```bash
K8S_VERSION=v1.32.3
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubectl"
```

Установите Kubernetes:

```bash
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Настройте сервер Kubernetes API

```bash
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

Внутренний IP-адрес будет использован другими членима кластера для общения с API сервером. Получить внутренний адрес
можно следующим образом:

```bash
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

Так как мы не настраивали yc на инстансах, а то следующий код нужно выполнить локально, а затем на серверах установить
переменную окружения.

```bash
yc vpc address get kubernetes-the-hard-way --format json | jq '.external_ipv4_address.address' -r
```

```bash
KUBERNETES_PUBLIC_ADDRESS=
```

Создайте `kube-apiserver.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=1 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### Настройте Kubernetes Controller Manager

Создайте `kube-controller-manager.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=127.0.0.1 \\
  --cluster-cidr=10.200.0.0/16 \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### Настройте Kubernetes Scheduler

Создайте `kube-scheduler.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

Создайте конфигурационный файл для Kubernetes Scheduler:

```bash
sudo mkdir -p /etc/kubernetes/config/

cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/etc/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

### Запустите сервисы

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

### Проверка

Проверьте статус сервисов:

```bash
sudo systemctl status kube-apiserver kube-controller-manager kube-scheduler
```

## Альтернатива: Высокодоступный Control Plane (для продакшена)

Для продакшен окружения рекомендуется использовать несколько control plane узлов. Вот пример конфигурации:

```bash
# Для кластера из 3 control plane узлов
--apiserver-count=3
--etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379
```

Дальше: [Развертывание воркеров Kubernetes](10-bootstrapping-kubernetes-workers.md)
